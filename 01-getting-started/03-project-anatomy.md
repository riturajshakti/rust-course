[← 1.2 Hello, Cargo: Your First Project](02-hello-cargo.md) · [Course Index](../README.md) · [1.4 rustfmt, Clippy, and rust-analyzer →](04-tooling.md)

# 1.3 — Anatomy of a Cargo Project

## What You'll Learn

- Binary crate vs library crate, and the path→target mapping cargo infers
- Why real projects split `lib.rs` + `main.rs` (tests and examples force it)
- `src/bin/`, `examples/`, `benches/`, `tests/`
- The `Cargo.toml` fields you actually set, and what an edition is

## Coming From Another Language

| Other | Rust |
|---|---|
| `package.json` `"main"` / `"bin"` | Inferred from file paths, not declared |
| `__init__.py` | `src/lib.rs` |
| Go: package per directory | Crate per `Cargo.toml`, modules inside |

The big difference from Node and Python: cargo **infers targets from file
paths**. You don't list entry points. Put a file at `src/bin/seed.rs` and you
have a second binary named `seed` with no config at all.

## Crate, package, target

Three words that get used loosely:

- **Package** — one `Cargo.toml`. The unit you publish.
- **Target** — one thing cargo builds: a binary, the library, a test, a bench.
- **Crate** — one compilation unit. A library crate or a binary crate.

A package has at most **one** library target and any number of binary targets.

## The path → target table

This is the whole convention. Memorize this table and you never configure a
build again.

| Path | Target | Name | Build / run with |
|---|---|---|---|
| `src/main.rs` | binary | package name | `cargo run` |
| `src/lib.rs` | library | package name | `cargo build` |
| `src/bin/seed.rs` | binary | `seed` | `cargo run --bin seed` |
| `src/bin/seed/main.rs` | binary | `seed` | `cargo run --bin seed` |
| `examples/demo.rs` | example | `demo` | `cargo run --example demo` |
| `tests/integration.rs` | test | `integration` | `cargo test` |
| `benches/simple.rs` | bench | `simple` | `cargo bench` |
| `build.rs` | build script | — | runs before your crate compiles |

Verify it on a real project with `cargo metadata`:

```
['lib']     anat      src/lib.rs
['bin']     anat      src/main.rs
['bin']     migrate   src/bin/migrate.rs
['bin']     seed      src/bin/seed.rs
['example'] demo      examples/demo.rs
['test']    integration tests/integration.rs
['bench']   simple    benches/simple.rs
```

Seven targets, zero lines of target configuration.

## Binary crate

`cargo new app` gives you `src/main.rs` — a binary crate. It has a `main`, it
compiles to an executable, and **nothing can import it**.

```rust
pub fn add(a: i64, b: i64) -> i64 {
    a + b
}

fn main() {
    println!("{}", add(1, 2));
}
```

Output:

```
3
```

`pub` on `add` does nothing useful here. There is no outside.

## Library crate

```bash
cargo new mathlib --lib
```

You get `src/lib.rs` with a sample test, no `main`, and `cargo run` has nothing
to run. A library exists to be imported — by other packages, and by your own
tests, examples, and binaries.

```rust
pub fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}

pub fn add(a: i64, b: i64) -> i64 {
    a + b
}
```

## Why real projects split lib.rs and main.rs

This is the single most important structural decision in a Rust project, and the
reason is concrete rather than stylistic: **`tests/` and `examples/` can only
import a library.**

Put everything in `src/main.rs` and try to test it:

```rust
pub fn add(a: i64, b: i64) -> i64 {
    a + b
}

fn main() {
    println!("{}", add(1, 2));
}
```

```rust
#[test]
fn adds() {
    assert_eq!(binonly::add(2, 2), 4);
}
```

`cargo test` fails to compile the test:

```
error[E0433]: cannot find module or crate `binonly` in this scope
 --> tests/it.rs:3:16
  |
3 |     assert_eq!(binonly::add(2, 2), 4);
  |                ^^^^^^^ use of unresolved module or unlinked crate `binonly`
  |
  = help: if you wanted to use a crate named `binonly`, use `cargo add binonly` to add it to your `Cargo.toml`
```

The crate is not importable, and `cargo add`-ing yourself is not the fix. The
fix is to move the logic into `src/lib.rs` and leave `main.rs` as a thin shell:

```rust
pub fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}

pub fn add(a: i64, b: i64) -> i64 {
    a + b
}
```

```rust
fn main() {
    println!("{}", anat::greet("raj"));
    println!("{}", anat::add(2, 3));
}
```

Output:

```
Hello, raj!
5
```

Note `main.rs` refers to the library by **package name** (`anat::`), not by
`crate::` or a relative path. From the binary's point of view the library is an
external dependency that happens to live in the same package.

Now the integration test works:

```rust
#[test]
fn adds() {
    assert_eq!(anat::add(2, 2), 4);
}
```

Output:

