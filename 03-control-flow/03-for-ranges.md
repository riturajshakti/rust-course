[← Course Index](../README.md)

# 3.3 — for Loops and Ranges

## What You'll Learn

- Every range form: `a..b`, `a..=b`, `.rev()`, `.step_by()`
- `for x in v` vs `&v` vs `&mut v` — and exactly what each hands you
- `.iter()`, `.iter_mut()`, `.into_iter()` and when `for` picks them for you
- `enumerate`, `zip`, `chars`, nested loops, and iterating maps

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `for (const x of arr)` | `for x in &arr` |
| JS `arr.forEach((x, i) =>)` | `for (i, x) in arr.iter().enumerate()` |
| Python `for i in range(5)` | `for i in 0..5` |
| Go `for i := 0; i < 5; i++` | `for i in 0..5` |

There is no C-style `for (init; cond; step)`. Everything goes through
`IntoIterator`, so `for` and iterator adapters are the same machinery.

## Exclusive and inclusive ranges

`a..b` excludes `b`; `a..=b` includes it.

```rust
for i in 0..5 { print!("{i} "); }
println!();
for i in 0..=5 { print!("{i} "); }
println!();
```

Output:

```
0 1 2 3 4 
0 1 2 3 4 5 
```

A backwards range is **empty**, not an error and not a countdown:

```rust
for i in 5..1 { print!("{i} "); }
println!("<- empty");
```

Output:

```
<- empty
```

Use `_` when the index is unused, or Clippy complains about `i`:

```rust
for _ in 0..3 { print!("x"); }
println!();
```

Output:

```
xxx
```

## rev and step_by

```rust
for i in (0..5).rev() { print!("{i} "); }
println!();
for i in (0..20).step_by(5) { print!("{i} "); }
println!();
for i in (0..=20).step_by(5) { print!("{i} "); }
println!();
for i in (0..10).rev().step_by(3) { print!("{i} "); }
println!();
```

Output:

```
4 3 2 1 0 
0 5 10 15 
0 5 10 15 20 
9 6 3 0 
```

Parens are required: `0..5.rev()` parses as `0..(5.rev())`. Order matters —
`.rev().step_by(3)` steps from the reversed end.

`char` is ordered, so ranges work on it too:

```rust
for c in 'a'..='e' { print!("{c} "); }
println!();
```

Output:

```
a b c d e 
```

## A range is a value, and an iterator

`0..5` is a `Range<i32>` struct with `start`/`end` fields. You can bind it, pass
it, store it, and call iterator methods on it directly.

```rust
let r = 2..7;
println!("{r:?} start={} end={} len={}", r.start, r.end, r.len());
println!("contains 5: {}", (2..7).contains(&5));
println!("sum {}", (1..=10).sum::<i32>());
println!("count {}", (1..=10).count());
println!("max {:?}", (1..=10).max());
println!("collected {:?}", (1..5).collect::<Vec<i32>>());
```

Output:

```
2..7 start=2 end=7 len=5
contains 5: true
sum 55
count 10
max Some(10)
collected [1, 2, 3, 4]
```

`RangeInclusive` hides its bounds behind methods instead of fields:

```rust
let ri = 1..=3;
println!("{:?} {:?}", ri.start(), ri.end());
```

Output:

```
1 3
```

A range is an ordinary parameter type too: `fn in_range(r: std::ops::Range<i32>,
n: i32) -> bool { r.contains(&n) }`.

A `Range` is consumed by iteration, so iterating one you want to keep needs
`.clone()` (cloning a range is two integers — free):

```rust
let ranges = vec![0..3, 10..12];
for r in &ranges {
    for i in r.clone() { print!("{i} "); }
}
println!();
```

Output:

```
0 1 2 10 11 
```

Range types: `a..b` is `Range`, `a..=b` is `RangeInclusive`, `a..` is
`RangeFrom` (infinite — only use with `zip`/`take`), `..b` is `RangeTo` and
`..` is `RangeFull`. The last two are not iterators; they only index slices.

