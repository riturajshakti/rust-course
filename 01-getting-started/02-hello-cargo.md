[← Course Index](../README.md)

# 1.2 — Hello, Cargo: Your First Project

## What You'll Learn

- `cargo new`, and `run` / `build` / `check` / `--release`
- `println!` formatting, and why it has a `!`
- What debug vs release actually changes at runtime
- `cargo add`, `Cargo.lock`, and what lives in `target/`

## Coming From Another Language

| Other | Rust |
|---|---|
| `npm init` / `go mod init` | `cargo new <name>` |
| `npm install axios` | `cargo add <crate>` |
| `package-lock.json` / `go.sum` | `Cargo.lock` |
| `node index.js` | `cargo run` |
| `tsc --noEmit` | `cargo check` |

Cargo is build tool, test runner, and package manager in one binary. There is no
separate `webpack`/`jest`/`tsc` to configure, and no `node_modules` — sources are
cached globally in `~/.cargo`, build artifacts go in `./target`.

## cargo new

```bash
cargo new hello
cd hello
```

Output:

```
    Creating binary (application) `hello` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
```

You get exactly three files:

```
hello/
├── Cargo.toml
├── .gitignore
└── src/
    └── main.rs
```

`cargo new` also runs `git init`. Use `--vcs none` to skip it, or `cargo init`
to set up a project in a directory that already exists.

```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2024"

[dependencies]
```

```rust
fn main() {
    println!("Hello, world!");
}
```

`fn main()` is the entry point. No class wrapper, no `if __name__ ==
"__main__"`, no exported handler.

## Running

```bash
cargo run
```

Output:

```
   Compiling hello v0.1.0 (/private/tmp/scratch/hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.31s
     Running `target/debug/hello`
Hello, world!
```

Three lines of cargo chatter, then your output. `--quiet` (or `-q`) suppresses
the chatter, which is what the rest of this course uses:

```bash
cargo run --quiet
```

Output:

```
Hello, world!
```

Pass arguments to your program after `--`:

```bash
cargo run --quiet -- --verbose input.txt
```

Without `--`, cargo would try to interpret `--verbose` as its own flag.

## build, check, run

| Command | Compiles | Links a binary | Runs it | Speed |
|---|---|---|---|---|
| `cargo check` | Yes | No | No | Fastest |
| `cargo build` | Yes | Yes | No | Slower |
| `cargo run` | Yes | Yes | Yes | Slower |

`cargo check` does type-checking and borrow-checking but skips code generation.
It is several times faster than `build`, which makes it the right command for a
tight edit-compile loop when you only want to know "does it compile".

```bash
cargo check
```

Output:

```
    Checking hello v0.1.0 (/private/tmp/scratch/hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.43s
```

`cargo build` produces `target/debug/hello`, which you can run directly — no
cargo involved:

```bash
cargo build
./target/debug/hello
```

Output:

```
Hello, world!
```

Rebuilds are incremental. Touch nothing and cargo does nothing:

```bash
cargo build
```

Output:

```
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
```

## println! and why it's a macro

The `!` means macro, not function. `println!` has to be one for two reasons:

- It takes a **variable** number of arguments, and Rust functions do not.
- The format string is checked **at compile time** against those arguments.

That second point is the real payoff — a format bug is a compile error, not a
runtime surprise or a silent `undefined`:

```rust
fn main() {
    println!("{} and {}", 1);
}
```

Does not compile:

```
error: 2 positional arguments in format string, but there is 1 argument
 --> src/main.rs:2:15
  |
2 |     println!("{} and {}", 1);
  |               ^^     ^^   -
```

Printing macros, all compile-time checked:

| Macro | Does |
|---|---|
| `println!` | stdout, with newline |
| `print!` | stdout, no newline |
| `eprintln!` | stderr, with newline |
| `eprint!` | stderr, no newline |
| `format!` | returns a `String` instead of printing |

`eprintln!` matters more than it looks: logs and diagnostics go to stderr so
that piping your program's real output to a file still works.

## Format arguments

Name a variable inside the braces and it is captured directly.

```rust
fn main() {
    let name = "raj";
    let n = 7;
    println!("Hello, {name}!");
    println!("{} squared is {}", n, n * n);
    println!("{0} {1} {0}", "a", "b");
}
```

Output:

```
Hello, raj!
7 squared is 49
a b a
```

Inline `{name}` only works for a bare variable. An expression still needs a
positional slot: `{}` with `n * n` after it.

Width, alignment, precision:

```rust
fn main() {
    println!("{:>8}|", "right");
    println!("{:<8}|", "left");
    println!("{:^8}|", "mid");
    println!("{:08.3}", 3.14159_f64);
    println!("{:width$}|", 1, width = 6);
    println!("{:.*}", 2, 1.23456);
}
```

Output:

```
   right|
left    |
  mid   |
0003.142
     1|
1.23
```

