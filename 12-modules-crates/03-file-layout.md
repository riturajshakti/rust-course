[← 12.2 use, pub use, and Re-exports](02-use-reexports.md) · [Course Index](../README.md) · [12.4 Cargo.toml in Depth →](04-cargo-toml.md)

# 12.3 — Multi-file and Multi-crate Layouts

## What You'll Learn

- `mod foo;` and the two directory layouts: `foo/mod.rs` vs `foo.rs` + `foo/`
- `lib.rs` as crate root, and what `pub` means there
- One package with a lib, a main bin, and extra bins in `src/bin/`
- A realistic backend directory tree you can copy

## Coming From Another Language

| Other | Rust |
|---|---|
| Node: a file is a module, resolved on import | A file is a module only after `mod foo;` |
| Python: `package/__init__.py` re-exports | `foo/mod.rs` or `foo.rs` plays that role |
| Go: directory = package, no declarations | Rust needs an explicit declaration chain |

Every file in `src/` must be reachable from the crate root by a chain of `mod`
declarations, or it is not compiled at all. Cargo does not scan directories —
except `src/bin/`, `examples/`, `tests/` and `benches/`, which it does.

## One module, one file

```rust
// src/main.rs
mod config;

fn main() {
    println!("{}", config::load());
}
```

```rust
// src/config.rs
pub fn load() -> String {
    String::from("config loaded")
}
```

Output:

```
config loaded
```

`mod config;` means "the body of module `config` is in `src/config.rs`". Nothing
else changes: visibility, paths, `super::`, all identical to an inline `mod`.

## The file must exist

```rust
mod missing;
```

```
error[E0583]: file not found for module `missing`
  --> src/main.rs:11:1
   |
11 | mod missing;
   | ^^^^^^^^^^^^
   |
   = help: to create the module `missing`, create file "src/missing.rs" or "src/missing/mod.rs"
   = note: if there is a `mod missing` elsewhere in the crate already, import it with `use crate::...` instead
```

That note is worth reading twice. `mod` *defines* a module. If the module
already exists elsewhere in the crate, you want `use`, not a second `mod` —
declaring the same file twice compiles it twice into two unrelated types.

## Two directory layouts

A module with children needs a directory. Both of these work:

```
src/db/mod.rs        <- older style
src/db/pool.rs

src/routes.rs        <- modern style
src/routes/users.rs
src/routes/health.rs
```

`src/db/mod.rs` and `src/routes.rs` contain exactly the same thing — the body of
that module, including the `mod` declarations for its children.

```rust
// src/routes.rs
pub mod users;
pub mod health;

pub fn register() {
    users::list();
    health::check();
}
```

Idiomatic choice: the modern `foo.rs` + `foo/` form. It avoids a project full of
tabs all named `mod.rs`. Both are fully supported; don't mix them for the same
module:

```
error[E0761]: file for module `routes` found at both "src/routes.rs" and "src/routes/mod.rs"
 --> src/main.rs:3:1
  |
3 | mod routes;
  | ^^^^^^^^^^^
  |
  = help: delete or rename one of them to remove the ambiguity
```

## A real tree, built out

```
src/
├── main.rs
├── config.rs
├── models.rs
├── models/
│   ├── user.rs
│   └── order.rs
├── db/
│   ├── mod.rs
│   └── pool.rs
├── routes.rs
└── routes/
    ├── users.rs
    └── health.rs
```

```rust
// src/main.rs
mod db;
mod models;
mod routes;

fn main() {
    let u = models::user::User::new(1, "raj");
    println!("{:?}", u);
    println!("{}", db::pool::connect("postgres://local"));
    routes::register();
}
```

```rust
// src/models.rs
pub mod user;
pub mod order;
```

```rust
// src/models/user.rs
#[derive(Debug)]
pub struct User {
    pub id: u64,
    pub name: String,
}

impl User {
    pub fn new(id: u64, name: &str) -> Self {
        User { id, name: name.to_string() }
    }
}
```

Sibling access uses `super::`:

```rust
// src/models/order.rs
use super::user::User;

#[derive(Debug)]
pub struct Order {
    pub id: u64,
    pub buyer: User,
}
```

Cross-tree access uses `crate::`:

```rust
// src/routes/users.rs
use crate::models::user::User;

pub fn list() {
    let all = vec![User::new(1, "raj"), User::new(2, "sam")];
    println!("GET /users -> {} rows", all.len());
}
```

Output:

```
User { id: 1, name: "raj" }
connected to postgres://local
GET /users -> 2 rows
GET /health -> ok
```

Note the tree above deliberately uses `db/mod.rs` and `routes.rs` at once —
different modules may use different styles in the same crate.