## Three ways to loop a Vec

This is the single most important table in the lesson.

| Form | Item type | Collection after |
|---|---|---|
| `for x in &v` | `&T` | still usable |
| `for x in &mut v` | `&mut T` | still usable, mutated |
| `for x in v` | `T` | **moved**, gone |

```rust
let v = vec![10, 20, 30];
for x in &v { print!("{x} "); }
println!("| v still usable, len {}", v.len());

let mut m = vec![1, 2, 3];
for x in &mut m { *x *= 10; }
println!("{m:?}");

for x in v { print!("{x} "); }
println!("| v moved");
```

Output:

```
10 20 30 | v still usable, len 3
[10, 20, 30]
10 20 30 | v moved
```

Note the `*x` in the `&mut` loop — the item is a `&mut i32`, so you dereference
to assign. Forgetting it:

```rust
for x in &mut m {
    x += 1;
}
```

Does not compile — intentional:

```
error[E0368]: binary assignment operation `+=` cannot be applied to type `&mut {integer}`
 --> src/main.rs:4:9
  |
4 |         x += 1;
  |         -^^^^^
  |         |
  |         cannot use `+=` on type `&mut {integer}`
  |
help: `+=` can be used on `{integer}` if you dereference the left-hand side
  |
4 |         *x += 1;
  |         +
```

And the by-value case, which is the error every newcomer hits:

```rust
let v = vec![String::from("a")];
for s in v {
    println!("{s}");
}
println!("{v:?}");
```

Does not compile — intentional:

```
error[E0382]: borrow of moved value: `v`
   --> src/main.rs:6:16
    |
  2 |     let v = vec![String::from("a")];
    |         - move occurs because `v` has type `Vec<String>`, which does not implement the `Copy` trait
  3 |     for s in v {
    |              - `v` moved due to this implicit call to `.into_iter()`
...
  6 |     println!("{v:?}");
    |                ^ value borrowed here after move
    |
note: the `for` loop is desugared into a call to `std::iter::IntoIterator::into_iter`, which takes ownership of the receiver `self`, which moves `v`
help: consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop
    |
  3 |     for s in &v {
    |              +
```

Idiomatic: default to `&v`. Use `v` only when you genuinely want to consume the
elements. Module 04 explains moves properly.

## Arrays behave differently

`[i32; 3]` is `Copy`, so a by-value loop does **not** consume it.

```rust
let a = [1, 2, 3];
for x in a { print!("{x} "); }
println!();
for x in &a { print!("{x} "); }
println!("| a is Copy, still usable: {a:?}");
```

Output:

```
1 2 3 
1 2 3 | a is Copy, still usable: [1, 2, 3]
```

An array of `String` would move, same as a `Vec`.

## iter, iter_mut, into_iter

`for` desugars to `IntoIterator::into_iter(expr)`. The three method forms are
the explicit spellings:

| Call | Item | Equivalent `for` |
|---|---|---|
| `v.iter()` | `&T` | `for x in &v` |
| `v.iter_mut()` | `&mut T` | `for x in &mut v` |
| `v.into_iter()` | `T` (consumes) | `for x in v` |

Printing `std::any::type_name` for each confirms it: `&i32`, `&i32`,
`&mut i32`, `i32`.

```rust
let names = vec![String::from("raj"), String::from("sam")];
for n in names.iter() { print!("{} ", n.len()); }
println!();
for n in names.into_iter() { print!("{} ", n.to_uppercase()); }
println!();
```

Output:

```
3 3 
RAJ SAM 
```

Use the method form when you are chaining adapters; use `&v` when you are just
looping. `(&v).into_iter()` is the same as `v.iter()`.

## enumerate

Gives `(index, item)` tuples. There is no index variable in a `for` otherwise.

```rust
let v = vec![10, 20, 30];
for (i, x) in v.iter().enumerate() { println!("{i}: {x}"); }
```

Output:

