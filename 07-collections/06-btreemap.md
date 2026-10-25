[← 7.5 HashMap Operations Reference](05-hashmap-operations.md) · [Course Index](../README.md) · [7.7 HashSet and BTreeSet →](07-sets.md)

# 7.6 — BTreeMap and Ordered Data

## What You'll Learn

- What `BTreeMap` gives you that `HashMap` does not: order and ranges
- `range(a..b)` and explicit `Bound`s
- `first_key_value`, `last_key_value`, `pop_first`, `pop_last`, `split_off`
- When the O(log n) is worth it

## Coming From Another Language

| Other | Rust |
|---|---|
| JS — nothing built in | `BTreeMap<K, V>` |
| Python `sortedcontainers.SortedDict` | `BTreeMap<K, V>` |
| Go — nothing built in | `BTreeMap<K, V>` |
| Java `TreeMap` | `BTreeMap<K, V>` |

Same API shape as `HashMap` — `insert`, `get`, `remove`, `entry` all behave
identically — plus ordered iteration and range queries.

```rust
use std::collections::BTreeMap;
```

## BTreeMap versus HashMap

| | `HashMap` | `BTreeMap` |
|---|---|---|
| Key bound | `Eq + Hash` | `Ord` |
| get / insert / remove | O(1) average | O(log n) |
| Iteration order | arbitrary, varies per map | sorted by key, always |
| Range queries | no | yes, O(log n + k) |
| Min / max key | O(n) scan | O(log n) |
| Memory per entry | higher (load factor headroom) | lower, but pointer-heavy nodes |
| Worst case | O(n) on a hash attack | O(log n) guaranteed |

Measured on 500,000 `u64` keys, release build:

```rust
use std::collections::{BTreeMap, HashMap};
use std::time::Instant;

let n = 500_000u64;
let mut h = HashMap::new();
let t = Instant::now();
for i in 0..n { h.insert(i, i); }
let hi = t.elapsed();
let t = Instant::now();
let mut s = 0u64;
for i in 0..n { s += h[&i]; }
let hg = t.elapsed();

let mut b = BTreeMap::new();
let t = Instant::now();
for i in 0..n { b.insert(i, i); }
let bi = t.elapsed();
let t = Instant::now();
let mut s2 = 0u64;
for i in 0..n { s2 += b[&i]; }
let bg = t.elapsed();

println!("HashMap  insert {:?}  lookup {:?}", hi, hg);
println!("BTreeMap insert {:?}  lookup {:?}", bi, bg);
println!("{}", s == s2);
```

Output:

```
HashMap  insert 25.094208ms  lookup 5.4605ms
BTreeMap insert 26.964833ms  lookup 15.626291ms
true
```

Lookups are about 3x slower, inserts roughly comparable at this size. The
`log n` is small and cache-friendly — `BTreeMap` is not a linked tree, it is a
B-tree with 11 elements per node. If you need ordering, the cost is modest
enough that you should not contort your design to avoid it.

Struct sizes:

```rust
println!("{}", std::mem::size_of::<BTreeMap<u32, u32>>());
println!("{}", std::mem::size_of::<HashMap<u32, u32>>());
```

Output:

```
24
48
```

## Iteration in key order

```rust
let mut m = BTreeMap::new();
m.insert("delta", 4);
m.insert("alpha", 1);
m.insert("charlie", 3);
m.insert("bravo", 2);

println!("{:?}", m);
for (k, v) in &m { println!("{k} = {v}"); }
println!("{:?}", m.keys().collect::<Vec<_>>());
println!("{:?}", m.values().collect::<Vec<_>>());
println!("{:?}", m.iter().rev().collect::<Vec<_>>());
```

Output:

```
{"alpha": 1, "bravo": 2, "charlie": 3, "delta": 4}
alpha = 1
bravo = 2
charlie = 3
delta = 4
["alpha", "bravo", "charlie", "delta"]
[1, 2, 3, 4]
[("delta", 4), ("charlie", 3), ("bravo", 2), ("alpha", 1)]
```

Every one of those is deterministic across runs — the whole point. `.rev()` is
free because the iterator is double-ended.

Construction is the same as `HashMap`:

```rust
let n = BTreeMap::from([(3, "c"), (1, "a"), (2, "b")]);
println!("{:?}", n);

let c: BTreeMap<i32, &str> = vec![(2, "b"), (1, "a")].into_iter().collect();
println!("{:?}", c);

let d: BTreeMap<i32, i32> = Default::default();
println!("{:?}", d);
```

Output:

```
{1: "a", 2: "b", 3: "c"}
{1: "a", 2: "b"}
{}
```

There is no `BTreeMap::with_capacity` — nodes are allocated as needed.

## Range queries

`range` takes anything implementing `RangeBounds<K>` and yields the entries
whose keys fall inside, in order.

