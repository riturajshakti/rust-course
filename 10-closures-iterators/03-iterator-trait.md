[← Course Index](../README.md)

# 10.3 — The Iterator Trait

## What You'll Learn

- `Iterator` has exactly one required method, `next()`
- `iter` vs `iter_mut` vs `into_iter` and the item type each yields
- `IntoIterator`, and what a `for` loop actually desugars to
- Laziness: an adapter chain with no consumer does nothing

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `[Symbol.iterator]`, `{value, done}` | `Iterator::next() -> Option<Item>` |
| Python `__iter__` / `__next__` + `StopIteration` | `next()` returning `None` |
| Go manual `for i := range` | `for x in collection` |

Two differences that matter. There is no exception to signal the end — `next`
returns `Option`. And the item type is part of the trait, so a `Vec<String>`
can be iterated three ways yielding `&String`, `&mut String`, or `String`.

## The trait

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // ~75 provided methods, all built on next()
}
```

That is the whole contract. Everything in lessons 10.4 and 10.5 — `map`,
`filter`, `sum`, `collect` — is a default method written in terms of `next`.

## Calling next() by hand

```rust
let v = vec![10, 20, 30];
let mut it = v.iter();
println!("{:?}", it.next());
println!("{:?}", it.next());
println!("{:?}", it.next());
println!("{:?}", it.next());
println!("{:?}", it.next());
```

Output:

```
Some(10)
Some(20)
Some(30)
None
None
```

`next` takes `&mut self`, so the iterator binding must be `mut`. Once exhausted
it keeps returning `None` for a `Vec` iterator, though that is not guaranteed in
general — `.fuse()` (lesson 10.6) makes it guaranteed.

## iter vs iter_mut vs into_iter

| Method | Receiver | Item for `Vec<T>` | Collection after |
|---|---|---|---|
| `.iter()` | `&self` | `&T` | still usable |
| `.iter_mut()` | `&mut self` | `&mut T` | still usable, elements may have changed |
| `.into_iter()` | `self` | `T` | consumed |

```rust
let v = vec![String::from("a"), String::from("b")];
for s in v.iter() { println!("ref {s}"); }

let mut v2 = vec![1, 2, 3];
for n in v2.iter_mut() { *n *= 10; }
println!("{v2:?}");

for s in v.into_iter() { println!("owned {s}"); }
```

Output:

```
ref a
ref b
[10, 20, 30]
owned a
owned b
```

`iter_mut` yields `&mut T`, so you dereference to assign: `*n *= 10`.

## into_iter consumes

```rust
let v = vec![String::from("a")];
for s in v { println!("{s}"); }
println!("{:?}", v);
```

Does not compile:

```
error[E0382]: borrow of moved value: `v`
   --> src/main.rs:4:22
    |
  2 |     let v = vec![String::from("a")];
    |         - move occurs because `v` has type `Vec<String>`, which does not implement the `Copy` trait
  3 |     for s in v { println!("{s}"); }
    |              - `v` moved due to this implicit call to `.into_iter()`
  4 |     println!("{:?}", v);
    |                      ^ value borrowed here after move
    |
note: the `for` loop is desugared into a call to `std::iter::IntoIterator::into_iter`, which takes ownership of the receiver `self`, which moves `v`
```

The note spells out the desugaring. `for x in v` moves; `for x in &v` does not.

## IntoIterator and the for loop

`for` does not use `Iterator` directly — it calls `IntoIterator::into_iter`.

```rust
for x in expr { body }
```

becomes, roughly:

```rust
let mut it = IntoIterator::into_iter(expr);
while let Some(x) = it.next() { body }
```

`Vec<T>` has three `IntoIterator` impls, which is why the three loop forms
differ:

| Loop | Impl used | Item |
|---|---|---|
| `for x in v` | `impl IntoIterator for Vec<T>` | `T` |
| `for x in &v` | `impl IntoIterator for &Vec<T>` | `&T` |
| `for x in &mut v` | `impl IntoIterator for &mut Vec<T>` | `&mut T` |

```rust
let v3 = vec![1, 2, 3];
for x in &v3 { print!("{x} "); }
println!();

let mut v4 = vec![1, 2, 3];
for x in &mut v4 { *x += 1; }
println!("{v4:?}");

for x in v3 { print!("{x} "); }
println!();
```

Output:

```
1 2 3 
[2, 3, 4]
1 2 3 
```

`&v` and `v.iter()` are the same thing. `&v` reads better in a `for`; `.iter()`
is required when you want to chain adapters.

An iterator is also `IntoIterator` (returning itself), so `for x in v.iter()`
and `for x in v.iter().map(f)` both work with no extra impl.

## Calling into_iter on a reference

`(&v).into_iter()` yields `&T`, not `T` — a classic surprise.

```rust
let v = vec![String::from("a")];
let r = &v;
let x: Vec<String> = r.into_iter().collect();
```

Does not compile:

```
error[E0277]: a value of type `Vec<String>` cannot be built from an iterator over elements of type `&String`
    --> src/main.rs:4:40
     |
   4 |     let x: Vec<String> = r.into_iter().collect();
     |                                        ^^^^^^^ value of type `Vec<String>` cannot be built from `std::iter::Iterator<Item=&String>`
     |
help: the trait `FromIterator<&String>` is not implemented for `Vec<String>`
      but trait `FromIterator<String>` is implemented for it
```

Fix with `.cloned()` or by owning `v` and calling `into_iter` on it.

## Other types that iterate

```rust
use std::collections::HashMap;

