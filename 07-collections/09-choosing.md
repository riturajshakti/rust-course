[← Course Index](../README.md)

# 7.9 — Choosing the Right Collection

## What You'll Learn

- A decision table that gets you to the right type in one question
- Big-O for every operation across all seven std collections
- Real memory overhead numbers
- When a sorted `Vec` beats a `HashMap` outright

## Coming From Another Language

In JS you have arrays and `Map`. In Python, `list` and `dict`. Rust gives you
seven, and the extra five exist because the trade-offs are real and the
compiler makes you pick. The good news: the default is almost always `Vec`,
and the second choice is almost always `HashMap`.

## The decision table

Answer the first question that applies.

| Question | Answer |
|---|---|
| Just a list of things, in order? | `Vec<T>` |
| Lookup by a key? | `HashMap<K, V>` |
| …and output must be sorted or ranged? | `BTreeMap<K, V>` |
| Only membership, no value? | `HashSet<T>` |
| …and sorted or ranged? | `BTreeSet<T>` |
| FIFO queue, or add/remove at both ends? | `VecDeque<T>` |
| Always need the largest (or smallest) next? | `BinaryHeap<T>` (+ `Reverse`) |
| Fixed length known at compile time? | `[T; N]` — not a collection at all |
| Built once, never resized, stored in bulk? | `Box<[T]>` |
| Splice in the middle constantly? | still `Vec` or `VecDeque`, not `LinkedList` |

Or by the operation that dominates your code:

| Dominant operation | Type |
|---|---|
| Iterate everything | `Vec` |
| Index by position | `Vec` |
| Push / pop at the end | `Vec` |
| Push / pop at the front | `VecDeque` |
| Look up by key | `HashMap` |
| Range of keys, min, max | `BTreeMap` |
| "Have I seen this?" | `HashSet` |
| "Give me the next highest priority" | `BinaryHeap` |
| Deterministic serialized output | `BTreeMap` / `BTreeSet` |

## Big-O table

`n` is the element count, `k` the number of elements a range or bulk operation
touches. "amortized" means occasional O(n) reallocation spread over many
calls.

| Operation | `Vec` | `VecDeque` | `HashMap` | `BTreeMap` | `HashSet` | `BTreeSet` | `BinaryHeap` |
|---|---|---|---|---|---|---|---|
| Index by position | O(1) | O(1) | — | — | — | — | — |
| Get by key / value | O(n) scan | O(n) scan | O(1)* | O(log n) | O(1)* | O(log n) | — |
| Insert at end | O(1) am. | O(1) am. | — | — | — | — | — |
| Insert at front | O(n) | O(1) am. | — | — | — | — | — |
| Insert in middle | O(n) | O(min(i, n-i)) | — | — | — | — | — |
| Insert by key | — | — | O(1)* | O(log n) | O(1)* | O(log n) | O(log n) push |
| Remove from end | O(1) | O(1) | — | — | — | — | — |
| Remove from front | O(n) | O(1) | — | — | — | — | — |
| Remove at index | O(n) ordered / O(1) `swap_remove` | O(min(i, n-i)) / O(1) swap | — | — | — | — | — |
| Remove by key | — | — | O(1)* | O(log n) | O(1)* | O(log n) | — |
| Min / max | O(n) | O(n) | O(n) | O(log n) | O(n) | O(log n) | O(1) peek max |
| Pop min / max | O(n) | O(n) | O(n) | O(log n) | O(n) | O(log n) | O(log n) |
| Range query | O(log n + k) if sorted | — | — | O(log n + k) | — | O(log n + k) | — |
| Iterate all | O(n), cache-friendly | O(n) | O(capacity) | O(n) | O(capacity) | O(n) | O(n) |
| Sorted iteration | O(n log n) to sort | O(n log n) | O(n log n) | **O(n)** | O(n log n) | **O(n)** | O(n log n) |
| Membership | O(n) | O(n) | O(1)* | O(log n) | O(1)* | O(log n) | O(n) |
| Append another | O(k) | O(k) | O(k) | O(k) | O(k) | O(k) | O(k) |

`*` average case. Worst case for the hash collections is O(n), which an
attacker controlling your keys cannot trigger with the default SipHash seed,
but a fast third-party hasher gives that protection up (lesson 7.5).

Two rows deserve emphasis. **`BTreeMap` iterates sorted in O(n)** while
`HashMap` needs an O(n log n) sort every time — if you sort on every read, you
picked wrong. And **`Vec` iteration is the fastest thing in the table** by a
constant factor the Big-O hides, because it is one contiguous run of memory.

## Memory overhead

Struct size on the stack (the handle, not the data):

```rust
use std::collections::*;

macro_rules! sz {
    ($($t:ty),*) => { $( println!("{:<28} {}", stringify!($t), std::mem::size_of::<$t>()); )* }
}

sz!(Vec<u64>, VecDeque<u64>, HashMap<u64,u64>, BTreeMap<u64,u64>,
    HashSet<u64>, BTreeSet<u64>, BinaryHeap<u64>, LinkedList<u64>, Box<[u64]>, String);
```

