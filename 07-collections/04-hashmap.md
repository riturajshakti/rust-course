[← 7.3 Sorting, Searching, and Dedup](03-sort-search.md) · [Course Index](../README.md) · [7.5 HashMap Operations Reference →](05-hashmap-operations.md)

# 7.4 — HashMap

## What You'll Learn

- Constructing a `HashMap`, and why `insert` returns an `Option`
- The entry API — the single most useful thing in this lesson
- Iteration, and that the order is genuinely arbitrary
- Key requirements (`Eq + Hash`), and `String` vs `&str` keys

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `new Map()` / `{}` | `HashMap::new()` |
| Python `dict` | `HashMap<K, V>` |
| Go `map[string]int` | `HashMap<String, i32>` |

Two differences that will bite. `m[k]` on a missing key **panics** rather than
returning `undefined`/`nil`/a zero value — use `get`, which returns
`Option<&V>`. And there is no `m[k] += 1` on a key that might not exist; that
is what the entry API is for.

`HashMap` is not in the prelude:

```rust
use std::collections::HashMap;
```

## Constructing

```rust
let mut m: HashMap<String, i32> = HashMap::new();
m.insert("a".to_string(), 1);
println!("{:?}", m);

let n = HashMap::from([("a", 1), ("b", 2)]);
println!("{}", n.len());

let p: HashMap<&str, i32> = HashMap::with_capacity(10);
println!("len={} cap>=10={}", p.len(), p.capacity() >= 10);

let q: HashMap<&str, i32> = vec![("x", 1), ("y", 2)].into_iter().collect();
println!("{}", q.len());

let r: HashMap<&str, i32> = HashMap::default();
println!("{:?}", r);
```

Output:

```
{"a": 1}
2
len=0 cap>=10=true
2
{}
```

`HashMap::from([..])` takes an array of tuples. `collect()` works on any
iterator of `(K, V)`, which is how you build a map from a `Vec` of pairs or a
`zip` of two iterators.

## insert returns the old value

```rust
let mut s = HashMap::new();
println!("{:?}", s.insert("k", 1));
println!("{:?}", s.insert("k", 2));
println!("{:?}", s);
```

Output:

```
None
Some(1)
{"k": 2}
```

`None` means the key was new. That return value is how you detect a
duplicate-key overwrite — there is no separate `try_insert` on stable, so
check `insert(...).is_none()` or use `entry` (below) if an existing value must
be preserved.

## get, index, get_key_value

```rust
let m = HashMap::from([("a", 1), ("b", 2)]);
println!("{:?}", m.get("a"));
println!("{:?}", m.get("z"));
println!("{}", m["a"]);
println!("{}", m.get("z").copied().unwrap_or(0));
println!("{:?}", m.get_key_value("b"));
println!("{}", m.contains_key("a"));
println!("{}", m.len());
println!("{}", m.is_empty());
```

Output:

```
Some(1)
None
1
0
Some(("b", 2))
true
2
false
```

`m["z"]` would panic with `key not found`. `get_key_value` hands back the
stored key too, which matters when your `Eq` is looser than identity (case
insensitive keys, for instance).

## get_mut

```rust
let mut m = HashMap::from([("a", 1)]);
if let Some(v) = m.get_mut("a") { *v += 100; }
println!("{:?}", m);
```

Output:

```
{"a": 101}
```

Only works if the key exists. For "increment, inserting 0 if absent", use the
entry API.

## The entry API

`entry(k)` returns an `Entry` — a handle to a slot that may or may not be
occupied. This is the idiomatic way to do read-modify-write in one lookup.

`or_insert(default)` returns `&mut V`, inserting `default` if vacant:

```rust
let mut m: HashMap<&str, i32> = HashMap::new();
*m.entry("a").or_insert(0) += 1;
*m.entry("a").or_insert(0) += 1;
*m.entry("b").or_insert(0) += 1;

let mut v: Vec<_> = m.iter().collect();
v.sort();
println!("{:?}", v);
```

Output:

```
[("a", 2), ("b", 1)]
```

`or_insert_with(f)` calls `f` only when vacant — use it when the default is
expensive. `or_default()` is `or_insert(Default::default())`:

```rust
let mut lists: HashMap<&str, Vec<i32>> = HashMap::new();
lists.entry("odd").or_insert_with(Vec::new).push(1);
lists.entry("odd").or_default().push(3);
lists.entry("even").or_default().push(2);

let mut v: Vec<_> = lists.into_iter().collect();
v.sort();
println!("{:?}", v);
```

Output:

```
[("even", [2]), ("odd", [1, 3])]
```

`and_modify(f)` mutates an existing value and chains:

```rust
let mut m: HashMap<&str, i32> = HashMap::new();
m.entry("hits").and_modify(|x| *x += 1).or_insert(1);
m.entry("hits").and_modify(|x| *x += 1).or_insert(1);
println!("{:?}", m);
```

Output:

```
{"hits": 2}
```

First call: vacant, `and_modify` does nothing, `or_insert(1)` runs. Second
call: occupied, `and_modify` bumps it to 2, `or_insert` does nothing.

