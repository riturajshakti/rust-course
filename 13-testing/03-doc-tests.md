[← 13.2 Integration Tests](02-integration-tests.md) · [Course Index](../README.md) · [13.4 Test Organization and Fixtures →](04-organization.md)

# 13.3 — Doc Tests

## What You'll Learn

- `///` examples are compiled and run by `cargo test` — documentation that cannot rot
- Hiding setup lines with `#`, and using `?` inside an example
- The attributes: `no_run`, `ignore`, `should_panic`, `compile_fail`, `text`
- Doc tests run for **library targets only**, and `#![doc = include_str!(...)]`

## Coming From Another Language

| Other | Rust |
|---|---|
| Python doctests (opt-in, `--doctest-modules`) | On by default in `cargo test` |
| JSDoc `@example` (never executed) | `///` code blocks, actually executed |
| Go example functions `ExampleFoo()` | `///` code blocks, no naming convention |

Python's doctests are the closest analogue, but Rust's are not opt-in: every
Rust block in a doc comment on a library target is compiled and run. An example
that drifts out of date fails your build.

## The basics

A fenced block inside `///` becomes a test. It is a **separate crate** that
links your library, so refer to items by their full public path.

```rust
/// Adds two numbers.
///
/// ```
/// assert_eq!(w13::add(2, 3), 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Output of `cargo test --doc`:

```
   Doc-tests w13

running 1 test
test src/lib.rs - add (line 11) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The test name is `<file> - <item path> (line N)`, where `N` is the line of the
opening fence. There is no test function to name.

Each block is wrapped in an implicit `fn main() { ... }`, so you can write
statements directly. `use` works as expected:

```rust
/// Doubles a number.
///
/// ```
/// use w13::double;
/// let x = double(21);
/// assert_eq!(x, 42);
/// println!("doubled to {x}");
/// ```
pub fn double(n: i32) -> i32 {
    n * 2
}
```

Output:

```
test src/lib.rs - double (line 20) ... ok
```

`println!` output is captured, exactly like a unit test — use
`cargo test --doc -- --nocapture` to see it.

## Module-level docs with //!

`//!` documents the *containing* item rather than the next one. At the top of
`src/lib.rs` it documents the crate, and its examples are tested too:

```rust
//! A tiny cart library.
//!
//! ```
//! let mut c = w13::Cart::new();
//! c.add("book", 60);
//! assert_eq!(c.len(), 1);
//! ```
```

Output:

```
test src/lib.rs - (line 3) ... ok
```

The item path is empty — just `src/lib.rs - (line 3)` — because it belongs to
the crate root.

## Assertions, not output matching

Unlike Python doctests, Rust does **not** compare printed output. A doc test
passes if it compiles and does not panic. So assert explicitly:

```rust
impl Cart {
    /// ```
    /// let c = w13::Cart::new();
    /// assert_eq!(c.len(), 0);
    /// ```
    pub fn new() -> Self {
        Cart { items: Vec::new() }
    }

    /// ```
    /// let mut c = w13::Cart::new();
    /// c.add("pen", 10).add("book", 20);
    /// assert_eq!(c.len(), 2);
    /// ```
    pub fn add(&mut self, name: &str, price: u32) -> &mut Self {
        self.items.push((name.to_string(), price));
        self
    }
}
```

Output:

```
test src/lib.rs - Cart::new (line 99) ... ok
test src/lib.rs - Cart::add (line 107) ... ok
```

An example with no assertion still tests that the code **compiles and runs**,
catching signature changes. An `assert_eq!` adds behaviour coverage at no cost
to the reader, so prefer it.

## Hiding setup with #

A line beginning with `# ` is compiled but **not rendered** in the generated
docs. Use it to keep examples short without making them fake.

```rust
/// Hidden setup with `#`.
///
/// ```
/// # fn make() -> Vec<i32> { vec![3, 1, 2] }
/// # let mut v = make();
/// v.sort();
/// assert_eq!(v, [1, 2, 3]);
/// ```
pub fn sorted_example() {}
```

Output:

```
test src/lib.rs - sorted_example (line 32) ... ok
```

The reader sees only `v.sort()` and the assertion; the compiler sees all four
lines. This is the right tool for boilerplate — building a client, opening a
connection, constructing an elaborate fixture. To render a literal leading `#`,
write `##`.

