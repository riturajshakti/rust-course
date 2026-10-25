[← 7.6 BTreeMap and Ordered Data](06-btreemap.md) · [Course Index](../README.md) · [7.8 VecDeque, BinaryHeap, LinkedList →](08-other-collections.md)

# 7.7 — HashSet and BTreeSet

## What You'll Learn

- `insert` returns `bool`, and why that makes it a perfect filter
- All six set operations with real output
- Deduplicating a `Vec` three ways
- `BTreeSet` for ordered iteration and ranges

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `new Set()` | `HashSet<T>` |
| Python `set` / `sorted` set | `HashSet<T>` / `BTreeSet<T>` |
| Go `map[string]struct{}` | `HashSet<String>` |

A `HashSet<T>` is literally a `HashMap<T, ()>` with a nicer API, and
`BTreeSet<T>` is a `BTreeMap<T, ()>`. Everything you know about the key
requirements carries over: `HashSet` needs `Eq + Hash`, `BTreeSet` needs
`Ord`.

```rust
use std::collections::{HashSet, BTreeSet};
```

## Constructing, and insert returning bool

```rust
let mut s: HashSet<i32> = HashSet::new();
println!("{}", s.insert(1));
println!("{}", s.insert(1));
println!("{}", s.len());

let t = HashSet::from([1, 2, 3]);
let u: HashSet<i32> = vec![1, 2, 2, 3].into_iter().collect();
println!("{} {}", t.len(), u.len());

let v: HashSet<i32> = (1..=3).collect();
println!("{}", v.len());

let w: HashSet<i32> = HashSet::with_capacity(8);
println!("{}", w.capacity() >= 8);
```

Output:

```
true
false
1
3 3
3
true
```

`insert` returns `true` if the value was **new**. That one `bool` is the most
useful thing in the type — it turns a set into a seen-filter without any
separate `contains` call.

## contains, get, remove, take

```rust
let s = HashSet::from([1, 2, 3]);
println!("{}", s.contains(&2));
println!("{}", s.contains(&9));
println!("{:?}", s.get(&2));
println!("{:?}", s.get(&9));

let mut m = s.clone();
println!("{}", m.remove(&2));
println!("{}", m.remove(&2));
println!("{:?}", m.take(&1));
println!("{:?}", m.take(&1));
println!("{}", m.len());

let mut r = s.clone();
println!("{:?}", r.replace(2));
r.retain(|x| *x > 1);
println!("{}", r.len());

let mut d = s.clone();
let mut dr: Vec<i32> = d.drain().collect(); dr.sort();
println!("{:?} {}", dr, d.len());

let mut c = s.clone();
c.clear();
println!("{}", c.is_empty());
```

Output:

```
true
false
Some(2)
None
true
false
Some(1)
None
1
Some(2)
2
[1, 2, 3] 0
true
```

`remove` gives you a `bool`; `take` gives you the stored value back as an
`Option<T>`. `get` and `take` matter when the stored value carries more than
its identity — a struct whose `Eq` only looks at an ID field, say. `replace`
inserts and returns whatever was there.

## The set operations

All six return lazy iterators, not sets. Every output below was sorted before
printing because `HashSet` order is arbitrary.

```rust
let a: HashSet<i32> = [1, 2, 3, 4].into();
let b: HashSet<i32> = [3, 4, 5, 6].into();
let srt = |mut v: Vec<i32>| { v.sort(); v };

println!("union        {:?}", srt(a.union(&b).copied().collect()));
println!("intersection {:?}", srt(a.intersection(&b).copied().collect()));
println!("difference   {:?}", srt(a.difference(&b).copied().collect()));
println!("b - a        {:?}", srt(b.difference(&a).copied().collect()));
println!("symmetric    {:?}", srt(a.symmetric_difference(&b).copied().collect()));
```

Output:

```
union        [1, 2, 3, 4, 5, 6]
intersection [3, 4]
difference   [1, 2]
b - a        [5, 6]
symmetric    [1, 2, 5, 6]
```

`difference` is not symmetric — `a.difference(&b)` is "in a but not b".
`symmetric_difference` is "in exactly one of them".

The three predicates:

```rust
println!("{}", a.is_subset(&b));
let small: HashSet<i32> = [1, 2].into();
println!("{} {}", small.is_subset(&a), a.is_superset(&small));

let disj: HashSet<i32> = [9, 10].into();
println!("{} {}", a.is_disjoint(&disj), a.is_disjoint(&b));
```