```rust
let m = BTreeMap::from([(1,"a"),(2,"b"),(3,"c"),(4,"d"),(5,"e")]);
println!("{:?}", m.range(2..4).collect::<Vec<_>>());
println!("{:?}", m.range(2..=4).collect::<Vec<_>>());
println!("{:?}", m.range(3..).collect::<Vec<_>>());
println!("{:?}", m.range(..3).collect::<Vec<_>>());
println!("{}", m.range(..).count());
println!("{:?}", m.range(10..20).collect::<Vec<_>>());
println!("{:?}", m.range(2..4).rev().collect::<Vec<_>>());
```

Output:

```
[(2, "b"), (3, "c")]
[(2, "b"), (3, "c"), (4, "d")]
[(3, "c"), (4, "d"), (5, "e")]
[(1, "a"), (2, "b")]
5
[]
[(3, "c"), (2, "b")]
```

An empty range is not an error. The bounds need not exist as keys — `range`
finds the position, so `m.range(2.5..)` style queries work on any `Ord` type.

## Explicit Bounds

`a..b` cannot express "exclusive at the low end". `Bound` can.

```rust
use std::ops::Bound;

let m = BTreeMap::from([(1,"a"),(2,"b"),(3,"c"),(4,"d"),(5,"e")]);
println!("{:?}", m.range((Bound::Excluded(2), Bound::Included(4))).collect::<Vec<_>>());
```

Output:

```
[(3, "c"), (4, "d")]
```

The three variants are `Included(x)`, `Excluded(x)`, `Unbounded`. Note that
`range` panics if the start bound is greater than the end bound, or if both
are `Excluded` on the same value.

`range_mut` is the mutable form:

```rust
let mut mm = BTreeMap::from([(1,"a"),(2,"b"),(3,"c"),(4,"d"),(5,"e")]);
for (_, v) in mm.range_mut(2..4) { *v = "X"; }
println!("{:?}", mm);
```

Output:

```
{1: "a", 2: "X", 3: "X", 4: "d", 5: "e"}
```

## First and last

```rust
let m = BTreeMap::from([(1,"a"),(2,"b"),(3,"c"),(4,"d"),(5,"e")]);
println!("{:?}", m.first_key_value());
println!("{:?}", m.last_key_value());

let e: BTreeMap<i32, i32> = BTreeMap::new();
println!("{:?} {:?}", e.first_key_value(), e.last_key_value());
```

Output:

```
Some((1, "a"))
Some((5, "e"))
None None
```

Both O(log n). On a `HashMap` you would write `m.iter().min_by_key(..)` and
pay O(n).

`pop_first` and `pop_last` remove and return, which makes a `BTreeMap` a
usable priority queue keyed on anything `Ord`:

```rust
let mut mm = BTreeMap::from([(1,"a"),(2,"b"),(3,"c"),(4,"d"),(5,"e")]);
println!("{:?}", mm.pop_first());
println!("{:?}", mm.pop_last());
println!("{:?}", mm);
```

Output:

```
Some((1, "a"))
Some((5, "e"))
{2: "b", 3: "c", 4: "d"}
```

`first_entry` and `last_entry` hand back an `OccupiedEntry` for in-place
mutation:

```rust
let mut mm = BTreeMap::from([(1,"a"),(2,"b"),(3,"c"),(4,"d"),(5,"e")]);
if let Some(mut e) = mm.first_entry() { *e.get_mut() = "FIRST"; }
if let Some(mut e) = mm.last_entry() { *e.get_mut() = "LAST"; }
println!("{:?}", mm);
```

Output:

```
{1: "FIRST", 2: "b", 3: "c", 4: "d", 5: "LAST"}
```

## split_off and append

`split_off(&k)` keeps everything below `k` and returns everything from `k` up.

```rust
let mut m = BTreeMap::from([(1,"a"),(2,"b"),(3,"c"),(4,"d")]);
let hi = m.split_off(&3);
println!("lo={:?} hi={:?}", m, hi);

let mut a = BTreeMap::from([(1,"a")]);
let mut b = BTreeMap::from([(2,"b")]);
a.append(&mut b);
println!("{:?} b={:?}", a, b);
```

Output:

```
lo={1: "a", 2: "b"} hi={3: "c", 4: "d"}
{1: "a", 2: "b"} b={}
```

`append` moves everything out of the other map, leaving it empty. On key
conflict, the appended map wins. The split key itself goes to the **high**
half.

## The same entry API

Everything from lesson 7.5 carries over unchanged.

```rust
let mut m: BTreeMap<&str, i32> = BTreeMap::new();
*m.entry("x").or_insert(0) += 1;
m.entry("y").or_default();
m.entry("x").and_modify(|v| *v += 10).or_insert(0);
println!("{:?}", m);
m.retain(|_, v| *v > 0);
println!("{:?}", m);
```

Output:

```
{"x": 11, "y": 0}
{"x": 11}
```

`or_insert_with`, `or_insert_with_key`, `Entry::Occupied` / `Vacant` matching,
`remove_entry`, `extend`, `into_keys`, `into_values`, `drain`-less `clear` —
all present. The only gaps versus `HashMap` are the capacity methods and
`drain()` (use `split_off(&min)` or `std::mem::take`).

