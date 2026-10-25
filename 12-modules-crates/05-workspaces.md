[← 12.4 Cargo.toml in Depth](04-cargo-toml.md) · [Course Index](../README.md) · [12.6 Features and Conditional Compilation →](06-features.md)

# 12.5 — Workspaces

## What You'll Learn

- `[workspace]`, `members`, `exclude`, `default-members`
- `[workspace.dependencies]` and `[workspace.package]` inheritance
- A shared `target/` and a single `Cargo.lock`
- `-p` vs `--workspace`, and when splitting a backend into crates helps

## Coming From Another Language

| Other | Rust |
|---|---|
| npm/pnpm workspaces, `workspace:*` | `[workspace]` + path deps |
| Python monorepo with editable installs | `path = "../other"` |
| Go modules in one repo | One `[workspace]`, many packages |

A workspace is several packages sharing one `Cargo.lock`, one `target/`, and one
dependency resolution. Each member is still a separate crate, so the module
boundaries between them are enforced by the compiler.

## The root manifest

```toml
# shop/Cargo.toml
[workspace]
resolver = "3"
members = ["crates/*"]
exclude = ["playground"]
default-members = ["crates/api"]

[workspace.package]
version = "0.3.0"
edition = "2024"
license = "MIT"
authors = ["Raj <raj@example.com>"]
repository = "https://github.com/example/shop"

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
anyhow = "1.0"
serde_json = "1.0"
shop-domain = { path = "crates/domain" }
shop-db = { path = "crates/db" }

[workspace.lints.rust]
unsafe_code = "forbid"

[profile.release]
lto = "thin"
```

Notes on the root:

- `members` accepts globs. `crates/*` picks up new members automatically.
- `exclude` keeps a directory out entirely — it becomes its own workspace.
- `resolver = "3"` is the edition-2024 resolver; set it explicitly at the root,
  because a root with no `[package]` does not inherit it from an edition.
- `[profile.*]` is **only** read from the root. Profiles in a member are ignored
  with a warning.

A *virtual manifest* is a root with `[workspace]` and no `[package]` — the root
directory holds no code of its own. That is the shape above, and the one to
prefer.

## Layout

```
shop/
├── Cargo.toml          <- [workspace], virtual
├── Cargo.lock          <- one, for all members
├── target/             <- one, for all members
├── crates/
│   ├── domain/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── db/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   └── api/
│       ├── Cargo.toml
│       └── src/main.rs
└── playground/         <- excluded: own Cargo.lock and target/
```

```
$ find . -name Cargo.toml -o -name Cargo.lock
./Cargo.lock
./Cargo.toml
./crates/api/Cargo.toml
./crates/db/Cargo.toml
./crates/domain/Cargo.toml
./playground/Cargo.toml
```

No per-member `Cargo.lock`, and `ls crates/*/target` finds nothing. Everything
lands in `shop/target/`. That is the main practical payoff: a dependency shared
by three members compiles once.

## Member manifests with inheritance

`field.workspace = true` pulls the value from `[workspace.package]`.

```toml
# crates/domain/Cargo.toml
[package]
name = "shop-domain"
version.workspace = true
edition.workspace = true
license.workspace = true
authors.workspace = true
repository.workspace = true

[dependencies]
serde = { workspace = true }

[lints]
workspace = true
```

`name` can never be inherited. `version`, `edition`, `license`, `authors`,
`repository`, `description`, `keywords`, `categories`, `rust-version`,
`homepage`, `documentation`, `readme`, `publish` and `exclude`/`include` all can.

`[lints] workspace = true` is the whole table at once — you do not list
individual lints.

```toml
# crates/db/Cargo.toml
[package]
name = "shop-db"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
shop-domain = { workspace = true }
anyhow = { workspace = true }

[lints]
workspace = true
```

Inherited dependencies can still add features locally:

```toml
tokio = { workspace = true, features = ["fs"] }
```

Features are additive on top of the workspace entry. You cannot *remove* one.

## The members

```rust
// crates/domain/src/lib.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Product {
    pub id: u64,
    pub name: String,
    pub cents: u32,
}

impl Product {
    pub fn new(id: u64, name: &str, cents: u32) -> Self {
        Product { id, name: name.to_string(), cents }
    }
    pub fn price(&self) -> String {
        format!("${}.{:02}", self.cents / 100, self.cents % 100)
    }
}
```

```rust
// crates/db/src/lib.rs
use anyhow::{bail, Result};
use shop_domain::Product;

pub fn fetch(id: u64) -> Result<Product> {
    match id {
        1 => Ok(Product::new(1, "Keyboard", 7999)),
        2 => Ok(Product::new(2, "Mouse", 2450)),
        _ => bail!("no product with id {id}"),
    }
}
```

```rust
// crates/api/src/main.rs
fn main() -> anyhow::Result<()> {
    for id in [1u64, 2, 9] {
        match shop_db::fetch(id) {
            Ok(p) => println!("{} {} {}", p.id, p.name, p.price()),
            Err(e) => println!("error: {e}"),
        }
    }
    let p = shop_domain::Product::new(3, "Monitor", 19900);
    println!("{}", serde_json::to_string(&p)?);
    Ok(())
}
```

