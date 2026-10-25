[← Course Index](../README.md)

# 10.7 — Writing Your Own Iterator

## What You'll Learn

- `impl Iterator` for your own struct: write `next`, get 70+ methods free
- `IntoIterator` for your own collection, all three forms (`T`, `&T`, `&mut T`)
- Returning `impl Iterator` and writing generic adapters
- `DoubleEndedIterator`, `ExactSizeIterator`, and iterating a tree

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `*[Symbol.iterator]()` generator | `impl Iterator` struct, or `iter::from_fn` |
| Python `yield` | explicit state in a struct |
| Go `for ... range` over a func | `impl Iterator` |

There are no generators in stable Rust, so a custom iterator is a struct that
holds its own position. That sounds heavier than `yield` — but in return you
get every adapter and consumer from lessons 10.4 to 10.6 for free, and the
whole thing optimizes away.

## Counter: the minimum

One associated type, one method.

```rust
struct Counter { n: u32 }

impl Counter {
    fn new() -> Self { Counter { n: 0 } }
}

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.n < 5 { self.n += 1; Some(self.n) } else { None }
    }
}

fn main() {
    println!("{:?}", Counter::new().collect::<Vec<u32>>());
    let mut c = Counter::new();
    println!("{:?} {:?}", c.next(), c.next());
    println!("{}", Counter::new().sum::<u32>());
}
```

Output:

```
[1, 2, 3, 4, 5]
Some(1) Some(2)
15
```

## What you get free

Nothing below was written for `Counter`. All of it falls out of `next`.

```rust
let total: u32 = Counter::new()
    .zip(Counter::new().skip(1))
    .map(|(a, b)| a * b)
    .filter(|x| x % 3 == 0)
    .sum();
println!("{total}");
```

Output:

```
18
```

The exception is methods needing extra traits:

```rust
println!("{:?}", Counter::new().rev().collect::<Vec<u32>>());
```

Does not compile:

```
error[E0277]: the trait bound `Counter: DoubleEndedIterator` is not satisfied
    --> src/main.rs:29:37
     |
  29 |     println!("{:?}", Counter::new().rev().collect::<Vec<u32>>());
     |                                     ^^^ unsatisfied trait bound
     |
help: the trait `DoubleEndedIterator` is not implemented for `Counter`
note: required by a bound in `rev`
     |
3441 |         Self: Sized + DoubleEndedIterator,
     |                       ^^^^^^^^^^^^^^^^^^^ required by this bound in `Iterator::rev`
```

## Fibonacci: an infinite iterator

An iterator need never return `None`. Here it does, but only when `u64`
overflows — the `?` on `checked_add` ends it cleanly.

```rust
struct Fib { a: u64, b: u64 }

impl Fib {
    fn new() -> Self { Fib { a: 0, b: 1 } }
}

impl Iterator for Fib {
    type Item = u64;
    fn next(&mut self) -> Option<u64> {
        let out = self.a;
        let next = self.a.checked_add(self.b)?;
        self.a = self.b;
        self.b = next;
        Some(out)
    }
}

fn main() {
    println!("{:?}", Fib::new().take(10).collect::<Vec<u64>>());
    println!("{}", Fib::new().count());
    println!("{:?}", Fib::new().take_while(|x| *x < 100).filter(|x| x % 2 == 0).collect::<Vec<u64>>());
    println!("{:?}", Fib::new().nth(20));
    println!("{:?}", Fib::new().position(|x| x > 1000));
}
```

Output:

```
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
92
[0, 2, 8, 34]
Some(6765)
Some(17)
```

`count()` terminating at 92 is the overflow guard working. Had `next` used
plain `+`, `count()` would panic in debug and `take(10)` would still be fine —
laziness means an infinite iterator is only a problem for an eager consumer.

## Implementing IntoIterator for your collection

Three impls, matching the three `Vec` forms. Each one delegates to the inner
`Vec`'s iterator, so `IntoIter` is just the right std type.

