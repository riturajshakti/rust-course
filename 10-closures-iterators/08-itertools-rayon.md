[← 10.7 Writing Your Own Iterator](07-custom-iterator.md) · [Course Index](../README.md) · [11.1 Box and Heap Allocation →](../11-smart-pointers/01-box.md)

# 10.8 — itertools and rayon

## What You'll Learn

- `itertools`: the adapters std leaves out — `unique`, `sorted`, `chunk_by`,
  `izip!`, `cartesian_product`, `tuple_windows`, `join`, `counts`, `batching`
- `rayon`: turning any iterator chain parallel by changing one method name
- When parallelism pays and when it costs you 500x
- The `Send + Sync` requirement and the errors it produces

## Coming From Another Language

| Other | Rust |
|---|---|
| Python `itertools` (stdlib) | the `itertools` crate |
| Lodash `_.groupBy`, `_.uniq` | `itertools::chunk_by`, `unique` |
| Go goroutines + `WaitGroup` for a parallel map | `rayon` `par_iter` |

These are the two crates almost every Rust project ends up adding. `itertools`
is pure convenience; `rayon` is a work-stealing thread pool behind an iterator
API.

## Adding them

```
cargo add itertools rayon
```

```toml
[dependencies]
itertools = "0.15.0"
rayon = "1.12.0"
```

Both work through a trait you must import:

```rust
use itertools::Itertools;
use rayon::prelude::*;
```

Every example below assumes `let v = vec![3, 1, 2, 3, 1];`.

## itertools: unique and dedup

`unique` removes all duplicates (hashes items, keeps first occurrences);
`dedup` removes only *adjacent* ones, like `Vec::dedup` but lazy.

```rust
println!("{:?}", v.iter().unique().collect::<Vec<_>>());
println!("{:?}", v.iter().unique_by(|x| *x % 2).collect::<Vec<_>>());
println!("{:?}", v.iter().dedup().collect::<Vec<_>>());
println!("{:?}", v.iter().sorted().dedup().collect::<Vec<_>>());
println!("{:?}", v.iter().dedup_with_count().collect::<Vec<_>>());
```

Output:

```
[3, 1, 2]
[3, 2]
[3, 1, 2, 3, 1]
[1, 2, 3]
[(1, 3), (1, 1), (1, 2), (1, 3), (1, 1)]
```

`unique_by` keeps the first item per key — `3` (odd) and `2` (even).
`dedup_with_count` yields `(run_length, item)`.

## itertools: sorted

`sorted` collects internally, sorts, and gives you an iterator back. It is
*not* lazy — it must see everything — but it keeps the chain readable.

```rust
println!("{:?}", v.iter().sorted().collect::<Vec<_>>());
println!("{:?}", v.iter().sorted_by_key(|x| -**x).collect::<Vec<_>>());
```

Output:

```
[1, 1, 2, 3, 3]
[3, 3, 2, 1, 1]
```

Also `sorted_by`, and `k_smallest(n)` when you only need the top few.

## itertools: chunk_by (grouping)

Groups **consecutive** items by a key, like Unix `uniq -c`. Sort first if you
want true grouping.

```rust
let data = vec![("a", 1), ("a", 2), ("b", 3), ("a", 4)];
for (key, group) in &data.iter().chunk_by(|(k, _)| *k) {
    println!("{key}: {:?}", group.map(|(_, v)| *v).collect::<Vec<i32>>());
}
```

Output:

```
a: [1, 2]
b: [3]
a: [4]
```

Two `"a"` groups, because the third `("a", 4)` was not adjacent. The full
group-by:

```rust
let groups: Vec<(char, Vec<&str>)> = vec!["apple", "fig", "avocado", "banana"]
    .into_iter()
    .sorted()
    .chunk_by(|w| w.chars().next().unwrap())
    .into_iter()
    .map(|(k, g)| (k, g.collect()))
    .collect();
println!("{groups:?}");
```

Output:

```
[('a', ["apple", "avocado"]), ('b', ["banana"]), ('f', ["fig"])]
```

