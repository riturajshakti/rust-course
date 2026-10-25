[← 11.7 Weak References and Cycles](../11-smart-pointers/07-weak-cycles.md) · [Course Index](../README.md) · [12.2 use, pub use, and Re-exports →](02-use-reexports.md)

# 12.1 — Modules and Visibility

## What You'll Learn

- `mod` blocks, nested modules, and the module tree
- Private by default, and the four `pub` forms
- Paths: `crate::`, `self::`, `super::`, absolute vs relative
- Why struct fields need their own `pub`, and what `E0603` looks like

## Coming From Another Language

| Other | Rust |
|---|---|
| JS: one file = one module, `export` | One *crate* = one module tree; files are optional |
| Python: `package/__init__.py` | `mod` declarations; no implicit file scan |
| Go: directory = package, capital = exported | `mod` block or file; `pub` = exported |

The big surprise: a module in Rust is a **namespace in the crate**, not a file.
A file becomes a module only because you wrote `mod foo;`. And there is no
"public by default" — everything starts private, including struct fields.

## A module is a named scope

```rust
mod network {
    pub fn connect() {
        println!("network::connect");
    }

    pub mod http {
        pub fn get(url: &str) {
            println!("GET {url}");
        }

        pub mod headers {
            pub fn set(k: &str, v: &str) {
                println!("{k}: {v}");
            }
        }
    }
}

fn main() {
    network::connect();
    network::http::get("/users");
    network::http::headers::set("Accept", "application/json");
}
```

Output:

```
network::connect
GET /users
Accept: application/json
```

That whole tree is one file. Modules nest arbitrarily deep.

## The module tree

The above crate has this tree, rooted at the implicit `crate` module:

```
crate
├── network
│   ├── connect
│   └── http
│       ├── get
│       └── headers
│           └── set
└── main
```

`crate` is `src/main.rs` for a binary, `src/lib.rs` for a library. Every path
in the crate resolves against this tree.

## Private by default

Drop one `pub` and it stops compiling.

```rust
mod db {
    fn connect() {
        println!("connecting");
    }
}

fn main() {
    db::connect();
}
```

Does not compile:

```
error[E0603]: function `connect` is private
 --> src/main.rs:8:9
  |
8 |     db::connect();
  |         ^^^^^^^ private function
  |
note: the function `connect` is defined here
 --> src/main.rs:2:5
  |
2 |     fn connect() {
  |     ^^^^^^^^^^^^
```

`E0603` is the error you will hit most often in this module. It always means
"the item exists, you just can't see it from here".

## A private module hides its public contents

`pub` on an item does nothing if a module on the path is private.

```rust
mod outer {
    mod secret {
        pub fn f() -> i32 { 1 }
    }
    pub fn use_it() -> i32 { secret::f() }
}

fn main() {
    println!("{}", outer::use_it());
    println!("{}", outer::secret::f());
}
```

The last line fails:

```
error[E0603]: module `secret` is private
  --> src/main.rs:10:27
   |
10 |     println!("{}", outer::secret::f());
   |                           ^^^^^^  - function `f` is not publicly re-exported
   |                           |
   |                           private module
   |
note: the module `secret` is defined here
  --> src/main.rs:2:5
   |
 2 |     mod secret {
   |     ^^^^^^^^^^
```

Visibility is checked at **every segment** of the path. `outer::use_it()` works
fine because the call to `secret::f` happens inside `outer`, where `secret` is
visible.

## Child modules can see their parents

A child can read anything in an ancestor, public or not. Visibility only
restricts the *downward* direction.

```rust
mod app {
    const NAME: &str = "svc";

    pub mod config {
        pub fn describe() -> String {
            format!("app named {}", super::NAME)
        }
    }
}

fn main() {
    println!("{}", app::config::describe());
}
```

Output:

```
app named svc
```

`NAME` has no `pub` at all and the child still reads it.

## pub(crate), pub(super), pub(in path)

Four graduated forms, from widest to narrowest:

| Form | Visible to |
|---|---|
| `pub` | Anyone, including other crates |
| `pub(crate)` | Everywhere in this crate |
| `pub(super)` | The parent module and its descendants |
| `pub(in crate::a::b)` | That specific module and its descendants |

