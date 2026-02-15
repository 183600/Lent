# SafeCpp - C++ with Ownership (Transpiles to C++17)

SafeCpp is a C++-compatible language extension that adds Rust-inspired **ownership** and **borrowing** checks on top of C++.
SafeCpp source files (`.scpp`) are transpiled to standard **C++17** (`.cpp`), enabling use of existing C++ libraries, build systems, and toolchains.

The SafeCpp compiler (`safecpp`) is written in **Haskell**.

> Design constraint: **Except for ownership-related annotations (`own/ref/mutref/raw/move/unsafe` + a few C++ attributes/pragmas), the syntax is standard C++**.

---

## Table of Contents

- [Overview](#overview)
- [Getting Started](#getting-started)
- [Language Extensions](#language-extensions)
  - [Ownership Qualifiers](#ownership-qualifiers)
  - [`move` Expression](#move-expression)
  - [Borrow Rules (NLL)](#borrow-rules-nll)
  - [Optional Lifetime Attributes (C++-syntax)](#optional-lifetime-attributes-c-syntax)
  - [`unsafe` Blocks](#unsafe-blocks)
  - [Interop Pragmas](#interop-pragmas)
- [Interop with C++](#interop-with-c)
- [Examples](#examples)
- [Limitations](#limitations)
- [Building the Compiler](#building-the-compiler)
- [File Extension](#file-extension)
- [License](#license)

---

## Overview

**SafeCpp = C++17 + compile-time ownership/borrow checking + transpile to C++17**

### Goals
- Catch common memory-safety bugs early (use-after-move, move-while-borrowed, illegal aliasing) in **checked code**
- Zero runtime overhead (all checks are compile-time; output is ordinary C++)
- Reuse existing C++ ecosystem (STL, third-party libs, build tools)
- Minimal and C++-shaped syntax: declarations and expressions remain C++

### Non-Goals / Clarifications
- SafeCpp is **not** a full C++ frontend: it does not implement C++ type checking, overload resolution, or template instantiation.
- SafeCpp does **not** attempt to prove all C++ is memory-safe (raw pointers, macros, and external code can break safety).
- Safety is guaranteed only for code that is successfully checked and does not cross `unsafe/raw/extern_cpp` boundaries.

---

## Getting Started

### Prerequisites
- GHC 9.4+ and Cabal 3.8+ (build the compiler)
- A C++17 compiler (clang++, g++, MSVC)

### Installation
```bash
git clone https://github.com/example/safecpp.git
cd safecpp
cabal build
cabal install
```

### Hello World
```cpp
// hello.scpp
#include <iostream>
#include <string>

int main() {
    own std::string msg = "Hello, SafeCpp!";
    std::cout << msg << "\n";
}
```

```bash
safecpp hello.scpp -o hello.cpp
g++ -std=c++17 hello.cpp -o hello
./hello
```

---

## Language Extensions

SafeCpp is a C++17 superset. It adds:

- **Contextual keywords** in declarations: `own`, `ref`, `mutref`, `raw`
- **A contextual keyword** in expressions: `move`
- An **`unsafe { ... }`** statement form (ownership-related)
- Optional **C++ attributes** `[[safecpp::...]]` for lifetimes / contracts (still valid C++ syntax; stripped during transpile)
- A few `#pragma safecpp ...` directives

No operators are redefined. In particular, `&` is still address-of and `*` is still dereference (standard C++).

### Ownership Qualifiers

#### `own` — owning variable
```cpp
own T x = expr;
own T x(args...);
```

Rules:
- `own` values are **single-owner** in checked code.
- Moving out requires `move x`.
- After move, `x` becomes **moved-from** (further use is a SafeCpp compile error, except re-assignment).

Example:
```cpp
#include <vector>
#include <utility>

int main() {
    own std::vector<int> v = {1,2,3};
    own std::vector<int> v2 = move v; // v is now invalid in SafeCpp
}
```

#### `ref` — immutable borrow (must be a C++ reference type)
```cpp
ref T& r = owner;
```

Transpiles to:
```cpp
const T& r = owner;
```

Rules (checked code):
- Multiple `ref` borrows of the same owner may coexist.
- While any borrow exists, the owner is **frozen**: you may not use the owner identifier directly. Use the borrow variable(s) instead.
  - This conservative rule keeps the checker implementable without full C++ semantic analysis.

Example:
```cpp
#include <string>
#include <iostream>

int main() {
    own std::string s = "hello";
    ref std::string& r = s;

    std::cout << r << "\n";  // OK
    // s += "!";              // ERROR (SafeCpp): owner `s` is frozen while borrowed
}
```

#### `mutref` — mutable borrow (must be a C++ reference type)
```cpp
mutref T& m = owner;
```

Transpiles to:
```cpp
T& m = owner;
```

Rules:
- Exactly one `mutref` borrow at a time.
- No `ref` borrow may coexist with a `mutref`.
- While `mutref` is live, the owner is frozen; only `m` may be used.

Example:
```cpp
#include <vector>

int main() {
    own std::vector<int> v = {1,2,3};
    mutref std::vector<int>& m = v;
    m.push_back(4);

    // ref std::vector<int>& r = v; // ERROR: cannot immutably borrow while mutably borrowed
}
```

#### `raw` — opt-out / unchecked
```cpp
raw T x = expr;   // value is not tracked
raw T* p = expr;  // raw pointer (unchecked)
raw T& r = expr;  // raw reference (unchecked)
```

Rules:
- No ownership/borrow checking is performed through `raw`.
- Using `raw` outside `unsafe` emits a warning by default.

---

### `move` Expression

```cpp
move expr
```

Transpiles to:
```cpp
std::move(expr)
```

Rules:
- `move x` transfers ownership out of an `own` variable `x`.
- Using `x` afterwards is a SafeCpp compile error (unless re-assigned).
- Moving is illegal if `x` is currently borrowed.

Example:
```cpp
#include <memory>

own std::unique_ptr<int> make() {
    own std::unique_ptr<int> p = std::make_unique<int>(42);
    return move p;
}
```

---

### Borrow Rules (NLL)

SafeCpp uses Non-Lexical Lifetimes (NLL) in checked code:
- A borrow ends at its **last use**, not necessarily at end of scope.
- This allows patterns like:

```cpp
#include <string>
#include <iostream>

int main() {
    own std::string s = "hello";
    ref std::string& r = s;
    std::cout << r << "\n"; // last use of r

    // borrow ended, `s` is no longer frozen
    // (SafeCpp allows re-using `s` after r's last use)
}
```

---

### Optional Lifetime Attributes (C++-syntax)

Because we keep C++ syntax, named lifetimes are expressed using C++ attributes (which are valid C++ and stripped during transpile).

#### Lifetime parameters (function/struct)
```cpp
[[safecpp::lt_params(a, b)]]
```

#### Lifetime annotation on a borrow
```cpp
ref [[safecpp::lt(a)]] T& x
mutref [[safecpp::lt(a)]] T& x
```

#### Where / outlives bounds
```cpp
[[safecpp::where("a: b, b: static")]]
```

Example (return borrow tied to an input):
```cpp
#include <vector>

[[safecpp::lt_params(a)]]
ref [[safecpp::lt(a)]] int& first(ref [[safecpp::lt(a)]] std::vector<int>& v) {
    // returns a reference into v
    return v[0];
}
```

If you omit lifetime attributes, SafeCpp applies elision rules (best-effort). For v0.1, complex cases may require explicit attributes.

---

### `unsafe` Blocks

```cpp
unsafe {
    // checked rules are suspended here
    raw int* p = (int*)0xDEADBEEF;
    *p = 42;
}
```

Semantics:
- Ownership/borrow checking is suspended in `unsafe {}`.
- `unsafe` blocks are recorded for auditing (`safecpp --audit`).

Transpiles to an ordinary C++ block `{ ... }` (keyword removed).

---

### Interop Pragmas

#### `#pragma safecpp extern_cpp`
Marks a region as “verbatim C++” (no checking inside; passed through).

```cpp
#pragma safecpp extern_cpp begin
// raw C++ area
void* malloc(size_t);
void free(void*);
#pragma safecpp extern_cpp end
```

#### `#pragma safecpp trust`
Allows you to declare **checked contracts** for external APIs using SafeCpp qualifiers/attributes.

```cpp
#pragma safecpp trust
void process(ref std::string& s); // treated as taking const std::string&
```

Untrusted external declarations are treated as opaque; calling them with `own/ref/mutref` values may require an `unsafe` block unless trusted.

---

## Interop with C++

- You can `#include` any C++ header.
- SafeCpp emits normal C++17. Build with any C++ toolchain.

Example:
```cpp
#include <nlohmann/json.hpp>
#include <string>

int main() {
    raw nlohmann::json j = {{"key", "value"}}; // unchecked value
    own std::string extracted = j["key"].get<std::string>();
}
```

---

## Examples

### Example 1: Ownership Transfer
```cpp
#include <iostream>
#include <vector>

void consume(own std::vector<int> data) {
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";
}

int main() {
    own std::vector<int> v = {1,2,3,4,5};
    consume(move v);
    // v is invalid in SafeCpp here
}
```

### Example 2: Borrow Conflict (mut vs ref)
```cpp
#include <vector>

int main() {
    own std::vector<int> v = {1,2,3};

    ref std::vector<int>& r = v;
    // mutref std::vector<int>& m = v; // ERROR: cannot mutably borrow while immutably borrowed

    (void)r;
}
```

### Example 3: NLL borrow ends at last use
```cpp
#include <string>
#include <iostream>

int main() {
    own std::string s = "hello";
    ref std::string& r = s;
    std::cout << r << "\n"; // last use of r

    // borrow ended (NLL), s can be moved now
    own std::string t = move s;
    std::cout << t << "\n";
}
```

---

## Limitations

SafeCpp intentionally avoids full C++ semantic analysis. As a result:

- Macro-heavy code and extremely complex template constructs may be only partially checked.
- Calls to untrusted external functions are conservatively treated as unsafe boundaries.
- Mutability detection is conservative: while a value is borrowed, the owner identifier is frozen; use the borrow variable instead.

Use `unsafe`, `raw`, or `extern_cpp` to interoperate with code the checker cannot reason about, and mark safe APIs with `#pragma safecpp trust`.

---

## Building the Compiler

```bash
git clone https://github.com/example/safecpp.git
cd safecpp
cabal build
cabal test
cabal install

# Usage
safecpp input.scpp -o output.cpp
safecpp input.scpp -o output.cpp --emit-ast
safecpp input.scpp -o output.cpp --no-check
safecpp --audit src/
```

---

## File Extension
SafeCpp source files use `.scpp`.

---

## License
MIT License
