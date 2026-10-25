[← 13.1 Unit Tests and assert Macros](01-unit-tests.md) · [Course Index](../README.md) · [13.3 Doc Tests →](03-doc-tests.md)

# 13.2 — Integration Tests

## What You'll Learn

- The `tests/` directory and why each file there is its own crate
- Importing your library with `use mycrate::` and what that hides
- Sharing helpers via `tests/common/mod.rs` — and why `tests/common.rs` is wrong
- Reading the four sections of `cargo test` output, and `--test <name>`

## Coming From Another Language

| Other | Rust |
|---|---|
| Jest tests in `__tests__/` importing the built package | `tests/*.rs` importing the crate |
| Go `package foo_test` (external test package) | `tests/*.rs` — closest analogue |
| pytest `tests/` importing the installed module | `tests/*.rs` with `use mycrate::` |

Unit tests (13.1) live inside your source and see everything. Integration tests
live in `tests/` and are **separate crates that link against yours**, so they see
exactly what a real consumer sees: your public API and nothing else.

## The layout

`tests/` sits next to `src/`, as a sibling — not inside it.

```
w13/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   └── main.rs
└── tests/
    ├── cart_api.rs
    ├── pricing.rs
    └── common/
        └── mod.rs
```

Cargo auto-discovers every `.rs` file directly in `tests/`. No registration, no
config. Here is the library under test:

```rust
#[derive(Debug, PartialEq)]
pub struct Cart {
    items: Vec<(String, u32)>,
}

fn apply_discount(total: u32) -> u32 {
    if total > 100 { total * 9 / 10 } else { total }
}

impl Cart {
    pub fn new() -> Self {
        Cart { items: Vec::new() }
    }

    pub fn add(&mut self, name: &str, price: u32) -> &mut Self {
        self.items.push((name.to_string(), price));
        self
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    pub fn total(&self) -> u32 {
        apply_discount(self.items.iter().map(|(_, p)| p).sum())
    }

    pub fn remove(&mut self, name: &str) -> Result<(), String> {
        match self.items.iter().position(|(n, _)| n == name) {
            Some(i) => {
                self.items.remove(i);
                Ok(())
            }
            None => Err(format!("no such item: {name}")),
        }
    }
}
```

Note `items` is a private field and `apply_discount` is a private function.
Both matter below.

## Each file is its own crate

No `#[cfg(test)]` and no `mod tests` wrapper. The file *is* the test crate, so
every `#[test]` goes at the top level, and you import your library by its
package name:

```rust
use w13::Cart;

#[test]
fn no_discount_under_threshold() {
    let mut c = Cart::new();
    c.add("mug", 100);
    assert_eq!(c.total(), 100);
}
```

Output of `cargo test --test pricing`:

```
     Running tests/pricing.rs (target/debug/deps/pricing-19a956306b456cfc)

running 1 test
test no_discount_under_threshold ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`#[cfg(test)]` would be redundant — the whole file is only ever compiled by
`cargo test`. Writing it is harmless but marks you as confused.

Because each file is a separate crate, each gets its own binary and runs in its
own process. Two integration files cannot share state, and they run in parallel
with each other.

## Private items are invisible

This is the hard boundary. Integration tests only reach `pub` items.

```rust
#[test]
fn reach_private() {
    assert_eq!(w13::apply_discount(200), 180);
}
```

Does not compile (intentional):

```
error[E0603]: function `apply_discount` is private
 --> tests/bad.rs:3:21
  |
3 |     assert_eq!(w13::apply_discount(200), 180);
  |                     ^^^^^^^^^^^^^^ private function
  |
note: the function `apply_discount` is defined here
 --> src/lib.rs:6:1
  |
6 | fn apply_discount(total: u32) -> u32 {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

The fix is *not* to make it `pub`. Test `apply_discount` from a `mod tests`
inside `src/lib.rs`, where it is reachable:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn discount_is_private_but_testable_here() {
        assert_eq!(apply_discount(200), 180);
        assert_eq!(apply_discount(50), 50);
    }
}
```

That is the division of labour: **unit tests for internals, integration tests
for the public contract.** Never widen visibility just to test something.

## A binary crate's code is also invisible

An integration test links against your **library** target only. Anything in
`src/main.rs` is unreachable.

```rust
fn main() {
    let mut c = w13::Cart::new();
    c.add("book", 60).add("pen", 60);
    println!("total = {}", c.total());
    helper();
}

fn helper() {
    println!("binary-only helper");
}
```

```rust
#[test]
fn reach_binary_helper() {
    w13::helper();
}
```

Does not compile (intentional):

```
error[E0425]: cannot find function `helper` in crate `w13`
 --> tests/bad.rs:8:10
  |
8 |     w13::helper();
  |          ^^^^^^ not found in `w13`
```