The `&` or `.into_iter()` on the `chunk_by` result is required — it returns a
`ChunkBy` struct, not an iterator, because each group borrows from it. This was
called `group_by` before itertools 0.13; that name still exists but is
deprecated and warns.

## itertools: counts

A one-liner for the word count from lesson 10.5.

```rust
let mut c: Vec<_> = v.iter().counts().into_iter().collect();
c.sort();
println!("{c:?}");
```

Output:

```
[(1, 2), (2, 1), (3, 2)]
```

`counts()` returns a `HashMap`, so sort before printing. `counts_by(f)` counts
by a key function instead of the item itself.

## itertools: join

`collect::<Vec<_>>().join(sep)` without the intermediate `Vec`. Works on any
`Display` item, so no `.to_string()` map either.

```rust
println!("{}", v.iter().join("-"));
```

Output:

```
3-1-2-3-1
```

## itertools: minmax

One pass for both, with explicit cases for 0 and 1 elements.

```rust
println!("{:?}", v.iter().minmax());
println!("{:?}", std::iter::empty::<i32>().minmax());
println!("{:?}", [5].iter().minmax());
```

Output:

```
MinMax(1, 3)
NoElements
OneElement(5)
```

`.into_option()` turns it into `Option<(T, T)>` if you don't care about the
one-element case.

## itertools: tuple_windows and tuples

`tuple_windows` is `slice::windows` for arbitrary iterators, with a *tuple*
item so each element is typed. `tuples` is non-overlapping.

```rust
println!("{:?}", v.iter().tuple_windows::<(_, _)>().collect::<Vec<_>>());
println!("{:?}", v.iter().tuple_windows::<(_, _, _)>().collect::<Vec<_>>());
println!("{:?}", v.iter().tuples::<(_, _)>().collect::<Vec<_>>());
```

Output:

```
[(3, 1), (1, 2), (2, 3), (3, 1)]
[(3, 1, 2), (1, 2, 3), (2, 3, 1)]
[(3, 1), (2, 3)]
```

`tuples` dropped the trailing `1` — like `chunks_exact`. The turbofish is
needed only when the arity can't be inferred from the closure that follows.

## itertools: cartesian_product, combinations, interleave

```rust
println!("{:?}", [1, 2].iter().cartesian_product(['a', 'b'].iter()).collect::<Vec<_>>());
println!("{:?}", [1, 2, 3].iter().combinations(2).collect::<Vec<_>>());
println!("{}", [1, 2, 3].iter().permutations(2).count());
println!("{:?}", [1, 2].iter().interleave([9, 9, 9].iter()).collect::<Vec<_>>());
```

Output:

```
[(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]
[[1, 2], [1, 3], [2, 3]]
6
[1, 9, 2, 9, 9]
```

`cartesian_product` requires the right side to be `Clone` (it is restarted per
left item). `combinations` and `permutations` buffer, so they allocate a `Vec`
per item.

## itertools: izip!

`zip` three or more sequences, with flat tuples instead of `((a, b), c)`.

```rust
use itertools::izip;

let names = ["a", "b", "c"];
let ages = [30, 40, 50];
let cities = ["X", "Y", "Z"];
for (n, a, city) in izip!(names, ages, cities) { println!("{n} {a} {city}"); }
println!("{:?}", izip!(1..4, 'a'..='c').collect::<Vec<_>>());
```

Output:

```
a 30 X
b 40 Y
c 50 Z
[(1, 'a'), (2, 'b'), (3, 'c')]
```

Like `zip`, it stops at the shortest input. `multizip` is the function form.

## itertools: batching

The escape hatch — a custom adapter from a closure that gets the raw iterator.
Anything the other adapters cannot express.

```rust
let nums = vec![1, 2, 3, 4, 5, 6, 7];
let batched: Vec<Vec<i32>> = nums.into_iter().batching(|it| {
    let b: Vec<i32> = it.take(3).collect();
    if b.is_empty() { None } else { Some(b) }
}).collect();
println!("{batched:?}");

let pairs: Vec<(i32, i32)> = (1..=6).batching(|it| match (it.next(), it.next()) {
    (Some(a), Some(b)) => Some((a, b)),
    _ => None,
}).collect();
println!("{pairs:?}");
```

