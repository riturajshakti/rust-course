[← 1.5 How Rust Thinks: A Mental Model](../01-getting-started/05-mental-model.md) · [Course Index](../README.md) · [2.2 Scalar Types: Integers, Floats, Bool, Char →](02-scalar-types.md)

# 2.1 — Variables, Mutability, and Shadowing

## What You'll Learn

- `let` is immutable by default; `mut` opts in
- When the compiler can infer a type and when you must annotate
- Deferred initialization, and the error when you read too early
- Shadowing: how it differs from `mut` and why it is the idiomatic parsing tool

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `const x = 5` | `let x = 5;` |
| JS `let x = 5` | `let mut x = 5;` |
| Python `x = 5` | `let mut x = 5;` (if reassigned) |
| Go `var x int` then assign | `let x: i32;` then assign once |

The default is flipped: you write extra characters to get mutability, not to
get immutability. And a variable can be *redeclared* in the same scope, which
no other mainstream language lets you do.

## Immutable by default

```rust
fn main() {
    let x = 5;
    println!("{x}");
}
```

Output:

```
5
```

Reassigning is a compile error, not a lint.

```rust
let x = 5;
x = 6;
```

Does not compile:

```
error[E0384]: cannot assign twice to immutable variable `x`
 --> src/main.rs:3:5
  |
2 |     let x = 5;
  |         - first assignment to `x`
3 |     x = 6;
  |     ^^^^^ cannot assign twice to immutable variable
  |
help: consider making this binding mutable
  |
2 |     let mut x = 5;
  |         +++
```

## mut

```rust
let mut y = 5;
println!("{y}");
y = 6;
println!("{y}");
```

Output:

```
5
6
```

`mut` allows reassignment and in-place mutation, but **not** a type change.
Compound assignment works as expected.

```rust
let mut count = 0;
count += 1;
count += 1;
println!("{count}");
```

Output:

```
2
```

There is no `++` or `--` in Rust; use `+= 1`.

## Inference

Most bindings need no annotation.

```rust
let a = 2.5;
let b: i64 = 10;
let c = "hi";
let d = b'A';
println!("{a} {b} {c} {d}");
```

Output:

```
2.5 10 hi 65
```

Defaults when nothing else constrains the value: integer literals become `i32`,
float literals `f64`. `"hi"` is `&str`, `b'A'` is a `u8` byte literal.

## Inference reads the whole function

The type is not decided at the `let`; it is decided by every use in the
function body. Here `n` and `total` become `u64` because of a call that happens
two lines later.

```rust
fn main() {
    let n = 1;
    let total = n * 1000;
    takes_u64(total);
}

fn takes_u64(v: u64) {
    println!("{v}");
}
```

Output:

```
1000
```

This is why Rust rarely needs annotations inside function bodies — but always
requires them on function signatures. Inference never crosses a `fn` boundary.

## When an annotation is required

If nothing in the function pins the type down, the compiler stops.

```rust
let v = Vec::new();
println!("{:?}", v);
```

Does not compile:

```
error[E0282]: type annotations needed for `Vec<_>`
 --> src/main.rs:2:9
  |
2 |     let v = Vec::new();
  |         ^   ---------- type must be known at this point
  |
help: consider giving `v` an explicit type, where the type for type parameter `T` is specified
  |
2 |     let v: Vec<T> = Vec::new();
  |          ++++++++
```

Methods like `parse()` are generic over their return type, so they need the
target type somewhere:

```rust
let guess: u32 = "42".parse().unwrap();
println!("{guess}");
```

Output:

```
42
```

Equivalent and also idiomatic: `"42".parse::<u32>().unwrap()`. Put the
annotation on the binding when the variable is reused, on the turbofish when
it is consumed immediately.

## Deferred initialization

`let` without a value is legal as long as every path assigns exactly once
before the first read. The annotation is usually required here, since there is
no initializer to infer from.

```rust
let b: i64 = 10;
let n;
if b > 5 {
    n = "big";
} else {
    n = "small";
}
println!("{n}");
```

Output:

```
big
```

Note `n` is still immutable — one assignment per path, never two. Reading it
before assignment:

```rust
let n: i32;
println!("{n}");
```

Does not compile:

```
error[E0381]: used binding `n` isn't initialized
 --> src/main.rs:3:16
  |
2 |     let n: i32;
  |         - binding declared here but left uninitialized
3 |     println!("{n}");
  |                ^ `n` used here but it isn't initialized
  |
help: consider assigning a value
  |
2 |     let n: i32 = 42;
  |                ++++
```

The flow analysis is per-path, so forgetting the `else` branch is caught too.
In practice `let x = if cond { a } else { b };` is preferred over deferred
init — see the next section and lesson 3.1.

## Shadowing

A second `let` with the same name creates a new variable, shadowing the first.