Output:

```
Vec<u64>                     24
VecDeque<u64>                32
HashMap<u64,u64>             48
BTreeMap<u64,u64>            24
HashSet<u64>                 48
BTreeSet<u64>                24
BinaryHeap<u64>              24
LinkedList<u64>              24
Box<[u64]>                   16
String                       24
```

Heap overhead per element is what actually matters at scale:

```rust
let n = 1000usize;
let v: Vec<(u64, u64)> = (0..n as u64).map(|i| (i, i)).collect();
println!("Vec slots {}  bytes/elem {}", v.capacity(), v.capacity() * 16 / n);

let h: HashMap<u64, u64> = (0..n as u64).map(|i| (i, i)).collect();
println!("HashMap slots {}  slots/elem {:.2}", h.capacity(), h.capacity() as f64 / n as f64);
```

Output:

```
Vec slots 1000  bytes/elem 16
HashMap slots 1792  slots/elem 1.79
```

The rules of thumb:

- **`Vec`**: zero per-element overhead when exactly sized; up to 2x wasted
  slots mid-growth. `shrink_to_fit()` or `into_boxed_slice()` reclaims it.
- **`VecDeque`**: like `Vec`, but capacity is always rounded, so expect some
  slack even after shrinking.
- **`HashMap` / `HashSet`**: roughly 1.8 slots allocated per entry to keep the
  load factor low, plus one control byte per slot. Budget ~2x the raw
  key-value bytes.
- **`BTreeMap` / `BTreeSet`**: nodes hold up to 11 entries, so overhead is a
  few pointers per 11 elements — lower than a hash map, but spread across
  separate allocations.
- **`BinaryHeap`**: it is a `Vec` underneath. Zero per-element overhead.
- **`LinkedList`**: two pointers *per element*, each in its own allocation.
  For `u64` data that is 24 bytes of overhead on 8 bytes of payload.

## Start with Vec, then HashMap

This is the actual advice, and it is not a cop-out.

**Start with `Vec`.** It is the fastest to iterate, the cheapest in memory, the
easiest to debug (`{:?}` prints in a stable order), the easiest to serialize,
and it requires nothing of `T`. Most collections in most programs are built
once, iterated a few times, and dropped — `Vec` is optimal for that, and a
linear scan over a few hundred elements is faster than hashing even one key.

**Move to `HashMap` when a key lookup is in a hot path** and the collection is
big enough that O(n) hurts — in practice from a few hundred elements, or when
lookups outnumber elements.

**Move to `BTreeMap` when order matters.** If you write
`let mut v: Vec<_> = map.iter().collect(); v.sort();` more than once, you
wanted a `BTreeMap` from the start.

**Everything else is a specific answer to a specific problem.** `VecDeque`
when you pop from the front, `BinaryHeap` when you need a priority queue,
sets when the value is nothing but the key. Reach for them when the shape of
your problem names them, not preemptively.

The mistake worth avoiding is the opposite one: reaching for `HashMap<u32, T>`
when the keys are dense small integers. `Vec<Option<T>>` indexed directly is
faster, smaller, and ordered for free.

## When a sorted Vec beats a map

A `Vec<(K, V)>` kept sorted and queried with `binary_search_by_key` is O(log n)
like a `BTreeMap`, but with `Vec` memory layout and `Vec` iteration speed. For
small-to-medium read-mostly data it beats both maps. One million lookups,
release build:

```rust
use std::collections::{HashMap, BTreeMap};
use std::time::Instant;

fn bench(n: u64, probes: u64) {
    let keys: Vec<u64> = (0..n).map(|i| i.wrapping_mul(2654435761) % 1_000_000).collect();
    let mut sorted: Vec<(u64, u64)> = keys.iter().map(|k| (*k, *k)).collect();
    sorted.sort_by_key(|p| p.0);
    sorted.dedup_by_key(|p| p.0);
    let hm: HashMap<u64, u64> = sorted.iter().copied().collect();
    let bm: BTreeMap<u64, u64> = sorted.iter().copied().collect();

    let mut s = 0u64;
    let t = Instant::now();
    for i in 0..probes {
        let k = keys[(i % n) as usize];
        if let Ok(j) = sorted.binary_search_by_key(&k, |p| p.0) { s += sorted[j].1; }
    }
    let a = t.elapsed();

    let t = Instant::now();
    for i in 0..probes { s += hm[&keys[(i % n) as usize]]; }
    let b = t.elapsed();

    let t = Instant::now();
    for i in 0..probes { s += bm[&keys[(i % n) as usize]]; }
    let c = t.elapsed();

    println!("n={:<7} sorted Vec {:>12?}   HashMap {:>12?}   BTreeMap {:>12?}",
             sorted.len(), a, b, c);
    std::hint::black_box(s);
}

for n in [8u64, 64, 1024, 100_000] { bench(n, 1_000_000); }
```

