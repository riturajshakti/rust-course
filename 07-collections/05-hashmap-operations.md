[← 7.4 HashMap](04-hashmap.md) · [Course Index](../README.md) · [7.6 BTreeMap and Ordered Data →](06-btreemap.md)

# 7.5 — HashMap Operations Reference

## What You'll Learn

- Every `HashMap` method, grouped by purpose, with signature and return type
- The full `Entry` / `OccupiedEntry` / `VacantEntry` surface
- Bulk operations: `extend`, `retain`, `drain`, `into_keys`, `into_values`
- The default hasher, when to replace it, and nested maps

## Coming From Another Language

Three facts that explain most of the API shape:

- Anything that can miss returns `Option`. `insert` returns the **old** value,
  `remove` returns the removed value, `get` returns `Option<&V>`.
- Lookups are generic over borrowed forms: a `HashMap<String, V>` is queried
  with `&str`, a `HashMap<Vec<u8>, V>` with `&[u8]`. No allocation to look
  something up.
- There is no `try_insert` on stable. `Entry` covers it, and much more.

Every example below sorts before printing where the output would otherwise be
order-dependent.

## Group 1: Create

| Method | Signature | Returns | Note |
|---|---|---|---|
| `HashMap::new` | `() -> HashMap<K,V>` | empty | no allocation |
| `HashMap::with_capacity` | `(usize) -> HashMap<K,V>` | empty | room for `n` without rehash |
| `HashMap::from` | `([(K,V); N]) -> HashMap<K,V>` | map | array of pairs |
| `HashMap::default` | `() -> HashMap<K,V>` | empty | same as `new` |
| `.collect()` | `(I: Iterator<Item=(K,V)>)` | map | from any pair iterator |
| `HashMap::with_hasher` | `(S) -> HashMap<K,V,S>` | empty | custom hasher |
| `HashMap::with_capacity_and_hasher` | `(usize, S)` | empty | both |
| `.clone()` | `(&self) -> HashMap<K,V>` | deep copy | `K, V: Clone` |

```rust
use std::collections::HashMap;
use std::collections::hash_map::RandomState;

let a: HashMap<&str, i32> = HashMap::new();
let b: HashMap<&str, i32> = HashMap::with_capacity(8);
let c = HashMap::from([("a", 1), ("b", 2)]);
let d: HashMap<&str, i32> = [("x", 1)].into_iter().collect();
let e: HashMap<&str, i32> = Default::default();
let f: HashMap<&str, i32> = HashMap::with_hasher(RandomState::new());
let g: HashMap<&str, i32> = HashMap::with_capacity_and_hasher(4, RandomState::new());

let ks = vec!["p", "q"];
let vs = vec![1, 2];
let h: HashMap<&str, i32> = ks.iter().copied().zip(vs).collect();

println!("{} {} {} {} {} {} {} {}",
    a.len(), b.capacity() >= 8, c.len(), d.len(),
    e.len(), f.len(), g.capacity() >= 4, h.len());

let j: HashMap<&str, i32> = c.iter().map(|(k, v)| (*k, v * 10)).collect();
let mut s: Vec<_> = j.into_iter().collect();
s.sort();
println!("{:?}", s);
```

Output:

```
0 true 2 1 0 0 true 2
[("a", 10), ("b", 20)]
```

`zip` + `collect` is the idiom for building a map from parallel key and value
lists. `iter().map(..).collect()` transforms a whole map in one expression.

## Group 2: Read

| Method | Signature | Returns | Note |
|---|---|---|---|
| `get` | `(&Q) -> Option<&V>` | value | `Q: Borrow`-compatible with `K` |
| `get_mut` | `(&Q) -> Option<&mut V>` | mutable value | key must exist |
| `get_key_value` | `(&Q) -> Option<(&K, &V)>` | stored key too | |
| `get_disjoint_mut` | `([&Q; N]) -> [Option<&mut V>; N]` | several at once | panics if keys overlap |
| `m[&k]` (`Index`) | `(&Q) -> &V` | **panics** if absent | read-only; no `IndexMut` |
| `contains_key` | `(&Q) -> bool` | | |
| `len` / `is_empty` | `() -> usize` / `bool` | | |
| `capacity` | `() -> usize` | slots before rehash | `>= len()` |
| `iter` | `() -> Iter<'_, K, V>` | `(&K, &V)` | arbitrary order |
| `iter_mut` | `() -> IterMut<'_, K, V>` | `(&K, &mut V)` | keys stay immutable |
| `keys` / `values` | `() -> Keys` / `Values` | `&K` / `&V` | |
| `values_mut` | `() -> ValuesMut` | `&mut V` | |
| `hasher` | `() -> &S` | the build-hasher | |