```rust
let z = 5;
let z = z + 1;
let z = z * 2;
println!("{z}");
```

Output:

```
12
```

Each `let` is a fresh binding, so the result stays immutable. This is not
mutation — the old `z` still exists, just unreachable by name.

## Shadowing can change the type

```rust
let spaces = "   ";
let spaces = spaces.len();
println!("{spaces}");
```

Output:

```
3
```

`mut` cannot do this:

```rust
let mut spaces = "   ";
spaces = spaces.len();
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:3:14
  |
2 |     let mut spaces = "   ";
  |                      ----- expected due to this value
3 |     spaces = spaces.len();
  |              ^^^^^^^^^^^^ expected `&str`, found `usize`
```

`mut` says "this slot can hold a different value"; shadowing says "this name
now means a different thing, possibly of a different type."

## Shadowing chains for parsing

The canonical use: carry one concept through several representations without
inventing `raw`, `raw_trimmed`, `raw_parsed`.

```rust
let raw = " 42 ";
let raw = raw.trim();
let raw: i32 = raw.parse().unwrap();
let raw = raw * 2;
println!("{raw}");
```

Output:

```
84
```

A full type swap is also fine:

```rust
let shadowed = 5;
let shadowed = "now a string";
println!("{shadowed}");
```

Output:

```
warning: unused variable: `shadowed`
  --> src/main.rs:12:9
   |
12 |     let shadowed = 5;
   |         ^^^^^^^^ help: if this is intentional, prefix it with an underscore: `_shadowed`
```

```
now a string
```

Shadowing a value you never read warns, because that first binding is dead.

## Block scope

```rust
let outer = 1;
{
    let outer = 2;
    println!("inner {outer}");
}
println!("outer {outer}");
```

Output:

```
inner 2
outer 1
```

Inner shadowing ends with the block. Top-level shadowing in the same scope
lasts to the end of that scope.

## Blocks are expressions

A `{ ... }` block evaluates to its final expression — no semicolon on that last
line.

```rust
let v = {
    let t = 3;
    t * t
};
println!("{v}");
```

Output:

```
9
```

Useful for keeping a multi-step computation's temporaries out of the outer
scope. Lesson 2.4 covers the statement/expression rule in general.

## Underscore bindings

An unused variable warns:

```rust
let total = 10;
println!("done");
```

```
warning: unused variable: `total`
 --> src/main.rs:2:9
  |
2 |     let total = 10;
  |         ^^^^^ help: if this is intentional, prefix it with an underscore: `_total`
```

```rust
let _unused = 99;
let _ = 100;
```

Compiles silently. The two are not identical: `_name` is a real binding that
holds the value (and drops it at end of scope); bare `_` binds nothing and
drops the value immediately.

## Quick Reference

| Form | Meaning |
|---|---|
| `let x = 5;` | Immutable binding, type inferred |
| `let x: u8 = 5;` | Immutable, annotated |
| `let mut x = 5;` | Reassignable, same type only |
| `let x;` … `x = 5;` | Deferred init, still immutable |
| `let x = x + 1;` | Shadow with a new binding |
| `let x: T = x.into();` | Shadow with a type change |
| `let x = { … };` | Block expression as a value |
| `let _x = v;` | Suppress unused warning, keep the value |
| `let _ = v;` | Discard immediately, bind nothing |
| `x += 1` | Compound assignment (no `++`) |

## Common Mistakes

**Reaching for `mut` to change a type.** `mut` keeps the type fixed; shadow
with a new `let` instead.

**Annotating everything out of habit.** Inference reads the whole function;
annotate only on signatures, on ambiguity, and where it documents intent.

**Using deferred init where an `if` expression fits.** Prefer
`let x = if cond { a } else { b };` — it cannot leave `x` uninitialized.

**Assuming shadowing mutates.** The original binding is untouched; a reference
taken to it still sees the old value.

**Writing `x++`.** Not valid Rust. `x += 1`.

## Exercises

1. Declare `let x = 10;`, try to reassign it, read the error, then fix it two
   ways: with `mut` and with shadowing.
2. Write `let n = 2;` and make the compiler infer `n: u8` by passing it to a
   function that takes `u8` — without annotating the `let`.
3. Take `" -17 "` and produce the `i32` `-34` using a chain of three shadowed
   `let` bindings with the same name.
4. Declare a variable with no initializer, assign it in both arms of an `if`,
   then delete one arm and read the error.
5. Write a block expression that computes the sum of three temporaries and
   binds only the sum to an outer variable. Confirm the temporaries are not
   visible after the block.

---

[← 1.5 How Rust Thinks: A Mental Model](../01-getting-started/05-mental-model.md) · [Course Index](../README.md) · [2.2 Scalar Types: Integers, Floats, Bool, Char →](02-scalar-types.md)