```rust
mod app {
    pub(crate) const NAME: &str = "svc";

    pub mod config {
        pub(super) fn secret() -> &'static str {
            "s3cr3t"
        }

        pub(in crate::app) fn internal() -> u32 {
            42
        }

        pub fn public_view() -> String {
            format!("{} {} {}", super::NAME, secret(), internal())
        }
    }

    pub fn boot() {
        println!("{}", config::secret());
        println!("{}", config::internal());
    }
}

fn main() {
    app::boot();
    println!("{}", app::config::public_view());
    println!("{}", app::NAME);
}
```

Output:

```
s3cr3t
42
svc s3cr3t 42
svc
```

`app::boot` can call both `secret` and `internal`; `main` cannot call either.
`pub(crate)` is the workhorse — use it for everything you want shared inside
your own crate but not promised to users.

## pub struct does not make fields pub

Each field needs its own `pub`.

```rust
mod bank {
    pub struct Account {
        pub id: u64,
        balance: i64,
    }

    impl Account {
        pub fn new(id: u64, balance: i64) -> Self {
            Account { id, balance }
        }
        pub fn balance(&self) -> i64 {
            self.balance
        }
    }
}

fn main() {
    let a = bank::Account::new(7, 500);
    println!("{} {}", a.id, a.balance());
}
```

Output:

```
7 500
```

Reading the private field directly gives `E0616`:

```rust
println!("{}", a.balance);
```

```
error[E0616]: field `balance` of struct `Account` is private
  --> src/main.rs:10:22
   |
10 |     println!("{}", a.balance);
   |                      ^^^^^^^ private field
```

And constructing with a struct literal gives a different code, `E0451`:

```rust
let a = bank::Account { id: 1, balance: 10 };
```

```
error[E0451]: field `balance` of struct `Account` is private
 --> src/main.rs:6:36
  |
6 |     let a = bank::Account { id: 1, balance: 10 };
  |                                    ^^^^^^^ private field
```

This is the standard way to force construction through a constructor: make one
field private and expose `new`.

## Enum variants are not independently private

Unlike struct fields, variants inherit the enum's visibility. A `pub enum` has
public variants and public variant fields.

```rust
mod bank {
    pub enum Status {
        Open,
        Frozen { reason: String },
    }
}

fn main() {
    let s = bank::Status::Frozen { reason: "fraud".into() };
    match s {
        bank::Status::Open => println!("open"),
        bank::Status::Frozen { reason } => println!("frozen: {reason}"),
    }
}
```

Output:

```
frozen: fraud
```

There is no `pub` to write on a variant, and no way to hide one. If you need a
non-exhaustive enum, use `#[non_exhaustive]`.

## Leaking a private type

If a `pub fn` returns a private type, the signature is unusable.

```rust
mod a {
    struct Hidden;
    pub fn make() -> Hidden { Hidden }
}

fn main() {
    let _ = a::make();
}
```

```
warning: type `Hidden` is more private than the item `make`
 --> src/main.rs:3:5
  |
3 |     pub fn make() -> Hidden { Hidden }
  |     ^^^^^^^^^^^^^^^^^^^^^^^ function `make` is reachable at visibility `pub(crate)`
  |
note: but type `Hidden` is only usable at visibility `pub(self)`
 --> src/main.rs:2:5
  |
2 |     struct Hidden;
  |     ^^^^^^^^^^^^^

error: type `Hidden` is private
 --> src/main.rs:7:13
  |
7 |     let _ = a::make();
  |             ^^^^^^^^^ private type
```

The warning fires at definition, the error at the call site. Match the
visibility of a type to the widest signature it appears in.

## Absolute and relative paths

Four prefixes. All four refer to the same function below.

```rust
fn top() -> &'static str { "top" }

mod outer {
    pub fn here() -> &'static str { "outer::here" }

    pub mod inner {
        pub fn call_all() {
            println!("{}", crate::top());
            println!("{}", super::here());
            println!("{}", self::local());
            println!("{}", local());
        }
        fn local() -> &'static str { "inner::local" }
    }
}

fn main() {
    outer::inner::call_all();
    crate::outer::inner::call_all();
}
```

Output:

```
top
outer::here
inner::local
inner::local
top
outer::here
inner::local
inner::local
```

- `crate::path` — absolute from the crate root.
- `super::path` — one level up; chain it as `super::super::`.
- `self::path` — explicit "this module"; identical to a bare name.
- bare `path` — relative to the current module.

