[← Course Index](../README.md)

# 12.7 — Publishing to crates.io

## What You'll Learn

- The pre-publish checklist and the metadata crates.io requires
- `cargo package`, inspecting the `.crate`, and `cargo publish --dry-run`
- Version bumping, yanking, and what you can never undo
- How docs.rs builds your docs, and publishing a workspace member

## Coming From Another Language

| Other | Rust |
|---|---|
| `npm publish` | `cargo publish` |
| `npm unpublish` (72h window) | Nothing. Versions are permanent |
| `npm deprecate` | `cargo yank` |
| Docs you host yourself | docs.rs builds them automatically, free |

Two differences that change how you work: **a published version can never be
removed or replaced**, and **crate names are global and first-come** — there is
no `@org/pkg` scoping. Everything below uses `--dry-run` and `cargo package`.

## The pre-publish checklist

1. `cargo test --all-features` and `cargo test --no-default-features` pass.
2. `cargo clippy --all-targets --all-features -- -D warnings` is clean.
3. `cargo fmt --check` is clean.
4. `cargo doc --no-deps` builds with no warnings; doc tests pass.
5. `cargo publish --dry-run` succeeds.
6. `cargo package --list` shows exactly the files you intend to ship.
7. Metadata is filled in: `description`, `license`, `repository`, `readme`,
   `keywords`, `categories`.
8. No `path` or `git` dependencies.
9. `README.md` and a `LICENSE` file exist.
10. `CHANGELOG.md` updated, version bumped, work committed and tagged.
11. The name is free: check `crates.io/crates/<name>` first.

## Required metadata

Hard-required: `name`, `version`, and `license` or `license-file`.
`description` is required in practice. Missing optional fields warn:

```
$ cargo publish --dry-run
    Updating crates.io index
warning: manifest has no description, license, license-file, documentation, homepage or repository
  |
  = note: see https://doc.rust-lang.org/cargo/reference/manifest.html#package-metadata for more info
```

A complete publishable manifest:

```toml
[package]
name = "pubdemo"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
description = "A tiny demo crate that adds two numbers"
license = "MIT OR Apache-2.0"
repository = "https://github.com/example/pubdemo"
documentation = "https://docs.rs/pubdemo"
homepage = "https://example.com/pubdemo"
readme = "README.md"
keywords = ["math", "addition", "demo"]
categories = ["mathematics"]
include = ["src/**/*", "README.md", "LICENSE-*", "Cargo.toml"]

[dependencies]

[package.metadata.docs.rs]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]
```

Field notes:

- `license` is an SPDX expression. `"MIT OR Apache-2.0"` is the ecosystem
  default — dual licensing lets users pick, and matches the standard library.
- `keywords`: max **5**, each max 20 chars, alphanumeric plus `-` and `_`.
- `categories`: max **5**, from the fixed list at `crates.io/category_slugs`.
  An invalid slug is rejected at upload.
- `readme` defaults to `README.md` and becomes your front page;
  `readme = false` suppresses it.
- `documentation` defaults to the docs.rs URL.
- `rust-version` is your MSRV; crates.io displays it and Cargo enforces it.

## The git-dirty gate

Cargo refuses to package uncommitted changes:

```
error: 2 files in the working directory contain changes that were not yet committed into git:

Cargo.toml
src/lib.rs

to proceed despite this and include the uncommitted changes, pass the `--allow-dirty` flag
```

Deliberate: the `.crate` records the commit it was built from, so "what's in
this release" is always answerable. Use `--allow-dirty` only locally.

## cargo package

Builds the `.crate` tarball and compiles it in isolation, without uploading.

```
$ cargo package
   Packaging pubdemo v0.1.0 (/.../pubdemo)
    Packaged 7 files, 2.2KiB (1.2KiB compressed)
   Verifying pubdemo v0.1.0 (/.../pubdemo)
   Compiling pubdemo v0.1.0 (/.../pubdemo/target/package/pubdemo-0.1.0)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.03s
```