let m: HashMap<&str, i32> = HashMap::from([("a", 1)]);
for (k, val) in &m { println!("{k}={val}"); }

println!("{:?}", Some(5).into_iter().collect::<Vec<_>>());
println!("{:?}", (1..4).collect::<Vec<_>>());

let arr = [1, 2, 3];
println!("{:?}", arr.into_iter().map(|x| x * 2).collect::<Vec<i32>>());
println!("{:?}", "héllo".chars().collect::<Vec<char>>());
```

Output:

```
a=1
[5]
[1, 2, 3]
[2, 4, 6]
['h', 'é', 'l', 'l', 'o']
```

Item types vary: `&HashMap` yields `(&K, &V)`, `Option` yields 0 or 1 items,
ranges *are* iterators already, arrays yield `T` by value from `into_iter`, and
`&str` has no `IntoIterator` — pick `.chars()` or `.bytes()` explicitly, since
there is no single right answer for text.

## Laziness

Adapters build a struct and return it. Nothing runs.

```rust
let v = vec![1, 2, 3];
v.iter().map(|x| { println!("mapping {x}"); x * 2 });
println!("nothing happened");
```

Compiles, with a warning:

```
warning: unused `Map` that must be used
 --> src/main.rs:3:5
  |
3 |     v.iter().map(|x| { println!("mapping {x}"); x * 2 });
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: iterators are lazy and do nothing unless consumed
```

And the program prints only:

```
nothing happened
```

## Laziness is element-at-a-time

Add a consumer and watch the interleaving. Each element is pulled all the way
through the chain before the next one starts — not one full pass per stage.

```rust
let v = vec![1, 2, 3];
let chain = v.iter()
    .map(|x| { println!("map {x}"); x * 2 })
    .filter(|x| { println!("  filter {x}"); *x > 2 });
println!("built, nothing ran");
let out: Vec<i32> = chain.collect();
println!("{out:?}");
```

Output:

```
built, nothing ran
map 1
  filter 2
map 2
  filter 4
map 3
  filter 6
[4, 6]
```

That also means short-circuiting consumers do less work:

```rust
let v2 = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
let first = v2.iter().map(|x| { println!("map {x}"); x * 2 }).find(|x| *x > 4);
println!("{first:?}");
```

Output:

```
map 1
map 2
map 3
Some(6)
```

Seven elements were never mapped. Python generators behave the same way; JS
array methods do not.

## Zero cost

An iterator chain compiles to the same machine code as the hand-written loop —
usually better, because indexing has a bounds check per access and the iterator
does not.

```rust
use std::time::Instant;

let v: Vec<u64> = (0..50_000_000).collect();

let t = Instant::now();
let mut sum = 0u64;
for i in 0..v.len() { sum += v[i]; }
let manual = t.elapsed();

let t = Instant::now();
let iter: u64 = v.iter().sum();
let iterated = t.elapsed();

println!("{sum} in {manual:?}");
println!("{iter} in {iterated:?}");
```

Output (`cargo run --release`; numbers are machine-specific):

```
1249999975000000 in 10.221292ms
1249999975000000 in 5.098084ms
```

Measure in release mode. In debug, iterator chains are *slower* because none of
the layers get inlined.

## size_hint

Iterators advertise their length so consumers can preallocate.

```rust
let mut it = vec![1, 2, 3].into_iter();
println!("{:?} {:?}", it.next(), it.size_hint());
```

Output:

```
Some(1) (2, Some(2))
```

`(lower_bound, Option<upper_bound>)`. `collect` into a `Vec` uses it to reserve
capacity once instead of growing repeatedly.

## Quick Reference

| Item | Meaning |
|---|---|
| `trait Iterator { type Item; fn next(&mut self) -> Option<Item>; }` | the whole contract |
| `.iter()` | yields `&T` |
| `.iter_mut()` | yields `&mut T` |
| `.into_iter()` | yields `T`, consumes the collection |
| `for x in v` | `IntoIterator for Vec<T>`, items `T` |
| `for x in &v` | items `&T`, same as `v.iter()` |
| `for x in &mut v` | items `&mut T`, same as `v.iter_mut()` |
| `.next()` | advance once; needs a `mut` binding |
| `.size_hint()` | `(lower, Option<upper>)` |
| adapter with no consumer | does nothing, warns `unused ... must be used` |

## Common Mistakes

**`for x in v` then using `v`.** The loop moved it; write `for x in &v`.

**Expecting `(&v).into_iter()` to yield `T`.** It yields `&T`; add `.cloned()`
or own the value first.

**Forgetting `mut` on a manual iterator.** `next` takes `&mut self`.

**Building a chain and not consuming it.** Nothing runs; the `must_use` warning
is telling you so.

**Benchmarking iterators in debug mode.** Always `--release`; the zero-cost
claim is about optimized builds.

## Exercises

1. Make a `Vec<&str>`, get an iterator, and call `next()` four times printing
   each `Option`.
2. For a `Vec<String>`, write three loops using `iter`, `iter_mut`, and
   `into_iter`, and print the item type each gives you (uppercase one, mutate
   one, move one out).
3. Build a `.map().filter()` chain with `println!` in both closures and no
   consumer. Record the warning, then add `.count()` and record the order the
   prints appear in.
4. Iterate a `HashMap<String, u32>` by reference and by value, and note the
   item type each yields.
5. Time `v.iter().map(|x| x * 2).sum::<u64>()` against an index loop in both
   debug and release. Explain the difference.

---

[← Course Index](../README.md)
