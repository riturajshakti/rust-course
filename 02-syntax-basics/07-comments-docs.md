[← 2.6 Constants, Statics, and Type Aliases](06-const-static.md) · [Course Index](../README.md) · [3.1 if, else if, and if as an Expression →](../03-control-flow/01-if-expressions.md)

# 2.7 — Comments and Documentation Comments

## What You'll Learn

- `//`, `/* */`, and the fact that block comments nest
- `///` documents the next item, `//!` documents the enclosing one
- Doc comments are Markdown, with conventional `#` sections
- Code blocks in docs are run by `cargo test` — your examples cannot rot
- `cargo doc --open`, `#[doc]`, and `missing_docs`

## Coming From Another Language

| Other | Rust |
|---|---|
| JSDoc `/** ... */` | `/// ...` above the item |
| Python docstring inside the body | `//!` inside, `///` above |
| Go doc comment `// Foo does ...` | `/// Returns ...` |
| Examples in docs are decoration | examples are **tests** |

The one genuinely different thing: documentation examples compile and run as
part of your test suite.

## Ordinary comments

```rust
fn main() {
    // line comment
    /* block comment */
    /* outer /* inner */ still outer */
    let x = 1 /* inline */ + 2;
    /* multi
       line
    */
    println!("{x}");
}
```

Output:

```
3
```

Block comments **nest**, unlike C and JavaScript — so commenting out a region
that already contains `/* */` works. In practice `//` is used for almost
everything; `rustfmt` leaves both alone and never reflows them.

## Comments should be rare

The code should carry the meaning. A comment earns its place only when the code
genuinely cannot:

```rust
let checksum = (sum & 0xFF) as u8;

let backoff_ms = 250;

std::thread::yield_now();
```

versus:

```rust
let checksum = (sum & 0xFF) as u8;

const RETRY_BACKOFF_MS: u64 = 250;

// Yield here or the worker starves the writer under contention — see #412.
std::thread::yield_now();
```

The second block replaced one comment with a named constant and kept the one
comment that records something the code cannot say: a non-obvious reason a
future reader would otherwise delete. Comments that restate the line (`// add
one to x`) are noise, go stale, and are worse than nothing. Prefer better
names, smaller functions, and a `///` doc comment on the public surface.

## /// documents the next item

```rust
/// Returns the area of a rectangle.
pub fn area(w: f64, h: f64) -> f64 {
    w * h
}

/// A shape.
#[derive(Debug)]
pub enum Shape {
    /// A square with the given side.
    Square(f64),
    /// A circle with the given radius.
    Circle(f64),
}
```

It attaches to whatever comes next — function, struct, field, enum variant,
module, trait method. With nothing after it, that is an error:

```rust
pub fn f() {}

/// This documents nothing.
```

Does not compile:

```
error: expected item after doc comment
 --> src/lib.rs:3:1
  |
3 | /// This documents nothing.
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^ this doc comment doesn't document anything
```

`/** ... */` is the block equivalent of `///` and is valid but rare.

## //! documents the enclosing item

```rust
//! # w02lib
//!
//! Crate-level docs with a list:
//!
//! - [`Parser`] parses
//! - [`parse_all`] parses many
```

At the top of `src/lib.rs` or `src/main.rs` this is the crate's front page; at
the top of a `mod` block or a module file it documents that module. The rule is
positional: `///` points down at the next item, `//!` points up at whatever
contains it. `/*! ... */` is the block form.

Note `[`Parser`]` — square brackets around a path create an intra-doc link that
rustdoc resolves and checks.

## Markdown and the conventional sections

Doc comments are CommonMark. Headings start at `#` (rustdoc renders them one
level down), and the section names below are a community convention that
rustdoc and reviewers expect.

```rust
/// Parses one line into a key and value.
///
/// # Arguments
///
/// * `line` - a `key=value` string
///
/// # Returns
///
/// `None` if there is no `=`.
///
/// # Examples
///
/// Basic use:
///
/// ```
/// use w02lib::Parser;
/// let p = Parser;
/// assert_eq!(p.parse("a=1"), Some(("a", "1")));
/// assert_eq!(p.parse("nope"), None);
/// ```
///
/// # Panics
///
/// Never panics.
pub struct Parser;
```

| Section | Use |
|---|---|
| `# Examples` | runnable examples — by far the most valuable |
| `# Panics` | conditions under which it panics |
| `# Errors` | what the `Err` variants mean |
| `# Safety` | mandatory on `unsafe fn` — the caller's obligations |
| `# Arguments` / `# Returns` | only when the signature is not self-evident |

Lists, tables, `inline code`, links, and emphasis all work. Indented or fenced
blocks become code.

## Doc examples are tests

`cargo test` compiles and runs every code block in a doc comment.

```
   Doc-tests w02lib

running 3 tests
test src/lib.rs - Parser (line 22) ... ok
test src/lib.rs - parse_all (line 51) ... ok
test src/lib.rs - Parser (line 31) ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Break one and the suite fails:

```rust
/// Adds one.
///
/// ```
/// assert_eq!(w02lib::inc(1), 3);
/// ```
pub fn inc(n: i32) -> i32 { n + 1 }
```

```
---- src/lib.rs - inc (line 3) stdout ----
Test executable failed (exit status: 101).

stderr:

thread 'main' panicked at doctest_bundle_2024.rs:6:1:
assertion `left == right` failed
  left: 2
 right: 3

