[← 12.5 Workspaces](05-workspaces.md) · [Course Index](../README.md) · [12.7 Publishing to crates.io →](07-publishing.md)

# 12.6 — Features and Conditional Compilation

## What You'll Learn

- `[features]`, `default`, optional dependencies, and `dep:` syntax
- Feature unification, why it bites, and the additive-only rule
- `#[cfg(...)]`, `cfg!`, `#[cfg_attr(...)]` beyond features
- `--features` / `--no-default-features` / `--all-features`, and CI matrices

## Coming From Another Language

| Other | Rust |
|---|---|
| npm `peerDependenciesMeta.optional` | `optional = true` + `[features]` |
| Python `pip install pkg[extra]` | `features = ["extra"]` |
| Go build tags | `#[cfg(...)]`, but compile-time only |

Features are compile-time flags that add code and dependencies. The rule that
shapes everything: they must be **purely additive**, because the whole
dependency graph gets *one* unified feature set per crate.

## A crate with an optional serde feature

```toml
[package]
name = "feat"
version = "0.1.0"
edition = "2024"

[features]
default = ["pretty"]
pretty = []
json = ["dep:serde", "dep:serde_json"]
metrics = []
full = ["pretty", "json", "metrics"]

[dependencies]
serde = { version = "1.0", features = ["derive"], optional = true }
serde_json = { version = "1.0", optional = true }
```

```rust
#[derive(Debug)]
#[cfg_attr(feature = "json", derive(serde::Serialize, serde::Deserialize))]
pub struct Point { pub x: i32, pub y: i32 }

impl Point {
    pub fn new(x: i32, y: i32) -> Self { Point { x, y } }

    #[cfg(feature = "pretty")]
    pub fn render(&self) -> String { format!("({}, {})", self.x, self.y) }

    #[cfg(not(feature = "pretty"))]
    pub fn render(&self) -> String { format!("{},{}", self.x, self.y) }

    #[cfg(feature = "json")]
    pub fn to_json(&self) -> String { serde_json::to_string(self).unwrap() }
}

pub fn enabled() -> Vec<&'static str> {
    let mut v = Vec::new();
    if cfg!(feature = "pretty") { v.push("pretty"); }
    if cfg!(feature = "json") { v.push("json"); }
    if cfg!(feature = "metrics") { v.push("metrics"); }
    if cfg!(debug_assertions) { v.push("debug_assertions"); }
    v
}
```

`render` has two mutually exclusive definitions — fine, since exactly one
survives `cfg`.

## Building it every way

```
$ cargo run
render:  (3, 4)
enabled: ["pretty", "debug_assertions"]
json:    <feature off>
```

```
$ cargo run --features json
render:  (3, 4)
enabled: ["pretty", "json", "debug_assertions"]
json:    {"x":3,"y":4}
```

```
$ cargo run --no-default-features
render:  3,4
enabled: ["debug_assertions"]
json:    <feature off>
```

```
$ cargo run --all-features
render:  (3, 4)
enabled: ["pretty", "json", "metrics", "debug_assertions"]
json:    {"x":3,"y":4}
```

```
$ cargo run --no-default-features --features full
render:  (3, 4)
enabled: ["pretty", "json", "metrics", "debug_assertions"]
json:    {"x":3,"y":4}
```

`full` is a *meta-feature* — a list of other features. `default` is the same
mechanism with a reserved name.

## The dependency cost, measured

The whole reason optional dependencies exist.

```
$ cargo tree
feat v0.1.0 (/.../feat)
```

```
$ cargo tree --features json
feat v0.1.0 (/.../feat)
├── serde v1.0.229
│   ├── serde_core v1.0.229
│   └── serde_derive v1.0.229 (proc-macro)
│       ├── proc-macro2 v1.0.107
│       │   └── unicode-ident v1.0.27
│       ├── quote v1.0.47
│       │   └── proc-macro2 v1.0.107 (*)
│       └── syn v3.0.9
│           ├── proc-macro2 v1.0.107 (*)
│           ├── quote v1.0.47 (*)
│           └── unicode-ident v1.0.27
└── serde_json v1.0.154
    ├── itoa v1.0.18
    ├── memchr v2.8.3
    ├── serde_core v1.0.229
    └── zmij v1.0.23
```

Zero dependencies versus eleven, including `syn` and `proc-macro2`. A user who
does not need JSON pays nothing.

## Optional dependencies and implicit features

`optional = true` creates an **implicit feature with the same name**, unless
you mention the dependency with `dep:`. Without it:

```toml
[features]
json = ["serde", "serde_json"]

[dependencies]
serde = { version = "1.0", optional = true }
serde_json = { version = "1.0", optional = true }
```

