[← 18.8 Performance Profiling](08-profiling.md) · [Course Index](../README.md)

# 18.9 — The Essential Crate Directory

## What You'll Learn

- The crate you reach for in each category, and when the alternative wins
- How to evaluate an unfamiliar crate before depending on it
- `cargo-deny`, MSRV, licenses, dependency count
- Where to go after this course

## Coming From Another Language

Rust's standard library is deliberately small: no HTTP client, no JSON, no async
runtime, no random numbers, no date type. Where Python has `datetime` and Go has
`net/http`, Rust has a crate — and usually one obvious winner the whole ecosystem
has settled on.

That convergence is the useful fact. `serde` is *the* serialization crate the way
`numpy` is *the* array library: alternatives exist and almost nobody uses them.
This directory is mostly a list of those defaults.

> Versions are the latest resolving as of writing, verified with
> `cargo add --dry-run`. Semver means `tokio = "1.54"` keeps working as 1.x
> advances — pin minor versions only when you have a reason.

## Runtime and async

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `tokio` | 1.54.1 | async runtime: tasks, IO, timers, sync, signals | any async program; the default |
| `futures` | 0.3.34 | combinators, `Stream`, `join_all`, `select` | when `tokio`'s own utilities aren't enough |
| `tokio-stream` | 0.1.19 | `Stream` adapters over tokio types | iterating a channel or interval as a stream |
| `tokio-util` | 0.7.20 | `CancellationToken`, codecs, `Framed` | graceful shutdown (18.4), custom protocols |
| `async-trait` | 0.1.92 | `async fn` in traits, pre-2024 style | object-safe async traits (`dyn`) |
| `pin-project-lite` | 0.2.17 | safe `Pin` projection without `unsafe` | hand-writing a `Future` or `Stream` |
| `async-stream` | 0.3.6 | `stream!` macro, yield in a loop | building a `Stream` imperatively |

**How to choose:** `tokio` unless something forces otherwise — axum, sqlx,
reqwest and most of the ecosystem assume it. `smol` and `async-std` exist and
are smaller; the cost is that half the crates below won't work. Native
`async fn` in traits (stable since 1.75) covers most of what `async-trait` did;
you still need the macro for `dyn Trait` objects. Modules 15 and 15.7 cover this.

## Web

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `axum` | 0.8.9 | HTTP server: routing, extractors, state | the default web framework (Module 17) |
| `tower` | 0.5.3 | the `Service` abstraction, middleware | writing reusable middleware |
| `tower-http` | 0.7.1 | tracing, CORS, compression, timeouts, auth layers | almost always, alongside axum |
| `hyper` | 1.12.0 | the HTTP/1 + HTTP/2 implementation underneath | only when you need raw HTTP |
| `hyper-util` | 0.1.21 | connection pooling, server/client helpers for hyper 1.x | custom clients or servers on hyper |
| `reqwest` | 0.13.5 | HTTP client: JSON, cookies, retries, multipart | calling other services (17.12) |
| `axum-extra` | 0.12.6 | typed headers, cookies, query extras | when axum's built-ins fall short |
| `tower-sessions` | 0.15.0 | session middleware with pluggable stores | cookie sessions instead of JWTs |

**How to choose:** `axum` is the default — it is maintained by the tokio team,
built on `tower` so middleware is shared, and its extractor model is ordinary
Rust traits rather than magic. `actix-web` is comparably fast and mature with a
different (actor-derived) API; `poem` and `salvo` are smaller. For a client,
`reqwest` with `rustls-tls` rather than `native-tls` — it drops the OpenSSL C
dependency, which matters for musl and Docker (18.6).

## Serialization

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `serde` | 1.0.229 | the derive-based serialization framework | always; everything below plugs into it |
| `serde_json` | 1.0.154 | JSON | APIs, config, logs |
| `toml` | 1.1.8 | TOML | config files (18.1), reading `Cargo.toml` |
| `csv` | 1.4.0 | streaming CSV reader/writer | imports, exports, data files |
| `rmp-serde` | 1.3.1 | MessagePack, compact binary | cache payloads, internal RPC |
| `serde_with` | 3.24.0 | ready-made `#[serde(with)]` adapters | timestamps-as-strings, `Duration`, base64 fields |
| `serde_yaml` | 0.9.34 | YAML | Kubernetes manifests, legacy config |

