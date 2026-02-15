
# SafeCpp Compiler — Implementation Plan (Haskell, transpile to C++17)

## Overview

`safecpp` is a source-to-source transpiler written in **Haskell**:

1. Parse `.scpp` (C++17 + ownership annotations)
2. Perform **ownership/borrow checking** (compile-time)
3. Emit `.cpp` (standard C++17, annotations erased)

Non-goal: implement full C++ semantic analysis (types/overload/templates). Downstream C++ compiler remains responsible for that.

To keep the project realistic, the checker is:
- **Intra-procedural** by default (within function bodies)
- Optionally **contract-based** across functions using `#pragma safecpp trust` and `[[safecpp::...]]` attributes

---

## Key Design Corrections vs Original Draft

1. **No new function/struct syntax** (`fn<'a>`, `struct<'a>`, `self` parameter removed).  
   Lifetimes/contracts are expressed via **C++ attributes** `[[safecpp::...]]` and pragmas.

2. **No ownership qualifiers inside type expressions** (e.g. `std::vector<own T>` is not C++).  
   Ownership qualifiers are only allowed in declaration specifiers (`own/ref/mutref/raw`).

3. **Conservative “owner freeze” rule** during borrows (implementable without full type info):  
   while a value is borrowed, the owner identifier cannot be used directly.

4. Parsing: do not rely on ad-hoc angle-bracket heuristics for real-world C++.  
   Recommended: use a real C++ concrete syntax parser such as **tree-sitter-cpp** for stable CST extraction.

---

## Architecture

```
┌─────────────┐   ┌───────────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────────┐
│  Source      │──▶│ Preprocessor   │──▶│   Parser/CST  │──▶│ Ownership     │──▶│ Rewriter    │
│ (.scpp)      │   │ relay (light)  │   │ extraction    │   │ Checker       │   │ (emit .cpp) │
└─────────────┘   └───────────────┘   └──────────────┘   └──────────────┘   └────────────┘
                                             │                    │
                                             ▼                    ▼
                                       ┌──────────┐        ┌─────────────┐
                                       │  IR/AST  │        │ Diagnostics  │
                                       └──────────┘        └─────────────┘
```

---

## Phase 0: Preprocessor Relay (Lightweight)

We do **not** implement macro expansion. Strategy:
- Preserve `#include/#define/#if...` verbatim in output.
- Interpret only:
  - `#pragma safecpp extern_cpp begin/end`
  - `#pragma safecpp trust`
  - `#pragma safecpp no_raw_warn`
  - `#pragma safecpp no_implicit_move`

Inside `extern_cpp` blocks:
- Skip checking
- Pass through verbatim

> Optional future mode: `--preprocess=clang` to run `clang -E` for more predictable parsing (but loses some original formatting).

---

## Phase 1: Parsing / CST Extraction

### Recommended approach: tree-sitter-cpp
C++ parsing is too context-sensitive for a hand-rolled parser to be robust. Use tree-sitter-cpp to obtain:
- Function definitions and bodies
- Variable declarations (including specifiers)
- Reference expressions / identifier usage
- Call expressions, member calls
- Block structure for scopes

Haskell integration:
- Use FFI to call tree-sitter C API
- Convert CST nodes into a simplified IR needed by the checker

If tree-sitter is not used initially, the project must explicitly document limited support (templates/macros will frequently break parsing).

---

## Phase 2: Build a Checker IR (minimal, ownership-oriented)

We only need a subset of C++ constructs.

```haskell
data Ownership = Own | Ref | MutRef | Raw | Unchecked
  deriving (Show, Eq)

data VarId = VarId { varName :: Text, varUniq :: Int }
  deriving (Show, Eq, Ord)

data Expr
  = EIdent VarId
  | EMove Expr            -- parsed from `move x`
  | ECall Expr [Expr]
  | EMember Expr Text
  | EOther Text           -- opaque fallback
  deriving (Show, Eq)

data Stmt
  = SVarDecl { sOwner :: Ownership, sName :: VarId, sInit :: Maybe Expr }
  | SAssign { sLhs :: Expr, sRhs :: Expr }
  | SExpr Expr
  | SReturn (Maybe Expr)
  | SIf Expr [Stmt] [Stmt]
  | SWhile Expr [Stmt]
  | SFor (Maybe Stmt) (Maybe Expr) (Maybe Expr) [Stmt]
  | SBlock [Stmt]
  | SUnsafe [Stmt]
  | SOpaque Text
  deriving (Show, Eq)

data FunctionIR = FunctionIR
  { fName     :: Text
  , fParams   :: [(Ownership, VarId)]
  , fBody     :: [Stmt]
  , fContracts :: Contracts  -- from trust/attributes, optional
  }
```

Ownership keywords are **contextual**:
- `own/ref/mutref/raw` recognized only in declaration specifier positions.
- `move` recognized only as a unary-prefix expression keyword.

---

## Phase 3: Ownership & Borrow Checking

### What we can soundly check without full C++ type semantics
1. **Use-after-move** on `own` variables
2. **Move while borrowed**
3. **Borrow exclusivity**:
   - many `ref` OR one `mutref`