Number bases, sign, scientific:

```rust
fn main() {
    println!("{:x} {:X} {:o} {:b}", 255, 255, 255, 255);
    println!("{:#x} {:#b}", 255, 5);
    println!("{:+} {:e}", 42, 1234.5_f64);
}
```

Output:

```
ff FF 377 11111111
0xff 0b101
+42 1.2345e3
```

`{}` requires the type to implement `Display`. Most of your own types won't, so
use `{:?}` (`Debug`) — that is what `#[derive(Debug)]` is for:

```rust
fn main() {
    println!("{:?}", vec![1, 2, 3]);
    println!("{{literal braces}}");
}
```

Output:

```
[1, 2, 3]
{literal braces}
```

Double the brace to escape it. Lesson 6.6 covers formatting exhaustively.

## Debug vs release

```bash
cargo build --release
./target/release/hello
```

`--release` turns on optimization and turns **off** two debug-only checks.
Integer overflow is the one that surprises people. In debug it panics; in
release it wraps.

```rust
fn add(a: u8, b: u8) -> u8 {
    a + b
}

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let bump: u8 = args.len() as u8 * 10;
    println!("{}", add(250, bump));
}
```

`cargo run --quiet`:

```
thread 'main' (13769311) panicked at src/main.rs:2:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`cargo run --release --quiet`:

```
4
```

Same binary logic, two different behaviours: `250 + 10` panics in debug and
wraps to `4` in release. The value goes through a function and a runtime-derived
argument on purpose — when the compiler can see both operands as constants it
rejects the program outright at compile time, in either profile:

```
error: this arithmetic operation will overflow
 --> src/main.rs:8:13
  |
8 |     let z = y + 10;
  |             ^^^^^^ attempt to compute `250_u8 + 10_u8`, which would overflow
```

When you want wrapping, say so — then both profiles agree:

```rust
fn main() {
    let x: u8 = 250;
    println!("{}", x.wrapping_add(10));
    println!("{:?}", x.checked_add(10));
    println!("{}", x.saturating_add(10));
}
```

Output:

```
4
None
255
```

| | `dev` (debug) | `release` |
|---|---|---|
| Optimization | `opt-level = 0` | `opt-level = 3` |
| Debug symbols | Yes | No |
| Overflow checks | Panic | Wrap |
| `debug_assert!` | Runs | Removed |
| Compile time | Fast | Slow |
| Output | `target/debug/` | `target/release/` |

Develop and test in debug so overflow bugs surface. Ship release. Benchmark
**only** in release — debug Rust can be 10-50x slower, and timings from it are
meaningless. Module 18.5 covers tuning the profiles.

## cargo add

```bash
cargo add rand
```

Output:

```
    Updating crates.io index
      Adding rand v0.10.3 to dependencies
             Features:
             + alloc
             + std
             + std_rng
             + sys_rng
             + thread_rng
             - chacha
             - log
             - serde
             - simd_support
             - unbiased
    Updating crates.io index
     Locking 8 packages to highest Rust 1.99.1 compatible versions
      Adding cfg-if v1.0.5
      Adding chacha20 v0.10.2
      Adding cpufeatures v0.3.1
      Adding getrandom v0.4.3
      Adding libc v0.2.191
      Adding r-efi v6.0.0
      Adding rand v0.10.3
      Adding rand_core v0.10.1
```

The `+`/`-` list is features — optional parts of the crate, covered in 12.6.
Your `Cargo.toml` now reads:

```toml
[dependencies]
rand = "0.10.3"
```

Using it:

```rust
use rand::RngExt;

fn main() {
    let roll = rand::rng().random_range(1..=6);
    println!("You rolled a {roll}");
}
```

Output:

```
You rolled a 2
```

`random_range` lives on the `RngExt` trait, so the `use` is required even though
you never name `RngExt` again — in Rust a trait must be in scope for its methods
to be callable. Drop that line and the method vanishes:

```
error[E0599]: no method named `random_range` found for struct `ThreadRng` in the current scope
   --> src/main.rs:2:28
    |
  2 |     let roll = rand::rng().random_range(1..=6);
    |                            ^^^^^^^^^^^^
    |
    = help: items from traits can only be used if the trait is in scope
help: trait `RngExt` which provides `random_range` is implemented but not in scope; perhaps you want to import it
    |
  1 + use rand::RngExt;
    |
```

This is the standard symptom of a missing trait import, and the compiler names
the exact line to add.

More `cargo add` forms:

```bash
cargo add serde --features derive
cargo add tokio --features full
cargo add --dev assert_cmd
cargo add anyhow@1.0
```

`--dev` puts it in `[dev-dependencies]` — available to tests, examples, and
benches, but not compiled into your shipped binary.

Related commands:

```bash
cargo remove rand
cargo update
cargo tree
```

`cargo update` bumps to the newest versions allowed by `Cargo.toml` and rewrites
`Cargo.lock`. `cargo tree` shows the dependency graph:

```
hello v0.1.0 (/private/tmp/scratch/hello)
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