**How to choose:** `serde` is not a choice, it is the interface. Pick the format
crate: `serde_json` for anything over HTTP, `toml` for human-edited config,
`rmp-serde` or `bincode` when size and speed beat readability. `simd-json` is a
faster drop-in parser for large documents. Note `serde_yaml` is no longer
actively developed — `serde_yml` and `serde_norway` are forks; prefer TOML for
new config. Module 17.1 covers serde properly.

## Database

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `sqlx` | 0.9.0 | async SQL with compile-time query checking | the default for Postgres/MySQL/SQLite |
| `diesel` | 2.3.14 | synchronous ORM, typed query DSL | you want the schema in Rust's type system |
| `sea-orm` | 2.0.4 | async ORM with entities and relations | you want an ActiveRecord-style ORM |
| `redis` | 1.8.0 | Redis client, sync and async | caching, rate limits, pub/sub |
| `deadpool` | 0.13.1 | generic async connection pool | pooling something that has no pool |
| `bb8` | 0.9.1 | generic async pool, alternative design | same need, different API |
| `moka` | 0.12.16 | in-process concurrent cache with TTL/LRU | caching before you reach for Redis |

**How to choose:** `sqlx` if you are comfortable writing SQL — `query!` validates
your statements against the real schema at compile time, which catches a typo'd
column before CI does, and `.sqlx/` plus `SQLX_OFFLINE` makes it work in Docker
and CI (18.6, 18.7). `diesel` if you want the compiler to construct queries for
you and don't mind the sync model (it has `diesel-async`). `sea-orm` if you
genuinely want an ORM with relations and migrations in Rust. `sqlx` and `diesel`
both ship pools; `deadpool`/`bb8` are for pooling other things. Modules 17.8 and
17.9 cover sqlx.

## Error handling

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `thiserror` | 2.0.21 | derive `Error` + `Display` on your own enums | library code, and any error a caller matches on |
| `anyhow` | 1.0.104 | one boxed error type with context and backtraces | application code, `main`, glue |
| `color-eyre` | — | `anyhow` with prettier reports | CLIs where the user reads the error |

**How to choose:** both, in the same project. `thiserror` for the typed errors a
caller must distinguish (`NotFound` vs `Conflict` — your `AppError` mapping to
HTTP status codes in 17.6 is exactly this). `anyhow::Result` for the places where
anything can go wrong and you just want `?` plus `.context("loading config")`.
The rule: a library returns `thiserror` types, a binary's `main` returns
`anyhow::Result`. Module 08 covers both.

## Logging and observability

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `tracing` | 0.1.44 | structured, span-based instrumentation | every service (18.2) |
| `tracing-subscriber` | 0.3.23 | the subscriber: `fmt`, `EnvFilter`, JSON | always, with `tracing` |
| `tracing-appender` | 0.2.5 | non-blocking writer, file rotation | writing logs to files |
| `metrics` | 0.24.6 | metrics facade: counters, gauges, histograms | every service (18.3) |
| `metrics-exporter-prometheus` | 0.18.3 | Prometheus text + bucket config | scraped by Prometheus |
| `opentelemetry` | 0.33.1 | OTel traces/metrics/logs API | distributed tracing across services |
| `tracing-opentelemetry` | 0.34.0 | bridges `tracing` spans to OTel | exporting your existing spans |
| `log` | 0.4 | the old logging facade | only to satisfy a dependency |

**How to choose:** `tracing`, not `log`. The reason is in 18.2: spans follow a
task across `.await`, so concurrent requests stay separable — a thread-local
`log` setup cannot do that. You still get `log`-based dependencies for free, via
the bridge that `tracing-subscriber` enables by default. Add OTel when you have
more than one service to correlate.

## Time and IDs

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `chrono` | 0.4.45 | dates, times, timezones, parsing, formatting | the common choice; `DateTime<Utc>` |
| `time` | 0.3.55 | same domain, smaller and stricter API | you want a leaner, more auditable crate |
| `uuid` | 1.29.0 | UUID v4/v7, parsing, serde | primary keys, request ids |
| `ulid` | 3.0.0 | sortable 128-bit ids, base32 | ids you want lexicographically time-ordered |
| `jiff` | — | newer datetime crate, tzdb-aware | greenfield code wanting a modern API |

