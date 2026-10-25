[← Course Index](../README.md)

# 12.2 — use, pub use, and Re-exports

## What You'll Learn

- Every `use` form: nested braces, `as`, `self`, globs
- `pub use` to re-export and build a facade API
- Why a trait must be *in scope* to call its methods
- Prelude modules and the ambiguity errors globs cause

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `import { a, b as c } from "m"` | `use m::{a, b as c};` |
| JS `export { a } from "m"` | `pub use m::a;` |
| Python `from m import *` | `use m::*;` (discouraged) |
| Go: import a package, call `pkg.Fn` | `use` only shortens a path; paths always work |

`use` is **not** a load instruction — nothing is fetched or executed. The module
tree already exists; `use` just creates a local alias into it. Writing the full
path works identically.

## Basic use and nested braces

```rust
use std::collections::HashMap;
use std::{fmt, io::{self, Read}};
use std::collections::BTreeMap as Sorted;

fn main() {
    let mut m: HashMap<&str, i32> = HashMap::new();
    m.insert("a", 1);
    println!("{:?}", m);

    let mut sorted = Sorted::new();
    sorted.insert(2, "two");
    sorted.insert(1, "one");
    println!("{:?}", sorted);

    let _: Option<fmt::Error> = None;
    let _: Option<io::Error> = None;
    fn _needs_read<R: Read>(_r: R) {}
}
```

Output:

```
{"a": 1}
{1: "one", 2: "two"}
```

Braces nest to any depth. `io::{self, Read}` imports both the `io` module itself
and the `Read` trait inside it — that is the `self` idiom, and it saves writing
two lines.

## as renaming

Two items with the same name collide.

```rust
mod a { pub fn run() {} }
mod b { pub fn run() {} }
use a::run;
use b::run;
```

```
error[E0252]: the name `run` is defined multiple times
 --> src/main.rs:4:5
  |
3 | use a::run;
  |     ------ previous import of the value `run` here
4 | use b::run;
  |     ^^^^^^ `run` reimported here
  |
  = note: `run` must be defined only once in the value namespace of this module
help: you can use `as` to change the binding name of the import
  |
4 | use b::run as other_run;
  |            ++++++++++++
```

Fix with `as`:

```rust
mod a { pub fn run() -> &'static str { "a" } }
mod b { pub fn run() -> &'static str { "b" } }

use a::run as run_a;
use b::run as run_b;

fn main() {
    println!("{} {}", run_a(), run_b());
}
```

Output:

```
a b
```

The alternative — and often the better choice — is to import the *parent* and
write `a::run()` / `b::run()`. The std library convention is to import types
directly (`HashMap`) but functions through their module (`io::stdin()`,
`mem::swap()`), because a bare `swap()` tells the reader nothing.

## Glob imports and why they're discouraged

```rust
mod a { pub fn run() -> &'static str { "a" } }
mod b { pub fn run() -> &'static str { "b" } }

use a::*;
use b::*;

fn main() {
    println!("{}", run());
}
```

```
error[E0659]: `run` is ambiguous
 --> src/main.rs:8:20
  |
8 |     println!("{}", run());
  |                    ^^^ ambiguous name
  |
  = note: ambiguous because of multiple glob imports of a name in the same module
note: `run` could refer to the function imported here
 --> src/main.rs:4:5
  |
4 | use a::*;
  |     ^^^^
  = help: consider adding an explicit import of `run` to disambiguate
note: `run` could also refer to the function imported here
 --> src/main.rs:5:5
  |
5 | use b::*;
  |     ^^^^
```

The deeper problem is silent: a glob'd crate adding a new public item in a patch
release can shadow or clash with one of your names. Two accepted uses of `*`:

- `use super::*;` inside a `#[cfg(test)] mod tests` block.
- importing a crate's documented `prelude` module.

## Prelude modules

A `prelude` is just a module full of `pub use`. The convention exists so users
can grab your common names in one line.