Output:

```
n=8       sorted Vec   1.324458ms   HashMap   4.069875ms   BTreeMap      1.948ms
n=64      sorted Vec   2.740667ms   HashMap    4.18925ms   BTreeMap   4.193416ms
n=1024    sorted Vec     5.7785ms   HashMap   4.061166ms   BTreeMap      8.255ms
n=100000  sorted Vec  21.210375ms   HashMap   6.065958ms   BTreeMap  60.722625ms
```

Read that from the top. At 8 and 64 entries the sorted `Vec` wins — the whole
thing fits in L1 and a `HashMap` pays a SipHash per probe regardless of size.
Around 1,000 entries `HashMap` pulls ahead. At 100,000 the hash map is 3.5x
faster than the sorted `Vec` and 10x faster than the `BTreeMap`, because
binary search starts missing cache on every step.

So use a sorted `Vec` when:

- the data is **small** (tens to low hundreds of entries),
- it is **built once and read many times** — inserting into the middle is
  O(n), which is what kills it for write-heavy use,
- you also want to **iterate in order** or do range queries,
- you care about memory, serialization, or cache behaviour,
- or `K` is awkward: no `Hash`, or an expensive one.

And do not use one when inserts and removals are frequent and interleaved with
lookups. That is exactly what `BTreeMap` is for.

For the small-map case the ecosystem has pre-built versions of this idea —
`indexmap` (insertion-ordered map with O(1) lookup), `smallvec` (inline
storage up to N elements, no heap allocation), and `arrayvec` (fixed capacity,
never allocates).

## Determinism checklist

If any of these describe your output, you need a `BTreeMap`/`BTreeSet` or an
explicit sort:

- it goes into a test assertion
- it is serialized to JSON, YAML, or TOML that gets diffed or committed
- it is a log line a human reads repeatedly
- it is hashed or signed
- it feeds a reproducible build

`HashMap` order varies between two maps in the same process, let alone between
runs (lesson 7.4 shows it).

## Quick Reference

| Type | Key bound | Ordered | Index | Lookup | Best at |
|---|---|---|---|---|---|
| `Vec<T>` | — | insertion | yes | O(n) | everything by default |
| `Box<[T]>` | — | insertion | yes | O(n) | immutable, stored in bulk |
| `VecDeque<T>` | — | insertion | yes | O(n) | FIFO, both-ends, ring buffer |
| `HashMap<K,V>` | `Eq + Hash` | no | — | O(1) | key lookup |
| `BTreeMap<K,V>` | `Ord` | by key | — | O(log n) | ordered keys, ranges, determinism |
| `HashSet<T>` | `Eq + Hash` | no | — | O(1) | membership, dedup |
| `BTreeSet<T>` | `Ord` | by value | — | O(log n) | ordered membership, ranges |
| `BinaryHeap<T>` | `Ord` | root only | — | — | priority queue, top-k |
| `LinkedList<T>` | — | insertion | **no** | O(n) | nothing, practically |
| sorted `Vec<(K,V)>` | `Ord` | by key | yes | O(log n) | small, read-mostly, ordered |

## Common Mistakes

**`HashMap<u32, T>` with dense small keys.** Use `Vec<T>` or
`Vec<Option<T>>` and index directly.

**Reaching for `HashMap` on 10 items.** A linear scan over a `Vec` is faster
and simpler.

**Sorting a `HashMap`'s contents on every read.** You wanted `BTreeMap`.

**Choosing `LinkedList` because insertion is "O(1)".** Finding the position is
O(n) and the cache misses dominate. `Vec` wins on real data.

**`Vec::remove(0)` as a queue.** O(n²) overall. `VecDeque::pop_front` is O(1).

**Optimizing the collection before measuring.** The allocation you removed was
not the bottleneck. Start with `Vec`, profile, then change one thing.

## Exercises

1. For each of these, name the collection and justify it in one line: an LRU
   cache's recency order; usernames seen in a log; a Dijkstra frontier; HTTP
   headers that must round-trip in order; a tag autocomplete index.
2. Implement the same word-frequency counter three ways — `Vec<(String, u32)>`
   with a linear scan, `HashMap`, and `BTreeMap` — and time all three on a
   10,000-word input.
3. Build a lookup of 50 string keys as both a `HashMap` and a sorted
   `Vec<(String, u32)>`, run a million lookups against each, and report which
   wins.
4. Write `fn needs_btree<T>(uses: &[&str]) -> bool` that returns true if the
   use list contains "range", "sorted", or "deterministic". Then apply it to
   three of your own existing `HashMap` uses.
5. Take a `HashMap<String, Vec<u32>>` with 100,000 entries and measure the
   memory difference after calling `shrink_to_fit()` on the map and on every
   inner `Vec`.

---

[← Course Index](../README.md)