```
0: 10
1: 20
2: 30
```

`enumerate` always starts at 0, even after `skip` — the index is the position in
the *enumerated* stream, so put `skip` after it to preserve real positions:

```rust
for (i, x) in v.iter().enumerate().skip(1) { println!("skip {i}: {x}"); }
```

Output:

```
skip 1: 20
skip 2: 30
```

For a 1-based counter, `zip(1..)` is cleaner than `i + 1`.

## zip

Pairs two iterators and **stops at the shorter one**.

```rust
let names = ["raj", "sam", "ana"];
let ages = [30, 25];
for (n, a) in names.iter().zip(ages.iter()) { println!("{n} is {a}"); }
```

Output:

```
raj is 30
sam is 25
```

`ana` is dropped silently. Zipping against an open range gives a counter:

```rust
for (n, a) in names.iter().zip(1..) { println!("{a}. {n}"); }
```

Output:

```
1. raj
2. sam
3. ana
```

## Iterating strings

There is no `for c in s`. Pick what you want explicitly.

```rust
for c in "héllo".chars() { print!("[{c}]"); }
println!();
for b in "hé".bytes() { print!("{b} "); }
println!();
for (i, c) in "héllo".char_indices() { print!("{i}:{c} "); }
println!();
for w in "the quick fox".split_whitespace() { print!("<{w}>"); }
println!();
for line in "a\nb\nc".lines() { print!("{line}|"); }
println!();
```

Output:

```
[h][é][l][l][o]
104 195 169 
0:h 1:é 3:l 4:l 5:o 
<the><quick><fox>a|b|c|
```

Note the byte indices from `char_indices` skip from 1 to 3 — `é` is two bytes.
Module 06 covers UTF-8 in full.

## Destructuring in the loop pattern

The `for` binding is a full pattern, not just a name.

```rust
let pairs = vec![(1, "one"), (2, "two")];
for (n, name) in &pairs { println!("{n} -> {name}"); }
for &(n, name) in &pairs { println!("{n} = {name}"); }
```

Output:

```
1 -> one
2 -> two
1 = one
2 = two
```

Both work: the first binds `n: &i32`, the second destructures through the
reference so `n: i32`. Lesson 3.5 covers patterns in depth.

## Nested for

```rust
for i in 1..=3 {
    for j in 1..=3 {
        print!("{:2} ", i * j);
    }
    println!();
}
```

Output:

```
 1  2  3 
 2  4  6 
 3  6  9 
```

For a `Vec<Vec<T>>`, `&grid` gives `&Vec<T>` rows, which iterate directly:

```rust
let grid = vec![vec![1, 2], vec![3, 4]];
for row in &grid {
    for cell in row { print!("{cell} "); }
}
println!();
```

Output:

```
1 2 3 4 
```

Labels work on `for` exactly as in lesson 3.2 — `'outer: for i in 0..3 { for c
in "ab".chars() { if i == 2 { break 'outer; } } }`.

## Slices

`&[T]` iterates like a `Vec`, and a slice of a `Vec` avoids copying.

```rust
fn sum_slice(xs: &[i32]) -> i32 {
    let mut t = 0;
    for x in xs { t += x; }
    t
}

let v = vec![1, 2, 3, 4, 5];
println!("{}", sum_slice(&v));
println!("{}", sum_slice(&v[1..3]));
for x in &v[..2] { print!("{x} "); }
println!();
```

Output:

```
15
5
1 2 
```

`t += x` works with `x: &i32` because `AddAssign<&i32>` exists for `i32`.

`chunks` and `windows` give you groups instead of single elements:

```rust
let v = vec![1, 2, 3, 4];
for chunk in v.chunks(2) { print!("{chunk:?} "); }
for w in v.windows(2) { print!("{w:?} "); }
println!();
```

Output:

```
[1, 2] [3, 4] [1, 2] [2, 3] [3, 4] 
```

## Iterating a map