## Using ? in a doc test

`?` needs a function returning `Result`, but the implicit wrapper returns `()`.
The idiom is a hidden trailing line that supplies the return type:

```rust
/// Parses a port.
///
/// ```
/// let p = w13::parse_port("8080")?;
/// assert_eq!(p, 8080);
/// # Ok::<(), std::num::ParseIntError>(())
/// ```
pub fn parse_port(s: &str) -> Result<u16, std::num::ParseIntError> {
    s.parse::<u16>()
}
```

Output:

```
test src/lib.rs - parse_port (line 42) ... ok
```

`Ok::<(), E>(())` is the turbofish form that pins both type parameters, so the
wrapper infers `Result<(), E>` and `?` compiles. It is hidden, so readers see a
clean example that just uses `?`.

The explicit alternative — write your own `fn main`. If a block contains
`fn main`, rustdoc uses it instead of wrapping:

```rust
/// ```
/// # fn main() -> Result<(), std::num::ParseIntError> {
/// let p = w13::parse_port("8080")?;
/// assert_eq!(p, 8080);
/// # Ok(())
/// # }
/// ```
```

`Ok::<(), E>(())` is shorter and more common; the `fn main` wrapper is clearer
when you need `Box<dyn std::error::Error>` for mixed error types.

## should_panic

Same meaning as the attribute in 13.1, written as a fence annotation:

```rust
/// Divides, panicking on zero.
///
/// ```should_panic
/// w13::div(1, 0);
/// ```
pub fn div(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("divide by zero");
    }
    a / b
}
```

Output:

```
test src/lib.rs - div (line 53) - should panic ... ok
```

Note the ` - should panic ` suffix. There is no `expected =` form for doc tests,
so any panic passes — use it sparingly and document the condition in prose.

## no_run

Compiled and type-checked, but never executed. For examples that would hit the
network, the filesystem, or loop forever.

```rust
/// Compiles but is not run.
///
/// ```no_run
/// let data = std::fs::read_to_string("/does/not/exist").unwrap();
/// println!("{data}");
/// ```
pub fn reads_a_file() {}
```

Output:

```
test src/lib.rs - reads_a_file (line 65) - compile ... ok
```

The suffix is ` - compile `, since compiling is all it did. `no_run` still
protects you from API drift — the best guarantee available for an example that
cannot run in CI.

## ignore

Neither compiled nor run. Reported as `ignored`.

```rust
/// Not even compiled.
///
/// ```ignore
/// this is not valid rust at all ???
/// ```
pub fn pseudocode() {}
```

Output:

```
test src/lib.rs - pseudocode (line 73) ... ignored
```

`ignore` is almost always wrong: it leaves an example with *no* verification,
which is exactly what rots. Reach for `no_run` first, or `text` if the block is
not Rust. Legitimate uses are rare — pseudocode, or a snippet for a feature
behind a `cfg` you cannot enable.

## compile_fail

Asserts the block does **not** compile. Useful for documenting what your API
deliberately forbids — type-safety invariants, lifetime restrictions.

```rust
/// This must NOT compile.
///
/// ```compile_fail
/// let s: String = 42;
/// ```
pub fn type_mismatch_demo() {}
```

Output:

```
test src/lib.rs - type_mismatch_demo (line 80) - compile fail ... ok
```

If the code *does* compile, the test fails:

```rust
/// ```compile_fail
/// let x = w13::add(1, 2);
/// ```
pub fn compile_fail_that_compiles() {}
```

Output (intentional failure):

```
---- src/lib.rs - compile_fail_that_compiles (line 138) stdout ----
Test compiled successfully, but it's marked `compile_fail`.
```

`compile_fail` passes on *any* compile error, including a typo in the example,
and is brittle across compiler versions. `compile_fail,E0308` pins the code.

## text and other non-Rust blocks

An unannotated fence is assumed to be Rust and will be compiled. For config
files, shell commands, or output samples, mark the language — anything rustdoc
does not recognize as Rust is rendered but never tested.

```rust
/// Config file format, not Rust.
///
/// ```text
/// host = localhost
/// port = 8080
/// ```
pub fn config_format() {}
```

No test appears in the output for this block. `text`, `json`, `toml`, `bash` and
`console` all behave this way. Forgetting the annotation on a non-Rust block is
the most common doc-test failure.

## The full output

`cargo test --doc` on the library above:

```
   Doc-tests w13