failures:
    src/lib.rs - inc (line 3)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out
```

Each block is a separate program, so it needs its own `use` statements and
refers to your crate by name. A line prefixed with `# ` is compiled but hidden
from the rendered page — use it for setup noise:

```rust
/// ```
/// # use w02lib::Parser;
/// # let p = Parser;
/// assert!(p.parse("k=v").is_some());
/// ```
```

Doc tests only run for library targets, not binaries. Lesson 13.3 covers them
in detail.

## Code block attributes

```rust
/// ```no_run
/// w02lib::area(-1.0, 1.0);
/// ```
///
/// ```should_panic
/// w02lib::area(-1.0, 1.0);
/// ```
///
/// ```ignore
/// this is not even rust
/// ```
```

```
running 4 tests
test src/lib.rs - never_run (line 45) - compile ... ok
test src/lib.rs - inc (line 38) ... ok
test src/lib.rs - panics_doc (line 52) - should panic ... ok
test src/lib.rs - area (line 9) ... ok

running 1 test
test src/lib.rs - ignored_doc (line 59) ... ignored
```

| Attribute | Effect |
|---|---|
| (none) | compile and run |
| `no_run` | compile only |
| `should_panic` | must panic to pass |
| `ignore` | skipped entirely — a smell; prefer `text` |
| `compile_fail` | must fail to compile |
| `text` | not Rust, never compiled |

## #[doc]

```rust
#[doc = "Doubles a number."]
#[doc = ""]
#[doc = "Same as writing `///` on two lines."]
pub fn double(n: i32) -> i32 { n * 2 }
```

`///` is sugar for `#[doc = "..."]`. The attribute form matters when the text is
generated — `#[doc = include_str!("../README.md")]` puts your README on the
crate's front page, keeping the two in sync. Related attributes:
`#[doc(hidden)]` to keep a public item out of the docs, and
`#[doc(alias = "...")]` to make it findable under another name.

## cargo doc

```
cargo doc --open
```

```
 Documenting w02lib v0.1.0 (/private/tmp/.../w02lib)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.06s
   Generated /private/tmp/.../w02lib/target/doc/w02lib/index.html
```

`--open` launches the browser; `--no-deps` skips dependency docs, which is much
faster. Without `--no-deps` you get local HTML for your whole dependency tree,
which is the fastest way to read an API offline and matches the exact versions
you compiled against.

## Requiring docs

```rust
#![warn(missing_docs)]

pub fn undocumented() {}

/// Documented.
pub fn documented() {}

#[allow(missing_docs)]
pub fn exempt() {}
```

```
warning: missing documentation for the crate
 --> src/lib.rs:1:1
  |
1 | / #![warn(missing_docs)]
...
note: the lint level is defined here
 --> src/lib.rs:1:9
  |
1 | #![warn(missing_docs)]
  |         ^^^^^^^^^^^^

warning: missing documentation for a function
 --> src/lib.rs:3:1
```

The lint covers public items only, including the crate itself — hence the first
warning, fixed with a `//!` block. `#[allow(missing_docs)]` exempts one item (or
a whole module, if placed on the `mod`). `#![deny(missing_docs)]` makes it a
hard error, which is reasonable for a published library and overkill for an
application.

## Quick Reference

| Form | Documents / does |
|---|---|
| `// text` | Line comment |
| `/* text */` | Block comment, nests |
| `/// text` | Doc comment on the next item |
| `//! text` | Doc comment on the enclosing item |
| `/** */` / `/*! */` | Block forms of the above |
| `#[doc = "..."]` | What `///` desugars to |
| `#[doc = include_str!("..")]` | Pull docs from a file |
| `#[doc(hidden)]` | Hide a public item from docs |
| `[`Item`]` | Intra-doc link, checked by rustdoc |
| ` ``` ` block | A doc test, run by `cargo test` |
| `# ` line in a block | Compiled but hidden |
| `cargo doc --no-deps --open` | Build and view your docs |
| `#![warn(missing_docs)]` | Require docs on public items |

## Common Mistakes

**Explaining the code in comments.** Rename the binding, extract a function, or
write a `///` doc instead; a comment that restates the line will go stale.

**Mixing up `///` and `//!`.** `///` is before the item, `//!` is inside the
thing it describes. A `///` with nothing after it is a compile error.

**Letting doc examples break.** They are tests — run `cargo test` before
publishing, and use `no_run` only when execution is genuinely impossible.

**Forgetting `use` lines in a doc example.** Each block is its own crate; add
them, hidden with `# ` if they clutter the page.

**Expecting doc tests on a binary.** Only library targets run them; move the
logic into `src/lib.rs`.

## Exercises

1. Nest a block comment three levels deep around working code and confirm it
   still compiles.
2. Create a library crate with a `//!` front page and one `///`-documented
   public function. Run `cargo doc --no-deps --open`.
3. Add an `# Examples` block with an `assert_eq!` to that function, run
   `cargo test`, then break the assertion and read the failure.
4. Add a function that panics on bad input, document it with `# Panics`, and
   prove the behaviour with a `should_panic` doc test.
5. Turn on `#![deny(missing_docs)]`, fix every resulting error, then add one
   private helper and confirm the lint ignores it.

---

[← 2.6 Constants, Statics, and Type Aliases](06-const-static.md) · [Course Index](../README.md) · [3.1 if, else if, and if as an Expression →](../03-control-flow/01-if-expressions.md)