`&map` yields `(&K, &V)` tuples. `HashMap` order is arbitrary and changes
between runs; `BTreeMap` is sorted by key.

```rust
use std::collections::BTreeMap;

let mut bt = BTreeMap::new();
bt.insert("a", 1);
bt.insert("b", 2);
for (k, v) in &bt { print!("{k}={v} "); }
println!();
for k in bt.keys() { print!("{k} "); }
println!();
for v in bt.values() { print!("{v} "); }
println!();
```

Output:

```
a=1 b=2 
a b 
1 2 
```

`HashMap` has the same `keys()`/`values()`/`values_mut()`/`iter_mut()` API; only
the ordering guarantee differs (Module 07).

## Adapters in the for header

Any iterator expression works, so filtering and mapping happen before the body.

```rust
let mut sum = 0;
for i in (1..=100).filter(|n| n % 3 == 0) { sum += i; }
println!("{sum}");

let total: i32 = (1..=5).map(|x| x * x).sum();
println!("{total}");
```

Output:

```
1683
55
```

Once the body is only an accumulation, the loop usually disappears entirely, as
in that second line.

Module 10 is the full adapter and consumer catalogue (`map`, `filter`, `fold`,
`take_while`, `flat_map`, `chain`, `peekable`, and the rest). This lesson covers
only the handful you need to write `for` loops.

## Quick Reference

| Form | Yields |
|---|---|
| `for i in 0..n` | `0` to `n-1` |
| `for i in 0..=n` | `0` to `n` |
| `for i in (0..n).rev()` | descending |
| `for i in (0..n).step_by(k)` | every k-th |
| `for c in 'a'..='z'` | chars |
| `for x in &v` | `&T`, `v` kept |
| `for x in &mut v` | `&mut T`, deref to assign |
| `for x in v` | `T`, `v` moved |
| `v.iter()` / `.iter_mut()` / `.into_iter()` | `&T` / `&mut T` / `T` |
| `for (i, x) in v.iter().enumerate()` | `(usize, &T)` |
| `for (a, b) in x.iter().zip(y.iter())` | stops at shorter |
| `for c in s.chars()` | `char` |
| `for (i, c) in s.char_indices()` | byte index + char |
| `for b in s.bytes()` | `u8` |
| `for w in s.split_whitespace()` | `&str` |
| `for (k, v) in &map` | `(&K, &V)` |
| `for c in v.chunks(n)` / `.windows(n)` | `&[T]` groups |
| `'l: for ..` + `break 'l` | labeled exit |

## Common Mistakes

**`for x in v` then using `v`.** The loop moved it (E0382). Use `&v`.

**Forgetting `*` in a `&mut` loop.** `x += 1` on a `&mut i32` is E0368. Write
`*x += 1`.

**`for i in 5..1` expecting a countdown.** Empty range. Use `(1..=5).rev()`.

**`0..5.rev()`.** Parses as `0..(5.rev())`. Wrap the range: `(0..5).rev()`.

**Mutating a collection while looping it.** `for x in &v { v.push(..) }` is a
borrow error, not a runtime surprise. Collect the changes and apply after, or
use indices.

**Relying on `HashMap` iteration order.** It is arbitrary and varies per run.
Use `BTreeMap`, or collect and sort.

## Exercises

1. Print the multiplication table for 1-12 as a grid, with `{:4}` alignment.
2. Print `0..=100` in steps of 7, descending, using one `for` header.
3. Given `let mut v = vec![1, 2, 3, 4, 5]`, double every element in place, then
   print the index and value of each odd element using `enumerate`.
4. Zip `["Mon","Tue","Wed"]` with `[12.5, 9.0]` and print each pair. Explain in
   one line what happened to `"Wed"`.
5. Count the vowels in `"programming in rust"` with a `for` over `.chars()`,
   then rewrite it as a single iterator chain with `.filter().count()`.
6. Write `fn longest(words: &[&str]) -> &str` using a `for` over the slice.

---

[← Course Index](../README.md)