Output:

```
false
true true
true false
```

Every set is a subset and a superset of itself, and the empty set is a subset
of everything and disjoint from everything.

## Operators

The four binary operations have operator forms on **references**, and they
return owned sets rather than iterators.

```rust
let srt = |mut v: Vec<i32>| { v.sort(); v };
println!("{:?}", srt((&a | &b).into_iter().collect()));
println!("{:?}", srt((&a & &b).into_iter().collect()));
println!("{:?}", srt((&a - &b).into_iter().collect()));
println!("{:?}", srt((&a ^ &b).into_iter().collect()));
println!("{}", a == HashSet::from([4, 3, 2, 1]));
```

Output:

```
[1, 2, 3, 4, 5, 6]
[3, 4]
[1, 2]
[1, 2, 5, 6]
true
```

`|` union, `&` intersection, `-` difference, `^` symmetric difference. The
`&a` is required — there is no `a | b` by value. Equality is content-based, so
insertion order never shows up.

## Deduplicating a Vec

Three idioms, three different guarantees about order.

```rust
let v = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3];

let mut uniq: Vec<i32> = v.iter().copied().collect::<HashSet<_>>().into_iter().collect();
uniq.sort();
println!("{:?}", uniq);

let ordered: Vec<i32> = v.iter().copied().collect::<BTreeSet<_>>().into_iter().collect();
println!("{:?}", ordered);

let mut seen = HashSet::new();
let first_seen: Vec<i32> = v.iter().copied().filter(|x| seen.insert(*x)).collect();
println!("{:?}", first_seen);

println!("{}", v.len() != v.iter().collect::<HashSet<_>>().len());
```

Output:

```
[1, 2, 3, 4, 5, 6, 9]
[1, 2, 3, 4, 5, 6, 9]
[3, 1, 4, 5, 9, 2, 6]
true
```

Pick by what you need. `HashSet` round-trip is fastest but the order is
garbage until you sort. `BTreeSet` gives you sorted output in one step. The
`seen.insert` filter is the only one that preserves original order, and it is
a single pass. The last line is the standard "does this contain duplicates"
check.

Compare with `sort` + `dedup` from lesson 7.3: that one mutates in place
without allocating a set, and wins when the data is already nearly sorted or
`T` is cheap to compare but expensive to hash.

## HashSet order is arbitrary

```rust
let a: HashSet<&str> = ["a","b","c","d"].into();
let b: HashSet<&str> = ["a","b","c","d"].into();
println!("{:?}", a.iter().collect::<Vec<_>>());
println!("{:?}", b.iter().collect::<Vec<_>>());
println!("{:?}", BTreeSet::from(["a","b","c","d"]).iter().collect::<Vec<_>>());
```

Output from one run:

```
["c", "a", "b", "d"]
["d", "a", "b", "c"]
["a", "b", "c", "d"]
```

The two `HashSet`s hold identical data and iterate differently. The
`BTreeSet` line is the only one you can rely on.

## BTreeSet: ordered iteration

```rust
let mut s = BTreeSet::new();
s.insert("delta");
s.insert("alpha");
s.insert("charlie");
println!("{:?}", s);
println!("{:?}", s.iter().rev().collect::<Vec<_>>());
println!("{:?} {:?}", s.first(), s.last());

let mut t = s.clone();
println!("{:?} {:?}", t.pop_first(), t.pop_last());
println!("{:?}", t);
```

Output:

```
{"alpha", "charlie", "delta"}
["delta", "charlie", "alpha"]
Some("alpha") Some("delta")
Some("alpha") Some("delta")
{"charlie"}
```

`first`/`last`/`pop_first`/`pop_last` are all O(log n), making a `BTreeSet` a
reasonable ordered work queue when you also need membership tests.

## Ranges on BTreeSet

```rust
let n: BTreeSet<i32> = (1..=10).collect();
println!("{:?}", n.range(3..6).collect::<Vec<_>>());
println!("{:?}", n.range(3..=6).collect::<Vec<_>>());
println!("{:?}", n.range(8..).collect::<Vec<_>>());
println!("{}", n.range(..5).count());

use std::ops::Bound;
println!("{:?}", n.range((Bound::Excluded(3), Bound::Unbounded)).take(3).collect::<Vec<_>>());
```

Output:

```
[3, 4, 5]
[3, 4, 5, 6]
[8, 9, 10]
4
[4, 5, 6]
```