Output:

```
[[1, 2, 3], [4, 5, 6], [7]]
[(1, 2), (3, 4), (5, 6)]
```

Note the first one keeps the short tail; `tuples` and `chunks_exact` would drop
it. Returning `None` ends the iterator, so the empty check is mandatory or it
loops forever.

## itertools: chunks

Fixed-size batches from a non-slice iterator. Same borrow dance as `chunk_by`.

```rust
println!("{:?}", [1, 2, 3].iter().chunks(2).into_iter()
    .map(|c| c.copied().sum::<i32>()).collect::<Vec<i32>>());
```

Output:

```
[3, 3]
```

## itertools quick reference

| Method | Does | Lazy |
|---|---|---|
| `unique()` / `unique_by(f)` | drop all duplicates, keep first | yes |
| `dedup()` / `dedup_with_count()` | drop adjacent duplicates | yes |
| `sorted()` / `sorted_by(f)` / `sorted_by_key(f)` | sort, return an iterator | no |
| `k_smallest(n)` | n smallest without a full sort | no |
| `chunk_by(f)` | group consecutive items by key | yes |
| `chunks(n)` | fixed-size batches | yes |
| `counts()` / `counts_by(f)` | `HashMap<Item, usize>` | no |
| `join(sep)` | `String`, no intermediate `Vec` | no |
| `minmax()` | `MinMaxResult`, one pass | no |
| `tuple_windows()` | overlapping tuples | yes |
| `tuples()` | non-overlapping tuples, drops the tail | yes |
| `cartesian_product(other)` | all pairs | yes |
| `combinations(k)` / `permutations(k)` | buffered `Vec` per item | yes |
| `interleave(other)` | alternate items | yes |
| `batching(f)` | custom adapter from a closure | yes |
| `izip!(a, b, c, ..)` | flat-tuple zip of any arity | yes |
| `exactly_one()` / `at_most_one()` | `Result`/`Option`, errors on count | no |
| `collect_vec()` | `collect::<Vec<_>>()` without the turbofish | no |

## rayon: par_iter

Change `iter` to `par_iter` and the chain runs across a thread pool. The API is
deliberately a mirror of `Iterator`.

```rust
use rayon::prelude::*;

let v: Vec<u64> = (1..=10).collect();
println!("{}", v.par_iter().sum::<u64>());
println!("{:?}", v.par_iter().map(|x| x * 2).collect::<Vec<u64>>());
println!("{:?}", v.par_iter().filter(|x| **x % 2 == 0).collect::<Vec<&u64>>());
println!("{}", v.clone().into_par_iter().map(|x| x * x).sum::<u64>());
println!("{:?}", v.par_iter().max());
println!("{}", v.par_iter().filter(|x| **x > 5).count());
println!("{}", v.par_iter().any(|x| *x > 9));
println!("{}", rayon::current_num_threads());
```

Output (thread count is machine-specific):

```
55
[2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
[2, 4, 6, 8, 10]
385
Some(10)
5
true
12
```

`par_iter` / `par_iter_mut` / `into_par_iter` mirror `iter` / `iter_mut` /
`into_iter`. The pool is global, sized to your core count, and created on first
use — there is nothing to set up.

## rayon: order is still deterministic

`collect`, `reduce`, `sum`, and `fold().sum()` all produce the same answer as
the sequential version. Work is split by index range, and results are
reassembled in order.

```rust
let v: Vec<u64> = (0..8).collect();
println!("{:?}", v.par_iter().map(|x| x * 2).collect::<Vec<u64>>());
println!("{:?}", v.par_iter().map(|x| x.to_string()).reduce(|| String::new(), |a, b| a + &b));
let (ev, od): (Vec<u64>, Vec<u64>) = v.par_iter().partition(|x| **x % 2 == 0);
println!("{ev:?} {od:?}");
println!("{}", v.par_iter().fold(|| 0u64, |a, x| a + x).sum::<u64>());
println!("{:?}", v.par_chunks(3).map(|c| c.iter().sum::<u64>()).collect::<Vec<u64>>());
```

