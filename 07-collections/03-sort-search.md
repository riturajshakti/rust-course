[← Course Index](../README.md)

# 7.3 — Sorting, Searching, and Dedup

## What You'll Learn

- `sort` vs `sort_unstable` vs `sort_by` vs `sort_by_key`, and which to reach for
- Sorting structs, sorting descending, and the float problem
- `binary_search` and the useful `Err(insert_index)` it returns
- `dedup` family, `retain`, and `is_sorted`

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `a.sort((x,y) => x-y)` | `v.sort()` or `v.sort_by(\|x,y\| x.cmp(y))` |
| Python `a.sort(key=len)` | `v.sort_by_key(\|s\| s.len())` |
| Go `sort.Slice(a, less)` | `v.sort_by(\|a,b\| ...)` |

Rust's `sort()` takes no arguments and needs no comparator because it requires
`T: Ord`. That one bound explains most of this lesson: integers and `String`
have it, `f64` does not.

## sort and sort_unstable

```rust
let mut v = vec![5, 3, 9, 1, 7];
v.sort();
println!("{:?}", v);

let mut v = vec![5, 3, 9, 1, 7];
v.sort_unstable();
println!("{:?}", v);
```

Output:

```
[1, 3, 5, 7, 9]
[1, 3, 5, 7, 9]
```

Both ascending, both in place, both require `T: Ord`.

**Stability** means equal elements keep their original relative order.
`sort` is stable (a merge sort, allocates a temporary buffer);
`sort_unstable` is not (a pattern-defeating quicksort, no allocation, usually
faster).

```rust
let mut v = vec![("b", 1), ("a", 1), ("c", 0)];
let mut s = v.clone();
s.sort_by_key(|p| p.1);
println!("stable:   {:?}", s);
v.sort_unstable_by_key(|p| p.1);
println!("unstable: {:?}", v);
```

Output:

```
stable:   [("c", 0), ("b", 1), ("a", 1)]
unstable: [("c", 0), ("b", 1), ("a", 1)]
```

On three elements they agree — unstable is not *randomized*, just
*unguaranteed*. Only `sort` promises that `("b",1)` stays ahead of `("a",1)`.
Default to `sort_unstable` unless you are sorting by a secondary key and
relying on a previous pass, or your elements carry identity you must preserve.

## sort_by: a full comparator

Return an `Ordering`. Swap the operands to reverse.

```rust
let mut v = vec![5, 3, 9, 1, 7];
v.sort_by(|a, b| b.cmp(a));
println!("{:?}", v);
```

Output:

```
[9, 7, 5, 3, 1]
```

## sort_by_key: a projection

Cheaper to write when you are sorting by one extracted value.

```rust
let mut v = vec!["hello", "hi", "hey!!"];
v.sort_by_key(|s| s.len());
println!("{:?}", v);

let mut v = vec![-5i32, 2, -1, 4];
v.sort_by_key(|x| x.abs());
println!("{:?}", v);

let mut v = vec![5, 3, 9];
v.sort_unstable_by_key(|x| -x);
println!("{:?}", v);
```

Output:

```
["hi", "hello", "hey!!"]
[-1, 2, 4, -5]
[9, 5, 3]
```

The key must be `Ord` and is computed **once per comparison**, so for an
expensive key (allocating a `String`, say) use `sort_by_cached_key`, which
computes it once per element:

```rust
let mut v = vec![3, 1, 2];
v.sort_by_cached_key(|x| x.to_string());
println!("{:?}", v);
```

Output:

```
[1, 2, 3]
```

`sort_by_key` also cannot return a key that borrows from the element — the
signature does not allow it. Use `sort_by` with `a.field.cmp(&b.field)` there.

## Sorting structs by field

```rust
#[derive(Debug, Clone)]
struct Emp { name: String, age: u32, salary: f64 }

let mut e = vec![
    Emp { name: "raj".into(), age: 30, salary: 90_000.0 },
    Emp { name: "sam".into(), age: 25, salary: 120_000.0 },
    Emp { name: "ana".into(), age: 30, salary: 75_000.0 },
];

e.sort_by_key(|x| x.age);
println!("{:?}", e.iter().map(|x| (&x.name, x.age)).collect::<Vec<_>>());

e.sort_by(|a, b| a.age.cmp(&b.age).then(a.name.cmp(&b.name)));
println!("{:?}", e.iter().map(|x| (&x.name, x.age)).collect::<Vec<_>>());

e.sort_by(|a, b| b.salary.partial_cmp(&a.salary).unwrap());
println!("{:?}", e.iter().map(|x| (&x.name, x.salary)).collect::<Vec<_>>());
```