**How to choose:** `chrono` if your dependencies already use it (sqlx and serde
integrate with both). `time` if you want a smaller surface and are not fighting
other crates. For ids, `uuid` with the `v7` feature gives you time-ordered UUIDs
— which is what you actually want for a database primary key, since random v4
keys fragment B-tree indexes. `ulid` is the same idea in a shorter, friendlier
encoding.

## Configuration and CLI

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `config` | 0.15.27 | layered config: defaults, files, env | services with per-environment config (18.1) |
| `figment` | 0.10.19 | same job, different API; Rocket's | you prefer defaults from a `Default` impl |
| `dotenvy` | 0.15.7 | load `.env` in development | local development only |
| `clap` | 4.6.7 | argument parsing via derive | any binary with flags |

**How to choose:** `config` for a service whose settings come from TOML plus
environment; `clap` for a tool whose settings come from the command line; both
when a service also takes flags. `dotenvy` is a development convenience and
should never be relied on in a container. Lesson 18.1 has worked examples of all
three.

## Crypto and auth

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `argon2` | 0.6.0 | Argon2id password hashing | hashing user passwords; the current default |
| `bcrypt` | 0.19.3 | bcrypt hashing | verifying hashes from a legacy system |
| `jsonwebtoken` | 11.1.0 | sign and verify JWTs | token auth (17.11) |
| `rustls` | 0.23.46 | TLS in pure Rust | TLS without OpenSSL; prefer it everywhere |
| `ring` | 0.17.14 | low-level primitives, used by rustls | you have a specific primitive need |
| `sha2` | 0.11.1 | SHA-256/512 | checksums, HMAC, content addressing |
| `aes-gcm` | 0.11.1 | authenticated symmetric encryption | encrypting data at rest |
| `base64` | 0.23.1 | base64 encode/decode | tokens, binary in JSON |

**How to choose:** `argon2` for passwords, with the tuning parameters set
deliberately — the default cost is a security decision, not a detail. Never
`sha2` for passwords: it is designed to be fast, which is the opposite of what
you want. `rustls` over `native-tls` on every crate that offers the choice: pure
Rust, no OpenSSL C build, and it makes musl static builds work (18.6). Write no
cryptography yourself; every crate here is a reviewed implementation of a
specified algorithm, and the gap between that and hand-rolled code is where
vulnerabilities live.

## Text

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `regex` | 1.13.1 | linear-time regex, no backtracking | pattern matching; safe on untrusted input |
| `unicode-segmentation` | 1.13.3 | grapheme, word and sentence boundaries | "characters" users recognise (Module 06) |
| `strsim` | 0.11.1 | Levenshtein, Jaro-Winkler, etc. | fuzzy matching, "did you mean" |
| `nom` | 8.0.0 | parser combinators | a real grammar, not a regex |
| `url` | 2.5.8 | RFC-compliant URL parsing | anything handling URLs |
| `idna` | 1.1.0 | internationalized domain names | validating email/host with non-ASCII |

**How to choose:** `regex` has no backtracking, so it cannot be made to hang on
crafted input — unlike PCRE or JS regex, it is safe on user data. The trade is no
backreferences or lookaround. When you need those, or a real grammar, use `nom`
rather than a bigger regex. `unicode-segmentation` matters more in Rust than
elsewhere because `String` indexes bytes: `"héllo".len()` is 6, and user-visible
character counts need graphemes (Module 06).

## Concurrency

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `rayon` | 1.12.0 | data parallelism: `.par_iter()` | CPU-bound work over a collection |
| `crossbeam` | 0.8.5 | channels, scoped threads, lock-free queues | thread-based concurrency beyond `std` |
| `parking_lot` | 0.13.0 | faster, smaller `Mutex`/`RwLock` | lock contention shows in a profile |
| `dashmap` | 6.2.1 | concurrent `HashMap`, sharded locks | a map many threads write to |
| `arc-swap` | 1.9.3 | atomically swap an `Arc` | config hot-reload, read-mostly state |
| `once_cell` | 1.21.4 | lazy statics | global initialized once |
| `smallvec` | 1.16.4 | `Vec` with inline storage | many short vectors in a hot path |