The "Verifying" step is the valuable one: it extracts the tarball and builds
*that*, so a forgotten `include` fails here instead of after upload. Always
check the file list:

```
$ cargo package --list
.cargo_vcs_info.json
Cargo.lock
Cargo.toml
Cargo.toml.orig
LICENSE-MIT
README.md
src/lib.rs
```

The source tree also held `fixtures/data.bin` and `scratch.bak`; `include` kept
both out. Run this before every first publish — shipping a `.env` or a 40MB
fixture is permanent.

Prefer `include` over `exclude`: enumerate what ships rather than guess what to
omit. Cargo also respects `.gitignore`, always ships `Cargo.toml`, and never
ships `target/`.

## Inspecting the .crate

It is a gzipped tar. Read it directly.

```
$ tar -tzf target/package/pubdemo-0.1.0.crate
pubdemo-0.1.0/.cargo_vcs_info.json
pubdemo-0.1.0/Cargo.lock
pubdemo-0.1.0/Cargo.toml
pubdemo-0.1.0/Cargo.toml.orig
pubdemo-0.1.0/LICENSE-MIT
pubdemo-0.1.0/README.md
pubdemo-0.1.0/src/lib.rs
```

Four files you did not write. `.cargo_vcs_info.json` pins the commit:

```
$ tar -xOzf target/package/pubdemo-0.1.0.crate pubdemo-0.1.0/.cargo_vcs_info.json
{
  "git": {
    "sha1": "5e1e65738f75a9d1e2e9fe889f3ced08c1582f6e"
  },
  "path_in_vcs": ""
}
```

`Cargo.toml.orig` is your manifest verbatim; `Cargo.toml` is normalized:

```
$ tar -xOzf target/package/pubdemo-0.1.0.crate pubdemo-0.1.0/Cargo.toml
# THIS FILE IS AUTOMATICALLY GENERATED BY CARGO
#
# When uploading crates to the registry Cargo will automatically
# "normalize" Cargo.toml files for maximal compatibility
# with all versions of Cargo and also rewrite `path` dependencies
# to registry (e.g., crates.io) dependencies.
#
# If you are reading this file be aware that the original Cargo.toml
# will likely look very different (and much more reasonable).
# See Cargo.toml.orig for the original contents.

[package]
edition = "2024"
rust-version = "1.85"
name = "pubdemo"
version = "0.1.0"
build = false
include = [
    "src/**/*",
    "README.md",
    "LICENSE-*",
    "Cargo.toml",
]
autolib = false
autobins = false
autoexamples = false
autotests = false
autobenches = false
description = "A tiny demo crate that adds two numbers"
...
[lib]
name = "pubdemo"
path = "src/lib.rs"

[dependencies]
```

What the rewrite did: every implicit target became explicit (`[lib]`,
`autobins = false`), and **`path` dependencies lose their path and resolve to
registry versions** — which is why path deps need a `version`.

`Cargo.lock` ships so `cargo install` of a binary is reproducible; library
consumers ignore it.

## cargo publish --dry-run

The full publish path, stopping before upload.

```
$ cargo publish --dry-run
    Updating crates.io index
   Packaging pubdemo v0.1.0 (/.../pubdemo)
    Packaged 7 files, 2.2KiB (1.2KiB compressed)
   Verifying pubdemo v0.1.0 (/.../pubdemo)
   Compiling pubdemo v0.1.0 (/.../pubdemo/target/package/pubdemo-0.1.0)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.16s
   Uploading pubdemo v0.1.0 (/.../pubdemo)
warning: aborting upload due to dry run
```

It does catch:

- Uncommitted changes.
- Missing `version` on a `path` dependency.
- A dependency not resolvable from the registry.
- Anything that fails to compile from the packaged tarball.
- `publish = false`:

```
$ cargo publish --dry-run
error: `pubdemo` cannot be published.
`package.publish` must be set to `true` or a non-empty list in Cargo.toml to publish.
```