Output:

```
[("sam", 25), ("raj", 30), ("ana", 30)]
[("sam", 25), ("ana", 30), ("raj", 30)]
[("sam", 120000.0), ("raj", 90000.0), ("ana", 75000.0)]
```

`.then(...)` chains tiebreakers; `.then_with(|| ...)` does the same but only
evaluates the second comparison when the first is `Equal`.

## Sorting descending

Three ways, all equivalent in result:

```rust
let mut v = vec![1, 2, 3];
v.sort_by(|a, b| b.cmp(a));
println!("{:?}", v);

let mut v = vec![1, 2, 3];
v.sort();
v.reverse();
println!("{:?}", v);

use std::cmp::Reverse;
let mut e = vec![("raj", 30), ("sam", 25), ("ana", 31)];
e.sort_by_key(|x| Reverse(x.1));
println!("{:?}", e);
```

Output:

```
[3, 2, 1]
[3, 2, 1]
[("ana", 31), ("raj", 30), ("sam", 25)]
```

`Reverse` is the idiomatic choice when you want `sort_by_key` ergonomics with
a descending order, and it is the only one that works cleanly on non-numeric
keys. `|x| -x` only works for signed numbers.

## Sorting floats

`f64` is not `Ord`, because `NaN` is not comparable with anything.

```rust
let mut v = vec![1.0f64, 0.5];
v.sort();
```

Does not compile:

```
error[E0277]: the trait bound `f64: Ord` is not satisfied
 --> src/main.rs:3:7
  |
3 |     v.sort();
  |       ^^^^ the trait `Ord` is not implemented for `f64`
```

Two fixes. `partial_cmp().unwrap()` is fine when you know there is no `NaN`:

```rust
let mut g = vec![3.2f64, 1.5, 0.1];
g.sort_by(|a, b| a.partial_cmp(b).unwrap());
println!("{:?}", g);
```

Output:

```
[0.1, 1.5, 3.2]
```

But it panics the moment one appears (deliberately broken):

```rust
let mut v = vec![1.0f64, f64::NAN, 0.5];
v.sort_by(|a, b| a.partial_cmp(b).unwrap());
```

```
thread 'main' panicked at src/main.rs:3:39:
called `Option::unwrap()` on a `None` value
```

`total_cmp` is the safe default — a total order that sorts `NaN` to the end:

```rust
let mut f = vec![3.2, 1.5, f64::NAN, 0.1];
f.sort_by(|a, b| a.total_cmp(b));
println!("{:?}", f);
```

Output:

```
[0.1, 1.5, 3.2, NaN]
```

## binary_search and Err(insert_index)

Requires a sorted vec. `Ok(i)` is a match; `Err(i)` is where the value *would*
go to keep it sorted.

```rust
let v = vec![1, 3, 5, 7, 9];
println!("{:?}", v.binary_search(&5));
println!("{:?}", v.binary_search(&6));
println!("{:?}", v.binary_search(&0));
println!("{:?}", v.binary_search(&100));
```

Output:

```
Ok(2)
Err(3)
Err(0)
Err(5)
```

That `Err` index is the point of the API — it gives you sorted insertion for
free:

```rust
let mut v = vec![1, 3, 5, 7, 9];
let x = 6;
match v.binary_search(&x) {
    Ok(i) => println!("found at {i}"),
    Err(i) => { v.insert(i, x); println!("inserted at {i} -> {:?}", v); }
}
```

Output:

```
inserted at 3 -> [1, 3, 5, 6, 7, 9]
```

On an **unsorted** vec it returns nonsense rather than erroring:

```rust
let unsorted = vec![5, 1, 9, 3];
println!("{:?}", unsorted.binary_search(&9));
```

Output:

```
Err(4)
```

The 9 is right there at index 2. Sorting is your responsibility.

## binary_search_by and binary_search_by_key

`binary_search_by` takes a closure returning how the *element* compares to the
target. For a descending vec, flip the operands:

```rust
let d = vec![9, 7, 5, 3, 1];
println!("{:?}", d.binary_search_by(|p| 5.cmp(p)));
```

Output:

```
Ok(2)
```

`binary_search_by_key` projects each element first:

```rust
let pairs = vec![(1, "a"), (3, "b"), (5, "c")];
println!("{:?}", pairs.binary_search_by_key(&3, |&(k, _)| k));
println!("{:?}", pairs.binary_search_by_key(&4, |&(k, _)| k));
```

Output:

```
Ok(1)
Err(2)
```