```rust
let mut m = HashMap::from([("a", 1), ("b", 2), ("c", 3)]);
println!("{:?}", m.get("a"));
println!("{:?}", m.get("zz"));
println!("{:?}", m.get_key_value("b"));
println!("{:?}", m.get_disjoint_mut(["a", "b"]));
println!("{}", m["c"]);
println!("{}", m.contains_key("a"));
println!("{} {}", m.len(), m.is_empty());

if let Some(v) = m.get_mut("a") { *v = 100; }
println!("{:?}", m.get("a"));

for (_, v) in m.iter_mut() { *v += 1; }
let mut it: Vec<_> = m.iter().collect(); it.sort();
println!("{:?}", it);

for v in m.values_mut() { *v *= 2; }
let mut it: Vec<_> = m.iter().collect(); it.sort();
println!("{:?}", it);

println!("{}", m.values().sum::<i32>());
println!("{:?}", m.iter().max_by_key(|(_, v)| **v));
println!("{:?}", m.iter().find(|(k, _)| **k == "b"));
```

Output:

```
Some(1)
None
Some(("b", 2))
[Some(1), Some(2)]
3
true
3 false
Some(100)
[("a", 101), ("b", 3), ("c", 4)]
[("a", 202), ("b", 6), ("c", 8)]
216
Some(("a", 202))
Some(("b", 6))
```

`get_disjoint_mut` is the escape hatch for the borrow checker when you need
two mutable values at once — it panics if you pass the same key twice.

There is no `IndexMut`, so `m["a"] = 5` does not compile. That is deliberate:
it would have to either panic or insert, and neither is obviously right.

## Group 3: Insert and extend

| Method | Signature | Returns | Note |
|---|---|---|---|
| `insert` | `(K, V) -> Option<V>` | previous value | `None` if the key was new |
| `extend` | `(I: IntoIterator<Item=(K,V)>)` | `()` | later duplicates win |
| `extend` (refs) | `(I: IntoIterator<Item=(&K,&V)>)` | `()` | `K, V: Copy` |
| `entry` | `(K) -> Entry<'_, K, V>` | a slot handle | see next group |

```rust
let mut m: HashMap<&str, i32> = HashMap::new();
println!("{:?}", m.insert("a", 1));
println!("{:?}", m.insert("a", 9));

m.extend([("b", 2), ("c", 3)]);
println!("{}", m.len());

let other = HashMap::from([("d", 4)]);
m.extend(other);
m.extend(vec![("e", 5)]);

let mut s: Vec<_> = m.iter().collect();
s.sort();
println!("{:?}", s);
```

Output:

```
None
Some(1)
3
[("a", 9), ("b", 2), ("c", 3), ("d", 4), ("e", 5)]
```

`extend` accepts arrays, `Vec`s, other maps, and any iterator of pairs. It
overwrites on conflict — there is no "keep existing" bulk form, so fold with
`entry` if you need that.

## Group 4: The Entry API in full

`entry(k)` consumes the key and hands back an `Entry`, which is either
`Occupied` or `Vacant`.

| Method | Signature | Returns | Note |
|---|---|---|---|
| `Entry::or_insert` | `(V) -> &mut V` | the value | inserts if vacant; `V` built eagerly |
| `Entry::or_insert_with` | `(F: FnOnce() -> V) -> &mut V` | the value | `F` runs only if vacant |
| `Entry::or_insert_with_key` | `(F: FnOnce(&K) -> V) -> &mut V` | the value | default derived from the key |
| `Entry::or_default` | `() -> &mut V` | the value | `V: Default` |
| `Entry::and_modify` | `(F: FnOnce(&mut V)) -> Entry` | the entry | chains; no-op if vacant |
| `Entry::key` | `() -> &K` | the key | works in either state |
| `OccupiedEntry::get` | `() -> &V` | | |
| `OccupiedEntry::get_mut` | `() -> &mut V` | | |
| `OccupiedEntry::into_mut` | `() -> &mut V` | longer-lived borrow | consumes the entry |
| `OccupiedEntry::insert` | `(V) -> V` | the old value | |
| `OccupiedEntry::remove` | `() -> V` | the value | |
| `OccupiedEntry::remove_entry` | `() -> (K, V)` | both | |
| `VacantEntry::insert` | `(V) -> &mut V` | the new value | |
| `VacantEntry::into_key` | `() -> K` | the key back | if you decide not to insert |