It does **not** catch server-side rules. A wildcard dependency and six keywords
both sail through a dry run and are rejected at upload:

```toml
keywords = ["a","b","c","d","e","f"]

[dependencies]
log = "*"
```

```
$ cargo publish --dry-run
    Packaged 7 files, 2.5KiB (1.3KiB compressed)
   Verifying pubdemo v0.1.0 (/.../pubdemo)
   Compiling log v0.4.34
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.32s
   Uploading pubdemo v0.1.0 (/.../pubdemo)
warning: aborting upload due to dry run
```

So verify the keyword/category/license rules by reading the manifest. A dry run
also never checks whether the **name is taken** or the **version already
exists** — both are known only at upload.

## Version bumping and semver obligations

The release flow:

```
1. Update CHANGELOG.md
2. Bump `version` in Cargo.toml
3. cargo test --all-features && cargo clippy -- -D warnings
4. cargo publish --dry-run
5. git commit -am "release v0.2.0" && git tag v0.2.0
6. cargo publish
7. git push && git push --tags
```

What bump a change requires (the detailed table is in 12.4):

| Change | Bump |
|---|---|
| Bug fix, internals only | patch |
| New `pub` item, new feature | minor |
| Remove/rename a `pub` item | major |
| Add a field to a public-fields struct, or an enum variant | major unless `#[non_exhaustive]` |
| Change a function signature | major |
| Bump a dep that appears in your public API | major |

Before a big release, run `cargo semver-checks` (`cargo install
cargo-semver-checks`): it diffs your public API against the last published
version and names the violations humans miss.

For `0.x` the minor is the breaking field. You cannot publish the same version
twice, nor one lower than what is already there; yanking does not free the
number.

## Yanking

`cargo yank` marks a version as "do not select for new resolutions".

```
$ cargo yank --version 0.1.1
$ cargo yank --version 0.1.1 --undo
```

What it does:

- Fresh resolutions and `cargo add` skip it.
- Existing `Cargo.lock` files that pin it **keep working**; downloads succeed.
- It stays visible on crates.io, marked yanked.

Yank for a security hole, a broken build, a leaked secret, or accidental
breakage. **Yanking is not deletion** — the code, docs and tarball stay public
forever. The right response to a bad release is to yank it *and* publish a fix
immediately; a yank alone leaves users with no upgrade path.

## What you cannot undo

| Want | Possible? |
|---|---|
| Delete a version | No |
| Replace a version's contents | No |
| Reuse a version number after yanking | No |
| Delete a crate entirely | No (only crates with no dependents, under 72h, by asking the team) |
| Rename a crate | No. Publish a new name; the old one stays |
| Remove a secret you committed | No. Rotate it. It is in the tarball forever |
| Transfer ownership | Yes: `cargo owner --add <user>`, then `--remove` yourself |
| Free a name you squatted | Only by contacting the crates.io team |

Treat your *first* publish as the careful one: check `cargo package --list`,
scan for secrets, be certain about the name. Rehearse with a `0.0.1` under a
scratch name if you want.

## docs.rs

Every published crate gets docs built and hosted at
`https://docs.rs/<crate>/<version>` automatically. How the build works:

- `cargo doc` on a **nightly** toolchain, sandboxed, with **no network** — a
  `build.rs` that downloads anything fails.
- Default features only, for `x86_64-unknown-linux-gnu`, under time and memory
  limits.
- A failed build shows its log on docs.rs. You cannot retry by republishing the
  same version, but you can trigger a rebuild from the crate's docs.rs page.

Configure it from your manifest:

```toml
[package.metadata.docs.rs]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]
targets = ["x86_64-unknown-linux-gnu", "x86_64-pc-windows-msvc"]
default-target = "x86_64-unknown-linux-gnu"
```

`all-features = true` is nearly always right: otherwise your optional APIs are
missing from the docs everyone reads.

## The docsrs cfg