`partition_point` returns the first index where a predicate stops holding — a
`binary_search` that never fails. On `vec![1, 3, 5, 7, 9]`,
`partition_point(|&x| x < 6)` is `3`.

## Linear search: contains, position, find, any, all

For unsorted data, or when the vec is small, scan it.

```rust
let v = vec![10, 20, 30, 20];
println!("{}", v.contains(&20));
println!("{:?}", v.iter().position(|&x| x == 20));
println!("{:?}", v.iter().rposition(|&x| x == 20));
println!("{:?}", v.iter().find(|&&x| x > 15));
println!("{:?}", v.iter().find_map(|&x| if x > 15 { Some(x * 2) } else { None }));
println!("{}", v.iter().any(|&x| x > 25));
println!("{}", v.iter().all(|&x| x > 5));
println!("{}", v.iter().filter(|&&x| x == 20).count());
println!("{:?}", v.iter().position(|&x| x == 99));
```

Output:

```
true
Some(1)
Some(3)
Some(20)
Some(40)
true
true
2
None
```

`position` gives the index, `find` the element, `find_map` a transformed
element; `any`/`all` short-circuit. All O(n) — and below a few dozen elements
they beat sort-plus-binary-search outright.

## min, max, and the _by_key variants

```rust
let v = vec![10, 20, 30, 20];
println!("{:?}", v.iter().max());
println!("{:?}", v.iter().min());

let words = vec!["apple", "fig", "cherry"];
println!("{:?}", words.iter().max_by_key(|s| s.len()));
println!("{:?}", words.iter().min_by_key(|s| s.len()));

let f = vec![1.5, 3.7, 0.2];
println!("{:?}", f.iter().copied().max_by(|a, b| a.total_cmp(b)));
println!("{}", f.iter().cloned().fold(f64::MIN, f64::max));

let e: Vec<i32> = vec![];
println!("{:?} {:?}", e.iter().max(), e.iter().min());
```

Output:

```
Some(30)
Some(10)
Some("cherry")
Some("fig")
Some(3.7)
3.7
None None
```

On an empty vec they return `None`. One subtlety: `max` returns the **last**
maximum on ties, `min` the **first**. Use `max_by` / `min_by` with
`total_cmp` for floats — there is no `max()` on an `f64` iterator.

Need the k-th element but not a full sort? `select_nth_unstable(k)` partitions
in O(n) and returns `(&mut below, &mut kth, &mut above)` — on `vec![5,2,8,1]`
with `k = 1` that is `([1], 2, [5, 8])`.

## dedup needs sorting first

`dedup` removes only **consecutive** duplicates.

```rust
let mut v = vec![3, 1, 3, 2, 1, 3];
v.dedup();
println!("naive dedup: {:?}", v);

let mut v = vec![3, 1, 3, 2, 1, 3];
v.sort();
v.dedup();
println!("sort+dedup:  {:?}", v);
```

Output:

```
naive dedup: [3, 1, 3, 2, 1, 3]
sort+dedup:  [1, 2, 3]
```

The first call removed nothing at all. `sort` then `dedup` is the standard
unique-values idiom for a `Vec`, and it is O(n log n).

## dedup_by and dedup_by_key

```rust
let mut v = vec!["Foo", "foo", "BAR", "bar", "baz"];
v.dedup_by(|a, b| a.eq_ignore_ascii_case(b));
println!("{:?}", v);

let mut v = vec![(1, 'a'), (1, 'b'), (2, 'c'), (1, 'd')];
v.dedup_by_key(|p| p.0);
println!("{:?}", v);

let mut v = vec![10, 11, 20, 21, 22, 30];
v.dedup_by_key(|x| *x / 10);
println!("{:?}", v);
```

Output:

```
["Foo", "BAR", "baz"]
[(1, 'a'), (2, 'c'), (1, 'd')]
[10, 20, 30]
```

`dedup_by` keeps the **first** of each run. Note the closure takes
`(&mut a, &mut b)` where `a` is the *later* element — the one that gets
removed when you return `true`.

If you need unique values but must preserve original order, a `HashSet` as a
seen-filter does it in one pass:

```rust
use std::collections::HashSet;

let v = vec![3, 1, 3, 2, 1];
let mut seen = HashSet::new();
let order_preserved: Vec<i32> = v.iter().copied().filter(|x| seen.insert(*x)).collect();
println!("{:?}", order_preserved);
```

Output:

```
[3, 1, 2]
```

`HashSet::insert` returns `true` the first time it sees a value, which makes
it a perfect `filter` predicate. Lesson 7.7 covers sets properly.

## retain