`or_insert_with_key(f)` passes the key to the default:

```rust
let mut m: HashMap<&str, usize> = HashMap::new();
m.entry("alpha").or_insert_with_key(|k| k.len());
println!("{:?}", m);
```

Output:

```
{"alpha": 5}
```

The win over `contains_key` + `get_mut` + `insert` is that `entry` hashes the
key **once**, and the borrow checker accepts it — a `contains_key` followed by
`get_mut` is two lookups, and the naive `if let Some(..) = m.get_mut(k) {} else { m.insert(..) }`
fails the borrow checker on older compilers and is just noisier on this one.

## remove and remove_entry

```rust
let mut m = HashMap::from([("a", 1), ("b", 2)]);
println!("{:?}", m.remove("a"));
println!("{:?}", m.remove("a"));
println!("{:?}", m.remove_entry("b"));
println!("{:?}", m);
```

Output:

```
Some(1)
None
Some(("b", 2))
{}
```

`remove` gives the value, `remove_entry` gives `(K, V)`. Both are `Option`,
and removing a missing key is not an error.

## Iteration, and arbitrary order

```rust
let m = HashMap::from([("a", 1), ("b", 2), ("c", 3)]);

let mut ks: Vec<_> = m.keys().collect(); ks.sort();
let mut vs: Vec<_> = m.values().collect(); vs.sort();
println!("{:?}", ks);
println!("{:?}", vs);

let mut pairs: Vec<_> = m.iter().collect(); pairs.sort();
println!("{:?}", pairs);

let mut mm = m.clone();
for v in mm.values_mut() { *v *= 10; }
let mut p: Vec<_> = mm.iter().collect(); p.sort();
println!("{:?}", p);

let mut owned: Vec<_> = mm.into_iter().collect(); owned.sort();
println!("{:?}", owned);
```

Output:

```
["a", "b", "c"]
[1, 2, 3]
[("a", 1), ("b", 2), ("c", 3)]
[("a", 10), ("b", 20), ("c", 30)]
[("a", 10), ("b", 20), ("c", 30)]
```

Every one of those was sorted before printing, deliberately. Iteration order
is **not** insertion order and **not** key order — it is a function of the
hash values and the random seed chosen per map. Two maps built from identical
data in the same program yield different orders:

```rust
let a = HashMap::from([("a",1),("b",2),("c",3),("d",4)]);
let b = HashMap::from([("a",1),("b",2),("c",3),("d",4)]);
println!("{:?}", a.keys().collect::<Vec<_>>());
println!("{:?}", b.keys().collect::<Vec<_>>());
```

Output from one run:

```
["a", "d", "c", "b"]
["a", "d", "b", "c"]
```

Your run will print something else. Never write a test, a log format, or a
serialized output that depends on this order — collect into a `Vec` and sort,
or use a `BTreeMap` (lesson 7.6).

`iter()` yields `(&K, &V)`; `iter_mut()` and `values_mut()` give mutable
values but never mutable keys (mutating a key would invalidate its hash
bucket). `into_iter()` yields owned `(K, V)` and consumes the map.

## Counting words

The canonical entry-API example.

```rust
let text = "the quick brown fox jumps over the lazy dog the fox";
let mut counts: HashMap<&str, usize> = HashMap::new();
for w in text.split_whitespace() {
    *counts.entry(w).or_insert(0) += 1;
}

let mut sorted: Vec<_> = counts.into_iter().collect();
sorted.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
for (w, c) in &sorted { println!("{c:2} {w}"); }
```

Output:

```
 3 the
 2 fox
 1 brown
 1 dog
 1 jumps
 1 lazy
 1 over
 1 quick
```

Count descending, then name ascending, so the output is deterministic.

## Grouping into HashMap<K, Vec<V>>

```rust
let people = vec![("raj", "pune"), ("sam", "delhi"), ("ana", "pune")];
let mut by_city: HashMap<&str, Vec<&str>> = HashMap::new();
for (n, c) in people {
    by_city.entry(c).or_default().push(n);
}

let mut g: Vec<_> = by_city.into_iter().collect();
g.sort();
println!("{:?}", g);
```

Output:

```
[("delhi", ["sam"]), ("pune", ["raj", "ana"])]
```

`or_default().push(x)` is the whole pattern. Within each group, order is
insertion order — the `Vec` preserves it even though the map does not.

## Key requirements: Eq + Hash

Any type can be a key if it derives both.

```rust
#[derive(Debug, PartialEq, Eq, Hash)]
struct Key { a: u8, b: char }

let mut m = HashMap::new();
m.insert(Key { a: 1, b: 'x' }, "hit");
println!("{:?}", m.get(&Key { a: 1, b: 'x' }));
println!("{:?}", m.get(&Key { a: 2, b: 'x' }));
```

Output:

```
Some("hit")
None
```

Without the derives:

```rust
#[derive(Debug)]
struct NoHash { x: i32 }

let mut m = HashMap::new();
m.insert(NoHash { x: 1 }, "v");
```

Does not compile:

```
error[E0277]: the trait bound `NoHash: Eq` is not satisfied
 --> src/main.rs:6:7
  |
6 |     m.insert(NoHash { x: 1 }, "v");
  |       ^^^^^^ --------------- `NoHash` doesn't satisfy the trait bound
  |       |
  |       the trait `Eq` is not implemented for `NoHash`
  |
help: consider annotating `NoHash` with `#[derive(Eq)]`
  |
3 + #[derive(Eq)]
4 | struct NoHash { x: i32 }
```

A second error follows for `Hash`. `f64` can never be a key — it is not `Eq`,
for the same `NaN` reason as in lesson 7.3. Tuples, `String`, `&str`, all
integers, `char`, `bool`, `Vec<T>` where `T: Hash`, and `Option<T>` all work
out of the box.

The invariant you must not break: equal keys must hash equally, and a key's
hash must not change while it is in the map. Deriving both traits gives you
that for free; hand-writing `Hash` or `PartialEq` on a key type is how people
get silently lost entries.

## String versus &str keys

A `HashMap<String, V>` can be looked up with a `&str` — no allocation needed.

```rust
let mut m: HashMap<String, i32> = HashMap::new();
m.insert("owned".to_string(), 1);
println!("{:?}", m.get("owned"));
println!("{:?}", m.get(&"owned".to_string()));
println!("{}", m.contains_key("owned"));

m.insert(String::from("x"), 2);
m.insert("y".into(), 3);
m.insert("z".to_owned(), 4);
println!("{}", m.len());
println!("{}", m.values().sum::<i32>());
println!("{:?}", m.values().max());
```

Output:

```
Some(1)
Some(1)
true
4
10
Some(4)
```

That `get("owned")` works because `get` is generic over `Q: Borrow`-compatible
types, and `String: Borrow<str>`. So: use `HashMap<String, V>` for anything
you own or store, and still query it with plain string literals.
`HashMap<&str, V>` only works when the keys outlive the map — fine for
literals, painful for parsed input.

## Quick Reference

| Operation | Form | Returns |
|---|---|---|
| New | `HashMap::new()` | empty |
| From pairs | `HashMap::from([(k,v), ..])` | map |
| From iterator | `iter.collect()` | map |
| Preallocate | `HashMap::with_capacity(n)` | empty |
| Insert | `m.insert(k, v)` | `Option<V>` (the old value) |
| Read | `m.get(&k)` | `Option<&V>` |
| Read, panicking | `m[&k]` | `&V` |
| Read key + value | `m.get_key_value(&k)` | `Option<(&K, &V)>` |
| Mutate existing | `m.get_mut(&k)` | `Option<&mut V>` |
| Upsert | `*m.entry(k).or_insert(d) += 1` | `&mut V` |
| Lazy default | `m.entry(k).or_insert_with(f)` | `&mut V` |
| `Default` default | `m.entry(k).or_default()` | `&mut V` |
| Modify if present | `m.entry(k).and_modify(f).or_insert(d)` | `&mut V` |
| Delete | `m.remove(&k)` | `Option<V>` |
| Delete with key | `m.remove_entry(&k)` | `Option<(K, V)>` |
| Membership | `m.contains_key(&k)` | `bool` |
| Size | `m.len()`, `m.is_empty()` | |
| Iterate pairs | `m.iter()`, `m.into_iter()` | `(&K,&V)` / `(K,V)` |
| Iterate keys | `m.keys()` | `&K` |
| Iterate values | `m.values()`, `m.values_mut()` | `&V` / `&mut V` |

## Common Mistakes

**Using `m[k]`.** It panics on a missing key. Use `m.get(k)`.

**`contains_key` then `insert` then `get_mut`.** Three hashes of the same key.
Use `entry`.

**Relying on iteration order.** It varies between maps and between runs.
Sort, or use `BTreeMap`.

**`HashMap<&str, V>` for parsed input.** The keys borrow from a buffer that
usually dies first. Use `String` keys.

**`f64` as a key.** It is not `Eq`. Quantize it to an integer, or use ordered
bits.

**Expecting `*m.get_mut(k).unwrap() += 1` to work on a new key.** `get_mut`
returns `None`. `*m.entry(k).or_insert(0) += 1` handles both cases.

## Exercises

1. Build a `HashMap<String, u32>` of three people's ages. Print one age via
   `get`, print a missing one with a default of 0, and print the map's size.
2. Count character frequencies in `"mississippi"` with the entry API, then
   print them sorted by count descending.
3. Given `vec![("a", 1), ("b", 2), ("a", 3)]`, build a
   `HashMap<&str, Vec<i32>>` grouping values by key, then print it sorted by
   key.
4. Write `fn merge(a: HashMap<String, i32>, b: HashMap<String, i32>) -> HashMap<String, i32>`
   that sums values for keys present in both.
5. Define `struct Point { x: i32, y: i32 }` with the right derives, use it as
   a key in a `HashMap<Point, char>` representing a grid, and look up both a
   present and an absent point.

---

[← 7.3 Sorting, Searching, and Dedup](03-sort-search.md) · [Course Index](../README.md) · [7.5 HashMap Operations Reference →](05-hashmap-operations.md)