Output:

```
[0, 2, 4, 6, 8, 10, 12, 14]
"01234567"
[0, 2, 4, 6] [1, 3, 5, 7]
28
[3, 12, 13]
```

`reduce` takes an **identity closure** rather than a first element, because
each worker needs its own starting value. Same for `fold`, whose result is a
parallel iterator of per-worker accumulators — hence the trailing `.sum()`.

The methods with no deterministic answer are named for it: `find_any` and
`position_any` return whichever match a worker hits first. Use `find_first` /
`position_first` when you need the sequentially-first one.

## rayon: par_sort

```rust
let mut s = vec![5, 3, 9, 1, 7];
s.par_sort();
println!("{s:?}");

let mut s2 = vec![5, 3, 9, 1, 7];
s2.par_sort_unstable_by_key(|x| -x);
println!("{s2:?}");

let mut m = vec![1, 2, 3];
m.par_iter_mut().for_each(|x| *x *= 100);
println!("{m:?}");
```

Output:

```
[1, 3, 5, 7, 9]
[9, 7, 5, 3, 1]
[100, 200, 300]
```

`par_sort` is a parallel merge sort; `par_sort_unstable` is a parallel
quicksort and is faster when stability doesn't matter. Both are only worth it
above roughly ten thousand elements.

## When parallelism pays

One API, three very different outcomes. The results are discarded through
`black_box` so the optimizer cannot delete the work.

```rust
use rayon::prelude::*;
use std::hint::black_box;
use std::time::Instant;

fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    let mut d = 2;
    while d * d <= n { if n % d == 0 { return false; } d += 1; }
    true
}

fn time<T>(label: &str, f: impl FnOnce() -> T) {
    let t = Instant::now();
    black_box(f());
    println!("{label}: {:?}", t.elapsed());
}

fn main() {
    println!("threads: {}", rayon::current_num_threads());

    let big: Vec<u64> = (2..400_000).collect();
    time("heavy seq", || big.iter().filter(|n| is_prime(**n)).count());
    time("heavy par", || big.par_iter().filter(|n| is_prime(**n)).count());

    let small: Vec<u64> = (0..2_000_000).collect();
    time("light seq", || small.iter().sum::<u64>());
    time("light par", || small.par_iter().sum::<u64>());

    let tiny: Vec<u64> = (0..100).collect();
    time("tiny  seq", || tiny.iter().sum::<u64>());
    time("tiny  par", || tiny.par_iter().sum::<u64>());
}
```

Output (`cargo run --release`; numbers are machine-specific — 12 logical cores
here):

```
threads: 12
heavy seq: 26.326292ms
heavy par: 3.040125ms
light seq: 228.334µs
light par: 311.083µs
tiny  seq: 84ns
tiny  par: 29.166µs
```

| Workload | Result |
|---|---|
| 400k items, real CPU work each | **8.7x faster** |
| 2M items, one add each | **1.4x slower** — memory-bound, not CPU-bound |
| 100 items | **350x slower** — pure pool overhead |

Rules of thumb: the win scales with `items x work_per_item`, so parallelize
when that product is milliseconds or more. A `sum` over integers is memory
bandwidth limited and will never speed up. Always measure in `--release`, and
always measure — the tiny case is a 350x regression written as a four-character
change.

## rayon requires Send + Sync

Items must be `Send` (movable between threads) and the closure `Sync`. `Rc` is
neither.

```rust
use rayon::prelude::*;
use std::rc::Rc;

let v: Vec<Rc<i32>> = vec![Rc::new(1), Rc::new(2)];
let s: i32 = v.par_iter().map(|x| **x).sum();
```

Does not compile:

```
error[E0599]: the method `par_iter` exists for struct `Vec<Rc<i32>>`, but its trait bounds were not satisfied
   --> src/main.rs:5:20
    |
  5 |     let s: i32 = v.par_iter().map(|x| **x).sum();
    |                    ^^^^^^^^
    |
    = note: the following trait bounds were not satisfied:
            `&Vec<Rc<i32>>: IntoParallelIterator`
            which is required by `Vec<Rc<i32>>: rayon::iter::IntoParallelRefIterator<'_>`