```rust
mod engine {
    pub mod parser {
        pub struct Ast(pub String);
        pub fn parse(s: &str) -> Ast { Ast(s.to_uppercase()) }
    }
    pub mod runtime {
        pub fn eval(a: &super::parser::Ast) -> usize { a.0.len() }
    }
}

pub use engine::parser::{parse, Ast};
pub use engine::runtime::eval;

mod prelude {
    pub use crate::{eval, parse, Ast};
}

fn main() {
    {
        use crate::prelude::*;
        let b: Ast = parse("world");
        println!("{} {}", b.0, eval(&b));
    }
    println!("{}", engine::parser::parse("deep").0);
}
```

Output:

```
WORLD 5
DEEP
```

Note the std prelude is already in scope in every file — that's why `Vec`,
`Option`, `String`, `Some` and `Box` need no `use`.

## pub use: re-exporting

A plain `use` is private — it is a binding in your module, not a path others can
travel.

```rust
mod wrapper {
    use std::collections::HashMap;
    pub fn make() -> HashMap<i32, i32> { HashMap::new() }
}

fn main() {
    let _m: wrapper::HashMap<i32, i32> = Default::default();
}
```

```
error[E0603]: struct import `HashMap` is private
   --> src/main.rs:8:22
    |
  8 |     let _m: wrapper::HashMap<i32, i32> = Default::default();
    |                      ^^^^^^^ private struct import
    |
note: the struct import `HashMap` is defined here...
   --> src/main.rs:2:9
    |
  2 |     use std::collections::HashMap;
    |         ^^^^^^^^^^^^^^^^^^^^^^^^^
```

Change it to `pub use std::collections::HashMap;` and `wrapper::HashMap` becomes
a real path. That is all a re-export is: a `use` that is also part of your
public surface.

## The facade pattern

This is the single most useful thing in this lesson. Organize internals however
you like in deep private modules, then publish a flat surface from the crate
root.

```rust
mod engine {
    pub mod parser {
        pub struct Ast(pub String);
        pub fn parse(s: &str) -> Ast { Ast(s.to_uppercase()) }
    }
    pub mod runtime {
        pub fn eval(a: &super::parser::Ast) -> usize { a.0.len() }
    }
}

pub use engine::parser::{parse, Ast};
pub use engine::runtime::eval;

fn main() {
    let a = parse("hello");
    println!("{} {}", a.0, eval(&a));
    println!("{}", engine::parser::parse("deep").0);
}
```

Output:

```
HELLO 5
DEEP
```

Users write `mycrate::parse` instead of `mycrate::engine::parser::parse`. Three
payoffs:

1. Your public API is one readable block at the top of `lib.rs`.
2. You can move `parser` anywhere without a breaking change.
3. Make `mod engine` private (drop its `pub`) and the *only* way in is through
   your re-exports — the deep paths disappear from the docs entirely.

That last point is the real design lever: `pub use` plus a private module is how
you expose exactly what you intend.

## A trait must be in scope to call its methods

This catches everyone. The `impl` existing is not enough; the **trait name** has
to be imported where the method call happens.

```rust
mod traits {
    pub trait Describe {
        fn describe(&self) -> String;
    }
}

pub struct Point { pub x: i32 }

impl traits::Describe for Point {
    fn describe(&self) -> String {
        format!("Point at {}", self.x)
    }
}

fn main() {
    let p = Point { x: 3 };
    println!("{}", p.describe());
}
```

Does not compile, even though the `impl` is right there:

```
error[E0599]: no method named `describe` found for struct `Point` in the current scope
  --> src/main.rs:17:22
   |
 3 |         fn describe(&self) -> String;
   |            -------- the method is available for `Point` here
...
 7 | pub struct Point { pub x: i32 }
   | ---------------- method `describe` not found for this struct
...
17 |     println!("{}", p.describe());
   |                      ^^^^^^^^ method not found in `Point`
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `Describe` which provides `describe` is implemented but not in scope; perhaps you want to import it
   |
 1 + use crate::traits::Describe;
   |
```

Take the suggestion:

```rust
use traits::Describe;

fn main() {
    let p = Point { x: 3 };
    println!("{}", p.describe());
    println!("{}", traits::Describe::describe(&p));
}
```

Output:

```
Point at 3
Point at 3
```

Method-call syntax needs the trait in scope. Fully qualified syntax never does.
The same thing happens with std: `write!` into a `String` fails until you
`use std::fmt::Write`, and `read_to_string` on a file fails until you
`use std::io::Read`.