`helper` is not private — it simply is not in the `w13` *library* crate at all.
This is precisely why the lib+main split from 1.3 is the standard shape: put all
real logic in `src/lib.rs` and let `src/main.rs` be a thin `fn main` that parses
arguments and calls into the library. Logic you leave in `main.rs` is logic you
cannot integration-test.

For a binary with no library, the usual workaround is to launch the compiled
binary as a subprocess — the `assert_cmd` crate wraps that (`Command::cargo_bin`
plus assertions on exit status and stdout). Restructuring into lib+main is
almost always the better answer.

## Testing a public API end to end

An integration test should drive the type the way a user would — a sequence of
public calls, then assertions on observable results.

```rust
use w13::Cart;

#[test]
fn new_cart_is_empty() {
    assert_eq!(Cart::new().len(), 0);
    assert_eq!(Cart::new().total(), 0);
}

#[test]
fn remove_round_trip() -> Result<(), String> {
    let mut c = Cart::new();
    c.add("book", 60).add("pen", 60);
    c.remove("pen")?;
    assert_eq!(c.len(), 1);
    assert_eq!(c.total(), 60);
    Ok(())
}

#[test]
fn remove_missing_errors() {
    let mut c = Cart::new();
    assert_eq!(c.remove("ghost"), Err("no such item: ghost".to_string()));
}
```

Output:

```
running 3 tests
test new_cart_is_empty ... ok
test remove_missing_errors ... ok
test remove_round_trip ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`Result`-returning tests and `?` work exactly as in 13.1. Note the assertions
touch only `len()`, `total()` and the returned `Result` — never `items`, which
is private. If a behaviour cannot be observed through the public API, an
integration test cannot check it, and that is a useful signal about your design.

A failure here looks the same as a unit-test failure, but the path points into
`tests/`:

```
---- wrong_expectation stdout ----

thread 'wrong_expectation' (13917748) panicked at tests/pricing.rs:14:5:
assertion `left == right` failed
  left: 450
 right: 500

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--test pricing`
```

The last line gives you the exact rerun command.

## Shared helpers: tests/common/mod.rs

Fixture builders get duplicated fast. Put them in a **subdirectory module**:

`tests/common/mod.rs`

```rust
use w13::Cart;

pub fn full_cart() -> Cart {
    let mut c = Cart::new();
    c.add("book", 60).add("pen", 60);
    c
}

pub fn empty_cart() -> Cart {
    Cart::new()
}
```

Then declare and use it from a test file:

`tests/cart_api.rs`

```rust
mod common;

use w13::Cart;

#[test]
fn builds_and_totals_with_discount() {
    let c = common::full_cart();
    assert_eq!(c.len(), 2);
    assert_eq!(c.total(), 108);
}

#[test]
fn remove_missing_errors() {
    let mut c = common::empty_cart();
    assert_eq!(c.remove("ghost"), Err("no such item: ghost".to_string()));
}
```

Output:

```
     Running tests/cart_api.rs (target/debug/deps/cart_api-af25a295eecca3bc)

running 2 tests
test remove_missing_errors ... ok
test builds_and_totals_with_discount ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Helpers must be `pub`, since `common` is a module and the test file is outside
it. `mod common;` goes in **every** file that wants the helpers — there is no
shared root to declare it once.

## Why tests/common.rs is wrong

Cargo treats every file *directly* in `tests/` as a test target. Name the helper
file `tests/common.rs` and Cargo builds and runs it as a test crate of its own:

```
     Running tests/common.rs (target/debug/deps/common-e37b4b7638db71be)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

A useless empty binary and a confusing `running 0 tests` section in every run.
The `common/mod.rs` form sidesteps it because Cargo only scans the top level of
`tests/`, so a *subdirectory* is never a test target. Any nested path works —
`tests/helpers/mod.rs`, `tests/support/mod.rs` — the name `common` is just
convention.

One consequence of each test file being its own crate: `common` is compiled
fresh into each one, so helpers a given file does not use are dead code *there*.

```
warning: function `full_cart` is never used
 --> tests/common/mod.rs:3:8
  |
3 | pub fn full_cart() -> Cart {
  |        ^^^^^^^^^
  |
  = note: `#[warn(dead_code)]` (part of `#[warn(unused)]`) on by default

warning: `w13` (test "pricing") generated 2 warnings
```

Harmless, and common enough that `#![allow(dead_code)]` at the top of
`common/mod.rs` is the normal response.

## Reading cargo test output

A plain `cargo test` on this project runs four kinds of target in order:

```
     Running unittests src/lib.rs (target/debug/deps/w13-02f05f83cf9a625d)

running 1 test
test tests::discount_is_private_but_testable_here ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/w13-e111fccba50aa532)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/cart_api.rs (target/debug/deps/cart_api-af25a295eecca3bc)

running 4 tests
test new_cart_is_empty ... ok
test remove_missing_errors ... ok
test remove_round_trip ... ok
test builds_and_totals_with_discount ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/pricing.rs (target/debug/deps/pricing-19a956306b456cfc)

running 1 test
test no_discount_under_threshold ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests w13

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Four things to read off it:

- `Running unittests src/lib.rs` — the `#[cfg(test)]` tests inside your library.
- `Running unittests src/main.rs` — the binary's own unit tests. `0 tests` here
  is normal, and it still appears.
- One `Running tests/<file>.rs` block **per integration file**, each a separate
  binary with its own hash suffix and its own result line.
- `Doc-tests <crate>` — documentation examples, covered in 13.3. Library
  targets only.

There is no grand total. Each section reports separately, and `cargo test` exits
non-zero if any section fails.

## Selecting what runs

| Command | Runs |
|---|---|
| `cargo test` | everything: lib, bin, all integration files, doc tests |
| `cargo test --lib` | library unit tests only |
| `cargo test --tests` | all test targets, skipping doc tests |
| `cargo test --test pricing` | one integration file |
| `cargo test --test cart_api remove` | one file, filtered by name |
| `cargo test --doc` | doc tests only |

`--test` takes the file name **without** `.rs`. Combining a target flag with a
name filter is the fastest iteration loop:

```
cargo test --test cart_api remove
```

Output:

```
     Running tests/cart_api.rs (target/debug/deps/cart_api-af25a295eecca3bc)

running 2 tests
test remove_missing_errors ... ok
test remove_round_trip ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 0.00s
```

Only `cart_api` was built and run; the other sections are absent entirely.

## Dev-dependencies

Integration tests may use `[dev-dependencies]`, which are compiled for tests,
examples and benchmarks but never shipped in your library:

```
cargo add --dev tempfile
```

```toml
[dev-dependencies]
tempfile = "3.27.0"
```

This is where `tempfile`, `assert_cmd`, `proptest` and `criterion` belong. They
do not appear in your crate's dependency tree for downstream users.

## Quick Reference

| Item | Meaning |
|---|---|
| `tests/foo.rs` | One integration test crate, auto-discovered |
| `use mycrate::X;` | Import your library by package name |
| no `#[cfg(test)]` needed | The whole file is test-only already |
| `pub` items only | Private items and `main.rs` code are invisible |
| `tests/common/mod.rs` | Shared helpers; must be `pub` |
| `mod common;` | Required in *each* file that uses them |
| `tests/common.rs` | **Wrong** — becomes its own `0 tests` target |
| `--test <name>` | Run one integration file (no `.rs`) |
| `--tests` | All test targets, skip doc tests |
| `Running unittests src/lib.rs` | Unit test section |
| `Running tests/x.rs` | One integration section per file |
| `Doc-tests <crate>` | Doc test section (13.3) |
| `[dev-dependencies]` | Available to tests, not shipped |

## Common Mistakes

**Making an item `pub` to test it.** Unit-test internals from `mod tests` inside
the source file instead; never widen your API for tests.

**Putting logic in `src/main.rs`.** Integration tests link the library target
only, so that code is untestable. Keep `main.rs` a thin wrapper (1.3).

**Naming the helper file `tests/common.rs`.** Cargo runs it as a test crate with
zero tests. Use `tests/common/mod.rs`.

**Forgetting `mod common;` in a new test file.** Each file is its own crate, so
the declaration does not carry over.

**Wrapping an integration test in `#[cfg(test)] mod tests`.** Redundant — put
`#[test]` functions at the top level of the file.

**Expecting integration files to share state.** Each is a separate process.
Build what you need per test, or use a fixture helper (13.4).

## Exercises

1. Add a library function `pub fn word_count(s: &str) -> usize` plus a private
   `fn tokenize`. Write a unit test for `tokenize` and an integration test for
   `word_count`, then try calling `tokenize` from `tests/` and read the error.
2. Create `tests/basic.rs` and `tests/edge_cases.rs`. Run only the second with
   `--test`, then run one test inside it by name.
3. Move a shared fixture into `tests/common/mod.rs` and use it from both files.
   Then rename it to `tests/common.rs` and find the extra section in the output.
4. Add a `src/main.rs` with a helper function and prove from `tests/` that it is
   unreachable. Move the helper into `src/lib.rs` and make the test pass.
5. Write an integration test for an error path that asserts on the exact `Err`
   value, using `assert_eq!` on the whole `Result`.

---

[← 13.1 Unit Tests and assert Macros](01-unit-tests.md) · [Course Index](../README.md) · [13.3 Doc Tests →](03-doc-tests.md)