Idiomatic choice: `crate::` for anything outside your own module, bare names
inside it. Reserve `super::` for sibling access in tightly coupled trees (a
`#[cfg(test)] mod tests` calling `super::*` is the canonical use).

## Modules do not map to files automatically

Creating `src/helpers.rs` does nothing. Nothing in the crate refers to it, so it
is not compiled at all — no error, no warning, just dead weight. The file
becomes a module only when some module declares it:

```rust
mod helpers;
```

Lesson 12.3 covers that layout in full.

The one exception is `src/bin/`. Cargo scans that directory and compiles each
`src/bin/*.rs` as a **separate binary crate**, no declaration needed. Those are
not modules of your crate — they are different crates that can only reach your
code through your library.

## Modules cannot be reopened

Two `mod` blocks with the same name in the same parent is an error, unlike
JS objects or Python module patching.

```rust
mod a { pub fn f() {} }
mod a { pub fn g() {} }
```

```
error[E0428]: the name `a` is defined multiple times
 --> src/main.rs:2:1
  |
1 | mod a { pub fn f() {} }
  | ----- previous definition of the module `a` here
2 | mod a { pub fn g() {} }
  | ^^^^^ `a` redefined here
  |
  = note: `a` must be defined only once in the type namespace of this module
```

## Grouping with inline modules

Inline modules are a cheap way to split one file into logical sections without
touching the filesystem.

```rust
mod math {
    pub fn double(x: i32) -> i32 { x * 2 }
}

mod stats {
    use crate::math::double;
    pub fn report(v: &[i32]) {
        let total: i32 = v.iter().map(|x| double(*x)).sum();
        println!("doubled sum = {total}");
    }
}

fn main() {
    stats::report(&[1, 2, 3]);
    println!("{}", math::double(10));
}
```

Output:

```
doubled sum = 12
20
```

Note `stats` must `use` or path-qualify `math::double` — sibling modules are not
in scope for each other.

## Quick Reference

| Syntax | Meaning |
|---|---|
| `mod name { ... }` | Inline module |
| `mod name;` | Load `name.rs` or `name/mod.rs` as this module |
| `pub` | Visible to all crates |
| `pub(crate)` | Visible within this crate |
| `pub(super)` | Visible to the parent module and below |
| `pub(in crate::a)` | Visible within `crate::a` and below |
| (none) | Visible only in this module and its children |
| `crate::a::b` | Absolute path from crate root |
| `super::b` | Parent module |
| `self::b` | This module |
| `pub struct S { pub a: T, b: U }` | `a` public, `b` private |
| `pub enum E { A, B }` | Variants always public |
| `E0603` | Item exists but is private |
| `E0616` | Private struct field read |
| `E0451` | Private field in a struct literal |

## Common Mistakes

**Expecting `src/foo.rs` to compile on its own.** You must write `mod foo;`
somewhere, or the file is ignored silently.

**Adding `pub` to the item but not the module.** Visibility is checked at every
path segment; a private ancestor module blocks everything inside it.

**Thinking `pub struct` exports its fields.** Each field needs its own `pub`,
or a getter.

**Using `super::` for everything.** It breaks the moment you move the module.
Prefer `crate::` for non-local paths.

**Returning a private type from a `pub fn`.** Compiles with a warning at the
definition and fails at every call site. Promote the type's visibility.

## Exercises

1. Build a crate with `mod geometry { pub mod shapes { ... } }`. Put
   `pub fn area(w: f64, h: f64) -> f64` in `shapes` and call it from `main`
   using an absolute path, then a relative one.
2. Add a private `fn validate` to `geometry` and call it from inside
   `geometry::shapes` using `super::`. Then try calling it from `main` and read
   the error code.
3. Define `pub struct Temp { pub celsius: f64, raw_reading: u16 }` inside a
   module. Add `Temp::new` and a `raw()` getter. Attempt a struct literal from
   `main` and note which error code appears.
4. Take a function you made `pub` and narrow it to `pub(crate)`, then to
   `pub(super)`. At each step, find a call site that stops compiling.
5. Make a `pub fn` in a module return a private struct. Compile it, read both
   the warning and the error, and fix it two ways: by promoting the struct, and
   by narrowing the function.

---

[← 11.7 Weak References and Cycles](../11-smart-pointers/07-weak-cycles.md) · [Course Index](../README.md) · [12.2 use, pub use, and Re-exports →](02-use-reexports.md)