## lib.rs is the crate root of a library

A package can have one library. Cargo finds it at `src/lib.rs` with no
declaration. `pub` in `lib.rs` is the only thing that is truly *public* — it
means "visible to other crates".

```rust
// src/lib.rs
mod internal;

pub use internal::greeting;

pub fn version() -> &'static str {
    env!("CARGO_PKG_VERSION")
}
```

```rust
// src/internal.rs
pub fn greeting(name: &str) -> String {
    format!("hello, {name}")
}
```

`mod internal;` has no `pub`, so `internal` is invisible outside the crate even
though `greeting` inside it is `pub`. The `pub use` is the only door.

## lib + bin in one package

Put `src/lib.rs` and `src/main.rs` side by side. The binary consumes the library
**as an external crate**, by package name.

```rust
// src/main.rs
fn main() {
    println!("{}", libbin::greeting("raj"));
    println!("v{}", libbin::version());
}
```

Output:

```
hello, raj
v0.1.0
```

Not `crate::greeting` — `libbin::greeting`. The bin and the lib are separate
compilation units, so from the bin's point of view your own library is just
another dependency.

That also means the bin is bound by your real public API:

```rust
fn main() {
    println!("{}", libbin::internal::greeting("db"));
}
```

```
error[E0603]: module `internal` is private
 --> src/bin/seed.rs:2:28
  |
2 |     println!("{}", libbin::internal::greeting("db"));
  |                            ^^^^^^^^  -------- function `greeting` is not publicly re-exported
  |                            |
  |                            private module
  |
note: the module `internal` is defined here
 --> src/lib.rs:1:1
  |
1 | mod internal;
```

This is why the lib + thin bin split is the standard shape for a service: the
binary is forced through the same surface external users get, and `src/lib.rs`
becomes testable and reusable.

## Multiple binaries

Every `src/bin/*.rs` is its own binary crate, picked up with no declaration.

```
src/
├── lib.rs
├── main.rs          -> binary named after the package
└── bin/
    ├── seed.rs      -> binary "seed"
    └── migrate.rs   -> binary "migrate"
```

```rust
// src/bin/seed.rs
fn main() {
    println!("seeding: {}", libbin::greeting("db"));
}
```

```
$ cargo run --bin seed
seeding: hello, db

$ cargo run --bin migrate
migrating with libbin v0.1.0
```

With more than one binary, a bare `cargo run` refuses to guess:

```
error: `cargo run` could not determine which binary to run. Use the `--bin` option to specify a binary, or the `default-run` manifest key.
available binaries: libbin, migrate, seed
```

Add `default-run` to `[package]` to pick one:

```toml
[package]
name = "libbin"
version = "0.1.0"
edition = "2024"
default-run = "libbin"
```

```
$ cargo run
hello, raj
v0.1.0
```

A binary needing several files of its own gets a directory with a `main.rs`:
`src/bin/worker/main.rs` plus `src/bin/worker/queue.rs`.

## The path attribute

`#[path]` overrides the filename lookup. Useful for vendored files or
platform-specific sources; otherwise avoid it, since it breaks the
module-path-equals-file-path intuition.

```rust
#[path = "../vendored/odd_name.rs"]
pub mod vendored;
```

Output:

```
from a path-attributed file
```

The common legitimate use is platform selection:

```rust
#[cfg(unix)]
#[path = "platform/unix.rs"]
mod platform;

#[cfg(windows)]
#[path = "platform/windows.rs"]
mod platform;
```

Both arms declare `mod platform`, but only one survives `cfg`.

## Other directories Cargo scans

| Directory | Compiled as |
|---|---|
| `src/lib.rs` | The library crate |
| `src/main.rs` | A binary named after the package |
| `src/bin/*.rs` | One binary each |
| `examples/*.rs` | One example binary each, `cargo run --example name` |
| `tests/*.rs` | One integration-test crate each (lesson 13.2) |
| `benches/*.rs` | One benchmark crate each |
| `build.rs` | Build script, compiled and run before your crate |

Everything in those directories links against your library the same way
`src/main.rs` does — by package name, through the public API only.

## A realistic backend layout

This is the shape most Axum/Actix services converge on.