Output:

```
1 Keyboard $79.99
2 Mouse $24.50
error: no product with id 9
{"id":3,"name":"Monitor","cents":19900}
```

Note the crate name is `shop-db` in TOML but `shop_db` in code — Cargo converts
hyphens to underscores for the Rust identifier.

## Path dependencies between members

Declared once at the root, consumed with `workspace = true`:

```toml
[workspace.dependencies]
shop-domain = { path = "crates/domain" }
shop-db = { path = "crates/db" }
```

For a workspace you intend to publish, give them both a path and a version:

```toml
shop-domain = { path = "crates/domain", version = "0.3.0" }
```

Cargo uses the path locally and the version when publishing, so the published
crate points at the registry.

Cycles are rejected outright. Adding `shop-db` to `domain`'s dependencies:

```
error: cyclic package dependency: package `shop-db v0.3.0 (/.../crates/db)` depends on itself. Cycle:
package `shop-db v0.3.0 (/.../crates/db)`
    ... which satisfies path dependency `shop-db` (locked to 0.3.0) of package `shop-domain v0.3.0 (/.../crates/domain)`
    ... which satisfies path dependency `shop-domain` (locked to 0.3.0) of package `shop-db v0.3.0 (/.../crates/db)`
    ... which satisfies path dependency `shop-db` (locked to 0.3.0) of package `shop-api v0.3.0 (/.../crates/api)`
```

This is the enforcement that modules cannot give you. `domain` physically cannot
import `db`:

```rust
pub fn leak(id: u64) -> String {
    format!("{:?}", shop_db::fetch(id))
}
```

```
error[E0433]: cannot find module or crate `shop_db` in this scope
  --> crates/domain/src/lib.rs:20:21
   |
20 |     format!("{:?}", shop_db::fetch(id))
   |                     ^^^^^^^ use of unresolved module or unlinked crate `shop_db`
   |
   = help: if you wanted to use a crate named `shop_db`, use `cargo add shop_db` to add it to your `Cargo.toml`
```

Inside one crate, a `models` module importing `crate::db` compiles fine and no
reviewer catches it. Across crates, the build fails.

## Running commands

| Command | Scope |
|---|---|
| `cargo build` | `default-members`, or all members if unset |
| `cargo build --workspace` | Every member |
| `cargo build -p shop-db` | One member (also `--package`) |
| `cargo build -p shop-db -p shop-api` | Several |
| `cargo build --exclude shop-api --workspace` | All but one |
| `cargo test --workspace --all-targets` | The usual CI line |
| `cargo run -p shop-api` | A specific binary's package |
| `cargo run` (from `crates/db/`) | That member only — cwd picks the package |

`cargo test --workspace` runs every member's suites:

```
$ cargo test --workspace
     Running unittests src/main.rs (target/debug/deps/shop_api-578d4ecedd165f64)
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
     Running unittests src/lib.rs (target/debug/deps/shop_db-8e72c550811eecf2)
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
     Running unittests src/lib.rs (target/debug/deps/shop_domain-397ad7f8b66bbd6d)
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## default-members

Without it, a bare `cargo build` at the root builds everything, and a bare
`cargo run` fails because several members have binaries. With
`default-members = ["crates/api"]`, bare commands target just that:

```
$ cargo run
1 Keyboard $79.99
2 Mouse $24.50
error: no product with id 9
{"id":3,"name":"Monitor","cents":19900}
```

Use it when one member is "the app" and the others are its libraries.
`--workspace` always overrides it.

## exclude

An excluded directory is skipped by `members` globs and becomes an independent
workspace:

```
$ cd playground && cargo build && ls
Cargo.lock
Cargo.toml
src
target
```

Its own lock, its own `target/`. Use `exclude` for scratch crates, vendored
third-party code, and anything that must not share your resolved dependency
versions.

## The not-a-member error

Create a package under the root without listing it and Cargo tells you exactly
what to do:

```
error: current package believes it's in a workspace when it's not:
current:   /.../shop/stray/Cargo.toml
workspace: /.../shop/Cargo.toml

