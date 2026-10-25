[← 18.4 Graceful Shutdown](04-graceful-shutdown.md) · [Course Index](../README.md) · [18.6 Docker and Deployment →](06-docker.md)

# 18.5 — Release Profiles and Build Optimization

## What You'll Learn

- Every `[profile.release]` knob, with measured build time and binary size
- Custom profiles, and `release-with-debug` for profiling
- `target-cpu=native` and why shipping it can crash on another machine
- `cargo build --timings`, incremental compilation, `sccache`

## Coming From Another Language

| Other | Rust |
|---|---|
| `NODE_ENV=production` (mostly logging) | `--release`: a genuinely different compile |
| `python -O` (barely anything) | 4-10x runtime difference, routinely |
| webpack minify settings | `[profile.release]` in `Cargo.toml` |

`cargo run` gives you a debug build: no optimization, overflow checks on, debug
info in. It can be an order of magnitude slower than release. Benchmarking a
debug build is the single most common Rust performance mistake — 18.8 has the
measurement to prove it.

## Measuring on a real project

> **These numbers are machine-specific.** Measured on an Apple M4
> (`rustc --print target-cpus` reports `apple-m4`), macOS, Rust 1.99.1, on a
> small Axum service: `axum`, `tokio` (`full`), `serde`, `serde_json`,
> `tracing`, `tracing-subscriber` — 2.5 MB default release binary. Build time is
> a **cold** build (`rm -rf target`) of the whole dependency graph. Your absolute
> numbers will differ; the *ratios* are what transfer.

Baseline:

| Profile | Cold build | Binary size |
|---|---|---|
| `dev` (debug) | 9.7 s | 10,899,736 B (10.4 MB) |
| `release` (default) | 8.3 s | 2,585,072 B (2.47 MB) |

The debug binary is **4.2x larger** — all debug info — and the debug build was
not even faster to produce here, because debug info generation costs about what
the dependency optimization did on a project this size.

## opt-level

| Setting | Cold build | Binary size | vs default |
|---|---|---|---|
| `opt-level = 0` (dev default) | 9.7 s | 10.4 MB | — |
| `opt-level = 1` | 8.0 s | 2,844,224 B | +10% size |
| `opt-level = 2` | — | — | — |
| `opt-level = 3` (release default) | 8.3 s | 2,585,072 B | baseline |
| `opt-level = "s"` | 9.5 s | 2,868,576 B | +11% |
| `opt-level = "z"` | 18.1 s | 3,086,160 B | +19% |

Two surprises worth internalising. `"s"` and `"z"` mean "optimize for size" and
here they made the binary **bigger**, because `3` inlines aggressively and then
LLVM deletes the code it inlined away, while `z` also disables loop vectorization
and auto-unrolling in ways that cost both speed and, on this project, size. And
`z` was more than twice as slow to build.

`opt-level = "z"` only wins combined with LTO (see the last table) and even then
costs runtime speed. Keep `3` unless you are targeting embedded.

| Value | Meaning |
|---|---|
| `0` | none |
| `1` | basic |
| `2` | most optimizations |
| `3` | all, aggressive inlining (release default) |
| `"s"` | optimize for size |
| `"z"` | size, plus no loop vectorization |

## lto and codegen-units

LTO optimizes across crate boundaries. Without it, each codegen unit is compiled
independently and cross-crate inlining is limited.

| Setting | Cold build | Binary size | vs default |
|---|---|---|---|
| default (`lto = false`, 16 CGUs) | 8.3 s | 2,585,072 B | — |
| `lto = "thin"` | 20.1 s | 2,565,024 B | −0.8% |
| `lto = "fat"` | 18.7 s | 2,003,840 B | **−22%** |
| `lto = "fat"`, `codegen-units = 1` | 14.9 s | 1,848,560 B | **−28%** |

`fat` LTO cut 22% of the binary for roughly 2.3x the build time. Adding
`codegen-units = 1` took another 6% off *and* was faster than `fat` alone here —
one unit means no cross-unit work to redo.

`codegen-units` is the parallelism/quality dial: the default 16 compiles faster
on many cores but optimizes less; `1` is maximum quality, no intra-crate
parallelism.

| `lto` value | Behaviour |
|---|---|
| `false` / `"off"` | none (release default) |
| `"thin"` | parallel, cheap, most of the benefit |
| `true` / `"fat"` | whole-program, slowest, best results |

Runtime effect is real but modest and workload-dependent — commonly 0-15% for a
web service, where most time is in syscalls and the database, not your code. LTO
is primarily a size win for a server. Measure with 18.8's tools before assuming
otherwise.