```rust
#[derive(Debug)]
struct Stack<T> { items: Vec<T> }

impl<T> Stack<T> {
    fn new() -> Self { Stack { items: Vec::new() } }
    fn push(&mut self, v: T) { self.items.push(v); }
    fn iter(&self) -> std::slice::Iter<'_, T> { self.items.iter() }
}

impl<T> IntoIterator for Stack<T> {
    type Item = T;
    type IntoIter = std::vec::IntoIter<T>;
    fn into_iter(self) -> Self::IntoIter { self.items.into_iter() }
}

impl<'a, T> IntoIterator for &'a Stack<T> {
    type Item = &'a T;
    type IntoIter = std::slice::Iter<'a, T>;
    fn into_iter(self) -> Self::IntoIter { self.items.iter() }
}

impl<'a, T> IntoIterator for &'a mut Stack<T> {
    type Item = &'a mut T;
    type IntoIter = std::slice::IterMut<'a, T>;
    fn into_iter(self) -> Self::IntoIter { self.items.iter_mut() }
}
```

Now all three `for` forms work, exactly as on a `Vec`:

```rust
let mut s = Stack::new();
s.push(1); s.push(2); s.push(3);

for x in &s { print!("{x} "); }
println!();
for x in &mut s { *x *= 10; }
println!("{:?}", s);
println!("{}", s.iter().sum::<i32>());
for x in s { print!("{x} "); }
println!();
```

Output:

```
1 2 3 
Stack { items: [10, 20, 30] }
60
10 20 30
```

Convention: also provide inherent `iter()` and `iter_mut()` methods. They are
what adapter chains need, since `&s.iter()` reads worse than `s.iter()`, and
every std collection offers them.

## Returning impl Iterator from a function

The cleanest way to hand out an iterator without naming its type.

```rust
fn evens(limit: u32) -> impl Iterator<Item = u32> {
    (0..limit).filter(|x| x % 2 == 0)
}

fn words_of(s: &str) -> impl Iterator<Item = &str> {
    s.split_whitespace().filter(|w| w.len() > 2)
}

fn main() {
    println!("{:?}", evens(10).collect::<Vec<u32>>());
    println!("{:?}", words_of("a bb ccc dddd").collect::<Vec<&str>>());
}
```

Output:

```
[0, 2, 4, 6, 8]
["ccc", "dddd"]
```

In `words_of`, the returned iterator borrows `s`, and the lifetime is inferred
from the single input reference. Borrow a **local** instead and it fails:

```rust
fn lines_of(path: &str) -> impl Iterator<Item = &str> {
    let text = String::from("a\nb");
    text.lines()
}
```

Does not compile:

```
error[E0515]: cannot return value referencing local variable `text`
 --> src/main.rs:3:5
  |
3 |     text.lines()
  |     ----^^^^^^^^
  |     |
  |     returns a value referencing data owned by the current function
  |     `text` is borrowed here
  |
  = help: use `.collect()` to allocate the iterator
```

Take the hint and return `Vec<String>`, or have the caller own the `String` and
pass a `&str` in.

As with closures, branching needs a box:

```rust
fn boxed(limit: u32, odd: bool) -> Box<dyn Iterator<Item = u32>> {
    if odd { Box::new((0..limit).filter(|x| x % 2 == 1)) }
    else { Box::new((0..limit).filter(|x| x % 2 == 0)) }
}

println!("{:?}", boxed(6, true).collect::<Vec<u32>>());
```

Output:

```
[1, 3, 5]
```

## Generic adapters over any iterator

Three useful bound shapes.

```rust
fn sum_doubled<I: IntoIterator<Item = i32>>(it: I) -> i32 {
    it.into_iter().map(|x| x * 2).sum()
}

fn every_other<I: Iterator>(it: I) -> impl Iterator<Item = I::Item> {
    it.enumerate().filter(|(i, _)| i % 2 == 0).map(|(_, x)| x)
}

fn top_n<I>(it: I, n: usize) -> Vec<I::Item>
where
    I: Iterator,
    I::Item: Ord,
{
    let mut v: Vec<I::Item> = it.collect();
    v.sort_unstable_by(|a, b| b.cmp(a));
    v.truncate(n);
    v
}

fn main() {
    println!("{}", sum_doubled(vec![1, 2, 3]));
    println!("{}", sum_doubled(1..4));
    println!("{:?}", every_other("a b c d e".split(' ')).collect::<Vec<&str>>());
    println!("{:?}", top_n(vec![5, 1, 9, 3].into_iter(), 2));
    println!("{:?}", top_n("pear fig apple".split(' '), 2));
}
```