```

The message points at the missing trait, not at `Send` — read it as "the item
type isn't thread-safe". Swap `Rc` for `Arc` (Module 11) and it compiles.

Mutating a captured variable is also out, because parallel closures must be
`Fn`, not `FnMut`:

```rust
let v = vec![1, 2, 3];
let mut out = Vec::new();
v.par_iter().for_each(|x| out.push(*x));
```

Does not compile:

```
error[E0596]: cannot borrow `out` as mutable, as it is a captured variable in a `Fn` closure
 --> src/main.rs:5:31
  |
4 |     let mut out = Vec::new();
  |         ------- `out` declared here, outside the closure
5 |     v.par_iter().for_each(|x| out.push(*x));
  |                  -------------^^^----------
  |                  |        |   |
  |                  |        |   cannot borrow as mutable
  |                  |        in this closure
  |                  expects `Fn` instead of `FnMut`
```

This is lesson 10.2's trait hierarchy enforcing thread safety. The fix is
almost always to stop mutating and `collect` instead — which is why the
parallel version of accumulating code is usually *shorter*.

## rayon quick reference

| Sequential | Parallel |
|---|---|
| `iter()` | `par_iter()` |
| `iter_mut()` | `par_iter_mut()` |
| `into_iter()` | `into_par_iter()` |
| `chunks(n)` | `par_chunks(n)` |
| `map`, `filter`, `filter_map`, `flat_map`, `flatten`, `inspect`, `cloned`, `copied`, `chain`, `zip`, `take`, `skip`, `rev`, `enumerate` | same names |
| `sum`, `product`, `count`, `min`, `max`, `min_by_key`, `max_by_key`, `collect`, `partition`, `unzip`, `for_each`, `any`, `all` | same names |
| `fold(init, f)` | `fold(\|\| init, f)` then `.sum()`/`.reduce(..)` |
| `reduce(f)` | `reduce(\|\| identity, f)` |
| `try_fold` | `try_fold(\|\| init, f)` |
| `find(p)` | `find_any(p)` or `find_first(p)` |
| `position(p)` | `position_any(p)` or `position_first(p)` |
| `v.sort()` | `v.par_sort()` |
| `scan`, `peekable`, `by_ref`, `cycle`, `step_by` | **no parallel version** (inherently sequential) |

Not available in parallel: anything order-dependent. `scan` needs the previous
item; `peekable` and `by_ref` need a single cursor.

## Common Mistakes

**Forgetting the prelude import.** No `use itertools::Itertools;` or
`use rayon::prelude::*;` and the methods simply don't exist on the iterator.

**Using `chunk_by` without sorting.** It groups *consecutive* items only;
`.sorted()` first for a real group-by.

**Iterating a `chunk_by`/`chunks` result directly.** Add `&` or
`.into_iter()`; the groups borrow the parent struct.

**Reaching for `par_iter` on small or memory-bound work.** Measure first — it
can be hundreds of times slower.

**Trying to push into a captured `Vec` from `par_iter().for_each()`.** Parallel
closures are `Fn`; `collect` instead.

## Exercises

1. For `vec!["bob", "alice", "bob", "carol"]`, use `counts` to get the tallies
   and print them sorted by count descending.
2. Group `vec![("eu","de"),("us","ca"),("eu","fr")]` into
   `Vec<(&str, Vec<&str>)>` with `sorted` + `chunk_by`.
3. Use `tuple_windows` to find the largest increase between adjacent elements
   of `vec![3, 8, 2, 9, 4]`, then use `batching` to split `1..=10` into runs
   that stop whenever the running total exceeds 15.
4. Write an `is_prime` count over `2..1_000_000` sequentially and with
   `par_iter`. Time both in release mode and compute the speedup against your
   core count.
5. Try `par_iter` on `Vec<Rc<String>>`, record the error, then make it work
   with `Arc`.

---

[← 10.7 Writing Your Own Iterator](07-custom-iterator.md) · [Course Index](../README.md) · [11.1 Box and Heap Allocation →](../11-smart-pointers/01-box.md)