## panic and strip

| Setting | Cold build | Binary size | vs default |
|---|---|---|---|
| default | 8.3 s | 2,585,072 B | — |
| `strip = "symbols"` | 11.1 s | 1,943,152 B | **−25%** |
| `panic = "abort"` | 12.1 s | 1,960,432 B | **−24%** |

Each on its own removes about a quarter of the binary, and both are nearly free
at runtime. These are the two settings every production profile should have —
`strip` unconditionally.

`strip` options: `"none"`, `"debuginfo"`, `"symbols"` (both). Stripping means a
panic backtrace shows addresses instead of function names, so keep an unstripped
copy of every binary you ship (see `release-with-debug` below).

`panic = "abort"` replaces unwinding with an immediate `SIGABRT`. It removes all
landing-pad code — hence the size drop — and can speed up hot paths slightly
since LLVM no longer reasons about unwind edges.

Before you set it, three consequences:

- `std::panic::catch_unwind` no longer works. If a dependency relies on it,
  behaviour changes.
- **A panic in one Tokio task kills the whole process** instead of being
  captured by `JoinHandle` as `Err(JoinError)`. For a web service this is a real
  decision: with `unwind`, one bad request's panic returns a 500 and the server
  lives; with `abort`, it takes down every in-flight request on the process.
- `#[should_panic]` tests cannot run, so `cargo test` needs the unwinding
  profile anyway.

For a service behind a supervisor that restarts instantly, `abort` is defensible
(you fail loudly rather than continuing in an unknown state). For a service
where one panicking endpoint shouldn't drop 200 other requests, keep `unwind`.
Axum turns a handler panic into a 500 with `CatchPanicLayer` (17.7) — which
requires unwinding.

## Everything together

| Profile | Cold build | Binary size | vs default |
|---|---|---|---|
| `release` default | 8.3 s | 2,585,072 B | — |
| `fat` + `cgu=1` + `abort` + `strip` | 14.1 s | 1,215,568 B | **−53%** |
| ...plus `opt-level = "z"` | 12.1 s | 904,608 B | **−65%** |

```toml
[profile.release]
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"
```

53% smaller for 1.7x the cold build time. That is the profile to ship for a
container image, minus `panic = "abort"` if you decided against it above.

The last row — adding `opt-level = "z"` — gets to 904,608 bytes, 65% off. Take
it only if image size genuinely dominates; you are trading real runtime
performance for ~310 KB, which for a server is a bad deal.

## Build time vs runtime

Cold build cost against what you get:

| Change | Build cost | Size benefit | Runtime benefit |
|---|---|---|---|
| `strip = "symbols"` | +34% | −25% | none |
| `panic = "abort"` | +46% | −24% | slight |
| `lto = "thin"` | +142% | −1% | small |
| `lto = "fat"` + `cgu=1` | +80% | −28% | small-moderate |
| `opt-level = "z"` | +118% | **+19%** (worse) | negative |

`strip` first — it is pure profit. Then `lto = "fat"` with `codegen-units = 1`.
`opt-level` stays at `3`.

Crucially, these costs land on **cold** builds, which means CI and Docker. Your
local edit-compile loop is incremental:

```
release incremental: 0.6s
debug incremental: 0.3s
```

Touching `src/main.rs` and rebuilding took 0.6 s in release and 0.3 s in debug.
Cold-build cost is a CI caching problem (18.7), not a developer-experience one.

## A custom profile for profiling

Profilers need symbols, so a stripped release binary is useless to them. Rather
than editing `[profile.release]`, add a profile that inherits from it.

```toml
[profile.release-with-debug]
inherits = "release"
debug = 1
strip = "none"
```

```bash
cargo build --profile release-with-debug
```

Measured:

```
release-with-debug: 9.4s  2,966,136 bytes
```

Artifacts land in `target/release-with-debug/`, so they never collide with your
real release build:

```
target/
  debug/
  release/
  release-with-debug/
```

| `debug` value | Contents |
|---|---|
| `0` / `false` | none |
| `1` / `"line-tables-only"` | line numbers — enough for flamegraphs |
| `2` / `true` | full debug info (dev default) |

`debug = 1` is what you want: profilers and backtraces resolve correctly, and the
binary is far smaller than `debug = 2`. Note it changes **nothing** about
optimization — the code is identical to release, only symbols were added. 18.8
uses this profile throughout.

Two other useful custom profiles:

```toml
[profile.dev]
opt-level = 1

[profile.dev.package."*"]
opt-level = 3

[profile.test]
inherits = "dev"
opt-level = 1
```