```
     Running tests/integration.rs (target/debug/deps/integration-5baa17107a42bf87)

running 1 test
test adds ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Three things follow from the split, all of them things you want:

- `tests/` and `examples/` can exercise your code.
- Multiple binaries share one implementation.
- The public API is forced to be explicit — only `pub` items escape `lib.rs`.

Unit tests inside `src/main.rs` (a `#[cfg(test)] mod tests` block) *do* work,
because they compile as part of the binary. It is only `tests/`, `examples/`,
and `benches/` — separate crates — that need a library. Module 13 covers the
distinction.

Rule of thumb: anything beyond a one-file toy gets `lib.rs` + `main.rs`, where
`main.rs` does argument parsing and calls into the library.

## src/bin/ for multiple binaries

One package, several executables, shared library code. Common for a server plus
its migration and seeding tools.

```
src/
├── lib.rs
├── main.rs            -> binary `anat`
└── bin/
    ├── migrate.rs     -> binary `migrate`
    └── seed.rs        -> binary `seed`
```

```rust
fn main() {
    println!("seeding: {}", anat::add(10, 32));
}
```

```bash
cargo run --quiet --bin seed
```

Output:

```
seeding: 42
```

Adding `src/bin/` files breaks bare `cargo run`, because now it is ambiguous:

```bash
cargo run
```

Output:

```
error: `cargo run` could not determine which binary to run. Use the `--bin` option to specify a binary, or the `default-run` manifest key.
available binaries: anat, migrate, seed
```

Fix it with `default-run`:

```toml
[package]
name = "anat"
version = "0.1.0"
edition = "2024"
default-run = "anat"
```

```bash
cargo run --quiet
```

Output:

```
Hello, raj!
5
```

`cargo build` always builds every binary; only `run` needs disambiguating.

## examples/

Compilable, runnable demos of your public API. They are real build targets, so
they cannot rot — `cargo test` type-checks them and CI fails if your API changes
underneath them. This is why examples beat code samples in a README.

```rust
fn main() {
    println!("{}", anat::greet("example"));
}
```

```bash
cargo run --quiet --example demo
```

Output:

```
Hello, example!
```

An example needing extra dependencies uses `[dev-dependencies]` — it is not
shipped to users of your library.

## tests/

Each file in `tests/` is its own crate, compiled separately, with access to only
your **public** API. That is the point: integration tests prove the API is
usable from outside.

```
tests/
├── integration.rs     -> own crate
└── api.rs             -> own crate
```

A shared helper goes in `tests/common/mod.rs`, not `tests/common.rs` — a bare
`.rs` file at the top level of `tests/` would become a test target of its own.

## benches/

```rust
fn main() {
    let start = std::time::Instant::now();
    let total: u64 = (1..=1_000_000u64).sum();
    println!("sum={total} in {:?}", start.elapsed());
}
```

```toml
[[bench]]
name = "simple"
harness = false
```

```bash
cargo bench
```

Output:

```
     Running benches/simple.rs (target/release/deps/simple-d9557901982ee922)
sum=500000500000 in 84ns
```

`harness = false` means "this is a plain `main`, don't wrap it in the test
harness" — required for Criterion and for hand-rolled timing like this.
`cargo bench` builds in **release** mode, which is the only mode worth timing.
Lesson 13.5 covers Criterion properly.

## build.rs

A `build.rs` at the package root is compiled and run *before* your crate, for
code generation, compiling C, or embedding build-time info. It talks to cargo by
printing directives to stdout:

```rust
fn main() {
    println!("cargo:rustc-env=BUILD_TIME={}", "2026-10-25");
    println!("cargo:rerun-if-changed=schema.sql");
}
```

`rerun-if-changed` matters — without it cargo re-runs the script on every build.
Build scripts use `[build-dependencies]`. Most projects never need one.

## Full layout

```
myapp/
├── Cargo.toml
├── Cargo.lock
├── rust-toolchain.toml
├── rustfmt.toml
├── .gitignore
├── build.rs
├── src/
│   ├── lib.rs
│   ├── main.rs
│   ├── config.rs
│   ├── db/
│   │   ├── mod.rs
│   │   └── queries.rs
│   └── bin/
│       ├── migrate.rs
│       └── seed.rs
├── tests/
│   ├── api.rs
│   └── common/
│       └── mod.rs
├── examples/
│   └── demo.rs
├── benches/
│   └── throughput.rs
└── target/
```

Only `src/`, `tests/`, `examples/`, `benches/` and the root files are special to
cargo. Everything else inside `src/` is modules, declared in code with `mod`,
not discovered from the filesystem — unlike Go or Python, creating
`src/config.rs` does nothing until some file says `mod config;`. Module 12 covers
that.

## Cargo.toml fields

```toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2024"
rust-version = "1.99"
authors = ["Raj <raj@example.com>"]
description = "A short one-liner for crates.io"
license = "MIT OR Apache-2.0"
repository = "https://github.com/you/myapp"
readme = "README.md"
keywords = ["web", "api"]
categories = ["web-programming"]
default-run = "myapp"
publish = false

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
local-crate = { path = "../local-crate" }
some-crate = { git = "https://github.com/x/y", branch = "main" }

[dev-dependencies]
assert_cmd = "2"

[build-dependencies]
cc = "1"

[profile.release]
lto = true
codegen-units = 1

[lints.clippy]
unwrap_used = "warn"
```