```rust
let mut s = String::new();
write!(s, "hello").unwrap();
```

```
error[E0599]: cannot write into `String`
   --> src/main.rs:3:12
    |
  3 |     write!(s, "hello").unwrap();
    |            ^
    |
note: must implement `io::Write`, `fmt::Write`, or have a `write_fmt` method
    = help: items from traits can only be used if the trait is in scope
help: trait `Write` which provides `write_fmt` is implemented but not in scope; perhaps you want to import it
    |
  1 + use std::fmt::Write;
    |
```

This is also why crates ship preludes: `use tokio::io::AsyncReadExt;` style
imports exist only to bring extension traits into scope.

## use inside a function

`use` is scoped like any item. Put it in a function to keep a one-off import
out of the file header.

```rust
use std::collections::hash_map::{self, HashMap};

fn main() {
    let mut m = HashMap::new();
    m.insert(1, "one");
    let e: hash_map::Entry<'_, i32, &str> = m.entry(2);
    e.or_insert("two");
    let mut keys: Vec<_> = m.keys().collect();
    keys.sort();
    println!("{:?}", keys);

    fn local() {
        use std::cmp::max;
        println!("{}", max(3, 9));
    }
    local();
}
```

Output:

```
[1, 2]
9
```

Idiomatic choice: file-level imports at the top. Function-level `use` is for
`use Enum::*` inside one `match`, or a trait needed by exactly one function.

## Importing enum variants

`use E::*` is the one glob nobody objects to, because it's local and explicit.

```rust
#[derive(Debug)]
enum Op { Add, Sub, Mul }

fn apply(op: Op, a: i32, b: i32) -> i32 {
    use Op::*;
    match op {
        Add => a + b,
        Sub => a - b,
        Mul => a * b,
    }
}

fn main() {
    println!("{}", apply(Op::Add, 2, 3));
    println!("{}", apply(Op::Mul, 2, 3));
}
```

Output:

```
5
6
```

## Quick Reference

| Form | Meaning |
|---|---|
| `use a::b;` | Bring `b` into scope (private binding) |
| `pub use a::b;` | Same, and re-export `b` as part of your API |
| `use a::{b, c};` | Multiple items from one path |
| `use a::{self, b};` | The module `a` plus `a::b` |
| `use a::b as c;` | Rename |
| `use a::*;` | Glob — avoid except preludes and tests |
| `use a as _;` | Import for its trait impls only, no name bound |
| `pub(crate) use a::b;` | Re-export only within this crate |
| `use Enum::*;` inside `fn` | Short variant names in one scope |
| `E0252` | Same name imported twice |
| `E0659` | Ambiguous name from overlapping globs |
| `E0599` + "trait is in scope" | Missing `use` for a trait |

## Common Mistakes

**Calling a trait method without importing the trait.** The `impl` is irrelevant;
`E0599` tells you the exact `use` line to add.

**Expecting a plain `use` to re-export.** It is private. Add `pub`.

**Two glob imports of overlapping modules.** `E0659` at the use site, and new
upstream items can break you silently. Import names explicitly.

**Importing bare function names.** `use std::mem::swap;` then `swap(a, b)` reads
worse than `mem::swap(a, b)`. Import types directly, functions via their module.

**Thinking `use` costs anything at runtime.** It is pure name resolution; there
is no load, no init, no code emitted.

## Exercises

1. Rewrite `use std::io::Write; use std::io::BufWriter; use std::io;` as a single
   nested-brace `use` with `self`.
2. Make two modules that both export `fn parse`. Import both into `main` three
   ways: with `as`, by importing the parent modules, and by full paths.
3. Define a trait `Doubler` with `fn doubled(&self) -> Self` in a submodule,
   implement it for `i32`, and call `5i32.doubled()` *without* importing it.
   Read the error, then fix it twice: with a `use`, and with fully qualified
   syntax.
4. Build a crate root that exposes `pub use` for three items buried in
   `internal::deep::stuff`, then make `mod internal` private. Confirm the deep
   path no longer resolves from outside.
5. Add a `prelude` module to exercise 4 and use it with a glob from a function.
   Then add a second item with a colliding name and watch `E0659` appear.

---

[← Course Index](../README.md)
