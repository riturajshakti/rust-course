[← 1.3 Anatomy of a Cargo Project](03-project-anatomy.md) · [Course Index](../README.md) · [1.5 How Rust Thinks: A Mental Model →](05-mental-model.md)

# 1.4 — rustfmt, Clippy, and rust-analyzer

## What You'll Learn

- `cargo fmt` and `cargo clippy`, and reading a real lint
- Lint levels: attributes, CLI flags, and `[lints]` in `Cargo.toml`
- `-D warnings` in CI, and wiring rust-analyzer to use clippy
- The extra cargo subcommands worth installing

## Coming From Another Language

| Other | Rust |
|---|---|
| Prettier | `cargo fmt` — built in, no config needed |
| ESLint / ruff / `go vet` | `cargo clippy` — ships with the toolchain |
| `tsserver` / `pyright` / `gopls` | `rust-analyzer` |

The difference from the JS world is that there is nothing to choose and nothing
to assemble. One formatter, one linter, one language server, all official, all
installed with the toolchain. No `.eslintrc`, no plugin hunting, no debate.

## cargo fmt

```bash
cargo fmt
```

No output, no config, formats every file in the package. Before:

```rust
fn main(){
let     v = vec![1,2,3];
    for i in 0..v.len() {
println!("{}",v[i]);
}
}
```

After:

```rust
fn main() {
    let v = vec![1, 2, 3];
    for i in 0..v.len() {
        println!("{}", v[i]);
    }
}
```

`--check` formats nothing, prints a diff, and exits non-zero if anything would
change. This is the CI form:

```bash
cargo fmt --check
```

Output:

```
Diff in /private/tmp/scratch/tool/src/main.rs:1:
-fn main(){
-let     v = vec![1,2,3];
+fn main() {
+    let v = vec![1, 2, 3];
     for i in 0..v.len() {
-println!("{}",v[i]);
-}
+        println!("{}", v[i]);
+    }
 }
```

Settle this at project start and never discuss it again. The defaults are fine;
the ecosystem uses them.

If you must change something, `rustfmt.toml` at the package root:

```toml
max_width = 60
```

```rust
fn main() {
    let message = format!(
        "{} {} {}",
        "a longish string", "another one", "and a third"
    );
    println!("{message}");
}
```

The same code at the default `max_width = 100` stays on one line. A handful of
other stable options exist (`tab_spaces`, `edition`, `use_field_init_shorthand`);
most interesting ones are nightly-only. Keeping `rustfmt.toml` absent is a
legitimate choice.

Skip formatting for one item when a hand-aligned table reads better:

```rust
#[rustfmt::skip]
const MATRIX: [[i32; 3]; 3] = [
    [1, 0, 0],
    [0, 1, 0],
    [0, 0, 1],
];
```

## cargo clippy

Clippy is a second compiler front-end with ~750 extra lints. It catches
non-idiomatic code, not just bugs, which makes it the fastest way to learn what
idiomatic Rust looks like.

```bash
cargo clippy
```

On the formatted code above:

```
    Checking tool v0.1.0 (/private/tmp/scratch/tool)
warning: the loop variable `i` is only used to index `v`
 --> src/main.rs:3:14
  |
3 |     for i in 0..v.len() {
  |              ^^^^^^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.99.0/index.html#needless_range_loop
  = note: `#[warn(clippy::needless_range_loop)]` on by default
help: consider using an iterator
  |