| Field | Why it matters |
|---|---|
| `name` | Crate name in code (hyphens become underscores) |
| `version` | SemVer; crates.io enforces it |
| `edition` | Language edition — see below |
| `rust-version` | MSRV; cargo refuses to build on older compilers |
| `license` | Required to publish |
| `publish = false` | Guard against accidentally publishing a private app |
| `default-run` | Which binary bare `cargo run` picks |

Note the name translation: a package named `my-app` is imported as `my_app::`.

Dependencies declared inline cover the cases you'll meet: a plain version, a
version with features, a local `path` for workspace-adjacent crates, and `git`
for an unpublished fix.

## Explicit target sections

Only needed when you break convention:

```toml
[lib]
name = "mylib"
path = "src/library.rs"
crate-type = ["rlib", "cdylib"]

[[bin]]
name = "cli"
path = "src/cli/entry.rs"
```

`[[bin]]` is a double bracket because it is an array — many binaries, one
library. `crate-type = ["cdylib"]` builds a C-compatible shared object, which is
how you produce a `.so`/`.dylib` or a WASM module. Prefer the conventional paths;
explicit sections are a smell outside of FFI and WASM.

## Editions

An edition is an opt-in bundle of breaking language changes. The compiler
supports all editions at once, and crates on different editions link together
freely — so there is no ecosystem split, which is the thing Python 2/3 got wrong.

| Edition | Brought |
|---|---|
| 2015 | The original |
| 2018 | `dyn Trait`, non-lexical lifetimes, module path changes |
| 2021 | Disjoint closure captures, `IntoIterator` for arrays |
| 2024 | `gen` keyword reserved, stricter `unsafe` attributes, RPIT lifetime capture |

Editions are not versions. `edition = "2024"` on Rust 1.99.1 is normal and
correct; a crate stuck on `edition = "2015"` still compiles today.

One edition-2024 change you will hit immediately if you copy older code:
attributes like `no_mangle` must now be wrapped in `unsafe(...)`.

```rust
#[unsafe(no_mangle)]
pub fn exported() {}
```

Writing `#[no_mangle]` instead gives:

```
error: unsafe attribute used without unsafe
 --> src/main.rs:2:3
  |
2 | #[no_mangle]
  |   ^^^^^^^^^ usage of unsafe attribute
```

Migrate with `cargo fix --edition` after bumping the field.

## Quick Reference

| Path | Becomes |
|---|---|
| `src/main.rs` | default binary |
| `src/lib.rs` | the library |
| `src/bin/x.rs` | binary `x` |
| `examples/x.rs` | example `x` |
| `tests/x.rs` | integration test crate `x` |
| `benches/x.rs` | bench `x` |
| `build.rs` | build script |

| Command | Does |
|---|---|
| `cargo new x --lib` | Library package |
| `cargo run --bin x` | Run a named binary |
| `cargo run --example x` | Run an example |
| `cargo build --bins` | All binaries |
| `cargo test` | Lib + bins + integration + doc tests |
| `cargo bench` | Benches, in release |
| `cargo metadata --no-deps` | Every inferred target, as JSON |
| `cargo fix --edition` | Migrate to a new edition |

## Common Mistakes

**Putting logic in `main.rs` and then trying to write `tests/`.** A binary crate
cannot be imported. Move it to `lib.rs`; keep `main.rs` thin.

**Adding `src/bin/*.rs` and losing bare `cargo run`.** Expected — set
`default-run` in `[package]`.

**Creating `src/config.rs` and expecting it to be picked up.** Cargo only infers
*targets* from paths. Modules inside `src/` need `mod config;`.

**`tests/common.rs` for shared helpers.** That becomes its own test target with
zero tests. Use `tests/common/mod.rs`.

**Importing your own library as `crate::` from `main.rs`.** They are separate
crates; use the package name with underscores, e.g. `my_app::greet()`.

## Exercises

1. Make a package with both `src/lib.rs` (a `pub fn double(n: i64) -> i64`) and
   `src/main.rs` that calls it. Confirm `main.rs` must use the package name.
2. Add `tests/it.rs` testing `double`. Then move `double` into `main.rs` instead
   and record the exact error `cargo test` produces.
3. Add `src/bin/report.rs` and `src/bin/cleanup.rs`. Run each with `--bin`, then
   run bare `cargo run`, read the error, and fix it with `default-run`.
4. Add `examples/usage.rs`. Change `double`'s signature to take a `&str` and show
   that `cargo test` now fails even though you never ran the example.
5. Run `cargo metadata --no-deps --format-version 1` and find every target your
   package exposes. Match each one to a row in the path→target table.

---

[← 1.2 Hello, Cargo: Your First Project](02-hello-cargo.md) · [Course Index](../README.md) · [1.4 rustfmt, Clippy, and rust-analyzer →](04-tooling.md)
