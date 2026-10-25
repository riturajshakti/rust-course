[← Course Index](../README.md)

# 12.4 — Cargo.toml in Depth

## What You'll Learn

- Every common `[package]` field and what it affects
- What `^1.2`, `~1.2`, `1.*` and `=1.2.0` actually allow — measured, not guessed
- `dev-`, `build-` and target-specific dependencies
- `[profile.*]`, `[lints]`, `[[bin]]`/`[lib]`/`[[example]]`, and `Cargo.lock`

## Coming From Another Language

| Other | Rust |
|---|---|
| `package.json` + `package-lock.json` | `Cargo.toml` + `Cargo.lock` |
| `^1.2.3` in npm | `^1.2.3` in Cargo — same meaning |
| `devDependencies` | `[dev-dependencies]` |
| `go.mod` + minimal version selection | Cargo picks the *highest* compatible |

Two surprises: a bare `"1.2"` **means** `"^1.2"`, and for `0.x` the minor acts
as the major — `0.2` and `0.3` are incompatible.

## A manifest with everything

This compiles; each section is explained below.

```toml
[package]
name = "manifest"
version = "0.2.0"
edition = "2024"
rust-version = "1.90"
authors = ["Raj <raj@example.com>"]
description = "A demo manifest showing every common field"
license = "MIT OR Apache-2.0"
repository = "https://github.com/example/manifest"
keywords = ["demo", "manifest", "cargo"]
categories = ["development-tools"]
readme = "README.md"
exclude = ["/fixtures", "*.bak"]

[dependencies]
semver = "1.0"
log = { version = "0.4", default-features = false }

[dev-dependencies]
tempfile = "3"

[build-dependencies]
cc = "1"

[target.'cfg(unix)'.dependencies]
libc = "0.2"

[lib]
name = "manifest"
path = "src/lib.rs"

[[bin]]
name = "mcli"
path = "src/main.rs"

[[example]]
name = "demo"
path = "examples/demo.rs"

[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
panic = "abort"
strip = "symbols"
debug = false

[profile.dev]
opt-level = 0
debug = true

[lints.rust]
unsafe_code = "forbid"

[lints.clippy]
unwrap_used = "warn"
```

```
$ cargo run --bin mcli
5

$ cargo run --example demo
example demo
```

## [package] fields

| Field | Required | Effect |
|---|---|---|
| `name` | yes | Crate name, registry name, default binary name |
| `version` | yes | SemVer; what downstream `^` ranges resolve against |
| `edition` | no (defaults 2015) | Language edition. Always set it — `2024` |
| `rust-version` | no | MSRV. Cargo refuses to build on older toolchains and avoids too-new deps |
| `authors` | no | Metadata only, not shown on crates.io any more |
| `description` | publish | One line shown in search results |
| `license` | publish | SPDX expression; `OR` for dual licensing |
| `license-file` | alt. | Path, for licenses with no SPDX id |
| `repository` | no | Link on crates.io and docs.rs |
| `homepage` / `documentation` | no | Extra links; `documentation` defaults to docs.rs |
| `keywords` | no | Max 5, 20 chars each, for search |
| `categories` | no | Max 5, from a fixed crates.io list |
| `readme` | no | Rendered as the crate's front page; defaults to `README.md` |
| `exclude` / `include` | no | What goes in the `.crate` tarball |
| `default-run` | no | Which binary a bare `cargo run` picks |
| `publish` | no | `false` blocks `cargo publish` entirely |

`rust-version` changes dependency resolution, not just docs:

```
$ cargo update --dry-run
    Updating crates.io index
     Locking 1 package to highest Rust 1.90 compatible version
    Updating log v0.4.20 -> v0.4.34
warning: not updating lockfile due to dry run
```

`exclude` and `include` are mutually exclusive; prefer `include`.

## Version requirements, measured

`semver` is the same crate Cargo uses. This resolves the same ranges Cargo does:

```rust
use semver::{Version, VersionReq};

fn show(req: &str, cands: &[&str]) {
    let r = VersionReq::parse(req).unwrap();
    let hits: Vec<&str> = cands
        .iter()
        .copied()
        .filter(|s| r.matches(&Version::parse(s).unwrap()))
        .collect();
    println!("{:<10} -> {:?}", req, hits);
}

fn main() {
    let c = ["1.2.0", "1.2.9", "1.3.0", "1.9.9", "2.0.0"];
    show("1.2", &c);
    show("^1.2", &c);
    show("^1.2.3", &c);
    show("~1.2", &c);
    show("~1.2.3", &c);
    show("1.*", &c);
    show("*", &c);
    show("=1.2.0", &c);
    show(">=1.3, <2", &c);

    let z = ["0.2.0", "0.2.5", "0.3.0", "0.3.1", "1.0.0"];
    println!();
    show("0.2", &z);
    show("^0.2.1", &z);
    show("0.0.3", &["0.0.3", "0.0.4"]);
}
```

Output:

```
1.2        -> ["1.2.0", "1.2.9", "1.3.0", "1.9.9"]
^1.2       -> ["1.2.0", "1.2.9", "1.3.0", "1.9.9"]
^1.2.3     -> ["1.2.9", "1.3.0", "1.9.9"]
~1.2       -> ["1.2.0", "1.2.9"]
~1.2.3     -> ["1.2.9"]
1.*        -> ["1.2.0", "1.2.9", "1.3.0", "1.9.9"]
*          -> ["1.2.0", "1.2.9", "1.3.0", "1.9.9", "2.0.0"]
=1.2.0     -> ["1.2.0"]
>=1.3, <2  -> ["1.3.0", "1.9.9"]

0.2        -> ["0.2.0", "0.2.5"]
^0.2.1     -> ["0.2.5"]
0.0.3      -> ["0.0.3"]
```

Key facts:

- `"1.2"` and `"^1.2"` are **identical** — caret is the default.
- `^1.2` allows up to `1.9.9`. It does *not* pin the minor.
- `^1.2.3` excludes `1.2.0`: the patch is a floor, not a pin.
- `~1.2` allows only `1.2.x`; `~1.2.3` allows `>=1.2.3, <1.3.0`.
- For `0.x` the first *non-zero* field is the breaking one.

Idiomatic choice: a bare `"1.0"` or `"0.4"`. Use `=` only to dodge a
known-broken release.

Cargo picks the **highest** version in range, unlike Go's minimal version
selection. With `rand = "0.10"` and `semver = "1.0"` declared:

```
$ cargo tree
manifest v0.2.0 (/.../manifest)
├── rand v0.10.3
│   ├── chacha20 v0.10.2
│   │   ├── cfg-if v1.0.5
│   │   └── rand_core v0.10.1
│   ├── getrandom v0.4.3
│   │   ├── cfg-if v1.0.5
│   │   ├── libc v0.2.191
│   │   └── rand_core v0.10.1
│   └── rand_core v0.10.1
└── semver v1.0.28
```

## Dependency forms

```toml
[dependencies]
semver = "1.0"

log = { version = "0.4", default-features = false, features = ["std"] }

tokio = { version = "1", features = ["rt-multi-thread", "macros"] }

helper = { path = "../helper" }

mycrate = { git = "https://github.com/me/mycrate", branch = "main" }
pinned  = { git = "https://github.com/me/mycrate", rev = "a1b2c3d" }
tagged  = { git = "https://github.com/me/mycrate", tag = "v0.3.0" }

internal = { version = "1.0", registry = "my-registry" }

serde_renamed = { package = "serde", version = "1.0" }

maybe = { version = "1", optional = true }
```

A `path` dependency is never published. A `git` dependency with no
`rev`/`tag`/`branch` tracks the default branch, pinned to a commit by
`Cargo.lock`. `package = ` renames a crate locally — the only way to depend on
two majors of the same crate at once.