`serde` becomes a public feature of your crate:

```
$ cargo tree --features serde
feat v0.1.0 (/.../feat)
└── serde v1.0.229
    ├── serde_core v1.0.229
    └── serde_derive v1.0.229 (proc-macro)
```

Almost never what you want: it leaks a dependency name into your API, so you
can never rename or drop it without a breaking change.

With `dep:`:

```toml
[features]
json = ["dep:serde", "dep:serde_json"]
```

The implicit feature disappears:

```
$ cargo run --features serde
error: package `feat v0.1.0 (/.../feat)` does not have feature `serde`

help: an optional dependency with that name exists, but the `features` table includes it with the "dep:" syntax so it does not have an implicit feature with that name
Dependency `serde` would be enabled by these features:
	- `json`
```

Idiomatic choice: always `dep:`. Your feature names are then your own.

## Enabling a dependency's features through yours

Three related syntaxes:

```toml
[features]
json = ["dep:serde", "dep:serde_json"]

derive = ["serde/derive"]

async = ["dep:tokio", "tokio/rt-multi-thread"]

tracing = ["dep:tracing", "tokio?/tracing"]
```

- `"serde/derive"` enables `serde`'s `derive` feature **and** turns on `serde`
  if it is optional.
- `"tokio?/tracing"` enables `tracing` on `tokio` *only if* something else
  already enabled `tokio` — the `?` avoids dragging in a dependency just to
  configure it.

## Feature unification, and why it bites

Cargo resolves **one** feature set per crate per build — the union of every
request in the graph. There is no way to have two copies with different
features.

Three members: `lightweight` depends on `feat` with `default-features = false`,
`heavy` with `features = ["json"]`, and `app` on both. `lightweight` alone:

```
$ cargo test -p lightweight -- --nocapture
running 1 test
["debug_assertions"]
```

Exactly what it asked for. Now the whole workspace:

```
$ cargo test --workspace -- --nocapture
running 1 test
["pretty", "json", "debug_assertions"]
```

**Same crate, same code, different features** — because a sibling asked for
them. `default-features = false` held only until someone else wanted the
defaults. This is the most surprising thing about Cargo features.

Consequences:

- You cannot opt out of a feature a sibling enables.
  `default-features = false` is a request, not a guarantee.
- `cargo test -p x` and `cargo test --workspace` can build `x` differently. CI
  that runs only one can miss a real failure.
- A dependency that compiles in your crate may fail once a user adds a second
  crate that shares it.

`resolver = "3"` limits this: features enabled only for build dependencies,
proc macros, or another target stay separate. It does **not** separate two
normal dependencies.

## The additive-only rule

Because of unification, a feature must only ever *add*. Changing or removing an
API breaks unrelated crates. The classic violation:

```rust
#[cfg(not(feature = "no-std"))]
pub fn alloc_name() -> String { String::from("std") }

#[cfg(feature = "no-std")]
pub fn alloc_name() -> &'static str { "no-std" }
```

Standalone it looks fine:

```
$ cargo run
std

$ cargo run --features no-std
no-std
```

A consumer written against the default:

```rust
pub fn owned() -> String { feat::alloc_name() }
```

```
$ cargo build -p lightweight
(ok)
```

Add a sibling that enables `no-std`, and the consumer breaks without changing
a line:

```
$ cargo build -p app
error[E0308]: mismatched types
 --> lightweight/src/lib.rs:1:28
  |
1 | pub fn owned() -> String { feat::alloc_name() }
  |                   ------   ^^^^^^^^^^^^^^^^^^ expected `String`, found `&str`
  |                   |
  |                   expected `String` because of return type
  |
help: try using a conversion method
--
error: could not compile `lightweight` (lib) due to 1 previous error
```

The author of `lightweight` did nothing wrong, and the error blames their code.
Rules that follow:

- Never change a signature or remove a `pub` item behind a feature.
- Name features for what they *add* (`std`, `serde`, `tls`), never what they
  remove. Make `std` a *default* feature and let users pass
  `--no-default-features`.
- Mutually exclusive features need a `compile_error!` guard:

```rust
#[cfg(all(feature = "tls-rustls", feature = "tls-native"))]
compile_error!("features `tls-rustls` and `tls-native` are mutually exclusive");
```

That still breaks unification, but fails loudly instead of silently.

## cfg beyond features

`#[cfg]` takes any configuration predicate, not just `feature`.