**How to choose:** `rayon` for CPU-bound parallelism — one-line `.par_iter()`,
measured at 10.8x in 18.8 — but **not** inside an async handler, where it fights
the Tokio runtime. For async code use `tokio::sync` (`Mutex`, `RwLock`, `mpsc`)
rather than these, because the std/parking_lot versions block the worker thread.
`std::sync::OnceLock` and `LazyLock` now cover most of `once_cell`'s purpose.
Modules 14 and 15.5 cover this ground.

## Testing

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `criterion` | 0.8.2 | statistical benchmarking with CIs | measuring an optimization (13.5, 18.8) |
| `proptest` | 1.12.0 | property testing, shrinking counterexamples | invariants over generated input (13.6) |
| `mockall` | 0.15.0 | generate mocks from traits | isolating a dependency behind a trait |
| `wiremock` | 0.6.5 | a real local HTTP server to assert against | testing code that calls an API (17.12) |
| `tempfile` | 3.27.0 | temp files and dirs, auto-deleted | filesystem tests |
| `insta` | 1.49.0 | snapshot testing with review workflow | large structured output, API responses |
| `cargo-nextest` | — | faster parallel test runner | any suite over a few seconds |

**How to choose:** `criterion` for "is it faster", `proptest` for "is it always
correct", `insta` for "did the output change". `wiremock` beats `mockall` for
HTTP clients because it exercises your real serialization and error handling
rather than a stubbed trait. All of these go in `[dev-dependencies]`, so they
never reach your production binary. Module 13 is the full treatment.

## Utility

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `itertools` | 0.15.0 | `chunks`, `group_by`, `dedup_by`, `cartesian_product` | the iterator adapter `std` is missing |
| `rand` | 0.10.3 | random numbers, distributions, shuffling | sampling, tokens, test data |
| `bytes` | 1.12.1 | `Bytes`/`BytesMut`, cheap refcounted slices | network buffers, zero-copy |
| `num-traits` | 0.2.19 | generic numeric traits | code generic over number types |
| `once_cell` | 1.21.4 | lazy initialization | globals |
| `derive_more` | 2.1.1 | derive `From`, `Display`, `Deref`, ... | cutting boilerplate on newtypes |
| `hex` | 0.4.3 | hex encode/decode | hashes, binary in text |
| `mime` | 0.3.17 | MIME type constants and parsing | content negotiation |

**How to choose:** `itertools` is the one to add reflexively — `std`'s iterators
are excellent and it fills the gaps (`chunks`, `tuple_windows`, `sorted_by_key`,
`itertools::izip!`). `bytes` matters when you handle network buffers: `Bytes` is
a refcounted slice, so cloning it copies no data. For cryptographic randomness
use `rand`'s `OsRng`, not the default thread RNG. Module 10 covers iterators.

## CLI

| Crate | Version | What it's for | When to reach for it |
|---|---|---|---|
| `clap` | 4.6.7 | args, subcommands, help, completions | any CLI |
| `indicatif` | 0.18.6 | progress bars and spinners | long-running tools |
| `console` | 0.16.6 | colors, styling, terminal detection | readable terminal output |
| `dialoguer` | — | prompts, confirmations, selects | interactive tools |
| `comfy-table` | — | aligned terminal tables | tabular output |

**How to choose:** `clap` with the `derive` feature — define a struct, get
parsing, `--help` and shell completions. `indicatif` and `console` compose with
it. All three respect `NO_COLOR` and detect whether stdout is a terminal, so
piping your tool's output stays clean.

## Evaluating an unfamiliar crate

A dependency is code you ship and are responsible for. Ten minutes of checking
is cheap.

**1. Downloads and recent trend.** On crates.io, look at recent downloads, not
all-time. High all-time with low recent means a crate the ecosystem has moved
off.

**2. Last release and open issues.** A crate with no release in two years is
either finished (`hex`, `mime` — fine) or abandoned (fine for a leaf utility,
not for your TLS stack). Check whether maintainers respond to issues.

**3. Dependency count.**

```bash
cargo tree -p <crate> --depth 3
cargo tree --duplicates
```

A utility crate pulling 80 transitive dependencies is a red flag: compile time,
audit surface, and 80 more chances of a supply-chain problem.
`cargo tree --duplicates` finds the same crate at two major versions, which
bloats your binary and sometimes causes type mismatches.