```rust
fn main() {
    println!("{}", helper::triple(5));
    let v = semver::Version::parse("1.2.3").unwrap();
    println!("{} {} {}", v.major, v.minor, v.patch);
}
```

Output:

```
15
1 2 3
```

crates.io will not accept a package with `path` or `git` dependencies — those
must be swapped for registry versions before publishing (12.7).

## dev-dependencies and build-dependencies

```toml
[dev-dependencies]
tempfile = "3"
criterion = "0.5"

[build-dependencies]
cc = "1"
```

- `[dev-dependencies]`: available to `tests/`, `benches/`, `examples/`, doc
  tests and `#[cfg(test)]` code. Not compiled for a normal build, not inherited
  by your users.
- `[build-dependencies]`: available only to `build.rs`, compiled for the *host*
  — which matters when cross-compiling.

## Target-specific dependencies

```toml
[target.'cfg(unix)'.dependencies]
libc = "0.2"

[target.'cfg(windows)'.dependencies]
windows-sys = "0.52"

[target.'cfg(target_arch = "wasm32")'.dependencies]
wasm-bindgen = "0.2"

[target.x86_64-unknown-linux-gnu.dependencies]
openssl-sys = "0.9"

[target.'cfg(unix)'.dev-dependencies]
nix = "0.29"
```

The key is a `cfg(...)` expression or a literal target triple.
`cfg(feature = "x")` is **not** allowed here — use optional dependencies (12.6).

## [profile.*]

Built-ins: `dev`, `release`, `test`, `bench` — the last two inherit from the
first two.

```toml
[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
panic = "abort"
strip = "symbols"
debug = false
```

| Key | Values | Meaning |
|---|---|---|
| `opt-level` | `0`-`3`, `"s"`, `"z"` | `3` = fast, `"s"`/`"z"` = small |
| `lto` | `false`, `"thin"`, `"fat"`/`true` | Cross-crate inlining; `"thin"` is the good default |
| `codegen-units` | integer | `1` = best code, slowest build. Default 16 release |
| `panic` | `"unwind"`, `"abort"` | `abort` drops unwinding tables; incompatible with `cargo test` |
| `strip` | `"none"`, `"debuginfo"`, `"symbols"` | Shrinks the binary |
| `debug` | `false`, `true`, `"line-tables-only"`, `0`-`2` | Debug info level |
| `debug-assertions` | bool | Enables `debug_assert!` and the `debug_assertions` cfg |
| `overflow-checks` | bool | Panic on integer overflow |
| `incremental` | bool | Faster rebuilds, slightly worse code |

`panic = "abort"` plus `cargo test --release` is a common trap: the harness
needs unwinding. Scope it to a custom profile.

Per-dependency overrides keep your crate fast to compile while optimizing the
slow dependency:

```toml
[profile.dev.package.image]
opt-level = 3

[profile.dev.package."*"]
opt-level = 2
```

Custom profiles inherit from a built-in:

```toml
[profile.dist]
inherits = "release"
lto = "fat"
codegen-units = 1
strip = "symbols"
```

`cargo build --profile dist` puts the output in `target/dist/`.

## [lints]

Lint levels in the manifest instead of `#![deny(...)]` in source. Per-package;
workspaces can share them (12.5).

```toml
[lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"
unused_crate_dependencies = "allow"

[lints.clippy]
unwrap_used = "warn"
pedantic = { level = "warn", priority = -1 }
```

`forbid` really does forbid:

```rust
pub fn danger() -> i32 {
    unsafe { 42 }
}
```

```
error: usage of an `unsafe` block
 --> src/lib.rs:4:5
  |
4 |     unsafe { 42 }
  |     ^^^^^^^^^^^^^
  |
  = note: requested on the command line with `-F unsafe-code`
```

And the clippy table takes effect on `cargo clippy`:

```rust
pub fn parse(s: &str) -> i32 { s.parse::<i32>().unwrap() }
```

