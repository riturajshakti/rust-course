[Course Index](../README.md) · [1.2 Hello, Cargo: Your First Project →](02-hello-cargo.md)

# 1.1 — Installing Rust and the Toolchain

## What You'll Learn

- What `rustup`, `rustc`, and `cargo` each are, and which one you actually type
- Installing Rust and reading the version strings
- Stable, beta, nightly, and pinning a project with `rust-toolchain.toml`
- Adding components and cross-compile targets

## Coming From Another Language

| Other | Rust |
|---|---|
| `nvm` / `pyenv` / `gvm` | `rustup` — installs and switches toolchains |
| `node` / `python` / `go build` | `rustc` — the compiler, rarely called by hand |
| `npm` / `pip` / `go` | `cargo` — build, test, run, dependencies |

One difference from Node and Python: there is no global package directory and no
virtualenv. Dependencies are per-project, resolved from `Cargo.toml`, and cached
in `~/.cargo`. Nothing to activate.

## Install

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Accept the default (option 1). It installs `rustup`, the `stable` toolchain, and
puts shims on your `PATH` via `~/.cargo/env`. Start a new shell, or:

```bash
source "$HOME/.cargo/env"
```

On Windows, download `rustup-init.exe` from <https://rustup.rs>. It will ask you
to install the Visual Studio C++ build tools — Rust needs the system linker.

Homebrew, apt, and other OS packages also ship Rust, but they give you a fixed
version with no `rustup`. Use `rustup` unless you have a specific reason not to.

## Verifying

```bash
rustc --version
cargo --version
rustup --version
```

Output:

```
rustc 1.99.1 (58cf8f9c3 2026-10-13)
cargo 1.99.1 (5f94df478 2026-08-27)
rustup 1.29.0 (2026-09-08)
```

The version string is `<semver> (<commit hash> <commit date>)`. `rustc` and
`cargo` share a version number; they ship together. If `rustc` works but
`cargo` does not, your `PATH` is picking up a stray compiler.

This course targets **1.99.1, edition 2024**.

## rustup vs rustc vs cargo

| Tool | You run it | What it does |
|---|---|---|
| `rustup` | Occasionally | Installs/updates toolchains, components, targets |
| `cargo` | Constantly | Builds, runs, tests, manages dependencies |
| `rustc` | Almost never | Compiles one file; `cargo` invokes it for you |

`rustc` directly is fine for a throwaway:

```bash
echo 'fn main() { println!("hi"); }' > one.rs
rustc one.rs && ./one
```

Output:

```
hi
```

Past one file, use `cargo`. It handles dependencies, which `rustc` does not.

The binaries in `~/.cargo/bin` are **shims**: thin wrappers that ask `rustup`
which toolchain applies to the current directory and forward the call. That is
how a per-project pin works without you switching anything.

```bash
rustup which rustc
```

Output:

```
/Users/you/.rustup/toolchains/stable-aarch64-apple-darwin/bin/rustc
```

## Channels: stable, beta, nightly

| Channel | Released | Use it for |
|---|---|---|
| `stable` | Every 6 weeks | Everything. The default. |
| `beta` | Next stable, in testing | Checking your code before a release lands |
| `nightly` | Daily | Unstable features, some tooling (`cargo expand`, miri) |

```bash
rustup toolchain install nightly
rustup toolchain list
```

Output:

```
stable-aarch64-apple-darwin (active, default)
nightly-aarch64-apple-darwin
```

Run one command on another channel with a `+channel` prefix — this is a
`rustup` shim feature, not a `cargo` flag:

```bash
cargo +nightly --version
rustc +beta --version
```

Set your global default:

```bash
rustup default stable
```

Update everything you have installed:

```bash
rustup update
```

Nothing in this course needs nightly. Stable Rust is the whole language for
application work; nightly exists for unfinished features.

## Pinning a project with rust-toolchain.toml

Drop this at the repo root and everyone — including CI — gets the same compiler
automatically, no instructions in the README.

```toml
[toolchain]
channel = "1.99.1"
components = ["rustfmt", "clippy"]
targets = ["x86_64-unknown-linux-musl"]
profile = "minimal"
```

Any `cargo` or `rustc` call inside this directory now uses 1.99.1. If that
toolchain is missing, `rustup` installs it on first use, along with the listed
components and targets.

```bash
rustup show
```

Output:

```
active toolchain
----------------
name: 1.99.1-aarch64-apple-darwin
active because: overridden by '/path/to/project/rust-toolchain.toml'
```

`channel` takes an exact version (`"1.99.1"`) or a channel name (`"stable"`).
Pin the exact version for applications — it makes builds reproducible and stops
a new release from breaking CI on a Tuesday. Libraries usually track `"stable"`.

A directory override without a file, if you need one:

```bash
rustup override set nightly
rustup override unset
```

`rust-toolchain.toml` is better: it is committed, so it applies to everybody.

## Components

Components are optional parts of a toolchain.

```bash
rustup component list --installed
```

Output:

```
cargo
clippy
rust-docs
rust-std
rustc
rustfmt
```

| Component | What it gives you |
|---|---|
| `rustfmt` | `cargo fmt` |
| `clippy` | `cargo clippy` |
| `rust-src` | Standard library source — rust-analyzer needs this |
| `rust-analyzer` | The language server |
| `rust-docs` | Offline docs, opened by `rustup doc` |
| `llvm-tools` | Coverage and profiling tooling |

```bash
rustup component add rust-src rust-analyzer
```

`rustfmt` and `clippy` come with the default profile. `rust-src` does not, and
its absence is the usual reason "go to definition" into `std` fails in an editor.