3 -     for i in 0..v.len() {
3 +     for <item> in &v {
  |

warning: useless use of `vec!`
 --> src/main.rs:2:13
  |
2 |     let v = vec![1, 2, 3];
  |             ^^^^^^^^^^^^^ help: you can use an array directly: `[1, 2, 3]`
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.99.0/index.html#useless_vec
  = note: `#[warn(clippy::useless_vec)]` on by default

warning: `tool` (bin "tool") generated 2 warnings (run `cargo clippy --fix --bin "tool" -p tool -- ` to apply 1 suggestion)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
```

Read the four parts of a clippy warning:

| Part | Content |
|---|---|
| Message | `the loop variable `i` is only used to index `v`` |
| Location | `src/main.rs:3:14` with the span underlined |
| `= help:` link | The lint's doc page, with rationale and examples |
| `= note:` | The lint name and its default level |

That last line is what you need to silence or escalate it: the lint is
`clippy::needless_range_loop`.

`cargo clippy --fix` applies the machine-applicable suggestions:

```bash
cargo clippy --fix --allow-dirty
```

Note the trailer said "apply 1 suggestion", not 2 — `needless_range_loop`'s
suggestion contains a `<item>` placeholder, so a human has to finish it. Clippy
distinguishes mechanical rewrites from ones needing judgement.

Lint everything, not just the default targets:

```bash
cargo clippy --all-targets --all-features
```

Without `--all-targets`, clippy skips tests, examples, and benches. Use it in CI.

## Lint levels

Four levels, in increasing severity:

| Level | Effect |
|---|---|
| `allow` | Silent |
| `warn` | Printed, build succeeds |
| `deny` | Compile error |
| `forbid` | Compile error that cannot be `allow`ed downstream |

Set them in three places, each overriding the one before it.

**In code**, per item or per crate:

```rust
#![warn(clippy::pedantic)]
#![allow(clippy::must_use_candidate)]

#[allow(clippy::needless_range_loop)]
fn legacy(v: &[i32]) {
    for i in 0..v.len() {
        println!("{}", v[i]);
    }
}
```

`#![...]` with the bang applies to the enclosing crate or module; `#[...]`
applies to the next item. Prefer narrow `#[allow]` on the one function over a
crate-wide one, and leave a note in the commit message explaining why.

**On the command line**, after `--`:

```bash
cargo clippy -- -W clippy::pedantic -A clippy::module_name_repetitions
cargo clippy -- -D warnings
```

**In `Cargo.toml`** — the modern answer, because it is committed and applies to
every invocation including rust-analyzer's:

```toml
[lints.rust]
unused_variables = "warn"
unsafe_code = "forbid"

[lints.clippy]
needless_range_loop = "deny"
useless_vec = "allow"
unwrap_used = "warn"
```

With that table, the same code from before gives one error and no `useless_vec`
warning at all:

```
error: the loop variable `i` is only used to index `v`
 --> src/main.rs:3:14
  |
3 |     for i in 0..v.len() {
  |              ^^^^^^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.99.0/index.html#needless_range_loop
  = note: requested on the command line with `-D clippy::needless-range-loop`

error: could not compile `tool` (bin "tool") due to 1 previous error
```

Use `[lints]` for project policy. Note the lint name uses underscores in TOML
but appears with hyphens in the `note:` line — same lint.

## Lint groups

| Group | Contains |
|---|---|
| `clippy::correctness` | Almost certainly bugs. `deny` by default. |
| `clippy::suspicious` | Probably wrong. `warn` by default. |
| `clippy::style` | Idiom. `warn` by default. |
| `clippy::complexity` | Needlessly convoluted. `warn` by default. |
| `clippy::perf` | Avoidable slowness. `warn` by default. |
| `clippy::pedantic` | Opinionated. `allow` by default. |
| `clippy::nursery` | Unpolished, false positives. `allow`. |
| `clippy::cargo` | `Cargo.toml` issues. `allow`. |

A reasonable starting policy for an application:

```toml
[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "warn"
expect_used = "warn"
panic = "warn"
```

`priority = -1` makes the group apply *before* individual lints, so a specific
entry can override a group-wide level. Omit it while also naming a lint that
belongs to the group and cargo rejects the ambiguity. Turning on `pedantic`
wholesale is noisy — expect to `allow` a dozen of its lints; that triage is
itself a good way to learn the idioms.

## -D warnings in CI

The pattern: warnings are fine locally, fatal in CI. That keeps your edit loop
quiet without letting warnings accumulate on `main`.

```bash
cargo clippy --all-targets --all-features -- -D warnings
```

Output:

```
error: useless use of `vec!`
 --> src/main.rs:2:13
  |
2 |     let v = vec![1, 2, 3];
  |             ^^^^^^^^^^^^^ help: you can use an array directly: `[1, 2, 3]`
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.99.0/index.html#useless_vec
  = note: `-D clippy::useless-vec` implied by `-D warnings`
  = help: to override `-D warnings` add `#[allow(clippy::useless_vec)]`

error: could not compile `tool` (bin "tool") due to 2 previous errors
```

Every warning became an error and the exit code is non-zero. A full CI check:

```yaml
- run: cargo fmt --check
- run: cargo clippy --all-targets --all-features -- -D warnings
- run: cargo test --all-features
```

The equivalent via environment variable, which also covers plain `cargo build`:

```bash
RUSTFLAGS="-D warnings" cargo build
```

Careful: changing `RUSTFLAGS` invalidates the whole build cache, so don't set it
in your shell profile. Module 18.7 covers CI in full.

## rust-analyzer

The language server: completion, go-to-definition, inline types, rename, and
inline error display. Install the "rust-analyzer" extension in VS Code, or the
equivalent LSP client for your editor. Do not install the old "Rust" extension.

Two settings worth changing immediately. In `.vscode/settings.json`:

```json
{
  "rust-analyzer.check.command": "clippy",
  "editor.formatOnSave": true,
  "[rust]": {
    "editor.defaultFormatter": "rust-lang.rust-analyzer"
  }
}
```

`rust-analyzer.check.command` defaults to `check`. Setting it to `clippy` means
every save surfaces clippy lints inline, in the editor, instead of only when you
remember to run the command. This is the single highest-value setting in Rust
tooling — it turns clippy from a chore into continuous feedback.

Other settings that earn their keep:

```json
{
  "rust-analyzer.cargo.features": "all",
  "rust-analyzer.inlayHints.typeHints.enable": true,
  "rust-analyzer.check.allTargets": true
}
```

Inlay type hints matter more in Rust than in TypeScript, because type inference
is aggressive — `let x = thing.iter().filter(...).collect();` has a type you
cannot guess, and the hint shows it.

If go-to-definition into `std` fails, you are missing a component:

```bash
rustup component add rust-src
```

rust-analyzer needs standard library *source*, not just the compiled `rlib`.
Other things to know when it misbehaves:

- It keeps its own `target/` subdirectory, so it will not fight your terminal
  `cargo build` for the lock.
- On a cold open of a large project it indexes for a minute; completions are
  incomplete until it finishes.
- "Restart server" fixes most stale-state weirdness. Deleting `target/` fixes
  the rest.
- It respects `[lints]` in `Cargo.toml` — another reason to prefer that over CLI
  flags.

## Extra cargo subcommands

Installed with `cargo install`, which builds from source into `~/.cargo/bin` and
makes them available as `cargo <name>`.

```bash
cargo install cargo-watch cargo-audit cargo-outdated cargo-nextest cargo-expand
```

| Tool | Command | Does |
|---|---|---|
| cargo-watch | `cargo watch -x check` | Re-run on file change |
| cargo-audit | `cargo audit` | Check `Cargo.lock` against the RustSec advisory DB |
| cargo-outdated | `cargo outdated` | Dependencies newer than your lockfile |
| cargo-nextest | `cargo nextest run` | Faster test runner, better output |
| cargo-expand | `cargo expand` | Show code after macro expansion |

The two to install today:

```bash
cargo watch -x clippy -x test
```

A proper watch loop — clippy then tests, on every save. For a web server,
`cargo watch -x run` is the Rust equivalent of `nodemon`.

```bash
cargo audit
```

Run it in CI on a schedule. It is how you find out a transitive dependency has a
CVE, and it reads `Cargo.lock`, so it sees your real resolved graph.

`cargo expand` is the one that makes macros comprehensible — it prints what
`#[derive(Debug)]` or `#[tokio::main]` actually generated. It requires nightly
to run, but works on stable code:

```bash
cargo +nightly expand
```

`cargo nextest` runs each test in its own process, so one segfault or
infinite-loop test doesn't take down the run, and it is meaningfully faster on
large suites. Note it does **not** run doc tests — keep `cargo test --doc`
alongside it.

## Documentation

```bash
cargo doc --open
```

Output:

```
 Documenting anat v0.1.0 (/private/tmp/scratch/anat)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.14s
   Generated /private/tmp/scratch/anat/target/doc/anat/index.html and 2 other files
```

Builds HTML docs for your crate **and every dependency**, then opens them. This
is better than browsing docs.rs because the versions match your lockfile
exactly. `--no-deps` restricts it to your crate and is much faster.

```bash
cargo doc --no-deps --document-private-items --open
```

`--document-private-items` includes non-`pub` items, which is what you want when
reading your own code rather than publishing an API.

## cargo tree

```bash
cargo tree
```

Output:

```
w01 v0.1.0 (/private/tmp/scratch/w01)
└── rand v0.10.3
    ├── chacha20 v0.10.2
    │   ├── cfg-if v1.0.5
    │   └── rand_core v0.10.1
    ├── getrandom v0.4.3
    │   ├── cfg-if v1.0.5
    │   ├── libc v0.2.191
    │   └── rand_core v0.10.1
    └── rand_core v0.10.1
```

| Flag | Use |
|---|---|
| `cargo tree -i <crate>` | Invert: who depends on this? |
| `cargo tree -d` | Show duplicate versions of the same crate |
| `cargo tree -e features` | Show which features pulled each crate in |

`-i` answers "why is this in my build at all", and `-d` explains surprise
compile-time blowups — two major versions of the same crate both get compiled.

## Quick Reference

| Command | Does |
|---|---|
| `cargo fmt` | Format everything |
| `cargo fmt --check` | CI: diff and fail |
| `cargo clippy` | Lint |
| `cargo clippy --all-targets --all-features` | Lint tests/examples too |
| `cargo clippy --fix` | Apply mechanical suggestions |
| `cargo clippy -- -D warnings` | CI: warnings are errors |
| `cargo doc --no-deps --open` | Build and open docs |
| `cargo tree -i <crate>` | Why is this dependency here |
| `cargo tree -d` | Duplicate versions |
| `cargo watch -x check` | Re-run on save |
| `cargo audit` | Security advisories |
| `cargo outdated` | Available upgrades |
| `cargo nextest run` | Faster tests |
| `cargo +nightly expand` | Post-macro source |

| Lint control | Where |
|---|---|
| `#[allow(clippy::x)]` | One item |
| `#![warn(clippy::x)]` | Crate or module |
| `-- -D warnings` | One command |
| `[lints.clippy] x = "deny"` | Committed project policy |

## Common Mistakes

**Running clippy without `--all-targets`.** Your tests and examples go
unlinted, then fail in CI.

**Leaving `rust-analyzer.check.command` at `check`.** You get type errors inline
but no clippy lints, and lose most of clippy's teaching value.

**Blanket `#![allow(clippy::all)]` to quiet the noise.** Silence the one lint on
the one item instead; a crate-wide allow turns off the lints that find real bugs.

**`#![deny(warnings)]` committed in source.** A future compiler adds a new
warning and your released crate stops building for everyone. Use `-D warnings`
in CI only.

**Missing `rust-src` and blaming rust-analyzer.** `rustup component add rust-src`
fixes go-to-definition into `std`.

**Setting `RUSTFLAGS` globally in your shell profile.** It invalidates the build
cache and triggers full rebuilds whenever it differs.

## Exercises

1. Write a deliberately mangled file — no indentation, no spaces after commas —
   then run `cargo fmt --check`, read the diff, and run `cargo fmt`.
2. Write `for i in 0..v.len() { println!("{}", v[i]); }`, run `cargo clippy`, and
   identify the lint name from the `note:` line. Fix it as clippy suggests.
3. Silence that lint three ways: `#[allow]` on the function, `-A` on the command
   line, and `[lints.clippy]` in `Cargo.toml`. Then escalate it to `deny` via
   `Cargo.toml` and confirm the build fails.
4. Add `[lints.clippy] unwrap_used = "warn"`, write a `.unwrap()` call, and
   confirm the warning appears both in the terminal and in your editor.
5. Add a dependency, run `cargo tree`, then `cargo tree -i cfg-if` to find every
   path that pulls it in. Run `cargo doc --no-deps --open` and read your own docs.

---

[← 1.3 Anatomy of a Cargo Project](03-project-anatomy.md) · [Course Index](../README.md) · [1.5 How Rust Thinks: A Mental Model →](05-mental-model.md)