```rust
let mut m: HashMap<&str, i32> = HashMap::new();
println!("{}", *m.entry("a").or_insert(1));
println!("{}", *m.entry("a").or_insert(99));
println!("{}", *m.entry("b").or_insert_with(|| 7));
println!("{}", *m.entry("c").or_default());

let mut n: HashMap<&str, usize> = HashMap::new();
println!("{}", *n.entry("hello").or_insert_with_key(|k| k.len()));

m.entry("a").and_modify(|v| *v += 10).or_insert(0);
println!("{:?}", m.get("a"));
println!("{}", m.entry("a").key());
```

Output:

```
1
1
7
0
5
Some(11)
a
```

The second `or_insert(99)` did nothing — the key existed.

### try_insert, via Entry

There is no `try_insert` on stable, but matching on the entry gives you the
same thing with full control:

```rust
use std::collections::hash_map::Entry;

let mut m = HashMap::from([("a", 1)]);
match m.entry("a") {
    Entry::Occupied(e) => println!("exists: {}", e.get()),
    Entry::Vacant(e) => { e.insert(5); println!("inserted"); }
}
match m.entry("z") {
    Entry::Occupied(e) => println!("exists: {}", e.get()),
    Entry::Vacant(e) => { e.insert(5); println!("inserted"); }
}

let mut s: Vec<_> = m.iter().collect();
s.sort();
println!("{:?}", s);
```

Output:

```
exists: 1
inserted
[("a", 1), ("z", 5)]
```

The whole `OccupiedEntry` surface in one block:

```rust
let mut m = HashMap::from([("a", 1)]);
if let Entry::Occupied(mut e) = m.entry("a") {
    println!("{} {}", e.key(), e.get());
    *e.get_mut() += 1;
    println!("{:?}", e.insert(50));
    println!("{:?}", e.remove());
}
println!("{:?}", m);

let mut m: HashMap<&str, i32> = HashMap::new();
if let Entry::Vacant(e) = m.entry("k") {
    println!("{}", e.key());
    println!("{}", *e.insert(3));
}
println!("{:?}", m);
```

Output:

```
a 1
2
50
{}
k
3
{"k": 3}
```

`e.insert(50)` returned the previous `2`; `e.remove()` then returned `50` and
emptied the map.

## Group 5: Remove and bulk removal

| Method | Signature | Returns | Note |
|---|---|---|---|
| `remove` | `(&Q) -> Option<V>` | the value | `None` if absent |
| `remove_entry` | `(&Q) -> Option<(K, V)>` | both | |
| `retain` | `(F: FnMut(&K, &mut V) -> bool)` | `()` | keeps where true; can mutate `V` |
| `drain` | `() -> Drain<'_, K, V>` | iterator of all `(K, V)` | empties the map, **keeps capacity** |
| `clear` | `()` | `()` | empties, keeps capacity |

```rust
let mut m = HashMap::from([("a",1),("b",2),("c",3),("d",4)]);
println!("{:?}", m.remove("a"));
println!("{:?}", m.remove("a"));
println!("{:?}", m.remove_entry("b"));

m.retain(|_, v| *v % 2 == 1);
let mut s: Vec<_> = m.iter().collect();
s.sort();
println!("{:?}", s);

let mut m = HashMap::from([("a",1),("b",2)]);
let mut drained: Vec<_> = m.drain().collect();
drained.sort();
println!("drained={:?} left={:?} cap_kept={}", drained, m, m.capacity() > 0);

let mut m = HashMap::from([("a",1),("b",2)]);
m.clear();
println!("{:?} len={} cap>0={}", m, m.len(), m.capacity() > 0);
```

Output:

```
Some(1)
None
Some(("b", 2))
[("c", 3)]
drained=[("a", 1), ("b", 2)] left={} cap_kept=true
{} len=0 cap>0=true
```

`retain` is the only way to remove while iterating — collecting the keys to
remove into a `Vec` first and then looping `remove` works but costs an extra
pass and allocation. `drain` differs from `clear` in handing you the items on
the way out, and from `into_iter` in leaving the map usable.

## Group 6: Convert and consume