Output:

```
12
12
["a", "c", "e"]
[9, 5]
["pear", "fig"]
```

Take `IntoIterator` in public APIs — it accepts `Vec<T>`, `&Vec<T>`, arrays,
ranges, and iterators, so callers never have to write `.iter()`. Take
`Iterator` when the function itself must be lazy and return an adapter.

## DoubleEndedIterator and ExactSizeIterator

Implement these and `rev`, `rfind`, `rfold`, `rposition`, and `len` light up.
Also override `size_hint` so `collect` preallocates correctly.

```rust
struct Range2 { lo: u32, hi: u32 }

impl Iterator for Range2 {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.lo < self.hi { let v = self.lo; self.lo += 1; Some(v) } else { None }
    }
    fn size_hint(&self) -> (usize, Option<usize>) {
        let n = (self.hi - self.lo) as usize;
        (n, Some(n))
    }
}

impl DoubleEndedIterator for Range2 {
    fn next_back(&mut self) -> Option<u32> {
        if self.lo < self.hi { self.hi -= 1; Some(self.hi) } else { None }
    }
}

impl ExactSizeIterator for Range2 {}
```

`ExactSizeIterator` has an empty body — its `len` is derived from `size_hint`,
which must be exact. That is the whole contract.

```rust
let r = || Range2 { lo: 0, hi: 5 };
println!("{:?}", r().collect::<Vec<u32>>());
println!("{:?}", r().rev().collect::<Vec<u32>>());
println!("{}", r().len());
println!("{:?}", r().rposition(|x| x % 2 == 0));
println!("{:?}", r().rfind(|x| *x < 3));

let mut it = r();
println!("{:?} {:?} {:?}", it.next(), it.next_back(), it.len());

let v: Vec<u32> = r().collect();
println!("{}", v.capacity());
```

Output:

```
[0, 1, 2, 3, 4]
[4, 3, 2, 1, 0]
5
Some(4)
Some(2)
Some(0) Some(4) 3
5
```

`next` and `next_back` walk toward each other from opposite ends and meet in
the middle — that is why `len` is 3 after one of each. And the collected `Vec`
has capacity exactly 5, no reallocation, because `size_hint` was honest.

The two invariants you must not break: `size_hint` upper bound must be correct
or `collect` will misbehave, and `next`/`next_back` must never yield the same
element twice.

## Iterating a recursive structure

A tree has no natural "next", so the iterator carries an explicit work stack.

```rust
struct Node { value: i32, children: Vec<Node> }

struct DepthFirst<'a> { stack: Vec<&'a Node> }

impl<'a> Iterator for DepthFirst<'a> {
    type Item = i32;
    fn next(&mut self) -> Option<i32> {
        let node = self.stack.pop()?;
        for child in node.children.iter().rev() { self.stack.push(child); }
        Some(node.value)
    }
}

impl Node {
    fn leaf(v: i32) -> Node { Node { value: v, children: vec![] } }
    fn depth_first(&self) -> DepthFirst<'_> { DepthFirst { stack: vec![self] } }
}

fn main() {
    let tree = Node {
        value: 1,
        children: vec![
            Node { value: 2, children: vec![Node::leaf(4), Node::leaf(5)] },
            Node::leaf(3),
        ],
    };
    println!("{:?}", tree.depth_first().collect::<Vec<i32>>());
    println!("{}", tree.depth_first().sum::<i32>());
    println!("{:?}", tree.depth_first().filter(|x| x % 2 == 0).max());
    println!("{}", tree.depth_first().count());
}
```

Output:

```
[1, 2, 4, 5, 3]
15
Some(4)
5
```