## Ord instead of Hash

A `BTreeMap` key needs `Ord`, not `Hash`. That is strictly easier for most
custom types, and it gives the ordering meaning:

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Ver(u32, u32);

let mut m = BTreeMap::new();
m.insert(Ver(1, 2), "old");
m.insert(Ver(1, 10), "new");
m.insert(Ver(0, 9), "ancient");
println!("{:?}", m);
println!("{:?}", m.keys().next());
```

Output:

```
{Ver(0, 9): "ancient", Ver(1, 2): "old", Ver(1, 10): "new"}
Some(Ver(0, 9))
```

Derived `Ord` on a tuple struct compares fields left to right, which is
exactly semver ordering here. Without the derive:

```rust
#[derive(Debug, PartialEq, Eq, Hash)]
struct NoOrd(u32);

let mut m = BTreeMap::new();
m.insert(NoOrd(1), "x");
```

Does not compile:

```
error[E0277]: the trait bound `NoOrd: Ord` is not satisfied
 --> src/main.rs:6:7
  |
6 |     m.insert(NoOrd(1), "x");
  |       ^^^^^^ -------- `NoOrd` doesn't satisfy the trait bound
  |       |
  |       the trait `Ord` is not implemented for `NoOrd`
```

`f64` keys are still out: `f64` is `PartialOrd` but not `Ord`.

## When ordering matters

Reach for `BTreeMap` when:

- **Output must be deterministic.** Serialized config, a snapshot test, a
  rendered report, a diffable file. A `HashMap` forces you to collect and sort
  every single time.
- **You query by range.** Time-series buckets, a sorted index, version
  lookups, "all orders between these dates".
- **You need min or max repeatedly.** O(log n) instead of a full scan.
- **Keys are `Ord` but not `Hash`,** or hashing them is expensive.
- **You need a worst-case bound.** No adversarial input can degrade it.

Stay with `HashMap` when you only ever do point lookups and the order is
irrelevant — it is the faster default, and lesson 7.9 has the full decision
table.

## Quick Reference

| Need | Method | Cost |
|---|---|---|
| Create | `new()`, `from([..])`, `.collect()` | |
| Read | `get(&k)`, `m[&k]`, `get_key_value(&k)` | O(log n) |
| Write | `insert(k, v)` | O(log n) |
| Upsert | `entry(k).or_insert(d)` etc. | O(log n) |
| Delete | `remove(&k)`, `remove_entry(&k)` | O(log n) |
| Delete by predicate | `retain(\|k, v\| ..)` | O(n) |
| Sorted iteration | `iter()`, `keys()`, `values()`, `.rev()` | O(n) |
| Mutable iteration | `iter_mut()`, `values_mut()` | O(n) |
| Range | `range(a..b)`, `range(a..=b)`, `range(..)` | O(log n + k) |
| Explicit bounds | `range((Bound::Excluded(a), Bound::Included(b)))` | |
| Mutable range | `range_mut(a..b)` | |
| Smallest / largest | `first_key_value()`, `last_key_value()` | O(log n) |
| Pop smallest / largest | `pop_first()`, `pop_last()` | O(log n) |
| Mutate extremes | `first_entry()`, `last_entry()` | O(log n) |
| Split at a key | `split_off(&k)` | O(n) |
| Merge | `append(&mut other)` | O(n) |
| Consume | `into_iter()`, `into_keys()`, `into_values()` | O(n) |

## Common Mistakes

**Expecting `with_capacity`.** It does not exist; there is nothing to
preallocate.

**`range(5..1)`.** A start bound above the end bound panics, it does not
return empty.

**Thinking `split_off(&k)` keeps `k`.** The key goes into the returned high
half.

**Using `BTreeMap` for an `f64` key.** `f64` is not `Ord`. Quantize to an
integer.

**Reaching for `HashMap` and then sorting on every output.** If you sort every
time you read, you wanted a `BTreeMap`.

## Exercises

1. Build a `BTreeMap<u32, &str>` of five events keyed by timestamp, print them
   in order, then print only those between two timestamps using `range`.
2. Use `Bound::Excluded` to print all entries strictly between two keys, and
   show it differs from `a..=b` on the same data.
3. Write `fn drain_ascending(m: &mut BTreeMap<i32, String>) -> Vec<(i32, String)>`
   using `pop_first` in a loop.
4. Define `struct SemVer(u32, u32, u32)` with the derives needed to be a
   `BTreeMap` key, insert four versions out of order, and print the newest via
   `last_key_value`.
5. Split a `BTreeMap<u32, u32>` of 1..=10 at key 6 with `split_off`, print
   both halves, then `append` them back and confirm the result equals the
   original.

---

[← 7.5 HashMap Operations Reference](05-hashmap-operations.md) · [Course Index](../README.md) · [7.7 HashSet and BTreeSet →](07-sets.md)