`[profile.dev.package."*"]` optimizes dependencies at `3` while keeping *your*
crate at `1` — dependencies rarely change, so you pay once and your tests stop
crawling. This is the best quality-of-life setting in this lesson for anyone
whose test suite does real work.

## target-cpu=native

By default `rustc` targets a baseline CPU, so the binary runs anywhere in the
architecture family. `native` lets LLVM use everything your machine has.

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

```
native build ok, size=2585072
```

Identical size on this project — unsurprising, since the gains are in
vectorized loops and this is an I/O-bound web service. Numeric code can see
10-30%.

```bash
rustc --print target-cpus
```

```
Available CPUs for this target:
    native          - Select the CPU of the current host (currently apple-m4).
    a64fx
    ampere1
```

**When it is unsafe to ship.** The binary may contain instructions the machine
running it does not have, and the failure mode is `SIGILL` — illegal instruction
— at an arbitrary point, with no useful message. Build on a newer host than you
deploy to and you have a crash that reproduces nowhere else.

- Never in a container image that could schedule on heterogeneous nodes.
- Never in a published binary or a release artifact.
- Never in a cloud build where the builder's instance type isn't pinned to the
  runtime's.
- Fine for a benchmark on the machine you are benchmarking on.
- Fine when you fully control the hardware and rebuild per deploy target.

The safe middle ground names a floor that your fleet is guaranteed to clear:

```bash
RUSTFLAGS="-C target-cpu=x86-64-v2" cargo build --release
```

`x86-64-v2` (SSE4.2, ~2009+) is essentially universal; `v3` adds AVX2 and is
safe on most modern cloud fleets but **not** on some older or burstable
instance families. Check before you commit to `v3`.

Put it in `.cargo/config.toml` so it is reproducible rather than living in
someone's shell:

```toml
[build]
rustflags = ["-C", "target-cpu=x86-64-v2"]
```

## cargo build --timings

Shows where a cold build actually goes.

```bash
cargo build --timings --release
```

```
      Timing report saved to /.../target/cargo-timings/cargo-timing-20261025T170910940Z-d6b2114f79959b7d.html
    Finished `release` profile [optimized] target(s) in 0.01s
```

Open the HTML. It gives a Gantt chart of every crate, a concurrency graph, and
the unit times. What to look for:

- One very wide bar late in the build — a bottleneck crate everything waits on.
  `sqlx-macros`, `diesel_derives` and `serde_derive` are common.
- A long tail where only one or two units run while your cores idle — that is
  the dependency graph's critical path, and `codegen-units` won't help.
- A crate you did not know you depended on taking 30 seconds. Check it with
  `cargo tree -i <crate>` and consider turning off the feature that pulls it in.

Feature trimming is usually the biggest cold-build win available. `tokio` with
`features = ["full"]` compiles considerably more than `["rt-multi-thread",
"macros", "net", "signal"]`, and most projects need only the latter.

## Incremental compilation

On for `dev`, **off** for `release`, which is why the cold-build numbers above
are what CI sees.

```toml
[profile.release]
incremental = true
```

That makes repeated local release builds much faster at some optimization
quality cost. Do not enable it for the artifact you ship, and do not enable it
in CI — the cache is per-machine and bigger than the build it saves.

`CARGO_INCREMENTAL=0` forces it off, which is what CI images should set:
incremental metadata is large, invalidates constantly, and poisons the
`target/` cache you are trying to restore in 18.7.

## sccache

`cargo install sccache` and set `RUSTC_WRAPPER=sccache` to cache compiled crates
across projects and, with an S3/Redis backend, across machines and CI runners —
a real win when many projects share a dependency set, though it cannot cache your
own crate's incremental work.

## debug_assertions in release

Release turns off `debug_assertions`, which silently changes behaviour.

```rust
fn add(a: u8, b: u8) -> u8 { a + b }

fn main() {
    println!("debug_assertions = {}", cfg!(debug_assertions));
    debug_assert!(1 == 2, "compiled out in release");
    println!("survived the debug_assert");

    let a: u8 = std::env::args().count() as u8 + 249;
    let b: u8 = 10;
    println!("wrapping_add = {}", a.wrapping_add(b));
    println!("checked_add  = {:?}", a.checked_add(b));
    println!("saturating   = {}", a.saturating_add(b));
    println!("plain a + b  = {}", add(a, b));
}
```

Debug output:

```
debug_assertions = true
thread 'main' (14064770) panicked at examples/r1.rs:5:5:
compiled out in release
```

Release output:

```
debug_assertions = false
survived the debug_assert
wrapping_add = 4
checked_add  = None
saturating   = 255
plain a + b  = 4
```

`250 + 10` on a `u8` **panicked in debug and silently wrapped to 4 in release**.
That is a real behaviour difference between your tests and production, and it is
why `checked_add`/`saturating_add`/`wrapping_add` exist: they state the intent
and behave identically in both profiles.

Force the check on in release when correctness matters more than the cycle:

```toml
[profile.release]
overflow-checks = true
debug-assertions = true
```

For a service handling money or quotas, `overflow-checks = true` in release is
cheap insurance.

## Reproducible builds, briefly

Same source plus same toolchain should give a byte-identical binary. What breaks
it: absolute paths baked into panic messages and debug info, and build
timestamps.

```toml
[build]
rustflags = ["--remap-path-prefix", "/home/builder/src=/src"]
```

Plus `strip = "symbols"` (removes most embedded paths), a committed
`Cargo.lock`, and a pinned toolchain:

```toml
# rust-toolchain.toml
[toolchain]
channel = "1.99.1"
components = ["rustfmt", "clippy"]
```

`rust-toolchain.toml` is the highest-value file here regardless of
reproducibility: it pins the compiler for every developer and every CI run, so
"works on my machine" stops being about toolchain drift. Commit it.

## A production Cargo.toml

```toml
[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
strip = "symbols"
panic = "unwind"

[profile.release-with-debug]
inherits = "release"
debug = 1
strip = "none"

[profile.dev.package."*"]
opt-level = 3
```

`panic = "unwind"` is explicit: it is the default, and writing it down records
that the 24% size saving was considered and declined so a single panicking
handler cannot drop every other in-flight request.

## Quick Reference

| Setting | Values | Effect (measured on this project) |
|---|---|---|
| `opt-level` | `0`-`3`, `"s"`, `"z"` | `3` is the default and was also smallest |
| `lto` | `false`, `"thin"`, `"fat"` | `"fat"` −22% size, +125% cold build |
| `codegen-units` | integer | `1` with fat LTO: −28% total |
| `panic` | `"unwind"`, `"abort"` | `abort` −24%, kills process on task panic |
| `strip` | `"none"`, `"debuginfo"`, `"symbols"` | −25%, no runtime cost |
| `debug` | `0`, `1`, `2` | `1` for profiling; no effect on optimization |
| `incremental` | bool | on for dev, leave off for release/CI |
| `overflow-checks` | bool | off in release by default; turn on if it matters |
| `debug-assertions` | bool | off in release by default |
| `inherits` | profile name | required for a custom profile |
| Custom profile | `--profile name` | output in `target/name/` |
| Timing report | `cargo build --timings` | HTML Gantt chart |
| CPU tuning | `-C target-cpu=x86-64-v2` | safe floor; `native` can `SIGILL` elsewhere |
| Shared cache | `RUSTC_WRAPPER=sccache` | cross-project/CI crate cache |

## Common Mistakes

**Benchmarking a debug build.** Up to 10x off, and it ranks optimizations
wrongly. Always `--release`. 18.8 measures this.

**Assuming `opt-level = "z"` shrinks the binary.** It made this one 19%
*bigger* and took twice as long. Measure.

**`panic = "abort"` on a web service without thinking.** One panicking handler
now kills every in-flight request on the process, and `CatchPanicLayer` stops
working.

**Shipping `target-cpu=native`.** `SIGILL` on any host older than the builder,
with no diagnostic. Use `x86-64-v2`.

**Editing `[profile.release]` to add `debug = 1` for profiling.** You then ship
a bloated binary or forget to revert. Use an inheriting profile.

**Stripping without keeping an unstripped copy.** Production backtraces become
hex addresses. Archive the `release-with-debug` artifact.

## Exercises

1. Measure your own project's cold release build time and binary size. Add
   `strip = "symbols"` and measure both again.
2. Try `lto = "thin"` and `lto = "fat"` with `codegen-units = 1`. Build a table
   like the one above and note that your ratios differ from this machine's.
3. Add a `release-with-debug` profile. Confirm the binary is bigger than
   `release` and lands in its own `target/` subdirectory.
4. Run `cargo build --timings --release` on a cold build and name the three
   slowest crates. Check whether a feature flag can drop any of them.
5. Write a function that overflows a `u8`. Run it in both profiles, then set
   `overflow-checks = true` in release and confirm the behaviour changes.

---

[← 18.4 Graceful Shutdown](04-graceful-shutdown.md) · [Course Index](../README.md) · [18.6 Docker and Deployment →](06-docker.md)