`rustdoc-args = ["--cfg", "docsrs"]` sets a `docsrs` cfg true only during the
docs.rs build. Two uses.

**Document feature-gated items even when the feature is off:**

```rust
#[cfg(any(feature = "json", docsrs))]
pub fn to_json(v: i32) -> String { format!("{{\"v\":{v}}}") }
```

Reproduce the docs.rs build locally:

```
$ RUSTDOCFLAGS="--cfg docsrs" cargo doc --no-deps --all-features
```

Declare the custom cfg so `unexpected_cfgs` stays useful:

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(docsrs)'] }
```

**Show a feature badge on each gated item.** Needs the nightly-only `doc_cfg`,
hence the `cfg_attr`:

```rust
#![cfg_attr(docsrs, feature(doc_cfg))]

#[cfg(feature = "json")]
#[cfg_attr(docsrs, doc(cfg(feature = "json")))]
pub fn to_json(v: i32) -> String { todo!() }
```

docs.rs uses nightly, so this works there. On stable it does not:

```
$ RUSTDOCFLAGS="--cfg docsrs" cargo doc --no-deps
error[E0554]: `#![feature]` may not be used on the stable release channel
 --> src/lib.rs:2:21
  |
2 | #![cfg_attr(docsrs, feature(doc_cfg))]
  |                     ^^^^^^^^^^^^^^^^
```

Expected: the `cfg_attr` means the `feature` attribute appears only when
`docsrs` is set, so a plain `cargo doc` is fine. Check it locally with
`cargo +nightly doc`.

docs.rs does not run your doc tests, so run them yourself:

```
$ cargo test --doc
running 2 tests
test src/lib.rs - add (line 9) ... ok
test src/lib.rs - (line 3) ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

A broken doc example ships in your README and front page. Lesson 13.3 covers
doc tests in full.

## README and license files

```
pubdemo/
├── Cargo.toml
├── README.md          <- rendered on crates.io
├── LICENSE-MIT
├── LICENSE-APACHE
├── CHANGELOG.md
└── src/lib.rs
```

- crates.io renders `README.md`, not `CHANGELOG.md` — link to it instead.
- `license` is metadata only. Ship the license *text* as files, and make sure
  `include` covers them (`"LICENSE-*"`).
- Keep the README's examples working by including it as docs:

```rust
#![doc = include_str!("../README.md")]
```

That turns your README into a tested document.

## Publishing a workspace member

Each member publishes separately. Two rules.

**Path dependencies need a `version`.** Without one:

```
$ cargo publish --dry-run -p shop-db
error: failed to verify manifest at `/.../crates/db/Cargo.toml`

Caused by:
  all dependencies must have a version requirement specified when publishing.
  dependency `shop-domain` does not specify a version
  Note: The published dependency will use the version from crates.io,
  the `path` specification will be removed from the dependency declaration.
```

Fix it at the workspace root:

```toml
[workspace.dependencies]
shop-domain = { path = "crates/domain", version = "0.3.0" }
shop-db = { path = "crates/db", version = "0.3.0" }
```

**Publish in dependency order.** `shop-db` depends on `shop-domain`, so
`shop-domain` must already exist on crates.io:

```
$ cargo publish --dry-run -p shop-db
   Packaging shop-db v0.3.0 (/.../crates/db)
    Updating crates.io index
error: failed to prepare local package for uploading

Caused by:
  no matching package named `shop-domain` found
  location searched: crates.io index
  required by package `shop-db v0.3.0 (/.../crates/db)`
```

Leaves first: `shop-domain`, then `shop-db`, then `shop-api`. Or use
`cargo publish --workspace`, which computes the order for you.

Members you never want published get `publish = false`:

```toml
[package]
name = "shop-testutil"
publish = false
```

`[workspace.package] version` plus `version.workspace = true` gives all members
one version number — simplest to reason about, at the cost of bumping crates
that did not change.

## cargo-release

`cargo install cargo-release` automates steps 1-7 of the release flow.