Offline documentation, no network needed:

```bash
rustup doc --std
```

## Cross-compile targets

A target triple is `<arch>-<vendor>-<os>-<abi>`. Adding one installs a
precompiled standard library for it.

```bash
rustup target add x86_64-unknown-linux-musl
rustup target list --installed
```

Output:

```
aarch64-apple-darwin
x86_64-unknown-linux-musl
```

Then build for it:

```bash
cargo build --release --target x86_64-unknown-linux-musl
```

The binary lands in `target/x86_64-unknown-linux-musl/release/` — note that
`--target` adds a directory level, which trips up Dockerfiles copied from
non-cross builds.

| Target | Why you'd want it |
|---|---|
| `x86_64-unknown-linux-gnu` | Standard Linux server |
| `x86_64-unknown-linux-musl` | Statically linked; runs on `scratch`/Alpine images |
| `aarch64-unknown-linux-gnu` | ARM servers (Graviton) |
| `wasm32-unknown-unknown` | WebAssembly |

`musl` is the one worth knowing for backend work. It links the C library
statically, so the result has no `libc` dependency and drops into an empty
container image. Module 18.6 covers the Docker side.

The catch: `rustup target add` gives you the Rust standard library, not a
**linker** for that platform. Cross-compiling from macOS to Linux still needs a
cross-linker. The practical answer is `cross`, which runs the build in a Docker
image that has one:

```bash
cargo install cross
cross build --release --target x86_64-unknown-linux-musl
```

Building for Linux *on* Linux needs only the target plus `musl-tools`.

## Directory layout

Two directories, both overridable via `RUSTUP_HOME` and `CARGO_HOME`.

`~/.rustup` — the toolchains themselves:

```
~/.rustup/
├── settings.toml              default toolchain, directory overrides
├── toolchains/
│   ├── stable-aarch64-apple-darwin/
│   │   ├── bin/               the real rustc, cargo, clippy-driver, rustfmt
│   │   └── lib/rustlib/       std for each installed target
│   └── 1.99.1-aarch64-apple-darwin/
└── downloads/
```

`~/.cargo` — the shims and the package cache:

```
~/.cargo/
├── bin/                       shims: cargo, rustc, + cargo install results
├── env                        the PATH snippet
├── config.toml                global cargo config
├── registry/
│   ├── cache/                 downloaded .crate archives
│   └── src/                   unpacked crate sources
└── git/                       checkouts of git dependencies
```

`~/.cargo/registry` is a shared cache across every project — a dependency is
downloaded once per machine, not once per project. It grows; a few GB is normal.

Reclaim space safely — both are caches and will be refetched or rebuilt:

```bash
rm -rf ~/.cargo/registry/cache
cargo clean          # deletes ./target in one project
```

Global config, if you want the `musl` target as the default output:

```toml
# ~/.cargo/config.toml
[build]
target = "x86_64-unknown-linux-musl"

[net]
git-fetch-with-cli = true
```

`git-fetch-with-cli` is the fix when private git dependencies fail to
authenticate — it makes cargo shell out to your real `git`, SSH keys and all.

## Uninstalling

```bash
rustup self uninstall
```

Removes `~/.rustup`, `~/.cargo`, and the `PATH` entry.

## Quick Reference

| Command | Does |
|---|---|
| `rustup update` | Update installed toolchains |
| `rustup default stable` | Set global default channel |
| `rustup show` | Active toolchain and why it's active |
| `rustup which rustc` | Real path behind the shim |
| `rustup toolchain install nightly` | Add a channel |
| `rustup toolchain list` | List installed channels |
| `rustup component add rust-src` | Add a component |
| `rustup component list --installed` | List components |
| `rustup target add <triple>` | Add a cross-compile std |
| `rustup target list --installed` | List targets |
| `rustup override set nightly` | Pin this directory (uncommitted) |
| `rustup doc --std` | Offline std docs |
| `cargo +nightly build` | One command on another channel |
| `rustup self uninstall` | Remove everything |

## Common Mistakes

**Installing Rust from `apt` or Homebrew, then wondering why `rustup` is
missing.** OS packages ship a fixed `rustc` with no toolchain manager. Remove it
and install via <https://rustup.rs>.

**`cargo: command not found` right after installing.** The installer edits your
shell profile but cannot change the running shell. Open a new terminal or
`source "$HOME/.cargo/env"`.

**Expecting `rustup target add` to make cross-compiling work.** It installs the
standard library, not a linker. Use `cross` for Linux-from-macOS builds.

**Committing `rustup override set` instead of `rust-toolchain.toml`.** An
override lives in your own `~/.rustup/settings.toml` and affects nobody else.
Commit the TOML file.

**Looking for the binary in `target/release/` after `--target`.** With
`--target` it moves to `target/<triple>/release/`.

## Exercises

1. Print `rustc`, `cargo`, and `rustup` versions. Confirm `rustc` and `cargo`
   share a version number.
2. Install the `nightly` toolchain and print its version with `+nightly` without
   changing your default. Confirm with `rustup show` that `stable` is still
   active.
3. Create an empty directory with a `rust-toolchain.toml` pinned to `"1.99.1"`.
   Run `rustup show` inside it and outside it; explain the `active because` line.
4. Add the `x86_64-unknown-linux-musl` target and list installed targets. Then
   find the directory under `~/.rustup/toolchains/.../lib/rustlib/` that appeared.
5. Run `rustup which rustc` and `rustup which cargo`. Explain why these differ
   from `which rustc` and `which cargo`.

---

[Course Index](../README.md) · [1.2 Hello, Cargo: Your First Project →](02-hello-cargo.md)