`.rev()` on the children is what makes it left-to-right, since a stack pops
last-in first. Swap `pop()` for `remove(0)` (a `VecDeque::pop_front` in
practice) and you get breadth-first. Note the iterator never recurses, so deep
trees cannot blow the stack — a real advantage over a recursive `visit`
function.

## Skipping the struct entirely

For a one-off, `iter::from_fn` or `iter::successors` replaces the whole impl.

```rust
use std::iter;

let mut state = (0u64, 1u64);
let fib: Vec<u64> = iter::from_fn(|| {
    let out = state.0;
    state = (state.1, state.0 + state.1);
    Some(out)
}).take(8).collect();
println!("{fib:?}");

let pow: Vec<u32> = iter::successors(Some(1u32), |x| x.checked_mul(3)).take(5).collect();
println!("{pow:?}");
```

Output:

```
[0, 1, 1, 2, 3, 5, 8, 13]
[1, 3, 9, 27, 81]
```

Write the struct when the iterator is part of your public API (named type,
documented, extra traits); use `from_fn` when it is an implementation detail.

## Quick Reference

| Goal | Write |
|---|---|
| New iterator type | `impl Iterator for S { type Item = T; fn next(&mut self) -> Option<T> }` |
| Make a collection `for`-able by value | `impl IntoIterator for S` |
| …by reference | `impl<'a> IntoIterator for &'a S` with `Item = &'a T` |
| …by mutable reference | `impl<'a> IntoIterator for &'a mut S` with `Item = &'a mut T` |
| Conventional accessors | inherent `fn iter(&self)` and `fn iter_mut(&mut self)` |
| Return an iterator | `-> impl Iterator<Item = T>` |
| Return one of several | `-> Box<dyn Iterator<Item = T>>` |
| Accept anything iterable | `<I: IntoIterator<Item = T>>` |
| Accept and adapt lazily | `<I: Iterator>` → `-> impl Iterator<Item = I::Item>` |
| Enable `rev`, `rfind`, `rfold` | `impl DoubleEndedIterator` with `next_back` |
| Enable `len`, `rposition` | exact `size_hint` + `impl ExactSizeIterator` |
| Preallocation in `collect` | override `fn size_hint` |
| Guarantee `None` is final | `impl FusedIterator` |
| Be a `collect` target | `impl FromIterator<T> for S` |
| One-off, no named type | `iter::from_fn` or `iter::successors` |

## Common Mistakes

**Forgetting `type Item`.** The compiler reports a missing associated type, not
a missing method.

**Returning `impl Iterator` that borrows a local.** E0515; collect to a `Vec`
or let the caller own the data.

**Implementing only `IntoIterator for S`.** Then `for x in &s` does not
compile. Write all three, plus inherent `iter`/`iter_mut`.

**Taking `I: Iterator` in a public API.** Callers must write `.iter()`
themselves; take `I: IntoIterator` instead.

**An `ExactSizeIterator` whose `size_hint` is the default.** The default upper
bound is `None`, so `len` will be wrong; override `size_hint` first.

## Exercises

1. Write `struct Countdown { from: u32 }` yielding `n, n-1, ..., 1`. Verify
   `collect`, `sum`, and `rev` (adding `DoubleEndedIterator` for the last).
2. Write a `Primes` iterator (trial division is fine) and print the first 10,
   then the first prime above 1000, with no `break`.
3. Build `struct Matrix { rows: usize, cols: usize, data: Vec<f64> }` with
   `iter()` over all cells and `row(i)` returning `impl Iterator<Item = &f64>`.
4. Implement all three `IntoIterator` forms for a `struct Ring<T>` wrapping a
   `VecDeque<T>`, then write all three `for` loops over it.
5. Write `fn interleave<I, J>(a: I, b: J) -> impl Iterator<Item = I::Item>`
   where both are `Iterator` with the same `Item`, alternating items.
6. Rewrite the `DepthFirst` tree iterator as breadth-first using a `VecDeque`,
   and confirm the order is `[1, 2, 3, 4, 5]`.

---

[← Course Index](../README.md)