**4. MSRV.** The minimum supported Rust version, usually in the README or
`rust-version` in `Cargo.toml`. A crate requiring a newer compiler than your
CI pins will fail the build. Also check whether they treat an MSRV bump as a
breaking change — some do not, so a patch release can break your build.

**5. License.**

```bash
cargo deny check licenses
```

`MIT OR Apache-2.0` is the Rust norm. A GPL dependency in a proprietary service
is a legal problem, and it is almost always transitive — nobody adds it on
purpose. This is the check most teams skip and later regret.

**6. `unsafe` and audit status.**

```bash
cargo install cargo-geiger && cargo geiger
```

`unsafe` is not disqualifying — `tokio` and `bytes` need it — but `unsafe` in a
crate that has no reason for it, with one maintainer and no reviews, is worth a
look. Check [RustSec](https://rustsec.org) advisories via `cargo deny`.

**7. Run `cargo-deny` in CI.** All of the above, continuously:

```bash
cargo install cargo-deny
cargo deny init
cargo deny check advisories bans licenses sources
```

`deny.toml`:

```toml
[advisories]
version = 2
yanked = "deny"

[bans]
multiple-versions = "warn"
wildcards = "deny"

[licenses]
version = 2
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "ISC", "Unicode-3.0"]

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

Lesson 18.7 wires this into GitHub Actions. `advisories` is the one that wakes
you up usefully: a new CVE against a dependency you have not touched.

**8. Read the source.** Rust's advantage here is that most crates are small
enough to read. If it is 400 lines of straightforward code, you can vendor or
replace it; if it is 40,000, you are taking on a real dependency. Decide
knowingly.

Quick checklist:

| Check | Command or place |
|---|---|
| Recent downloads | crates.io, "recent downloads" |
| Last release | crates.io versions tab |
| Maintenance | GitHub issues and commit recency |
| Transitive deps | `cargo tree -p <crate> --depth 3` |
| Version conflicts | `cargo tree --duplicates` |
| MSRV | README or `rust-version` |
| License | `cargo deny check licenses` |
| Advisories | `cargo deny check advisories` |
| `unsafe` usage | `cargo geiger` |
| Docs quality | docs.rs — are items documented, with examples? |

## A starting Cargo.toml for a service

Everything the Module 17 stack plus this module needs:

```toml
[dependencies]
axum = "0.8.9"
tokio = { version = "1.54.1", features = ["macros", "rt-multi-thread", "net", "signal", "time"] }
tower = "0.5.3"
tower-http = { version = "0.7.1", features = ["trace", "cors", "compression-gzip", "timeout"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.154"
sqlx = { version = "0.9.0", default-features = false, features = ["runtime-tokio", "tls-rustls", "postgres", "macros", "migrate", "chrono", "uuid"] }
thiserror = "2.0.21"
anyhow = "1.0.104"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter", "json"] }
metrics = "0.24.6"
metrics-exporter-prometheus = "0.18.3"
config = "0.15.27"
dotenvy = "0.15.7"
uuid = { version = "1.29.0", features = ["v7", "serde"] }
chrono = { version = "0.4.45", features = ["serde"] }
argon2 = "0.6.0"
jsonwebtoken = "11.1.0"
tokio-util = { version = "0.7.20", features = ["rt"] }

[dev-dependencies]
wiremock = "0.6.5"
criterion = "0.8.2"
tempfile = "3.27.0"
```

Note what is *not* there: no `futures`, no `hyper`, no `log`, no `once_cell`,
no `lazy_static`. Add a dependency when you need it. Note also the explicit
`tokio` feature list rather than `full`, and `default-features = false` on
`sqlx` with `tls-rustls` — both cut cold build time meaningfully (18.5, 18.7)
and the rustls choice is what makes musl builds work (18.6).

## Where to go next

You now have the whole path: the language (Modules 01-12), testing (13),
concurrency and async (14-15), macros and `unsafe` (16), a real backend (17), and
production concerns (18). Nothing in this course depends on reading anything
else.

What to do with it:

**Build the service end to end.** Take the Module 17 API and add every part of
this module: layered config with a validated `AppConfig` (18.1), `tracing` with
JSON output and request ids (18.2), `/metrics` and split liveness/readiness
probes (18.3), the four-step graceful shutdown (18.4), a tuned release profile
(18.5), a `cargo-chef` Dockerfile with Compose for Postgres (18.6), and the CI
workflow with a Postgres service container (18.7). That assembly is the real
exercise; each lesson alone is a piece of it.

**Then profile it under load.** Point a load generator at it, use 18.8's
discipline, and find out what your service actually spends time on. It will not
be what you expect, and learning to be wrong about that cheaply is the most
transferable skill in this module.

**Read the source of one crate you depend on.** Pick `axum` or `tower`. Both are
readable, and seeing how extractors and `Service` are built from ordinary traits
— no magic, no reflection — is what makes the type system click.

**Pick a problem where Rust's properties matter.** A CLI tool that must start
instantly, a service where tail latency matters, a data pipeline that must not
run out of memory, something embedded. Rust's cost is up front: you fight the
borrow checker to get guarantees you then stop thinking about. Choose work where
that trade pays.

**And keep the measurement habit.** The theme running through this entire
module is that production decisions are empirical. Release profiles, image
sizes, allocation counts, latency — every table in these nine lessons came from
running something and reading the number, including the ones where the expected
answer turned out to be wrong (`opt-level = "z"` making the binary bigger,
`lines.clone()` costing 1%). Do that with your own system and you will be right
more often than people who reason from first principles about performance.

## Quick Reference

| Need | Reach for |
|---|---|
| Async runtime | `tokio` |
| HTTP server | `axum` + `tower-http` |
| HTTP client | `reqwest` (with `rustls-tls`) |
| Serialization | `serde` + `serde_json` |
| SQL | `sqlx` (or `diesel` for an ORM) |
| Caching | `moka` in-process, `redis` shared |
| Typed errors | `thiserror` |
| Application errors | `anyhow` |
| Logging | `tracing` + `tracing-subscriber` |
| Metrics | `metrics` + `metrics-exporter-prometheus` |
| Dates | `chrono` (or `time`) |
| IDs | `uuid` with `v7` |
| Config | `config` + `dotenvy` |
| CLI args | `clap` |
| Passwords | `argon2` |
| TLS | `rustls` |
| Regex | `regex` |
| CPU parallelism | `rayon` |
| Concurrent map | `dashmap` |
| Benchmarks | `criterion` |
| Property tests | `proptest` |
| HTTP mocking | `wiremock` |
| Iterator extras | `itertools` |
| Dependency auditing | `cargo-deny` |

## Common Mistakes

**`tokio = { features = ["full"] }` everywhere.** It compiles considerably more
than you need. List the features — the service `Cargo.toml` above shows a
realistic set.

**Adding `futures` out of habit.** `tokio` and std cover most of it. Add it when
a compile error tells you to.

**`native-tls` instead of `rustls`.** Pulls in OpenSSL, adds a C build step,
grows the image, and breaks musl static builds.

**`sha2` or `bcrypt` for new password hashing.** `sha2` is far too fast to be a
password hash. Use `argon2`.

**`uuid` v4 as a primary key.** Random keys fragment B-tree indexes. Use `v7`,
which is time-ordered.

**Never running `cargo deny`.** License and advisory problems are both
transitive and both found in seconds.

**Reaching for a crate before checking `std`.** `OnceLock`, `LazyLock`,
`std::sync::mpsc`, `Vec::chunks`, `sort_by_key` and `Iterator`'s own adapters
cover more than most people realise.

## Exercises

1. Run `cargo tree --duplicates` on a project of yours. Identify one crate
   appearing at two versions and find which dependency pins the older one.
2. Set up `cargo deny` with the `deny.toml` above and run all four checks. Fix
   or explicitly allow every finding.
3. Replace `features = ["full"]` on `tokio` with the minimum set your code
   needs. Measure the cold build time before and after.
4. Pick a crate you have never used from this directory, work through all eight
   evaluation steps, and write down whether you would depend on it.
5. Build the full service described in "Where to go next" — Module 17's API plus
   every lesson in Module 18 — then load-test it and report what the profile
   says the bottleneck is.

---

[← 18.8 Performance Profiling](08-profiling.md) · [Course Index](../README.md)