```
$ cargo release patch --dry-run
$ cargo release minor --execute
```

It bumps versions, updates intra-workspace requirements, runs your checks,
commits, tags, publishes and pushes. Configure it in the manifest:

```toml
[workspace.metadata.release]
shared-version = true
tag-name = "v{{version}}"
pre-release-commit-message = "release {{version}}"
```

Its value is on a workspace, where it handles the ordering and inter-member
version bumps. Always `--dry-run` first; `--execute` publishes.

## Other registries

`cargo publish --registry my-registry` publishes elsewhere. To allow only a
private registry:

```toml
[package]
publish = ["my-registry"]
```

`publish = false` blocks publishing entirely — the right setting for every
binary, internal service and test-helper crate.

## Quick Reference

| Command | Does |
|---|---|
| `cargo package` | Build and verify the `.crate`, no upload |
| `cargo package --list` | List files that would ship |
| `cargo publish --dry-run` | Full path, stops before upload |
| `cargo publish` | Upload. Permanent |
| `cargo publish -p NAME` | One workspace member |
| `cargo publish --workspace` | All members, in dependency order |
| `cargo publish --allow-dirty` | Skip the git-clean check |
| `cargo yank --version X` | Hide a version from new resolutions |
| `cargo yank --version X --undo` | Un-yank |
| `cargo owner --add USER` | Add an owner |
| `cargo login` | Store your API token |
| `cargo semver-checks` | Diff public API vs last release (external) |
| `cargo release minor --execute` | Automated release (external) |

| Manifest key | Effect on publishing |
|---|---|
| `license` / `license-file` | Required |
| `description` | Required in practice; warned if absent |
| `readme` | Crate front page |
| `keywords` | Max 5, 20 chars each |
| `categories` | Max 5, from the fixed slug list |
| `include` / `exclude` | What ships in the tarball |
| `publish = false` | Blocks `cargo publish` |
| `publish = ["reg"]` | Allows only that registry |
| `[package.metadata.docs.rs]` | Configures the docs.rs build |

## Common Mistakes

**Publishing before running `cargo package --list`.** Shipping a `.env`, a
fixtures directory, or a key is permanent and public.

**Thinking `cargo yank` deletes a version.** It only hides it from new
resolutions. The code stays downloadable forever. Rotate leaked secrets.

**Expecting `--dry-run` to catch everything.** Wildcard deps, six keywords, bad
category slugs, a taken name and a duplicate version are all server-side.

**Leaving `path` dependencies without a `version`.** The manifest check rejects
it. Give workspace path deps both fields.

**Publishing a workspace in the wrong order.** Leaves first, or use
`cargo publish --workspace`.

**Not setting `all-features = true` for docs.rs.** Your optional API is then
invisible in the docs everyone actually reads.

**Forgetting `publish = false` on internal crates.** One stray command and your
service code is public and unremovable.

## Exercises

1. Take a scratch crate, fill in every `[package]` field this lesson lists, and
   run `cargo publish --dry-run` until the metadata warning disappears.
2. Add a `fixtures/` directory with a large file and a `.env`. Use `include` to
   keep both out, and prove it with `cargo package --list`.
3. Run `cargo package`, then `tar -tzf` the result. Read `Cargo.toml` from
   inside the tarball and list three ways it differs from yours.
4. Add `#![doc = include_str!("../README.md")]` to `lib.rs` with a code fence in
   the README. Run `cargo test --doc` and watch it run as a test. Then break the
   example and confirm the test fails.
5. Build a two-member workspace with a path dependency and no `version`. Run
   `cargo publish --dry-run -p` the dependent and read the exact error. Fix it,
   re-run, and read the next error.
6. Add `[package.metadata.docs.rs]` with `all-features = true` and a
   `cfg(docsrs)`-gated item. Build locally with
   `RUSTDOCFLAGS="--cfg docsrs" cargo doc --no-deps` and confirm the item appears
   with the feature off.

---

[← Course Index](../README.md)