this may be fixable by adding `stray` to the `workspace.members` array of the manifest located at: /.../shop/Cargo.toml
Alternatively, to keep it out of the workspace, add the package to the `workspace.exclude` array, or add an empty `[workspace]` table to the package's manifest.
```

An empty `[workspace]` table in the member's own manifest is the third option —
it declares that package its own workspace root.

## When to split a backend into crates

The real question is: **do two parts of this system want different
dependencies?** If yes, split. If no, use modules.

Worth splitting:

| Crate | Dependencies it should have | Why the split |
|---|---|---|
| `domain` | `serde`, `thiserror`, `chrono` | Pure types and rules. Cannot touch SQL or HTTP, enforced |
| `db` | `sqlx`, `domain` | All queries. Swap the driver without touching domain |
| `api` | `axum`, `tower`, `db`, `domain` | HTTP only |
| `worker` | `domain`, `db`, a queue client | Shares logic with `api`, ships separately |
| `cli` | `clap`, `db` | Ops tooling, no HTTP |
| `migrations` | `sqlx-cli` glue | Runs standalone |

Concrete payoffs beyond architecture purity:

- **Parallel and incremental builds.** Touching a handler recompiles `api`, not
  `domain`. Members build in parallel once their deps are done.
- **Two deployables from one codebase.** `api` and `worker` share `domain` with
  no copy-paste.
- **Test isolation.** `cargo test -p domain` runs in a second with no database.
- **Feature isolation.** `domain` can be `no_std` or wasm-compatible while `db`
  is not.

Not worth splitting:

- A service under ~5k lines. Modules are free; crates have overhead.
- A split you cannot describe as a dependency *arrow*. If `api` and `db` would
  need each other, the boundary is wrong — you are describing one crate.
- Splitting `handlers` from `routes` from `middleware`. Those are layers of the
  same crate.
- Separating code purely to reuse it "someday". A workspace member you never
  depend on twice has given you nothing but a `Cargo.toml` to maintain.

The honest signal to split: you reach for `pub(crate)` to stop a layer from
being imported, and wish the compiler would enforce it. That is a crate
boundary.

Start as one crate with clean modules. The move to a workspace is then almost
mechanical: `src/domain/` becomes `crates/domain/src/`, `crate::domain::X`
becomes `shop_domain::X`, and everything the compiler complains about was a
layering violation you already had.

## Shared tooling at the root

A workspace root is also the place for things you would otherwise repeat:

```toml
[workspace.lints.clippy]
unwrap_used = "warn"

[workspace.metadata.release]
shared-version = true

[profile.release]
lto = "thin"
codegen-units = 1

[patch.crates-io]
sqlx = { git = "https://github.com/launchbadge/sqlx", rev = "abc1234" }
```

`[patch]` replaces a crate for the **entire** graph, including transitive uses —
the standard way to test an upstream fix before it is released. It only works at
the workspace root.

## Quick Reference

| Key | Where | Meaning |
|---|---|---|
| `[workspace]` | root | Declares the workspace |
| `members` | root | Paths or globs of member packages |
| `exclude` | root | Directories that are not members |
| `default-members` | root | What bare commands target |
| `resolver` | root | `"3"` for edition 2024 |
| `[workspace.package]` | root | Shared `[package]` field values |
| `[workspace.dependencies]` | root | Shared dependency declarations |
| `[workspace.lints.*]` | root | Shared lint levels |
| `[profile.*]` | root only | Codegen; ignored in members |
| `[patch.crates-io]` | root only | Graph-wide dependency replacement |
| `field.workspace = true` | member | Inherit that `[package]` field |
| `dep = { workspace = true }` | member | Inherit that dependency |
| `[lints] workspace = true` | member | Inherit all lint tables |
| `[workspace]` (empty) | member | Opt this package out as its own root |
| `-p NAME` | CLI | One package |
| `--workspace` | CLI | All packages |
| `--exclude NAME` | CLI | With `--workspace`, skip one |

## Common Mistakes

**Putting `[profile.release]` in a member.** Only the root is read; you get a
warning and no effect.

**Forgetting `resolver = "3"` on a virtual manifest.** A root with no `[package]`
has no edition to infer it from, so you silently get the old resolver.

**A path dependency with no `version` in a workspace you publish.** crates.io
rejects it. Give path deps both fields.

**Expecting `cargo build` at the root to build everything once
`default-members` is set.** It narrows bare commands. Use `--workspace` in CI.

**Splitting into crates to express layers you could express with modules.** If
there is no dependency arrow, there is no crate boundary — just more manifests.

**Trying to turn off a feature a workspace dependency enables.** Member entries
are additive. Keep the workspace entry minimal and add features per member.

## Exercises

1. Build a three-member workspace `core`, `store`, `cli` where `store` depends
   on `core` and `cli` depends on both. Use `[workspace.package]` so only `name`
   appears in each member.
2. Move `serde` into `[workspace.dependencies]` with
   `features = ["derive"]` and consume it from two members with
   `workspace = true`. Then have one member add `features = ["rc"]` on top.
3. Add a dependency to `store` and try to use it from `core`. Read the error,
   then confirm `cargo tree -p core` does not list it.
4. Set `default-members` to just `cli`. Verify a bare `cargo build` skips the
   others and `cargo build --workspace` does not.
5. Add a `scratch/` package and build it without listing it. Read the error, then
   fix it three ways: `members`, `exclude`, and an empty `[workspace]` table.
6. Make `core` depend on `store` to create a cycle. Read the error and note
   which path dependency Cargo blames.

---

[← 12.4 Cargo.toml in Depth](04-cargo-toml.md) · [Course Index](../README.md) · [12.6 Features and Conditional Compilation →](06-features.md)