```rust
#[cfg(target_os = "macos")]
fn platform() -> &'static str { "macos" }

#[cfg(target_os = "linux")]
fn platform() -> &'static str { "linux" }

#[cfg(windows)]
fn platform() -> &'static str { "windows" }

#[cfg(all(unix, target_pointer_width = "64"))]
fn width() -> &'static str { "unix 64-bit" }

#[cfg(not(all(unix, target_pointer_width = "64")))]
fn width() -> &'static str { "something else" }

#[cfg(any(target_arch = "x86_64", target_arch = "aarch64"))]
const ARCH_OK: bool = true;

fn main() {
    println!("platform     {}", platform());
    println!("width        {}", width());
    println!("arch_ok      {ARCH_OK}");
    println!("cfg os       {}", cfg!(target_os = "macos"));
    println!("cfg arch     {}", cfg!(target_arch = "aarch64"));
    println!("cfg family   {}", cfg!(target_family = "unix"));
    println!("cfg endian   {}", cfg!(target_endian = "little"));
    println!("cfg ptr      {}", cfg!(target_pointer_width = "64"));
    println!("cfg debug    {}", cfg!(debug_assertions));
    println!("cfg test     {}", cfg!(test));
    println!("env ARCH     {}", std::env::consts::ARCH);
    println!("env OS       {}", std::env::consts::OS);
}
```

Output on an Apple Silicon Mac:

```
platform     macos
width        unix 64-bit
arch_ok      true
cfg os       true
cfg arch     true
cfg family   true
cfg endian   true
cfg ptr      true
cfg debug    true
cfg test     false
env ARCH     aarch64
env OS       macos
```

In release `debug_assertions` flips:

```
$ cargo run --release
cfg debug    false
```

The predicates worth knowing:

| Predicate | Values |
|---|---|
| `target_os` | `"linux"`, `"macos"`, `"windows"`, `"android"`, `"ios"`, `"wasi"`, … |
| `target_arch` | `"x86_64"`, `"aarch64"`, `"wasm32"`, `"arm"`, … |
| `target_family` | `"unix"`, `"windows"`, `"wasm"` |
| `unix` / `windows` | Shorthand for `target_family` |
| `target_pointer_width` | `"16"`, `"32"`, `"64"` |
| `target_endian` | `"little"`, `"big"` |
| `target_env` | `"gnu"`, `"musl"`, `"msvc"`, `""` |
| `debug_assertions` | Set unless `debug-assertions = false` |
| `test` | Set when building the test harness |
| `doc` / `doctest` | Set when rustdoc is running |
| `feature = "x"` | A Cargo feature |
| `panic` | `"unwind"` or `"abort"` |

Combine with `all`, `any`, `not`, nested freely.

## cfg! vs #[cfg]

`#[cfg]` removes code before type checking. `cfg!` expands to `true`/`false` —
**both branches must still compile**.

```rust
fn main() {
    let mut log: Vec<&str> = Vec::new();

    #[cfg(debug_assertions)]
    log.push("debug build");

    #[cfg(not(debug_assertions))]
    log.push("release build");

    let level = if cfg!(debug_assertions) { "trace" } else { "warn" };
    println!("{log:?} level={level}");
}
```

Output:

```
["debug build"] level=trace
```

```
$ cargo run --release
["release build"] level=warn
```

Use `cfg!` when both branches are valid and you want a plain `if`; `#[cfg]`
when one branch would not compile at all. `#[cfg]` works on statements and
expressions, not just items.

## cfg_attr

Applies an attribute conditionally.

```rust
#[cfg_attr(debug_assertions, derive(Debug))]
struct Loud { v: i32 }

#[cfg_attr(test, allow(dead_code))]
fn maybe_unused() {}

#[cfg_attr(not(debug_assertions), inline(always))]
fn hot(x: i32) -> i32 { x + 1 }

fn main() {
    let l = Loud { v: 1 };
    #[cfg(debug_assertions)]
    println!("{:?}", l);
    #[cfg(not(debug_assertions))]
    println!("no Debug in release: {}", l.v);
    println!("{}", hot(1));
    maybe_unused();
}
```

Output:

```
Loud { v: 1 }
2
```

```
$ cargo run --release
no Debug in release: 1
2
```

The common use is conditional derives — how every crate offers optional serde
support:

```rust
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub struct Config { pub host: String }
```

Two more:

```rust
#![cfg_attr(docsrs, feature(doc_cfg))]

#[cfg_attr(docsrs, doc(cfg(feature = "json")))]
pub fn to_json() {}
```

`docsrs` is set by docs.rs, and makes the docs show which feature gates each
item (12.7).

## The unexpected_cfgs warning

Reference a feature that does not exist and rustc catches it:

```
warning: unexpected `cfg` condition value: `pretty`
  --> src/lib.rs:11:11
   |
11 |     #[cfg(feature = "pretty")]
   |           ^^^^^^^^^^^^^^^^^^
   |
   = note: expected values for `feature` are: `json`, `serde`, and `serde_json`
   = help: consider adding `pretty` as a feature in `Cargo.toml`
   = note: `#[warn(unexpected_cfgs)]` on by default
