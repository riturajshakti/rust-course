# The Rust Tour

A hands-on Rust course for developers who already ship software in another language.
Every lesson is short, example-driven, and shows real program output.

- **Target toolchain:** Rust 1.99.1, edition 2024
- **Backend stack taught:** Axum + SQLx + Tokio
- **Format:** 18 modules, 110 lessons. Read in order; each lesson stands alone.

Check the `Done` column to track your progress as files land.

---

## Module 01 — Getting Started ✅

| # | Lesson | Done |
|---|--------|------|
| 1.1 | [Installing Rust and the Toolchain](01-getting-started/01-install-toolchain.md) | ✅ |
| 1.2 | [Hello, Cargo: Your First Project](01-getting-started/02-hello-cargo.md) | ✅ |
| 1.3 | [Anatomy of a Cargo Project](01-getting-started/03-project-anatomy.md) | ✅ |
| 1.4 | [rustfmt, Clippy, and rust-analyzer](01-getting-started/04-tooling.md) | ✅ |
| 1.5 | [How Rust Thinks: A Mental Model](01-getting-started/05-mental-model.md) | ✅ |

## Module 02 — Syntax Basics ✅

| # | Lesson | Done |
|---|--------|------|
| 2.1 | [Variables, Mutability, and Shadowing](02-syntax-basics/01-variables.md) | ✅ |
| 2.2 | [Scalar Types: Integers, Floats, Bool, Char](02-syntax-basics/02-scalar-types.md) | ✅ |
| 2.3 | [Tuples and Arrays](02-syntax-basics/03-tuples-arrays.md) | ✅ |
| 2.4 | [Functions, Parameters, and Return Values](02-syntax-basics/04-functions.md) | ✅ |
| 2.5 | [Numeric Operators and Type Casting](02-syntax-basics/05-operators-casting.md) | ✅ |
| 2.6 | [Constants, Statics, and Type Aliases](02-syntax-basics/06-const-static.md) | ✅ |
| 2.7 | [Comments and Documentation Comments](02-syntax-basics/07-comments-docs.md) | ✅ |

## Module 03 — Control Flow ✅