| Method | Signature | Returns | Note |
|---|---|---|---|
| `into_iter` | `(self) -> IntoIter<K, V>` | owned `(K, V)` | consumes the map |
| `into_keys` | `(self) -> IntoKeys<K, V>` | owned `K` | consumes |
| `into_values` | `(self) -> IntoValues<K, V>` | owned `V` | consumes |
| `iter().collect()` | | `Vec`, `BTreeMap`, another `HashMap` | |
| `==` (`PartialEq`) | `(&self, &Self) -> bool` | | same pairs, order irrelevant |

```rust
let m = HashMap::from([("a",1),("b",2)]);
let mut k: Vec<_> = m.clone().into_keys().collect(); k.sort();
let mut v: Vec<_> = m.clone().into_values().collect(); v.sort();
println!("{:?} {:?}", k, v);

let mut p: Vec<_> = m.clone().into_iter().collect(); p.sort();
println!("{:?}", p);

let a = HashMap::from([("a",1),("b",2)]);
let b = HashMap::from([("b",2),("a",1)]);
println!("{}", a == b);
println!("{}", a == HashMap::from([("a",1)]));
```

Output:

```
["a", "b"] [1, 2]
[("a", 1), ("b", 2)]
true
false
```

Equality compares contents, not layout, so insertion order never matters. Use
`into_values()` rather than `values().cloned()` when you are discarding the
map anyway — it moves instead of cloning.

## Group 7: Capacity

| Method | Signature | Returns | Note |
|---|---|---|---|
| `capacity` | `() -> usize` | | |
| `reserve` | `(usize)` | `()` | room for `n` **more** |
| `try_reserve` | `(usize) -> Result<(), TryReserveError>` | | fallible |
| `shrink_to_fit` | `()` | `()` | down to fit `len` |
| `shrink_to` | `(usize)` | `()` | down to at least `n` |

```rust
let mut m: HashMap<&str, i32> = HashMap::new();
m.reserve(100);
println!("cap>=100 {}", m.capacity() >= 100);
m.insert("a", 1);
m.shrink_to_fit();
println!("len={} cap={}", m.len(), m.capacity());

let mut n: HashMap<&str, i32> = HashMap::with_capacity(100);
n.insert("a", 1);
n.shrink_to(10);
println!("cap>=10 {}", n.capacity() >= 10);
println!("{}", n.try_reserve(5).is_ok());
```

Output:

```
cap>=100 true
len=1 cap=3
cap>=10 true
true
```

Capacity for a hash map is not the number of buckets — the implementation
keeps headroom so the load factor stays low, which is why `shrink_to_fit` on a
one-element map reported 3.

## Nested maps

`or_default()` composes, which makes two-level maps painless.

```rust
let mut nested: HashMap<&str, HashMap<&str, i32>> = HashMap::new();
nested.entry("outer").or_default().insert("inner", 1);
nested.entry("outer").or_default().insert("inner2", 2);

println!("{:?}", nested.get("outer").and_then(|m| m.get("inner")));
println!("{:?}", nested.get("nope").and_then(|m| m.get("inner")));

*nested.entry("outer").or_default().entry("inner").or_insert(0) += 100;
println!("{}", nested["outer"]["inner"]);
println!("{}", nested.values().map(|m| m.len()).sum::<usize>());
```

Output:

```
Some(1)
None
101
2
```

`and_then` is the chained-lookup tool — it flattens the `Option<Option<&V>>`
you would otherwise get. Note `nested["a"]["b"]` panics at either level, so
reserve it for keys you know exist.

## The default hasher

`HashMap`'s default is SipHash-1-3 with a per-process random seed, chosen for
**HashDoS resistance**: an attacker who controls your keys cannot craft
collisions, because they do not know the seed. The costs are that it is slower
than a non-cryptographic hash, and that iteration order varies between maps.

Replace it when keys are trusted (internal IDs, enum discriminants, parsed
config) and hashing shows up in a profile. The ecosystem options are
`ahash` (fast, still somewhat DoS-resistant) and `rustc-hash`/`fxhash` (the
fastest, from the compiler itself, no DoS resistance). Both plug in as a type
alias:

```text
use std::collections::HashMap;
use rustc_hash::FxBuildHasher;

type FastMap<K, V> = HashMap<K, V, FxBuildHasher>;
let mut m: FastMap<u64, u64> = FastMap::default();
```