```
warning: used `unwrap()` on a `Result` value
 --> src/lib.rs:2:32
  |
2 | pub fn parse(s: &str) -> i32 { s.parse::<i32>().unwrap() }
  |                                ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: if this value is an `Err`, it will panic
  = help: consider using `expect()` to provide a better panic message
  = note: requested on the command line with `-W clippy::unwrap-used`
```

`priority = -1` on a lint *group* lets individual lints override it; without it
you get a conflict error.

## Target sections

The defaults match the standard layout; write these only to override.

```toml
[lib]
name = "manifest"
path = "src/lib.rs"
crate-type = ["lib"]
doctest = true
bench = false

[[bin]]
name = "mcli"
path = "src/main.rs"

[[bin]]
name = "worker"
path = "src/bin/worker/main.rs"

[[example]]
name = "demo"
path = "examples/demo.rs"

[[test]]
name = "api"
path = "tests/api.rs"
harness = true

[[bench]]
name = "throughput"
harness = false
```

`[lib]` is singular; the rest are TOML arrays-of-tables, one block each.

`crate-type` matters when you leave pure Rust:

| Value | Produces |
|---|---|
| `"lib"` / `"rlib"` | Normal Rust library (default) |
| `"cdylib"` | `.so`/`.dylib`/`.dll` for FFI and wasm |
| `"staticlib"` | `.a` for linking into C |
| `"proc-macro"` | Procedural macro crate (16.4) |

`harness = false` means "this file has its own `main`" — what Criterion needs.

## Cargo.lock semantics

The exact resolved version and checksum of every package in the graph (from the
`rand` tree above):

```
[[package]]
name = "cfg-if"
version = "1.0.5"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "4e7648175b45a9a48536d676f68d918270699102aa8dab5496df06904c914600"

[[package]]
name = "chacha20"
version = "0.10.2"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "65c35e4b699c7e15ccbe7ee35c005e4fc0a278d22238a2857e6ce2dadeda1b06"
dependencies = [
 "cfg-if",
 "cpufeatures",
 "rand_core",
]
```

Rules:

- Commit it, for binaries and libraries alike. Consumers ignore your lockfile,
  so for a library it only pins your own CI.
- Cargo rewrites it when `Cargo.toml` changes and otherwise leaves versions
  alone — a fresh `cargo build` bumps nothing.
- `cargo build --locked` fails if the lock would need changing. Use it in CI.
  `--offline` and `--frozen` are stricter still.

## cargo update

```
$ cargo update --dry-run
    Updating crates.io index
     Locking 1 package to highest Rust 1.90 compatible version
    Updating log v0.4.20 -> v0.4.34
warning: not updating lockfile due to dry run
```

`cargo update` moves versions **within** the ranges in `Cargo.toml`. It never
edits the manifest, so it can never pull a breaking major.

```
$ cargo update semver --precise 1.0.27
    Updating crates.io index
 Downgrading semver v1.0.28 -> v1.0.27
note: pass `--verbose` to see 1 unchanged dependencies behind latest
```

Useful forms:

| Command | Does |
|---|---|
| `cargo update` | Bump everything within range |
| `cargo update semver` | Bump one package |
| `cargo update semver --precise 1.0.27` | Pin exactly, including downgrades |
| `cargo update --breaking` | Bump across majors *and* edit `Cargo.toml` |

`cargo upgrade` is not built in:

```
$ cargo upgrade
error: no such command: `upgrade`

help: a command with a similar name exists: `update`
```

It ships with `cargo-edit`, and rewrites the requirements in `Cargo.toml`
itself — which `cargo update` will not do.

## SemVer in practice

What you owe downstream users:

| Change | Required bump |
|---|---|
| Bug fix, no API change | patch |
| New `pub` item, new trait impl for your own type | minor |
| New default feature | minor |
| Removing or renaming any `pub` item | major |
| Adding a field to a `pub struct` with public fields | major |
| Adding a variant to a `pub enum` | major unless `#[non_exhaustive]` |
| Adding a required method to a `pub trait` | major unless defaulted |
| Narrowing a function's argument types | major |
| Raising `rust-version` | minor by convention, major if you're strict |
| Bumping a dependency that appears in your public API | major |

That last one catches people: if you return `serde_json::Value`, your users'
`serde_json` must match yours. Defend with `#[non_exhaustive]`, private fields
behind getters, and sealed traits.

## Quick Reference

| Section | Purpose |
|---|---|
| `[package]` | Identity, metadata, edition, MSRV |
| `[dependencies]` | Normal deps |
| `[dev-dependencies]` | Tests, benches, examples, doc tests |
| `[build-dependencies]` | `build.rs` only, host-compiled |
| `[target.'cfg(..)'.dependencies]` | Platform-conditional deps |
| `[features]` | Optional functionality (12.6) |
| `[profile.dev/release/test/bench]` | Codegen settings |
| `[profile.X.package.Y]` | Per-dependency codegen override |
| `[lints.rust]` / `[lints.clippy]` | Lint levels for this package |
| `[lib]` | Library target overrides |
| `[[bin]]` / `[[example]]` / `[[test]]` / `[[bench]]` | One block per target |
| `[workspace]` | Workspace definition (12.5) |
| `[patch]` / `[replace]` | Swap a dep graph-wide |

| Requirement | Allows |
|---|---|
| `"1.2"` / `"^1.2"` | `>=1.2.0, <2.0.0` |
| `"^1.2.3"` | `>=1.2.3, <2.0.0` |
| `"~1.2"` | `>=1.2.0, <1.3.0` |
| `"~1.2.3"` | `>=1.2.3, <1.3.0` |
| `"1.*"` | `>=1.0.0, <2.0.0` |
| `"*"` | Anything (rejected by crates.io) |
| `"=1.2.0"` | Exactly `1.2.0` |
| `">=1.3, <2"` | Explicit range |
| `"0.2"` | `>=0.2.0, <0.3.0` |
| `"0.0.3"` | Exactly `0.0.3` |

## Common Mistakes

**Writing `~1.2` to "stay on 1.2".** That is what it does, and it usually means
you miss bug fixes in `1.3`. A bare `"1.2"` is almost always what you wanted.

**Treating `^0.2` like `^1.2`.** For `0.x` the minor is the breaking field;
`^0.2` will not take `0.3`.

**Setting `panic = "abort"` in `[profile.release]` then running
`cargo test --release`.** The harness needs unwinding. Use a custom profile.

**Omitting `edition`.** The default is 2015, not the latest. Always set
`edition = "2024"`.

**Committing a `path` or `git` dependency you intend to publish.** crates.io
rejects both. Swap them for registry versions first.

**Forgetting that `cargo update` cannot cross a major.** It only moves inside
your declared ranges. Edit `Cargo.toml` (or use `--breaking`) for majors.

## Exercises

1. Write a manifest with `rust-version = "1.75"` and a dependency whose latest
   release needs a newer compiler. Run `cargo update --dry-run` and read what
   version Cargo locks.
2. Add `semver` as a dependency and write a program that prints which of
   `0.1.9`, `0.2.0`, `1.0.0`, `1.0.1`, `2.0.0` satisfy `^0.1`, `^1.0`, `~1.0`,
   and `>=0.2, <2`.
3. Add a `[profile.dist]` that inherits from `release` with `lto = "fat"` and
   `strip = "symbols"`. Build with and without it and compare binary sizes.
4. Add `[lints.clippy]` with `unwrap_used = "deny"`, then write a function that
   calls `.unwrap()`. Confirm `cargo clippy` fails, and fix it with `?`.
5. Create a `path` dependency on a sibling crate, then run
   `cargo publish --dry-run` and read the exact rejection.

---

[← Course Index](../README.md)
