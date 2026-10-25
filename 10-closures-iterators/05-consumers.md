[← Course Index](../README.md)

# 10.5 — Consumers: collect, fold, and Friends

## What You'll Learn

- `collect` into every standard container, plus `Result` and `Option`
- `fold`, `try_fold`, `reduce` — and when each is right
- The short-circuiting family: `any`, `all`, `find`, `position`
- `max_by_key` and friends, `partition`, `unzip`, `extend`, iterator comparison

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `.reduce(f, init)` | `fold(init, f)` |
| JS `.reduce(f)` (no init) | `reduce(f)` → `Option` |
| Python `sum()`, `any()`, `all()` | `.sum()`, `.any()`, `.all()` |
| Python `list(gen)` | `.collect::<Vec<_>>()` |

A **consumer** calls `next()` until it is satisfied and returns something that
is not an iterator. One per chain, at the end.

## collect needs a target type

```rust
let v = vec![1, 2, 3];
let out = v.iter().map(|x| x * 2).collect();
```

Does not compile:

```
error[E0283]: type annotations needed
    --> src/main.rs:3:9
     |
   3 |     let out = v.iter().map(|x| x * 2).collect();
     |         ^^^                           ------- type must be known at this point
     |
     = note: the type must implement `FromIterator<i32>`
help: consider giving `out` an explicit type
     |
   3 |     let out: Vec<_> = v.iter().map(|x| x * 2).collect();
```

Two spellings: annotate the binding (`let out: Vec<i32> = ...`) or the call
(`.collect::<Vec<i32>>()`). Use the binding when it is a `let`, the turbofish
when it is inline in a `println!` or argument.

## collect into containers

```rust
use std::collections::{HashSet, BTreeSet, VecDeque};

let v = vec![3, 1, 2, 3];

let a: Vec<i32> = v.iter().copied().collect();
let b: VecDeque<i32> = v.iter().copied().collect();
let c: HashSet<i32> = v.iter().copied().collect();
let d: BTreeSet<i32> = v.iter().copied().collect();
println!("{a:?} {b:?} {d:?}");

let mut cs: Vec<_> = c.into_iter().collect();
cs.sort();
println!("{cs:?}");
```

Output:

```
[3, 1, 2, 3] [3, 1, 2, 3] {1, 2, 3}
[1, 2, 3]
```

`HashSet` iteration order is arbitrary, so the only way to print it
deterministically is to collect and sort (or use `BTreeSet`, which is ordered
by construction).

## collect into String

Works from `&str`, `String`, or `char` items.

```rust
let s: String = vec!["a", "b"].into_iter().collect();
let s2: String = "hello".chars().rev().collect();
println!("{s} {s2}");
```

Output:

```
ab olleh
```

## collect into maps

Items must be `(K, V)` tuples.

```rust
use std::collections::{HashMap, BTreeMap};

let m: HashMap<&str, usize> = vec!["apple", "fig"].into_iter().map(|w| (w, w.len())).collect();
let mut ms: Vec<_> = m.into_iter().collect();
ms.sort();
println!("{ms:?}");

let bm: BTreeMap<&str, usize> = vec!["apple", "fig"].into_iter().map(|w| (w, w.len())).collect();
println!("{bm:?}");
```

Output:

```
[("apple", 5), ("fig", 3)]
{"apple": 5, "fig": 3}
```

Duplicate keys: the **last** one wins, silently. `BTreeMap` prints in key order
with no sorting step, which is often reason enough to choose it for output.

## collect into Result and Option

The headline trick: `collect` can turn an iterator of `Result`s into a
`Result` of a collection, stopping at the first error.

```rust
let ok: Result<Vec<i32>, _> = vec!["1", "2"].into_iter().map(|s| s.parse::<i32>()).collect();
let bad: Result<Vec<i32>, _> = vec!["1", "x", "3"].into_iter().map(|s| s.parse::<i32>()).collect();
println!("{ok:?}");
println!("{bad:?}");
```

Output:

```
Ok([1, 2])
Err(ParseIntError { kind: InvalidDigit })
```

Note it short-circuits — `"3"` was never parsed. Keep every outcome instead by
collecting into `Vec<Result<..>>`:

```rust
let each: Vec<Result<i32, _>> = vec!["1", "x"].into_iter().map(|s| s.parse::<i32>()).collect();
println!("{each:?}");
```