```

Without this lint a typo silently disables the code forever. Never `allow` it.
For a custom cfg from a build script or `RUSTFLAGS`, declare it:

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(docsrs)'] }
```

## CLI flags

| Flag | Effect |
|---|---|
| `--features a,b` | Enable `a` and `b` on top of defaults |
| `-F a,b` | Short form |
| `--features pkg/a` | Enable feature `a` on dependency `pkg` |
| `--no-default-features` | Drop the `default` list |
| `--all-features` | Enable every feature of every selected package |
| `--no-default-features --features x` | Exactly `x` and its implications |

In a workspace `--features` applies to the selected packages, so pair it with
`-p`. `--all-features` enables everything everywhere, which is why it passes
when narrower combinations fail.

## Checking feature combos in CI

`--all-features` and the default set are not enough. An item gated on `json`
that uses a helper gated on `pretty` compiles under both, but not under
`--no-default-features --features json`. A minimum bar:

```
$ cargo check
ok
$ cargo check --no-default-features
ok
$ cargo check --features=json
ok
$ cargo check --features=metrics
ok
$ cargo check --no-default-features --features=json
ok
$ cargo check --all-features
ok
```

Script it, or use `cargo-hack`, which generates the matrix for you:

```
cargo install cargo-hack
cargo hack check --feature-powerset --no-dev-deps
cargo hack check --each-feature --no-dev-deps
```

`--each-feature` is linear and catches most problems; `--feature-powerset` is
exponential — fine for 4 features, not 15. `--no-dev-deps` matters because
dev-dependencies can enable features that mask a missing gate. Test `--locked`
and your MSRV in the same job too.

## Quick Reference

| Form | Meaning |
|---|---|
| `[features] x = []` | A plain feature |
| `[features] default = ["x"]` | Enabled unless `--no-default-features` |
| `[features] full = ["a", "b"]` | Meta-feature |
| `dep = { optional = true }` | Optional dependency (+ implicit feature) |
| `x = ["dep:serde"]` | Enable the dependency, no implicit feature |
| `x = ["serde/derive"]` | Enable a feature on a dependency |
| `x = ["tokio?/tracing"]` | Only if `tokio` is already enabled |
| `#[cfg(feature = "x")]` | Compile this item only with `x` |
| `#[cfg(all(a, any(b, not(c))))]` | Combined predicate |
| `cfg!(feature = "x")` | Runtime-looking bool; both branches compile |
| `#[cfg_attr(x, derive(Debug))]` | Conditional attribute |
| `compile_error!("...")` | Fail the build with a message |
| `--features a,b` / `-F a,b` | Add features |
| `--no-default-features` | Drop defaults |
| `--all-features` | Everything |
| `cargo hack --each-feature` | Feature matrix in CI |

## Common Mistakes

**Assuming `default-features = false` is honored.** Unification means a sibling
can re-enable the defaults. It is a request, not a guarantee.

**Naming a feature for what it removes.** `no-std` is backwards; make `std` a
default feature instead.

**Changing a signature or dropping a `pub` item behind a feature.** It breaks
crates that never enabled it. Features add, only ever add.

**Forgetting `dep:`.** You get a public feature named after your dependency, and
you can never rename or drop that dependency again without a major bump.

**Testing only the default and `--all-features`.** Both can pass while a real
combination fails. Check each feature on its own.

**Ignoring `unexpected_cfgs`.** A typo'd feature name silently compiles nothing
and gives no error, forever.

## Exercises

1. Build a crate with a `color` feature that changes a `render()` method's
   output. Run it with and without, then with `--no-default-features`.
2. Add `serde` as an optional dependency gated behind a `json` feature using
   `dep:`. Compare `cargo tree` with and without the feature. Then drop the
   `dep:` and confirm `--features serde` starts working.
3. Add a `full` meta-feature covering all your others. Verify
   `--no-default-features --features full` equals `--all-features`.
4. Write a function with two `#[cfg]` bodies whose return types differ, consume
   it from a second crate, and then have a third crate enable the other feature.
   Reproduce the `E0308`.
5. Print a table of `cfg!` values for `target_os`, `target_arch`,
   `target_pointer_width`, `debug_assertions` and `test`. Compare debug, release,
   and `cargo test`.
6. Write a shell loop that runs `cargo check` over every subset of your features
   and reports pass/fail. Then introduce a bug that only fails under one subset.

---

[← 12.5 Workspaces](05-workspaces.md) · [Course Index](../README.md) · [12.7 Publishing to crates.io →](07-publishing.md)
