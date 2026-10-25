[← 2.2 Scalar Types: Integers, Floats, Bool, Char](02-scalar-types.md) · [Course Index](../README.md) · [2.4 Functions, Parameters, and Return Values →](04-functions.md)

# 2.3 — Tuples and Arrays

## What You'll Learn

- Tuples: `.0` access, destructuring, nesting, and the unit type `()`
- Arrays are fixed-length, and the length is part of the type
- Out of bounds: caught at compile time for a literal index, panics otherwise
- Array vs slice vs `Vec`, and which to put in a function signature

## Coming From Another Language

| Other | Rust |
|---|---|
| Python `t = (1, "a")` | `let t = (1, "a");` |
| JS array `[1, 2, 3]` (growable) | `Vec<i32>` (module 07), not `[i32; 3]` |
| Go `[3]int` / `[]int` | `[i32; 3]` / `&[i32]` |
| TS tuple `[number, string]` | `(i32, &str)` |

Rust's `[T; N]` is Go's array, not JS's. It cannot grow, lives on the stack,
and `[i32; 3]` and `[i32; 4]` are **different types**. The growable one is
`Vec<T>`.

## Tuples

```rust
fn main() {
    let tup: (i32, f64, char) = (500, 6.4, 'x');
    println!("{} {} {}", tup.0, tup.1, tup.2);
    println!("{:?}", tup);
}
```

Output:

```
500 6.4 x
(500, 6.4, 'x')
```

Mixed types, fixed length. `.0` indices are literals only — `tup.i` with a
variable does not exist, because each element has its own type.

## Destructuring

```rust
let tup = (500, 6.4, 'x');
let (a, b, c) = tup;
println!("{a} {b} {c}");

let (x, _, _) = tup;
println!("{x}");
```

Output:

```
500 6.4 x
500
```

`_` skips an element. This is pattern matching; module 03 goes further.

## Mutating and swapping

```rust
let mut m = (1, 2);
m.0 = 10;
println!("{:?}", m);

let mut sw = (1, 2);
sw = (sw.1, sw.0);
println!("{:?}", sw);
```

Output:

```
(10, 2)
(2, 1)
```

The second form is the idiomatic swap for two locals — no temporary needed.

## One-element tuples need a trailing comma

```rust
let single = (5,);
println!("{:?} {}", single, single.0);
```

Output:

```
(5,) 5
```

`(5)` is just `5` in parentheses. The comma is what makes it a tuple.

## Nested tuples

```rust
let nested = ((1, 2), (3, (4, 5)));
println!("{} {} {:?}", nested.0.1, nested.1.1.0, nested);
let ((p, q), (r, (s, t))) = nested;
println!("{p}{q}{r}{s}{t}");
```

Output:

```
2 4 ((1, 2), (3, (4, 5)))
12345
```

`.0.1` chains; destructuring nests to match.

## Tuples compare lexicographically

```rust
let pair = (1, 2);
println!("{} {}", pair == (1, 2), pair < (1, 3));
```

Output:

```
true true
```

`==` and ordering are derived element-wise, left to right, as long as every
element type supports them. This makes tuples the standard sort key.

## The unit type

```rust
let unit = ();
println!("{:?} {}", unit, size_of::<()>());
```

Output:

```
() 0
```

`()` is both the type and its only value, and it occupies zero bytes. It is
what a function with no return type returns, and what a `println!` statement
evaluates to.

## Tuples as return values

The main reason tuples exist: returning two things without declaring a struct.

```rust
fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divmod(17, 5);
println!("{q} {r}");
```

Output:

```
3 2
```

Beyond three elements, name the fields with a struct — `.3` and `.4` are
unreadable. Tuples go up to 12 elements with full trait support:

```rust
let big = (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12);
println!("{:?}", big);
```

Output:

```
(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12)
```

## Tuple layout

```rust
println!("{} {} {}", size_of::<(u8, u8)>(), size_of::<(u8, u32)>(), size_of::<(i32, f64, char)>());
```

Output:

```
2 8 16
```

Padding and field order are the compiler's business, same as structs.

## Arrays

```rust
let a = [1, 2, 3, 4, 5];
let b: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0; 5];
let months = ["Jan", "Feb", "Mar"];
println!("{:?} {:?} {:?} {:?}", a, b, zeros, months);
println!("{} {} {}", a[0], a[4], a.len());
println!("{} {}", a.is_empty(), months.len());
```