Same semantics as `BTreeMap::range` — any `RangeBounds`, O(log n) to find the
start, then a straight walk. This is the thing `HashSet` simply cannot do.

`split_off` and `append` are here too:

```rust
let mut m = BTreeSet::from([1, 2, 3, 4]);
let hi = m.split_off(&3);
println!("{:?} {:?}", m, hi);

let mut a = BTreeSet::from([1, 2]);
let mut b = BTreeSet::from([3]);
a.append(&mut b);
println!("{:?} {:?}", a, b);
```

Output:

```
{1, 2} {3, 4}
{1, 2, 3} {}
```

And the set operations yield sorted iterators:

```rust
let x = BTreeSet::from([1, 2, 3]);
let y = BTreeSet::from([2, 3, 4]);
println!("{:?}", x.union(&y).collect::<Vec<_>>());
println!("{:?}", x.intersection(&y).collect::<Vec<_>>());
println!("{:?}", x.symmetric_difference(&y).collect::<Vec<_>>());
```

Output:

```
[1, 2, 3, 4]
[2, 3]
[1, 4]
```

No sorting needed before printing — that is the whole advantage.

## Sizes

```rust
println!("{}", std::mem::size_of::<HashSet<u64>>());
println!("{}", std::mem::size_of::<BTreeSet<u64>>());
```

Output:

```
48
24
```

## Quick Reference

| Need | `HashSet` | `BTreeSet` |
|---|---|---|
| Bound on `T` | `Eq + Hash` | `Ord` |
| Create | `new`, `from([..])`, `.collect()`, `with_capacity(n)` | `new`, `from([..])`, `.collect()` |
| Add | `insert(x) -> bool` | same |
| Replace | `replace(x) -> Option<T>` | same |
| Test | `contains(&x) -> bool`, `get(&x) -> Option<&T>` | same |
| Delete | `remove(&x) -> bool`, `take(&x) -> Option<T>` | same |
| Delete by predicate | `retain(\|x\| ..)` | same |
| Empty | `clear()`, `drain()` | `clear()` |
| Size | `len()`, `is_empty()` | same |
| Union | `a.union(&b)` or `&a \| &b` | same, sorted |
| Intersection | `a.intersection(&b)` or `&a & &b` | same, sorted |
| Difference | `a.difference(&b)` or `&a - &b` | same, sorted |
| Symmetric difference | `a.symmetric_difference(&b)` or `&a ^ &b` | same, sorted |
| Predicates | `is_subset`, `is_superset`, `is_disjoint` | same |
| Iterate | `iter()`, `into_iter()` — arbitrary order | sorted, double-ended |
| Extremes | — (O(n) scan) | `first`, `last`, `pop_first`, `pop_last` |
| Range | — | `range(a..b)` |
| Split / merge | — | `split_off(&x)`, `append(&mut other)` |
| Lookup cost | O(1) average | O(log n) |

## Common Mistakes

**Calling `contains` before `insert`.** `insert` already tells you via its
`bool`. Two hashes for one decision.

**Printing a `HashSet` in a test assertion.** The order varies. Collect and
sort, or use `BTreeSet`.

**`a | b` without the references.** The operators are implemented on `&Set`.

**Expecting `difference` to be symmetric.** It is directional; you want
`symmetric_difference`.

**Collecting a set operation and expecting a set.** They yield iterators —
`.collect::<HashSet<_>>()` or use the operator forms.

**Reaching for a `HashSet<T>` where `T` is one byte.** For small dense integer
domains a `[bool; N]` or a bitset is far faster; a set has per-element
overhead.

## Exercises

1. Given two `Vec<&str>` of tags, print the tags in both, in only the first,
   and in either — all sorted.
2. Write `fn has_duplicates<T: Eq + std::hash::Hash>(v: &[T]) -> bool` using a
   `HashSet` in one pass that short-circuits.
3. Deduplicate `vec!["b","a","c","a","b"]` three ways: `HashSet`, `BTreeSet`,
   and an order-preserving `seen` filter. Print all three.
4. Build a `BTreeSet<u32>` of 1..=50, then print every value between 20 and 30
   inclusive, and the count of values below 10.
5. Model two users' permission sets as `HashSet<&str>`. Print whether one is a
   subset of the other, their shared permissions, and the permissions unique
   to each.

---

[← 7.6 BTreeMap and Ordered Data](06-btreemap.md) · [Course Index](../README.md) · [7.8 VecDeque, BinaryHeap, LinkedList →](08-other-collections.md)