The predicate form of removal — one pass, order preserved.

```rust
let mut v = vec![1, 2, 3, 4, 5, 6];
v.retain(|x| x % 3 != 0);
println!("{:?}", v);
```

Output:

```
[1, 2, 4, 5]
```

`retain` visits elements in order exactly once, so a stateful closure (an
index counter, an external iterator of flags) is reliable.

## is_sorted

```rust
println!("{}", vec![1, 2, 3].is_sorted());
println!("{}", vec![3, 1, 2].is_sorted());
println!("{}", vec![1, 1, 2].is_sorted());
println!("{}", vec![3, 2, 1].is_sorted_by(|a, b| a >= b));
println!("{}", vec!["aa", "b"].is_sorted_by_key(|s| s.len()));

let e: Vec<i32> = vec![];
println!("{} {}", e.is_sorted(), vec![1].is_sorted());
```

Output:

```
true
false
true
true
false
true true
```

Equal neighbours count as sorted. `is_sorted_by` takes a `bool`-returning
closure (not an `Ordering`) meaning "a comes before b". Empty and one-element
vecs are always sorted. Useful as a debug assertion before a
`binary_search`.

## Quick Reference

| Need | Method |
|---|---|
| Ascending, stable | `sort()` |
| Ascending, fastest | `sort_unstable()` |
| Custom comparator | `sort_by(\|a,b\| ...)` → `Ordering` |
| By one field | `sort_by_key(\|x\| x.field)` |
| By an expensive key | `sort_by_cached_key(\|x\| ...)` |
| Descending | `sort_by_key(\|x\| Reverse(k))` or `sort_by(\|a,b\| b.cmp(a))` |
| Multi-key | `sort_by(\|a,b\| a.k1.cmp(&b.k1).then(a.k2.cmp(&b.k2)))` |
| Floats | `sort_by(\|a,b\| a.total_cmp(b))` |
| Find in sorted | `binary_search(&x)` → `Ok(i)` / `Err(insert_at)` |
| Find in sorted, projected | `binary_search_by_key(&k, \|x\| x.k)` |
| Insertion point | `partition_point(\|x\| pred)` |
| Membership | `contains(&x)` |
| Index of first/last match | `iter().position(p)` / `rposition(p)` |
| First matching element | `iter().find(p)` |
| Exists / all | `iter().any(p)` / `all(p)` |
| Extremes | `iter().max()`, `min()`, `max_by_key`, `min_by_key` |
| k-th element, no full sort | `select_nth_unstable(k)` |
| Collapse adjacent dups | `dedup()`, `dedup_by`, `dedup_by_key` |
| All unique | `sort(); dedup();` |
| Unique, order kept | `HashSet` seen-filter |
| Remove by predicate | `retain(p)` |
| Check order | `is_sorted()`, `is_sorted_by_key` |

## Common Mistakes

**`dedup()` without sorting.** It only collapses adjacent duplicates, so it
can silently do nothing.

**`binary_search` on unsorted data.** It returns a meaningless `Ok`/`Err`
rather than an error. Sort first, or use `position`.

**`v.sort()` on a `Vec<f64>`.** `f64` is not `Ord`. Use
`sort_by(|a,b| a.total_cmp(b))`.

**`partial_cmp().unwrap()` on data that may contain `NaN`.** It panics. Use
`total_cmp`.

**Assuming `sort_unstable` reorders equal elements.** It is allowed to, not
guaranteed to — never rely on either behaviour; use `sort` when order among
equals matters.

**`sort_by_key` with a borrowing key.** `|x| &x.name` will not compile; use
`sort_by(|a, b| a.name.cmp(&b.name))`.

## Exercises

1. Sort `vec![("raj", 30), ("ana", 25), ("sam", 30)]` by age descending, then
   by name ascending as a tiebreaker. Print the result.
2. Given a sorted `Vec<i32>`, write `fn insert_sorted(v: &mut Vec<i32>, x: i32)`
   using `binary_search` that keeps the vec sorted and does nothing if `x` is
   already present.
3. Sort `vec![3.1, f64::NAN, 0.5, 2.0]` with `total_cmp`, print it, then try
   `partial_cmp().unwrap()` and capture the panic message.
4. Write two `unique` functions for `Vec<String>`: one via `sort` + `dedup`,
   one via a `HashSet` seen-filter. Show that they differ in output order.
5. For `vec![5, 3, 8, 1, 9, 2]`, print the max, the min, the index of the
   first element greater than 4, whether all elements are positive, and
   whether the vec is sorted — in six lines.

---

[← Course Index](../README.md)