Output:

```
[1, 2, 3, 4, 5] [1, 2, 3, 4, 5] [0, 0, 0, 0, 0] ["Jan", "Feb", "Mar"]
1 5 5
false 3
```

`[0; 5]` is the repeat form: value, semicolon, count. The count must be a
compile-time constant.

```rust
let n = 5;
let a = [0; n];
```

Does not compile:

```
error[E0435]: attempt to use a non-constant value in a constant
 --> src/main.rs:3:17
  |
3 |     let a = [0; n];
  |                 ^ non-constant value
  |
help: consider using `const` instead of `let`
  |
2 -     let n = 5;
2 +     const n: /* Type */ = 5;
  |
```

Length mismatches are type errors:

```rust
let a: [i32; 5] = [1, 2, 3];
```

```
error[E0308]: mismatched types
 --> src/main.rs:2:23
  |
2 |     let a: [i32; 5] = [1, 2, 3];
  |            --------   ^^^^^^^^^ expected an array with a size of 5, found one with a size of 3
```

An empty array needs an annotation, since nothing constrains `T`:

```rust
let empty: [i32; 0] = [];
println!("{:?} {}", empty, empty.len());
```

Output:

```
[] 0
```

## Mutation and safe access

```rust
let mut m = [0; 3];
m[1] = 42;
println!("{:?}", m);

let a = [1, 2, 3, 4, 5];
println!("{:?} {:?}", a.first(), a.get(10));
println!("{:?} {:?}", a.last(), a.get(2));
```

Output:

```
[0, 42, 0]
Some(1) None
Some(5) Some(3)
```

`a[i]` panics out of range; `a.get(i)` returns `Option`. Use `get` whenever the
index comes from outside the function.

## Out of bounds

With a literal index, constant propagation catches it at compile time:

```rust
let a = [1, 2, 3, 4, 5];
println!("{}", a[10]);
```

Does not compile:

```
error: this operation will panic at runtime
 --> src/main.rs:3:20
  |
3 |     println!("{}", a[10]);
  |                    ^^^^^ index out of bounds: the length is 5 but the index is 10
  |
  = note: `#[deny(unconditional_panic)]` on by default
```

Hide the index behind a function and it becomes a runtime panic instead — the
bounds check is always there, it is just the *diagnostic* that depends on what
the compiler can see.

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let i = 10;
    println!("{}", a[index(i)]);
}
fn index(i: usize) -> usize { i }
```

Output:

```
thread 'main' (13790976) panicked at src/main.rs:4:20:
index out of bounds: the len is 5 but the index is 10
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Indices must be `usize`, not any integer:

```rust
let a = [1, 2, 3];
let i: i32 = 1;
println!("{}", a[i]);
```

```
error[E0277]: the type `[{integer}]` cannot be indexed by `i32`
   --> src/main.rs:4:22
    |
  4 |     println!("{}", a[i]);
    |                      ^ slice indices are of type `usize` or ranges of `usize`
```

## Iteration

```rust
let a = [1, 2, 3, 4, 5];
for v in a { print!("{v} "); }
println!();
for v in &a { print!("{v} "); }
println!();
for (i, v) in a.iter().enumerate() { print!("{i}:{v} "); }
println!();
```

Output:

```
1 2 3 4 5 
1 2 3 4 5 
0:1 1:2 2:3 3:4 4:5 
```

`for v in a` consumes the array by value (fine here — `i32` is `Copy`);
`for v in &a` borrows and yields `&i32`. No manual index needed, so no bounds
check in the loop.

## Array methods

```rust
let a = [1, 2, 3, 4, 5];
println!("{} {:?} {}", a.iter().sum::<i32>(), a.iter().max(), a.contains(&3));