Output:

```
[Ok(1), Err(ParseIntError { kind: InvalidDigit })]
```

| Target | Behaviour |
|---|---|
| `Result<Vec<T>, E>` | all-or-nothing, first error wins, short-circuits |
| `Vec<Result<T, E>>` | one entry per item, nothing lost |
| `Vec<T>` via `filter_map(.ok())` | errors silently dropped |

`Option` works the same way:

```rust
let opt: Option<Vec<i32>> = vec![Some(1), Some(2)].into_iter().collect();
let opt2: Option<Vec<i32>> = vec![Some(1), None].into_iter().collect();
println!("{opt:?} {opt2:?}");
```

Output:

```
Some([1, 2]) None
```

## sum, product, count, last, nth

```rust
let v = vec![1, 2, 3, 4, 5];
let e: Vec<i32> = vec![];

println!("{} {}", v.iter().sum::<i32>(), e.iter().sum::<i32>());
println!("{} {}", v.iter().product::<i32>(), e.iter().product::<i32>());
println!("{} {}", v.iter().count(), e.iter().count());
println!("{:?} {:?}", v.iter().last(), e.iter().last());
println!("{:?} {:?}", v.iter().nth(2), v.iter().nth(99));
```

Output:

```
15 0
120 1
5 0
Some(5) None
Some(3) None
```

Empty `sum` is 0, empty `product` is 1. `nth` is 0-based and *consumes* the
items it skips. For a `Vec` prefer `v[2]` or `v.last()` — `nth`/`last` on an
iterator are O(n).

`sum` panics on overflow in debug builds:

```rust
let v: Vec<u8> = vec![200, 100];
println!("{}", v.iter().sum::<u8>());
```

Output:

```
thread 'main' panicked at .../iter/traits/accum.rs:206:1:
attempt to add with overflow
```

Sum into a wider type: `v.iter().map(|x| *x as u32).sum::<u32>()`.

## fold

The general reduction. Takes an initial accumulator, so it is total — no
`Option`.

```rust
println!("{}", v.iter().fold(0, |a, x| a + x));
println!("{}", v.iter().fold(String::new(), |mut a, x| { a.push_str(&x.to_string()); a }));
println!("{}", v.iter().fold(1i64, |a, x| a * *x as i64));
```

Output:

```
15
12345
120
```

The accumulator is passed *by value* and returned, which is why the `String`
version takes `mut a` and gives it back.

## fold into a HashMap

The idiomatic word count.

```rust
use std::collections::HashMap;

let text = "the quick the lazy the dog";
let counts = text.split_whitespace().fold(HashMap::new(), |mut m, w| {
    *m.entry(w).or_insert(0) += 1;
    m
});
let mut cs: Vec<_> = counts.into_iter().collect();
cs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
println!("{cs:?}");
```

Output:

```
[("the", 3), ("dog", 1), ("lazy", 1), ("quick", 1)]
```

Sorted by count descending, then name, because `HashMap` order is arbitrary.

## reduce

`fold` with the first element as the initial value. Returns `Option` because an
empty iterator has no first element.

```rust
println!("{:?}", v.iter().copied().reduce(|a, b| a.max(b)));
println!("{:?}", e.iter().copied().reduce(|a, b| a + b));
```

Output:

```
Some(5)
None
```

Use `reduce` when accumulator and item are the same type and you have no
sensible identity; use `fold` otherwise.

## try_fold

Stops at the first `Err`/`None` and returns it.

```rust
let tf: Result<i32, String> = v.iter()
    .try_fold(0i32, |a, x| if *x < 4 { Ok(a + x) } else { Err(format!("too big: {x}")) });
println!("{tf:?}");

let tf2: Option<i32> = v.iter().try_fold(0i32, |a, x| a.checked_add(*x));
println!("{tf2:?}");
```

Output:

```
Err("too big: 4")
Some(15)
```

`try_fold` with `checked_add` is the overflow-safe sum.

## for_each and try_for_each

`fold` with no accumulator. Prefer a plain `for` loop unless you are at the end
of a long chain, where `for_each` reads better than wrapping it in parentheses.

```rust
v.iter().for_each(|x| print!("{x} "));
println!();

let r: Result<(), String> = v.iter().try_for_each(|x| {
    if *x < 3 { println!("ok {x}"); Ok(()) } else { Err(format!("stop at {x}")) }
});
println!("{r:?}");
```