running 9 tests
test src/lib.rs - reads_a_file (line 65) - compile ... ok
test src/lib.rs - sorted_example (line 32) ... ok
test src/lib.rs - Cart::new (line 99) ... ok
test src/lib.rs - div (line 53) - should panic ... ok
test src/lib.rs - Cart::add (line 107) ... ok
test src/lib.rs - (line 3) ... ok
test src/lib.rs - parse_port (line 42) ... ok
test src/lib.rs - add (line 11) ... ok
test src/lib.rs - double (line 20) ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s


running 2 tests
test src/lib.rs - pseudocode (line 73) ... ignored
test src/lib.rs - type_mismatch_demo (line 80) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.03s

all doctests ran in 0.86s; merged doctests compilation took 0.12s
```

Two `running` blocks, which surprises people: ordinary doc tests are batched
into one compilation for speed, while `ignore` and `compile_fail` blocks cannot
be, so they form a second group. The trailing line reports that batching.

## Reading failures

Three distinct failure shapes. A failed assertion reports the child process's
exit status, then its stderr:

```
---- src/lib.rs - broken_assert (line 124) stdout ----
Test executable failed (exit status: 101).

stderr:

thread 'main' (13935847) panicked at src/lib.rs:5:1:
assertion `left == right` failed
  left: 4
 right: 5
```

`src/lib.rs:5:1` is the line *within the generated test*, not your file — the
useful locator is `(line 124)` in the test name.

A compile error shows the full rustc diagnostic against the example's own lines:

```
---- src/lib.rs - broken_compile (line 131) stdout ----
error[E0308]: mismatched types
   --> src/lib.rs:133:26
    |
133 | let x: i32 = w13::add(1, "two");
    |              --------    ^^^^^ expected `i32`, found `&str`
    |              |
    |              arguments to this function are incorrect
    |