Only `new()` and `with_capacity()` are unavailable on a custom-hasher map —
use `default()` and `with_capacity_and_hasher()`. Everything else is
identical. Here is the same shape with a hand-rolled hasher, so it runs
without a dependency:

```rust
use std::collections::HashMap;
use std::hash::{BuildHasherDefault, Hasher};

#[derive(Default)]
struct Weak(u64);

impl Hasher for Weak {
    fn finish(&self) -> u64 { self.0 }
    fn write(&mut self, bytes: &[u8]) {
        for b in bytes {
            self.0 = self.0.wrapping_mul(31).wrapping_add(*b as u64);
        }
    }
}

let mut m: HashMap<u64, u64, BuildHasherDefault<Weak>> = HashMap::default();
m.insert(1, 10);
m.insert(2, 20);
println!("{:?} {:?}", m.get(&1), m.get(&3));
```

Output:

```
Some(10) None
```

Do not ship that one — it is a demonstration of the plumbing, not a good hash.

## Quick Reference

| Need | Method | Returns |
|---|---|---|
| Create empty | `new`, `default`, `with_capacity(n)` | map |
| Create from data | `from([..])`, `.collect()`, `zip().collect()` | map |
| Read | `get(&k)` | `Option<&V>` |
| Read or panic | `m[&k]` | `&V` |
| Read with key | `get_key_value(&k)` | `Option<(&K,&V)>` |
| Several mutably | `get_disjoint_mut([..])` | `[Option<&mut V>; N]` |
| Exists | `contains_key(&k)` | `bool` |
| Set | `insert(k, v)` | `Option<V>` old |
| Set many | `extend(iter)` | `()` |
| Upsert counter | `*m.entry(k).or_insert(0) += 1` | `&mut V` |
| Upsert lazily | `entry(k).or_insert_with(f)` | `&mut V` |
| Upsert from key | `entry(k).or_insert_with_key(f)` | `&mut V` |
| Insert-if-absent only | `match m.entry(k) { Vacant(e) => .., .. }` | |
| Modify if present | `entry(k).and_modify(f).or_insert(d)` | `&mut V` |
| Group into vecs | `entry(k).or_default().push(v)` | |
| Delete | `remove(&k)`, `remove_entry(&k)` | `Option<V>` / `Option<(K,V)>` |
| Delete by predicate | `retain(\|k, v\| ..)` | `()` |
| Empty and take | `drain()` | iterator |
| Empty | `clear()` | `()` |
| Iterate | `iter`, `iter_mut`, `keys`, `values`, `values_mut` | borrowed |
| Consume | `into_iter`, `into_keys`, `into_values` | owned |
| Memory | `reserve`, `shrink_to_fit`, `shrink_to`, `capacity` | |
| Compare | `a == b` | content equality |

## Common Mistakes

**`m[&k] = v`.** There is no `IndexMut`. Use `insert` or `entry`.

**`contains_key` then `insert`.** Two hashes. Use `entry` and match on
`Vacant`.

**Looping `m.remove(k)` over collected keys.** Use `retain`.

**`values().cloned().collect()` on a map you are discarding.** Use
`into_values()` — no clone.

**`reserve(n)` as a total size.** It is `n` additional entries.

**`get_disjoint_mut(["a", "a"])`.** Duplicate keys panic.

**Swapping in `fxhash` for keys that come from the network.** You lose HashDoS
resistance; a crafted key set can degrade every lookup to O(n).

## Exercises

1. Build a `HashMap<String, Vec<u32>>` of scores per student from
   `vec![("raj", 90), ("sam", 80), ("raj", 70)]` using only `entry`. Print it
   sorted by name.
2. Write `fn invert(m: &HashMap<String, u32>) -> HashMap<u32, String>` that
   swaps keys and values. Then state what happens when two keys share a value,
   and change it to return `HashMap<u32, Vec<String>>` instead.
3. Given two maps, produce a third containing only keys present in both, with
   the values summed. Use `retain` or `filter_map`, then print it sorted.
4. Use `Entry::Vacant` matching to write
   `fn insert_if_absent(m: &mut HashMap<String, i32>, k: String, v: i32) -> bool`
   that returns whether it inserted.
5. Build `HashMap<&str, HashMap<&str, u32>>` of per-city per-product sales,
   add to it with chained `entry().or_default().entry()`, then print total
   sales per city sorted by city name.

---

[← 7.4 HashMap](04-hashmap.md) · [Course Index](../README.md) · [7.6 BTreeMap and Ordered Data →](06-btreemap.md)