Output:

```
1 2 3 4 5 
ok 1
ok 2
Err("stop at 3")
```

A `for` loop can `break` and `return`; `for_each` cannot. That is the deciding
factor most of the time.

## any and all

Both short-circuit. Note the empty-iterator results.

```rust
println!("{} {}", v.iter().any(|x| *x > 4), v.iter().any(|x| *x > 9));
println!("{} {}", v.iter().all(|x| *x > 0), v.iter().all(|x| *x > 1));
println!("{} {}", e.iter().any(|x| *x > 0), e.iter().all(|x| *x > 0));
```

Output:

```
true false
true false
false true
```

Empty `any` is `false`, empty `all` is `true` — vacuous truth, same as Python.

## find, find_map, position, rposition

```rust
let words = vec!["apple", "fig", "banana", "kiwi"];

println!("{:?} {:?}", v.iter().find(|x| **x % 2 == 0), v.iter().find(|x| **x > 9));
println!("{:?}", words.iter().find_map(|w| w.strip_prefix("ban")));
println!("{:?} {:?}", v.iter().position(|x| x == &3), v.iter().position(|x| x == &9));
println!("{:?}", v.iter().rposition(|x| x % 2 == 1));
```

Output:

```
Some(2) None
Some("ana")
Some(2) None
Some(4)
```

`find` returns the item, `position` the index. `find_map` is
`filter_map().next()` — it returns the first successful *transformation*, which
is cleaner than `find` followed by a redundant unwrap. `rposition` searches
from the back and needs a `DoubleEndedIterator + ExactSizeIterator`.

## max, min, and the _by variants

```rust
println!("{:?} {:?}", v.iter().max(), v.iter().min());
println!("{:?} {:?}", e.iter().max(), e.iter().min());
println!("{:?}", words.iter().max_by_key(|w| w.len()));
println!("{:?}", words.iter().min_by_key(|w| w.len()));
```

Output:

```
Some(5) Some(1)
None None
Some("banana")
Some("fig")
```

Floats are not `Ord`, so `max()` does not exist on them — use `max_by` with
`partial_cmp`:

```rust
let fs = vec![1.5f64, -2.0, 0.3];
println!("{:?}", fs.iter().copied().max_by(|a, b| a.partial_cmp(b).unwrap()));
println!("{:?}", fs.iter().copied().min_by(|a, b| a.partial_cmp(b).unwrap()));
```

Output:

```
Some(1.5)
Some(-2.0)
```

That `unwrap` panics on `NaN`; use `f64::total_cmp` instead if NaN is possible.

Ties break asymmetrically, which bites in practice: `max*` returns the **last**
maximum, `min*` the **first** minimum.

```rust
let ties = vec!["aa", "bb", "cc"];
println!("{:?} {:?}", ties.iter().max_by_key(|s| s.len()), ties.iter().min_by_key(|s| s.len()));
```

Output:

```
Some("cc") Some("aa")
```

| Variant | Takes | Use for |
|---|---|---|
| `max()` / `min()` | nothing | `Ord` items |
| `max_by_key(f)` / `min_by_key(f)` | item → `Ord` key | compare by one field |
| `max_by(f)` / `min_by(f)` | `(a, b) -> Ordering` | floats, custom orders |

## partition

Split into two collections by a predicate.

```rust
let (ev, od): (Vec<i32>, Vec<i32>) = v.iter().partition(|x| **x % 2 == 0);
println!("{ev:?} {od:?}");
```

Output:

```
[2, 4] [1, 3, 5]
```

True side first. Any pair of `Default + Extend` containers works, including
two `HashSet`s.

## unzip

Inverse of `zip`. Needs tuple items.

```rust
let words = vec!["apple", "fig", "banana", "kiwi"];
let (lens, firsts): (Vec<usize>, Vec<char>) =
    words.iter().map(|w| (w.len(), w.chars().next().unwrap())).unzip();
println!("{lens:?} {firsts:?}");
```

Output:

```
[5, 3, 6, 4] ['a', 'f', 'b', 'k']
```

## extend

Append an iterator to an existing collection instead of building a new one.