note: function defined here
   --> src/lib.rs:14:8
    |
 14 | pub fn add(a: i32, b: i32) -> i32 {
    |        ^^^

error: aborting due to 1 previous error

Couldn't compile the test.
```

Here the line numbers *do* point at your source. And the summary:

```
failures:
    src/lib.rs - broken_assert (line 124)
    src/lib.rs - broken_compile (line 131)
    src/lib.rs - compile_fail_that_compiles (line 138)

test result: FAILED. 10 passed; 3 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: doctest failed, to rerun pass `--doc`
```

## Library targets only

Doc tests run for `src/lib.rs` and nothing else. Examples on items in
`src/main.rs` are **never executed**:

```rust
/// This doc test is never run.
///
/// ```
/// assert_eq!(1, 2);
/// ```
fn unused_doc() {}

fn main() {
    unused_doc();
    println!("{}", w13::add(1, 2));
}
```

Output of `cargo test`:

```
     Running unittests src/main.rs (target/debug/deps/w13-e111fccba50aa532)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests w13

running 10 tests
test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

An obviously false `assert_eq!(1, 2)`, and everything passes — the `Doc-tests`
section covers only the library's ten examples. Same lib+main lesson as 13.2.
Doc tests also only run for `pub` items, since the generated test is an external
crate; a private function's example is not tested.

## Testing your README

Duplicating the crate overview between `README.md` and `//!` docs guarantees
drift. Include the file instead, and its Rust blocks become doc tests:

`README.md`

````text
# w13

A tiny cart library.

```rust
let mut c = w13::Cart::new();
c.add("book", 60);
assert_eq!(c.len(), 1);
```
````

`src/lib.rs`

```rust
#![doc = include_str!("../README.md")]
```

Output:

```
running 10 tests
test src/lib.rs - (line 12) ... ok
test src/lib.rs - (line 19) ... ok
```

Your README's examples are now verified by CI. Two notes: attribution says
`src/lib.rs`, not `README.md`, so a failing line number is relative to the
included file; and README examples need the full crate path
(`w13::Cart::new()`), since they compile as an external crate.

## Running and filtering

| Command | Effect |
|---|---|
| `cargo test` | includes the `Doc-tests` section |
| `cargo test --doc` | doc tests only |
| `cargo test --tests` | all other test targets, **skips** doc tests |
| `cargo test --doc Cart` | substring filter on the test name |
| `cargo test --doc -- --nocapture` | show `println!` from examples |

Filtering works on the generated name, so the item path is matchable:

```
cargo test --doc Cart
```

Output:

```
test src/lib.rs - Cart::new (line 100) ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 8 filtered out; finished in 0.00s
```

Doc tests are the slowest part of a suite — each batch is a compile plus a link.
`cargo test --lib` while iterating, full `cargo test` in CI, is the usual
rhythm.

## Living documentation

The payoff: rename a function or change a parameter order, and every example
referencing it fails to compile. You cannot ship documentation describing an API
you no longer have.

Two habits: put an example on **every** `pub` item (`#![warn(missing_docs)]` at
the crate root flags the gaps, `cargo doc --open` shows the result), and write
examples as the API's first consumer. An example that is awkward to write is
telling you the API is awkward to use.

## Quick Reference

| Form | Behaviour |
|---|---|
| ` ``` ` | Compiled and run |
| ` ```no_run ` | Compiled only; shows as `- compile` |
| ` ```ignore ` | Neither; shows as `ignored` |
| ` ```should_panic ` | Must panic; shows as `- should panic` |
| ` ```compile_fail ` | Must **not** compile; shows as `- compile fail` |
| ` ```compile_fail,E0308 ` | Must fail with that exact error code |
| ` ```text ` / `json` / `bash` | Rendered, never tested |
| `///` | Documents the next item |
| `//!` | Documents the containing item (crate or module) |
| `# line` | Compiled, hidden from rendered docs |
| `## line` | Renders a literal leading `#` |
| `# Ok::<(), E>(())` | Lets `?` work in the implicit `fn main` |
| `# fn main() -> Result<..> {` | Explicit wrapper, for mixed error types |
| `#![doc = include_str!("../README.md")]` | Tests your README's examples |
| `cargo test --doc` | Doc tests only |
| `cargo test --tests` | Skip doc tests |

## Common Mistakes

**Forgetting the language annotation on a non-Rust block.** An unannotated fence
is compiled as Rust and fails. Mark it `text`, `toml`, `bash`.

**Referring to items by short path.** The doc test is an external crate, so
`add(2, 3)` fails — write `w13::add(2, 3)` or add a `use`.

**Using `?` without the hidden `Ok` line.** The implicit wrapper returns `()`.
Add `# Ok::<(), E>(())`.

**Reaching for `ignore` to silence a failing example.** That removes all
verification. Use `no_run` if it merely cannot execute.

**Documenting items in `src/main.rs` and expecting tests to run.** Doc tests are
library-only; those examples are silently never executed.

**Expecting printed output to be checked.** Rust does not compare output, only
pass/panic. Assert explicitly.

**Leaving `compile_fail` unpinned.** It passes on any error, including a typo in
your own example. Pin the code with `compile_fail,E0308`.

## Exercises

1. Write `pub fn slugify(s: &str) -> String` with a `///` example that asserts
   the result. Run `cargo test --doc` and confirm the test name and line number.
2. Add an example needing three lines of setup. Hide the setup with `#` and
   verify with `cargo doc --open` that only the interesting lines render.
3. Write `pub fn parse_config(s: &str) -> Result<u16, ...>` with a doc example
   using `?`. Make it compile two ways: the hidden `Ok::<(), E>(())` line, then
   an explicit hidden `fn main`.
4. Add one block each of `should_panic`, `no_run`, `compile_fail` and `text`.
   Predict each one's suffix in the output before running, then check.
5. Add `#![doc = include_str!("../README.md")]` with a Rust block in the README.
   Break the README example and confirm `cargo test` fails.

---

[← 13.2 Integration Tests](02-integration-tests.md) · [Course Index](../README.md) · [13.4 Test Organization and Fixtures →](04-organization.md)