| # | Lesson | Done |
|---|--------|------|
| 3.1 | [if, else if, and if as an Expression](03-control-flow/01-if-expressions.md) | ✅ |
| 3.2 | [loop, while, and Loop Labels](03-control-flow/02-loop-while.md) | ✅ |
| 3.3 | [for Loops and Ranges](03-control-flow/03-for-ranges.md) | ✅ |
| 3.4 | [match: Rust's Superpower](03-control-flow/04-match.md) | ✅ |
| 3.5 | [Pattern Matching in Depth](03-control-flow/05-patterns.md) | ✅ |
| 3.6 | [if let, while let, and let else](03-control-flow/06-if-let-chains.md) | ✅ |

## Module 04 — Ownership and Borrowing ✅

| # | Lesson | Done |
|---|--------|------|
| 4.1 | [Stack, Heap, and Why Ownership Exists](04-ownership/01-why-ownership.md) | ✅ |
| 4.2 | [Move Semantics and Copy Types](04-ownership/02-moves-copy.md) | ✅ |
| 4.3 | [References and Borrowing Rules](04-ownership/03-borrowing.md) | ✅ |
| 4.4 | [Mutable Borrows and the Borrow Checker](04-ownership/04-mutable-borrows.md) | ✅ |
| 4.5 | [Slices](04-ownership/05-slices.md) | ✅ |
| 4.6 | [Lifetimes: The Basics](04-ownership/06-lifetimes-basics.md) | ✅ |
| 4.7 | [Lifetime Elision and Struct Lifetimes](04-ownership/07-lifetime-elision.md) | ✅ |
| 4.8 | [Clone, Copy, and Drop](04-ownership/08-clone-drop.md) | ✅ |

## Module 05 — Structs and Enums ✅

| # | Lesson | Done |
|---|--------|------|
| 5.1 | [Defining and Instantiating Structs](05-structs-enums/01-structs.md) | ✅ |
| 5.2 | [Methods and Associated Functions](05-structs-enums/02-methods.md) | ✅ |
| 5.3 | [Tuple Structs and Unit Structs](05-structs-enums/03-tuple-unit-structs.md) | ✅ |
| 5.4 | [Enums and Variants with Data](05-structs-enums/04-enums.md) | ✅ |
| 5.5 | [Option: No More Null](05-structs-enums/05-option.md) | ✅ |
| 5.6 | [Deriving Common Traits](05-structs-enums/06-derive.md) | ✅ |
| 5.7 | [Builder Pattern and Default](05-structs-enums/07-builder-default.md) | ✅ |

## Module 06 — Strings and Text ✅

| # | Lesson | Done |
|---|--------|------|
| 6.1 | [String vs &str](06-strings/01-string-vs-str.md) | ✅ |
| 6.2 | [Creating and Growing Strings](06-strings/02-creating-strings.md) | ✅ |
| 6.3 | [String Operations Reference](06-strings/03-string-operations.md) | ✅ |
| 6.4 | [Searching, Splitting, and Replacing](06-strings/04-search-split.md) | ✅ |
| 6.5 | [UTF-8, Chars, Bytes, and Grapheme Safety](06-strings/05-utf8-chars.md) | ✅ |
| 6.6 | [Formatting and the format! Family](06-strings/06-formatting.md) | ✅ |
| 6.7 | [Parsing and Converting Text](06-strings/07-parsing.md) | ✅ |

## Module 07 — Collections ✅

| # | Lesson | Done |
|---|--------|------|
| 7.1 | [Vec: The Workhorse](07-collections/01-vec.md) | ✅ |
| 7.2 | [Vec Operations Reference](07-collections/02-vec-operations.md) | ✅ |
| 7.3 | [Sorting, Searching, and Dedup](07-collections/03-sort-search.md) | ✅ |
| 7.4 | [HashMap](07-collections/04-hashmap.md) | ✅ |
| 7.5 | [HashMap Operations Reference](07-collections/05-hashmap-operations.md) | ✅ |
| 7.6 | [BTreeMap and Ordered Data](07-collections/06-btreemap.md) | ✅ |
| 7.7 | [HashSet and BTreeSet](07-collections/07-sets.md) | ✅ |
| 7.8 | [VecDeque, BinaryHeap, LinkedList](07-collections/08-other-collections.md) | ✅ |
| 7.9 | [Choosing the Right Collection](07-collections/09-choosing.md) | ✅ |

## Module 08 — Error Handling ✅

| # | Lesson | Done |
|---|--------|------|
| 8.1 | [panic! and When to Use It](08-error-handling/01-panic.md) | ✅ |
| 8.2 | [Result and Propagation](08-error-handling/02-result.md) | ✅ |
| 8.3 | [The ? Operator](08-error-handling/03-question-operator.md) | ✅ |
| 8.4 | [Combinators on Option and Result](08-error-handling/04-combinators.md) | ✅ |
| 8.5 | [Custom Error Types](08-error-handling/05-custom-errors.md) | ✅ |
| 8.6 | [thiserror and anyhow](08-error-handling/06-thiserror-anyhow.md) | ✅ |
| 8.7 | [Error Handling Strategy for Libraries vs Apps](08-error-handling/07-strategy.md) | ✅ |

## Module 09 — Generics and Traits ✅

| # | Lesson | Done |
|---|--------|------|
| 9.1 | [Generic Functions and Structs](09-generics-traits/01-generics.md) | ✅ |
| 9.2 | [Defining and Implementing Traits](09-generics-traits/02-traits.md) | ✅ |
| 9.3 | [Trait Bounds and where Clauses](09-generics-traits/03-trait-bounds.md) | ✅ |
| 9.4 | [Default Methods and Supertraits](09-generics-traits/04-default-supertraits.md) | ✅ |
| 9.5 | [Trait Objects and Dynamic Dispatch](09-generics-traits/05-trait-objects.md) | ✅ |
| 9.6 | [Operator Overloading and Standard Traits](09-generics-traits/06-std-traits.md) | ✅ |
| 9.7 | [From, Into, TryFrom, and Conversions](09-generics-traits/07-conversions.md) | ✅ |
| 9.8 | [Associated Types and Generic Traits](09-generics-traits/08-associated-types.md) | ✅ |

## Module 10 — Closures and Iterators ✅

| # | Lesson | Done |
|---|--------|------|
| 10.1 | [Closures and Capture Modes](10-closures-iterators/01-closures.md) | ✅ |
| 10.2 | [Fn, FnMut, FnOnce](10-closures-iterators/02-fn-traits.md) | ✅ |
| 10.3 | [The Iterator Trait](10-closures-iterators/03-iterator-trait.md) | ✅ |
| 10.4 | [Adapters: map, filter, and Friends](10-closures-iterators/04-adapters.md) | ✅ |
| 10.5 | [Consumers: collect, fold, and Friends](10-closures-iterators/05-consumers.md) | ✅ |
| 10.6 | [Iterator Operations Reference](10-closures-iterators/06-iterator-reference.md) | ✅ |
| 10.7 | [Writing Your Own Iterator](10-closures-iterators/07-custom-iterator.md) | ✅ |
| 10.8 | [itertools and rayon](10-closures-iterators/08-itertools-rayon.md) | ✅ |

## Module 11 — Smart Pointers and Interior Mutability ✅

| # | Lesson | Done |
|---|--------|------|
| 11.1 | [Box and Heap Allocation](11-smart-pointers/01-box.md) | ✅ |
| 11.2 | [Rc and Shared Ownership](11-smart-pointers/02-rc.md) | ✅ |
| 11.3 | [RefCell and Interior Mutability](11-smart-pointers/03-refcell.md) | ✅ |
| 11.4 | [Arc and Mutex](11-smart-pointers/04-arc-mutex.md) | ✅ |
| 11.5 | [Deref, AsRef, and Borrow](11-smart-pointers/05-deref.md) | ✅ |
| 11.6 | [Cow: Clone on Write](11-smart-pointers/06-cow.md) | ✅ |
| 11.7 | [Weak References and Cycles](11-smart-pointers/07-weak-cycles.md) | ✅ |

## Module 12 — Modules, Crates, and Cargo ✅

| # | Lesson | Done |
|---|--------|------|
| 12.1 | [Modules and Visibility](12-modules-crates/01-modules.md) | ✅ |
| 12.2 | [use, pub use, and Re-exports](12-modules-crates/02-use-reexports.md) | ✅ |
| 12.3 | [Multi-file and Multi-crate Layouts](12-modules-crates/03-file-layout.md) | ✅ |
| 12.4 | [Cargo.toml in Depth](12-modules-crates/04-cargo-toml.md) | ✅ |
| 12.5 | [Workspaces](12-modules-crates/05-workspaces.md) | ✅ |
| 12.6 | [Features and Conditional Compilation](12-modules-crates/06-features.md) | ✅ |
| 12.7 | [Publishing to crates.io](12-modules-crates/07-publishing.md) | ✅ |

## Module 13 — Testing and Documentation ✅

| # | Lesson | Done |
|---|--------|------|
| 13.1 | [Unit Tests and assert Macros](13-testing/01-unit-tests.md) | ✅ |
| 13.2 | [Integration Tests](13-testing/02-integration-tests.md) | ✅ |
| 13.3 | [Doc Tests](13-testing/03-doc-tests.md) | ✅ |
| 13.4 | [Test Organization and Fixtures](13-testing/04-organization.md) | ✅ |
| 13.5 | [Benchmarking and Criterion](13-testing/05-benchmarks.md) | ✅ |
| 13.6 | [Property Testing and Fuzzing](13-testing/06-property-testing.md) | ✅ |

## Module 14 — Concurrency ✅

| # | Lesson | Done |
|---|--------|------|
| 14.1 | [Threads and join](14-concurrency/01-threads.md) | ✅ |
| 14.2 | [Channels and Message Passing](14-concurrency/02-channels.md) | ✅ |
| 14.3 | [Shared State with Mutex and RwLock](14-concurrency/03-shared-state.md) | ✅ |
| 14.4 | [Send, Sync, and Thread Safety](14-concurrency/04-send-sync.md) | ✅ |
| 14.5 | [Atomics](14-concurrency/05-atomics.md) | ✅ |
| 14.6 | [Scoped Threads and Thread Pools](14-concurrency/06-scoped-pools.md) | ✅ |

## Module 15 — Async and Tokio ✅

| # | Lesson | Done |
|---|--------|------|
| 15.1 | [async/await and Futures](15-async-tokio/01-async-await.md) | ✅ |
| 15.2 | [The Tokio Runtime](15-async-tokio/02-tokio-runtime.md) | ✅ |
| 15.3 | [Spawning Tasks and JoinSet](15-async-tokio/03-tasks.md) | ✅ |
| 15.4 | [select!, Timeouts, and Cancellation](15-async-tokio/04-select-timeouts.md) | ✅ |
| 15.5 | [Async Channels and Synchronization](15-async-tokio/05-async-channels.md) | ✅ |
| 15.6 | [Streams](15-async-tokio/06-streams.md) | ✅ |
| 15.7 | [Async Traits and Pinning](15-async-tokio/07-async-traits-pin.md) | ✅ |
| 15.8 | [Blocking Code in Async Contexts](15-async-tokio/08-blocking.md) | ✅ |

## Module 16 — Macros and Unsafe ✅

| # | Lesson | Done |
|---|--------|------|
| 16.1 | [Declarative Macros with macro_rules!](16-macros-unsafe/01-macro-rules.md) | ✅ |
| 16.2 | [Procedural Macros Overview](16-macros-unsafe/02-proc-macros.md) | ✅ |
| 16.3 | [Writing a Derive Macro](16-macros-unsafe/03-derive-macro.md) | ✅ |
| 16.4 | [unsafe Rust and Raw Pointers](16-macros-unsafe/04-unsafe.md) | ✅ |
| 16.5 | [FFI: Calling C and Being Called](16-macros-unsafe/05-ffi.md) | ✅ |

## Module 17 — Backend Development with Axum ✅

| # | Lesson | Done |
|---|--------|------|
| 17.1 | [Serde: Serialization and Deserialization](17-backend-axum/01-serde.md) | ✅ |
| 17.2 | [Your First Axum Server](17-backend-axum/02-first-server.md) | ✅ |
| 17.3 | [Routing, Path, and Query Extractors](17-backend-axum/03-routing-extractors.md) | ✅ |
| 17.4 | [JSON Requests and Responses](17-backend-axum/04-json.md) | ✅ |
| 17.5 | [Shared State and Dependency Injection](17-backend-axum/05-state.md) | ✅ |
| 17.6 | [Error Handling in Axum](17-backend-axum/06-errors.md) | ✅ |
| 17.7 | [Middleware and tower Layers](17-backend-axum/07-middleware.md) | ✅ |
| 17.8 | [SQLx: Connecting to Postgres](17-backend-axum/08-sqlx.md) | ✅ |
| 17.9 | [Queries, Transactions, and Migrations](17-backend-axum/09-queries-migrations.md) | ✅ |
| 17.10 | [Validation and Request Guards](17-backend-axum/10-validation.md) | ✅ |
| 17.11 | [Authentication with JWT](17-backend-axum/11-auth-jwt.md) | ✅ |
| 17.12 | [HTTP Clients with reqwest](17-backend-axum/12-reqwest.md) | ✅ |
| 17.13 | [Testing an Axum Application](17-backend-axum/13-testing-axum.md) | ✅ |

## Module 18 — Production Rust ✅

| # | Lesson | Done |
|---|--------|------|
| 18.1 | [Configuration and Environment](18-production/01-config.md) | ✅ |
| 18.2 | [Logging and Tracing](18-production/02-tracing.md) | ✅ |
| 18.3 | [Metrics and Health Checks](18-production/03-metrics-health.md) | ✅ |
| 18.4 | [Graceful Shutdown](18-production/04-graceful-shutdown.md) | ✅ |
| 18.5 | [Release Profiles and Build Optimization](18-production/05-release-builds.md) | ✅ |
| 18.6 | [Docker and Deployment](18-production/06-docker.md) | ✅ |
| 18.7 | [CI with GitHub Actions](18-production/07-ci.md) | ✅ |
| 18.8 | [Performance Profiling](18-production/08-profiling.md) | ✅ |
| 18.9 | [The Essential Crate Directory](18-production/09-crate-directory.md) | ✅ |

---

## Lesson Format

Every lesson follows the same shape so you always know where to look:

1. **What You'll Learn** — the 3-5 takeaways
2. **Coming From Another Language** — how this maps to JS/TS, Python, or Go
3. **Explanation** — the concept, in small steps
4. **Examples** — runnable code with its actual output
5. **Common Mistakes** — the errors you will hit, and the fix
6. **Exercises** — a few problems to confirm it stuck

## How to Use This Course

```bash
# Create a scratch project to try the examples in
cargo new rust-practice
cd rust-practice

# Run examples as you read
cargo run

# Keep these on while you learn — the compiler is the best tutor here
cargo clippy
cargo fmt
```

Work the modules in order. Modules 01-08 are the foundation and worth not rushing;
ownership (04) is the one that takes real time to click. Modules 17-18 are where
your full-stack experience starts paying off directly.