```rust
let mut target = vec![0];
target.extend(v.iter().copied());
target.extend([9, 9]);
println!("{target:?}");
```

Output:

```
[0, 1, 2, 3, 4, 5, 9, 9]
```

`collect` is `Default::default()` then `extend`. Reach for `extend` when the
destination already exists.

## Joining into a String

There is no `join` on iterators — it is a slice method.

```rust
let csv = vec![1, 2, 3].iter().map(|x| x.to_string()).collect::<Vec<_>>().join("-");
println!("{csv}");
```

Output:

```
1-2-3
```

Lesson 10.8 shows `itertools::join`, which skips the intermediate `Vec`.

## Comparing iterators

Element-wise lexicographic comparison, without building collections.

```rust
println!("{}", [1, 2, 3].iter().eq([1, 2, 3].iter()));
println!("{}", [1, 2, 3].iter().eq([1, 2].iter()));
println!("{:?}", [1, 2, 3].iter().cmp([1, 2, 4].iter()));
println!("{}", [1, 2, 3].iter().lt([1, 2, 4].iter()));
println!("{}", [1, 2].iter().lt([1, 2, 0].iter()));
println!("{:?}", [1.0f64, 2.0].iter().partial_cmp([1.0, 3.0].iter()));
```

Output:

```
true
false
Less
true
true
Some(Less)
```

Also `ne`, `le`, `gt`, `ge`. A shorter prefix is `Less` than a longer sequence
it starts, exactly like string comparison. Use `partial_cmp` for floats.

## Quick Reference

| Consumer | Returns |
|---|---|
| `collect()` | any `FromIterator` type |
| `sum()` / `product()` | `T`; 0 / 1 when empty |
| `count()` | `usize` |
| `last()` / `nth(n)` | `Option<Item>` |
| `fold(init, f)` | accumulator, always |
| `reduce(f)` | `Option<Item>` |
| `try_fold(init, f)` | `Result`/`Option`, short-circuits |
| `for_each(f)` | `()` |
| `try_for_each(f)` | `Result`/`Option`, short-circuits |
| `any(p)` / `all(p)` | `bool`, short-circuits |
| `find(p)` | `Option<Item>` |
| `find_map(f)` | `Option<U>` |
| `position(p)` / `rposition(p)` | `Option<usize>` |
| `max()` / `min()` | `Option<Item>`, needs `Ord` |
| `max_by_key(f)` / `min_by_key(f)` | `Option<Item>` |
| `max_by(f)` / `min_by(f)` | `Option<Item>` |
| `partition(p)` | `(B, B)` |
| `unzip()` | `(B, C)` |
| `extend(iter)` | `()`, on the collection |
| `eq` / `cmp` / `lt` / `partial_cmp` | `bool` / `Ordering` / `Option<Ordering>` |

## Common Mistakes

**Calling `collect()` with no target type.** Annotate the binding or
turbofish the call.

**Printing a collected `HashMap`/`HashSet` directly.** Order is arbitrary;
collect into a `Vec` and sort, or use the `BTree` version.

**Using `fold` where `sum`/`max`/`count` exists.** The named consumer is
clearer and sometimes faster.

**Expecting `max_by_key` to return the first tie.** It returns the last;
`min_by_key` returns the first.

**Collecting into `Vec<Result<T, E>>` when you wanted all-or-nothing.** Annotate
`Result<Vec<T>, E>` to short-circuit on the first error.

## Exercises

1. Collect `1..=10` into a `Vec`, a `HashSet`, a `BTreeSet`, and a `String` of
   digits. Print each deterministically.
2. Parse `vec!["1","2","three","4"]` three ways: `Result<Vec<i32>, _>`,
   `Vec<Result<i32, _>>`, and `Vec<i32>` dropping failures. Print all three.
3. Write the word-count fold for a sentence, then print the top two words by
   count with ties broken alphabetically.
4. Given `Vec<(&str, f64)>` of products and prices, find the most expensive
   with `max_by`, the total with `sum`, and split cheap/expensive at 10.0 with
   `partition`.
5. Use `try_fold` with `checked_mul` to compute a factorial, returning `None`
   on overflow. Find the largest `n` that fits in `u32`.
6. Compare `[1,2,3]` against `[1,2,3,0]` with `eq`, `cmp`, and `lt`. Predict
   each result before running it.

---

[← Course Index](../README.md)