```
myservice/
├── Cargo.toml
├── .env
├── migrations/
│   └── 0001_init.sql
├── src/
│   ├── main.rs            thin: load config, build router, serve
│   ├── lib.rs             pub use of the app's real surface
│   ├── error.rs           AppError + IntoResponse + From conversions
│   ├── config.rs          env parsing into a Config struct
│   ├── state.rs           AppState { db: PgPool, cfg: Config }
│   ├── db.rs              pub mod user; pub mod order;
│   ├── db/
│   │   ├── user.rs        queries returning models::User
│   │   └── order.rs
│   ├── models.rs
│   ├── models/
│   │   ├── user.rs        the domain type + serde derives
│   │   └── order.rs
│   ├── routes.rs          pub fn router() -> Router, merges submodules
│   ├── routes/
│   │   ├── users.rs       handlers only
│   │   ├── orders.rs
│   │   └── health.rs
│   └── middleware/
│       ├── mod.rs
│       └── auth.rs
├── tests/
│   └── api.rs             integration tests against the public router()
└── examples/
    └── seed.rs
```

Rules that keep it from rotting:

- `main.rs` holds no logic. It calls `lib.rs`.
- `routes/*` never touches SQL; it calls `db/*`.
- `models/*` depends on nothing in the crate — it is the leaf.
- `error.rs` sits at the root because everything imports it.
- Every layer boundary is `pub(crate)` unless it is genuinely part of the
  library's API, in which case it is `pub` and re-exported from `lib.rs`.

When a single `src/` gets unwieldy, the next step is not more nesting — it is
splitting into crates in a workspace. Lesson 12.5.

## Multi-crate: the two ways

A *package* is one `Cargo.toml`. A *crate* is one compilation unit.

- One package, many crates: a lib plus N bins, as above. They share one
  `Cargo.toml`, one version, one dependency set.
- Many packages: a workspace. Separate `Cargo.toml` files, separate dependency
  sets, path dependencies between them. Covered in 12.5.

Reach for a workspace when two parts of the system want *different
dependencies* — e.g. your domain logic should not be able to import `sqlx`. The
compiler enforces that split; a module boundary does not.

## Quick Reference

| Need | Write |
|---|---|
| Module in its own file | `mod foo;` + `src/foo.rs` |
| Module with children (modern) | `mod foo;` + `src/foo.rs` + `src/foo/*.rs` |
| Module with children (legacy) | `mod foo;` + `src/foo/mod.rs` + `src/foo/*.rs` |
| Child declarations | Go in `foo.rs` / `foo/mod.rs`, not the parent |
| Sibling path | `use super::sibling::Item;` |
| Cross-tree path | `use crate::a::b::Item;` |
| Library root | `src/lib.rs`, no declaration needed |
| Bin using own lib | `use mypackage::Item;` (external-crate path) |
| Extra binary | `src/bin/name.rs`, run with `--bin name` |
| Multi-file binary | `src/bin/name/main.rs` |
| Pick default binary | `default-run = "name"` in `[package]` |
| Non-standard filename | `#[path = "..."] mod foo;` |
| `E0583` | `mod foo;` with no matching file |
| `E0761` | Both `foo.rs` and `foo/mod.rs` exist |

## Common Mistakes

**Creating `src/foo.rs` and expecting it to compile.** Without `mod foo;` in the
parent it is dead text. No warning either.

**Declaring the same file with `mod` from two places.** You get two separate
modules with duplicated, incompatible types. The second place wants `use`.

**Putting `mod bar;` in the parent instead of in `foo.rs`.** `mod` declarations
for `src/foo/bar.rs` belong inside module `foo`.

**Writing `crate::greeting` in `src/main.rs` of a lib+bin package.** From the
binary, your library is an external crate: `mypackage::greeting`.

**Assuming `src/bin/*.rs` can see your private modules.** It cannot — it goes
through the public API like any consumer. That is the point.

## Exercises

1. Convert a single-file program with three inline `mod` blocks into
   `src/a.rs`, `src/b.rs`, `src/c.rs`. Keep the output identical.
2. Give module `b` two children using `src/b.rs` + `src/b/`. Then convert it to
   `src/b/mod.rs` and confirm nothing else changes. Create both at once and
   read the error code.
3. Split a program into `src/lib.rs` + `src/main.rs`. Make one module private in
   `lib.rs` and re-export one item from it. Confirm `main.rs` can only reach the
   re-export.
4. Add `src/bin/seed.rs` and `src/bin/report.rs`. Run each with `--bin`, then
   run a bare `cargo run` and fix the error with `default-run`.
5. Build the backend layout above as empty stubs: `routes/users.rs` with a
   handler that calls a `db/user.rs` function that returns a
   `models/user.rs` struct. Make `db` `pub(crate)` and `models` `pub`, and
   verify from `src/main.rs` which paths resolve.

---

[← 12.2 use, pub use, and Re-exports](02-use-reexports.md) · [Course Index](../README.md) · [12.4 Cargo.toml in Depth →](04-cargo-toml.md)