Eight packages were locked but only six appear here: `r-efi` and `cpufeatures`
are platform-specific and don't apply to this machine.

## Version requirements

`rand = "0.10.3"` is a **caret** requirement — it means "compatible with
0.10.3", not "exactly 0.10.3".

| In Cargo.toml | Accepts |
|---|---|
| `"0.10.3"` | `>=0.10.3, <0.11.0` |
| `"1.2.3"` | `>=1.2.3, <2.0.0` |
| `"=0.10.3"` | Exactly 0.10.3 |
| `"~0.10.3"` | `>=0.10.3, <0.11.0` |
| `"*"` | Anything (don't) |

Pre-1.0 crates treat the **minor** version as breaking, so `0.10` to `0.11` is
not automatic. This is why upgrading a 0.x crate is a real code change: `rand`
renamed its core trait to `RngExt` in a minor bump.

## Cargo.lock

`cargo add` also wrote `Cargo.lock`, recording the exact resolved version and
checksum of every package in the graph.

```toml
# This file is automatically @generated by Cargo.
# It is not intended for manual editing.
version = 4

[[package]]
name = "rand"
version = "0.10.3"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "..."
```

`Cargo.toml` says what you want; `Cargo.lock` says what you got. Never edit it
by hand — use `cargo update`.

**Commit it for binaries. Commit it for libraries too.** The old advice to
gitignore it in libraries is obsolete: a library's lockfile is ignored by
downstream consumers anyway, so committing it costs nothing and makes your own
CI reproducible.

## target/

```
target/
├── CACHEDIR.TAG
├── debug/
│   ├── hello              your binary
│   ├── deps/              compiled dependencies
│   ├── incremental/       incremental compilation cache
│   ├── build/             build-script output
│   └── examples/
└── release/
```

```bash
du -sh target target/debug target/release
```

Output:

```
 34M	target
 20M	target/debug
 13M	target/release
```

34 MB for a hello-world with one dependency. It is all regenerable cache —
`CACHEDIR.TAG` is there so backup tools skip it, and `cargo new` already added
`target` to `.gitignore`. Never commit it.

```bash
cargo clean
cargo clean --release
```

Debug binaries are larger because they carry debug symbols:

```bash
ls -l target/debug/hello target/release/hello
```

Output:

```
576136 target/debug/hello
455552 target/release/hello
```

## Quick Reference

| Command | Does |
|---|---|
| `cargo new <name>` | New binary project |
| `cargo new --lib <name>` | New library project |
| `cargo init` | Project in current directory |
| `cargo check` | Type-check only, fastest |
| `cargo build` | Compile to `target/debug/` |
| `cargo build --release` | Optimized, to `target/release/` |
| `cargo run` | Build and run |
| `cargo run -q` | Build and run, no cargo output |
| `cargo run -- <args>` | Pass args to your program |
| `cargo add <crate>` | Add a dependency |
| `cargo add --dev <crate>` | Add a dev-dependency |
| `cargo remove <crate>` | Drop a dependency |
| `cargo update` | Bump within `Cargo.toml` ranges |
| `cargo tree` | Dependency graph |
| `cargo clean` | Delete `target/` |

## Common Mistakes

**Benchmarking a debug build.** Debug Rust is often 10-50x slower. Any
performance number without `--release` is noise.

**Forgetting `--` before your program's arguments.** `cargo run --verbose` is a
cargo flag; `cargo run -- --verbose` reaches your program.

**Calling a trait method without importing the trait.** `random_range` needs
`use rand::RngExt;` in scope. `error[E0599]: no method named ...` on a type that
clearly has it means a missing trait import.

**Expecting `"0.10.3"` to pin the version.** It is a caret range. Use `"=0.10.3"`
if you truly need one exact version, or rely on `Cargo.lock`.

**Gitignoring `Cargo.lock`, or committing `target/`.** Exactly backwards. Commit
the lockfile, ignore the build directory.

## Exercises

1. `cargo new greeter`. Make it print your name, the current year, and a
   formatted float to 2 decimal places using a single `println!`.
2. Print the same number as decimal, hex, octal, and binary with `{:#x}`-style
   alternate forms. Then right-align each in a field of width 12.
3. Write a program whose output differs between `cargo run` and
   `cargo run --release`. Explain which check `--release` turned off.
4. `cargo add rand`, then roll two dice and print their sum. Delete the
   `use rand::RngExt;` line and record the exact error you get.
5. Run `cargo tree` and `du -sh target`. Then `cargo clean`, run `cargo check`,
   and compare `du -sh target` again — explain the difference versus `cargo build`.

---

[← Course Index](../README.md)