4. **Owner freeze rule** (conservative):
   - while a borrow is alive, the owner identifier cannot be used directly
5. `raw` warnings, `unsafe` audit

### CFG + Dataflow
Borrow checking requires control-flow awareness.

- Build a CFG per function
- Run forward dataflow for move states
- Run liveness analysis for borrow variables to compute NLL regions

#### States
```haskell
data OwnerState = Alive | Moved SourcePos deriving (Show, Eq)

data BorrowKind = BImm | BMut deriving (Show, Eq)

data Borrow = Borrow
  { bOwner  :: VarId
  , bVar    :: VarId
  , bKind   :: BorrowKind
  , bRegion :: Set NodeId
  }
```

#### Rules (checked code)
- Creating `ref` borrow of owner `x` is illegal if any `mutref` active
- Creating `mutref` borrow is illegal if any borrow active
- `move x` is illegal if any borrow of `x` is active
- Any use of `x` while borrowed is illegal (owner frozen), except:
  - creating another `ref` borrow of `x` (when only immutable borrows exist)
  - re-assignment that ends its previous state is allowed only when no borrow exists

This conservative “freeze” rule avoids needing to know whether `x.foo()` is const/mutating.

---

## Phase 4: Optional Contracts & Lifetimes (Attribute-based)

Because we keep pure C++ syntax, lifetimes are represented as attributes:

- `[[safecpp::lt_params(a,b)]]` on functions/structs
- `[[safecpp::lt(a)]]` on `ref/mutref` parameters/returns/fields
- `[[safecpp::where("a: b")]]` on functions

Implementation strategy (incremental):
- v0.1: intra-procedural borrows only, ignore named lifetimes except recording
- v0.2+: check return-borrow relationships for functions marked `#pragma safecpp trust`:
  - e.g. “return lifetime equals param 1 lifetime”
- Call-site checking uses only lexical region approximation in caller (no type inference)

---

## Phase 5: Rewriter / Code Generation (Token-preserving)

Instead of “pretty-printing” C++, prefer token-preserving rewrite to keep formatting and macros intact.

Rewrite rules:
- Remove `own/ref/mutref/raw` specifiers from declarations
  - `ref T&` => ensure output is `const T&` (add `const` if missing)
  - `mutref T&` => output `T&`
  - `own T` => output `T`
  - `raw` => output just the underlying C++ type
- Replace `move x` => `std::move(x)`
- Replace `unsafe { ... }` => `{ ... }` (drop keyword)
- Strip or comment-out `#pragma safecpp ...` so downstream compilers won’t warn
- Strip `[[safecpp::...]]` attributes

If `std::move` is emitted, ensure `<utility>` is available:
- Either inject `#include <utility>` at the top (configurable), or
- Require users to include it (documented). (Injecting is recommended for UX.)

---

## Project Structure (suggested)

```
safecpp/
├── app/Main.hs
├── src/SafeCpp/
│   ├── Driver.hs
│   ├── PreprocessorRelay.hs
│   ├── Parse/
│   │   ├── TreeSitter.hs        # FFI wrapper
│   │   └── ExtractIR.hs         # CST -> IR
│   ├── IR.hs
│   ├── CFG.hs
│   ├── Check/
│   │   ├── Move.hs
│   │   ├── Borrow.hs
│   │   └── Contracts.hs
│   ├── Rewrite.hs              # token-preserving rewrite
│   ├── Diagnostic.hs
│   └── Pretty.hs
├── test/...
├── README.md
├── PLAN.md
└── safecpp.cabal
```

---

## Milestones

### M1 (Weeks 1-3): Rewriter-only transpile
- Parse enough to find ownership keywords in declarations and `move/unsafe`
- Emit compilable C++17 (strip annotations, emit std::move)
- `--no-check` works reliably

### M2 (Weeks 4-6): Move checking (CFG)
- Build function-level CFG
- Detect use-after-move, move-while-borrowed (basic)

### M3 (Weeks 7-10): Borrow checking + NLL
- Liveness analysis for borrow vars
- Enforce ref/mutref exclusivity
- Implement owner-freeze rule

### M4 (Weeks 11-14): Interop boundaries
- `extern_cpp` blocks passthrough
- `raw` warnings + audit
- `#pragma safecpp trust` for external declarations (basic)

### M5 (Weeks 15-18): Optional contracts/lifetimes (attribute-based)
- Parse/strip attributes
- Limited cross-function checking for trusted APIs

---

## Diagnostics (examples)

Use Rust-like error layout and stable error codes:

- E001 use-after-move
- E002 borrow conflict (mut vs imm)
- E003 multiple mutable borrows
- E004 move while borrowed
- W001 raw used outside unsafe
- W002 untrusted external call requires unsafe

---

## Notes on Soundness

SafeCpp can only guarantee the rules it checks. Any of the following reduce guarantees:
- `raw` pointers/references
- `unsafe` blocks
- `extern_cpp` blocks
- Untrusted external functions that may store references globally

The compiler should surface these boundaries clearly (warnings + `--audit`).