let mut s = [5, 3, 1, 4];
s.sort();
println!("{:?}", s);
s.reverse();
println!("{:?}", s);
println!("{}", [1, 2, 3] == [1, 2, 3]);
```

Output:

```
15 Some(5) true
[1, 3, 4, 5]
[5, 4, 3, 1]
true
```

Arrays get all the slice methods for free (via deref), which is most of what
you would want. Module 07 lists them with `Vec`.

## 2D arrays

```rust
let grid = [[1, 2, 3], [4, 5, 6]];
println!("{:?} {} {}", grid, grid[1][2], grid.len() * grid[0].len());
for row in &grid {
    for cell in row { print!("{cell} "); }
    println!();
}
```

Output:

```
[[1, 2, 3], [4, 5, 6]] 6 6
1 2 3 
4 5 6 
```

The type is `[[i32; 3]; 2]` — read it outside-in: 2 rows of 3. It is one flat
contiguous block, not an array of pointers.

## Slices

A slice `&[T]` is a borrowed view: pointer + length, no ownership, no fixed `N`.

```rust
let a = [1, 2, 3, 4, 5];
let slice = &a[1..4];
println!("{:?} {} {:?} {:?}", slice, slice.len(), &a[..2], &a[3..]);
println!("{} {}", size_of::<[i32; 5]>(), size_of::<&[i32]>());
```

Output:

```
[2, 3, 4] 3 [1, 2] [4, 5]
20 16
```

Ranges are half-open: `1..4` is indices 1, 2, 3. The slice itself is 16 bytes
(pointer + `usize` length) regardless of how much it points at. Lesson 4.5
covers slices and their lifetimes properly.

## Array vs slice vs Vec

| | `[T; N]` | `&[T]` | `Vec<T>` |
|---|---|---|---|
| Length | fixed, in the type | runtime, in the value | runtime, growable |
| Storage | inline (stack if local) | borrowed | heap |
| Owns data | yes | no | yes |
| Can push | no | no | yes |

Take `&[T]` in parameters: it accepts arrays, `Vec`s, and sub-ranges of either.

```rust
fn total(xs: &[i32]) -> i32 {
    xs.iter().sum()
}

let a = [1, 2, 3, 4, 5];
let v = vec![10, 20, 30];
println!("{}", total(&a));
println!("{}", total(&v));
println!("{}", total(&a[1..3]));
```

Output:

```
15
60
5
```

`&Vec<i32>` as a parameter type would reject the first and third call for no
benefit. Module 07 covers `Vec`.

## Quick Reference

| Form | Meaning |
|---|---|
| `(a, b)` | Tuple literal |
| `(a,)` | One-element tuple |
| `()` | Unit value and unit type |
| `t.0` | Tuple element by literal index |
| `let (a, b) = t;` | Destructure |
| `[1, 2, 3]` | Array literal |
| `[v; N]` | Array of `N` copies of `v` |
| `[T; N]` | Array type |
| `a[i]` | Index, panics out of range |
| `a.get(i)` | `Option<&T>` |
| `a.first()` / `a.last()` | `Option<&T>` |
| `a.len()` / `a.is_empty()` | Size queries |
| `&a[1..4]` | Slice, half-open range |
| `&a[..]` | Whole array as a slice |
| `&[T]` | Slice type — use in parameters |
| `[[T; C]; R]` | 2D array, `R` rows of `C` |

## Common Mistakes

**Expecting arrays to grow.** `[T; N]` has no `push`. Use `Vec<T>`.

**Indexing a tuple with a variable.** `t.0` is a literal field, not an index
operation; each element can have a different type.

**Writing `(5)` and expecting a tuple.** Needs the trailing comma: `(5,)`.

**Taking `&Vec<T>` in a signature.** Take `&[T]` — it accepts more callers and
costs nothing.

**Indexing with an `i32`.** Indices are `usize`; cast or use `usize` throughout.

## Exercises

1. Write `fn min_max(xs: &[i32]) -> (i32, i32)` and call it with an array, a
   slice of that array, and a `vec!`.
2. Build `[[u8; 4]; 3]`, fill it so each cell holds `row * 4 + col`, and print
   it as a grid.
3. Index an array out of bounds with a literal, read the compile error, then
   move the index into a variable passed through a function and read the panic.
4. Destructure `((1, 'a'), (2.5, true))` into four named bindings in one `let`.
5. Sort a `[(String, u32); 3]` of names and scores by score descending, using
   the tuple's own ordering where you can.

---

[← 2.2 Scalar Types: Integers, Floats, Bool, Char](02-scalar-types.md) · [Course Index](../README.md) · [2.4 Functions, Parameters, and Return Values →](04-functions.md)
