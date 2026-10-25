[← 10.5 Consumers: collect, fold, and Friends](05-consumers.md) · [Course Index](../README.md) · [10.7 Writing Your Own Iterator →](07-custom-iterator.md)

# 10.6 — Iterator Operations Reference

## What You'll Learn

- Every `Iterator` method you will realistically use, grouped by category
- Which are lazy adapters and which are eager consumers
- The grouping and windowing operations that live on *slices*, not iterators
- A "which one do I want?" decision table

## Coming From Another Language

This is the lesson to bookmark. JS has about 20 array methods, Python splits
across builtins plus `itertools`; Rust puts roughly 75 methods on one trait,
and they split cleanly into **lazy adapters** (return an iterator) and **eager
consumers** (return a value). The tables below mark which is which.

Every example assumes:

```rust
let v = vec![1, 2, 3, 4, 5];
let e: Vec<i32> = vec![];
```

## Reading the tables

| Column | Meaning |
|---|---|
| Lazy | returns an iterator; nothing runs until a consumer pulls |
| Eager | runs immediately, returns a value |
| Short-circuits | may stop before the end of the input |

## Adapters: one-to-one and filtering

All lazy.

| Method | Signature | Returns | Example → output |
|---|---|---|---|
| `map` | `map<B>(f: FnMut(Item) -> B)` | `Map` of `B` | `v.iter().map(\|x\| x * 2)` → `[2,4,6,8,10]` |
| `filter` | `filter(p: FnMut(&Item) -> bool)` | `Filter` of `Item` | `v.iter().filter(\|x\| **x > 3)` → `[4,5]` |
| `filter_map` | `filter_map<B>(f: FnMut(Item) -> Option<B>)` | `FilterMap` of `B` | `["1","x"].iter().filter_map(\|s\| s.parse::<i32>().ok())` → `[1]` |
| `map_while` | `map_while<B>(f: FnMut(Item) -> Option<B>)` | `MapWhile` of `B` | `["1","x","3"].iter().map_while(\|s\| s.parse::<i32>().ok())` → `[1]` |
| `inspect` | `inspect(f: FnMut(&Item))` | same items | `v.iter().inspect(\|x\| println!("{x}"))` → prints, passes through |
| `cloned` | `cloned()` where `Item = &T: Clone` | `Cloned` of `T` | `v.iter().cloned()` → `[1,2,3,4,5]` as `i32` |
| `copied` | `copied()` where `Item = &T: Copy` | `Copied` of `T` | `v.iter().copied()` → `[1,2,3,4,5]` as `i32` |

```rust
println!("{:?}", v.iter().map(|x| x * 2).collect::<Vec<i32>>());
println!("{:?}", v.iter().copied().filter(|x| x % 2 == 1).collect::<Vec<i32>>());
```

Output:

```
[2, 4, 6, 8, 10]
[1, 3, 5]
```

## Adapters: flattening

All lazy.

| Method | Signature | Returns | Example → output |
|---|---|---|---|
| `flat_map` | `flat_map<U: IntoIterator>(f: FnMut(Item) -> U)` | items of each `U` | `["a b","c"].iter().flat_map(\|s\| s.split(' '))` → `["a","b","c"]` |
| `flatten` | `flatten()` where `Item: IntoIterator` | items of each item | `vec![vec![1,2],vec![3]].into_iter().flatten()` → `[1,2,3]` |

`flatten` removes exactly one level, and works on `Option`/`Result` items
because both are iterators of 0 or 1.

```rust
println!("{:?}", vec![Some(1), None, Some(3)].into_iter().flatten().collect::<Vec<i32>>());
println!("{:?}", vec![vec![vec![1]], vec![vec![2]]].into_iter().flatten().flatten().collect::<Vec<i32>>());
```

Output:

```
[1, 3]
[1, 2]
```

## Adapters: positional and combining

All lazy.

| Method | Signature | Returns | Example → output |
|---|---|---|---|
| `enumerate` | `enumerate()` | `(usize, Item)` from 0 | `['a','b'].iter().enumerate()` → `[(0,'a'),(1,'b')]` |
| `zip` | `zip<U: IntoIterator>(other: U)` | `(Item, U::Item)`, stops at shorter | `(1..9).zip('a'..='c')` → `[(1,'a'),(2,'b'),(3,'c')]` |
| `chain` | `chain<U: IntoIterator<Item=Item>>(other: U)` | self then other | `(1..3).chain(8..10)` → `[1,2,8,9]` |
| `take` | `take(n: usize)` | first `n` | `v.iter().take(2)` → `[1,2]` |
| `take_while` | `take_while(p: FnMut(&Item) -> bool)` | until `p` first fails | `v.iter().take_while(\|x\| **x < 3)` → `[1,2]` |
| `skip` | `skip(n: usize)` | all but first `n` | `v.iter().skip(3)` → `[4,5]` |
| `skip_while` | `skip_while(p: FnMut(&Item) -> bool)` | from first failure on | `v.iter().skip_while(\|x\| **x < 3)` → `[3,4,5]` |
| `step_by` | `step_by(n: usize)` | first, then every nth | `(0..10).step_by(4)` → `[0,4,8]` |
| `rev` | `rev()` requires `DoubleEndedIterator` | back to front | `v.iter().rev()` → `[5,4,3,2,1]` |
| `cycle` | `cycle()` requires `Clone` | repeats forever | `[1,2].iter().cycle().take(5)` → `[1,2,1,2,1]` |
| `scan` | `scan<St,B>(init: St, f: FnMut(&mut St, Item) -> Option<B>)` | each `Some`; `None` ends it | `v.iter().scan(0,\|a,x\|{*a+=x;Some(*a)})` → `[1,3,6,10,15]` |

```rust
println!("{:?}", (0..10).step_by(4).collect::<Vec<i32>>());
println!("{:?}", v.iter().scan(0, |a, x| { *a += x; Some(*a) }).collect::<Vec<i32>>());
println!("{:?}", v.iter().zip(v.iter().skip(1)).collect::<Vec<_>>());
```

Output:

```
[0, 4, 8]
[1, 3, 6, 10, 15]
[(1, 2), (2, 3), (3, 4), (4, 5)]
```

That last pattern — `zip(skip(1))` — is the pairwise-windows trick when you
have an iterator rather than a slice.

## Zipping against a range

Ranges are infinite-ish iterators, which makes several idioms possible.

```rust
println!("{:?}", v.iter().zip(1..).map(|(x, i)| format!("{i}.{x}")).collect::<Vec<String>>());
println!("{:?}", v.iter().zip(0..).filter(|(_, i)| i % 2 == 0).map(|(x, _)| *x).collect::<Vec<i32>>());
```

Output:

```
["1.1", "2.2", "3.3", "4.4", "5.5"]
[1, 3, 5]
```

`zip(1..)` is 1-based enumerate. `zip(0..)` then filtering on the index is how
you take every nth of a non-range iterator (`step_by` works too, and is
clearer).

## Adapters: control

| Method | Lazy? | Signature | What it does |
|---|---|---|---|
| `peekable` | lazy | `peekable()` | adds `.peek() -> Option<&Item>` and `.next_if(p)` |
| `by_ref` | lazy | `by_ref() -> &mut Self` | lets you adapt without consuming the iterator |
| `fuse` | lazy | `fuse()` | guarantees `None` forever after the first `None` |

Most iterators already behave as if fused; `fuse` matters for hand-written ones
that misbehave.

```rust
struct Flaky(u32);
impl Iterator for Flaky {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        self.0 += 1;
        if self.0 % 2 == 0 { None } else { Some(self.0) }
    }
}

let mut a = Flaky(0);
println!("{:?} {:?} {:?} {:?}", a.next(), a.next(), a.next(), a.next());
let mut b = Flaky(0).fuse();
println!("{:?} {:?} {:?} {:?}", b.next(), b.next(), b.next(), b.next());
```

Output:

```
Some(1) None Some(3) None
Some(1) None None None
```

`peekable` also gives `next_if`, a conditional consume:

```rust
let mut it = v.iter().peekable();
while it.next_if(|x| **x < 3).is_some() {}
println!("{:?}", it.collect::<Vec<_>>());
```

Output:

```
[3, 4, 5]
```

## Consumers: reduction

All eager. `Acc` is any type; `Item` must be `Sum`/`Product`/`Ord` where noted.

| Method | Signature | Returns | Example → output |
|---|---|---|---|
| `sum` | `sum<S: Sum<Item>>()` | `S`; `0` when empty | `v.iter().sum::<i32>()` → `15` |
| `product` | `product<P: Product<Item>>()` | `P`; `1` when empty | `v.iter().product::<i32>()` → `120` |
| `count` | `count()` | `usize` | `v.iter().count()` → `5` |
| `fold` | `fold<B>(init: B, f: FnMut(B, Item) -> B)` | `B`, always | `v.iter().fold(0,\|a,x\| a+x)` → `15` |
| `reduce` | `reduce(f: FnMut(Item, Item) -> Item)` | `Option<Item>` | `v.iter().copied().reduce(i32::max)` → `Some(5)` |
| `try_fold` | `try_fold<B,R: Try>(init: B, f)` | `R`, short-circuits | `v.iter().try_fold(0i32,\|a,x\| a.checked_add(*x))` → `Some(15)` |
| `rfold` | `rfold<B>(init: B, f)` needs `DoubleEnded` | `B`, from the back | `v.iter().rfold(String::new(),\|a,x\| a + &x.to_string())` → `"54321"` |
| `try_rfold` | `try_rfold<B,R: Try>(init, f)` | `R` | same, short-circuiting |

`sum` and `product` also work over `Option`/`Result` items, collapsing them:

```rust
let so: Option<i32> = vec![Some(1), Some(2)].into_iter().sum();
let sr: Result<i32, &str> = vec![Ok(1), Err("bad"), Ok(3)].into_iter().sum();
println!("{so:?} {sr:?}");
```

Output:

```
Some(3) Err("bad")
```

## Consumers: search and test

All eager, all short-circuiting.

| Method | Signature | Returns | Example → output |
|---|---|---|---|
| `any` | `any(p: FnMut(Item) -> bool)` | `bool`; `false` when empty | `v.iter().any(\|x\| *x > 4)` → `true` |
| `all` | `all(p: FnMut(Item) -> bool)` | `bool`; `true` when empty | `v.iter().all(\|x\| *x > 0)` → `true` |
| `find` | `find(p: FnMut(&Item) -> bool)` | `Option<Item>` | `v.iter().find(\|x\| **x % 2 == 0)` → `Some(2)` |
| `rfind` | `rfind(p)` needs `DoubleEnded` | `Option<Item>` | `v.iter().rfind(\|x\| **x < 3)` → `Some(2)` |
| `find_map` | `find_map<B>(f: FnMut(Item) -> Option<B>)` | `Option<B>` | `["x","3"].iter().find_map(\|s\| s.parse::<i32>().ok())` → `Some(3)` |
| `position` | `position(p: FnMut(Item) -> bool)` | `Option<usize>` | `v.iter().position(\|x\| *x == 3)` → `Some(2)` |
| `rposition` | `rposition(p)` needs `DoubleEnded + ExactSize` | `Option<usize>` | `v.iter().rposition(\|x\| x % 2 == 1)` → `Some(4)` |
| `nth` | `nth(n: usize)` | `Option<Item>`, consumes the skipped | `v.iter().nth(2)` → `Some(3)` |
| `last` | `last()` | `Option<Item>`, O(n) | `v.iter().last()` → `Some(5)` |

## Consumers: min and max

All eager, all full passes.

| Method | Signature | Tie goes to |
|---|---|---|
| `max` / `min` | `max()` requires `Item: Ord` | last / first |
| `max_by_key` / `min_by_key` | `max_by_key<B: Ord>(f: FnMut(&Item) -> B)` | last / first |
| `max_by` / `min_by` | `max_by(f: FnMut(&Item,&Item) -> Ordering)` | last / first |

```rust
let words = vec!["aa", "bb", "cc"];
println!("{:?} {:?}", words.iter().max_by_key(|s| s.len()), words.iter().min_by_key(|s| s.len()));

let fs = vec![1.5f64, f64::NAN, 0.3];
println!("{:?}", fs.iter().copied().max_by(f64::total_cmp));
```

Output:

```
Some("cc") Some("aa")
Some(NaN)
```

Floats have no `Ord`, so `max()` does not exist on them. `total_cmp` sorts
`NaN` above everything; `partial_cmp(..).unwrap()` panics on it instead.

## Consumers: building collections

All eager.

| Method | Signature | Returns | Example → output |
|---|---|---|---|
| `collect` | `collect<B: FromIterator<Item>>()` | `B` | `v.iter().copied().collect::<Vec<i32>>()` → `[1,2,3,4,5]` |
| `partition` | `partition<B: Default+Extend>(p)` | `(B, B)`, true side first | `v.iter().partition(\|x\| **x%2==0)` → `([2,4],[1,3,5])` |
| `unzip` | `unzip<A,B,FA,FB>()` where `Item=(A,B)` | `(FA, FB)` | `[(1,'a')].into_iter().unzip()` → `([1],['a'])` |
| `for_each` | `for_each(f: FnMut(Item))` | `()` | `v.iter().for_each(\|x\| print!("{x}"))` |
| `try_for_each` | `try_for_each<R: Try>(f)` | `R`, short-circuits | returns first `Err` |

`collect` targets you will actually use:

| Target type | Needs |
|---|---|
| `Vec<T>`, `VecDeque<T>`, `LinkedList<T>` | `Item = T` |
| `String` | `Item` is `char`, `&str`, or `String` |
| `HashSet<T>`, `BTreeSet<T>` | `T: Hash+Eq` / `T: Ord` |
| `HashMap<K,V>`, `BTreeMap<K,V>` | `Item = (K, V)`; last duplicate key wins |
| `Result<Vec<T>, E>` | `Item = Result<T, E>`; first `Err` wins, short-circuits |
| `Vec<Result<T, E>>` | keeps every outcome |
| `Option<Vec<T>>` | `Item = Option<T>`; first `None` wins |
| `()` | `Item = ()`; drains the iterator |
| `Cow<str>` | `Item` is `char` or `&str` |

`extend` is `collect` into an existing collection — `collection.extend(iter)`.

## Consumers: comparison

Eager, element-wise, lexicographic. No allocation.

| Method | Returns | Example → output |
|---|---|---|
| `eq` / `ne` | `bool` | `[1,2].iter().eq([1,2].iter())` → `true` |
| `cmp` | `Ordering`, needs `Ord` | `[1,2].iter().cmp([1,3].iter())` → `Less` |
| `partial_cmp` | `Option<Ordering>` | `[1.0].iter().partial_cmp([f64::NAN].iter())` → `None` |
| `lt` / `le` / `gt` / `ge` | `bool` | `[1,2].iter().lt([1,2,0].iter())` → `true` |
| `eq_by` / `cmp_by` | custom comparator | |

## Consumers: ordering checks

| Method | Returns | Example → output |
|---|---|---|
| `is_sorted` | `bool` | `v.iter().is_sorted()` → `true` |
| `is_sorted_by` | `bool` with a comparator | |
| `is_sorted_by_key` | `bool` with a key fn | `v.iter().is_sorted_by_key(\|x\| -x)` → `false` |

```rust
println!("{} {}", v.iter().is_sorted(), [3, 1].iter().is_sorted());
```

Output:

```
true false
```

## Inspection and metadata

| Method | Returns | Note |
|---|---|---|
| `size_hint` | `(usize, Option<usize>)` | lower and optional upper bound |
| `len` | `usize`, needs `ExactSizeIterator` | exact, no iteration |
| `as_slice` | `&[T]` on `slice::Iter` | the unconsumed remainder |

```rust
println!("{:?}", (1..10).size_hint());
println!("{:?}", (1..10).filter(|x| x % 2 == 0).size_hint());
println!("{:?}", "abc".chars().size_hint());
println!("{}", v.iter().map(|x| x * 2).len());
```

Output:

```
(9, Some(9))
(0, Some(9))
(1, Some(3))
3
```

`filter` cannot know how many pass, so its lower bound drops to 0. `map`
preserves the count exactly, so `len` works through it. `chars` knows a UTF-8
string of 3 bytes holds between 1 and 3 chars.

## Sources: making an iterator from nothing

| Function | Yields | Example → output |
|---|---|---|
| `iter::once(x)` | one item | `iter::once(5)` → `[5]` |
| `iter::once_with(f)` | one item, lazily | `iter::once_with(\|\| String::new())` → `[""]` |
| `iter::empty()` | nothing | `iter::empty::<i32>()` → `[]` |
| `iter::repeat(x)` | `x` forever, needs `Clone` | `iter::repeat(7).take(3)` → `[7,7,7]` |
| `iter::repeat_n(x, n)` | `x`, `n` times | `iter::repeat_n(7, 3)` → `[7,7,7]` |
| `iter::repeat_with(f)` | `f()` forever | `iter::repeat_with(\|\| 1).take(2)` → `[1,1]` |
| `iter::from_fn(f)` | each `Some` from `f()` | closure-driven iterator |
| `iter::successors(first, f)` | `first`, `f(prev)`, … | `successors(Some(1u32), \|x\| x.checked_mul(3)).take(5)` → `[1,3,9,27,81]` |
| `a..b`, `a..=b` | the range | `(1..4)` → `[1,2,3]` |

```rust
use std::iter;

let mut n = 1;
println!("{:?}", iter::from_fn(|| { n *= 2; if n < 20 { Some(n) } else { None } }).collect::<Vec<i32>>());
println!("{:?}", iter::successors(Some(1u32), |x| x.checked_mul(3)).take(5).collect::<Vec<_>>());
```

Output:

```
[2, 4, 8, 16]
[1, 3, 9, 27, 81]
```

`successors` is the clean way to write a sequence defined by its predecessor —
no struct, no `impl Iterator`.

## Grouping and windowing live on slices

There are no `windows`, `chunks`, `dedup`, `sort`, or `group_by` methods on
`Iterator`. They need random access or a buffer, so they are **slice** or
**`Vec`** methods. Collect first, or use `itertools` (lesson 10.8).

| Method | On | Returns | Example → output |
|---|---|---|---|
| `windows(n)` | `&[T]` | overlapping slices of `n` | `v.windows(2)` → `[[1,2],[2,3],[3,4],[4,5]]` |
| `chunks(n)` | `&[T]` | non-overlapping, last may be short | `v.chunks(2)` → `[[1,2],[3,4],[5]]` |
| `chunks_exact(n)` | `&[T]` | drops the short tail; `.remainder()` gives it | `v.chunks_exact(2)` → `[[1,2],[3,4]]` |
| `rchunks(n)` | `&[T]` | chunks from the end | `v.rchunks(2)` → `[[4,5],[2,3],[1]]` |
| `split(p)` | `&[T]` | slices between matches | `v.split(\|x\| *x==3)` → `[[1,2],[4,5]]` |
| `split_first` / `split_last` | `&[T]` | `Option<(&T, &[T])>` | `v.split_first()` → `Some((1,[2,3,4,5]))` |
| `dedup` | `Vec<T>` | removes *consecutive* duplicates, in place | `[1,1,2,2,3].dedup()` → `[1,2,3]` |
| `sort` / `sort_unstable` | `Vec<T>` | in place | |
| `retain(p)` | `Vec<T>` | filter in place, no allocation | |
| `concat` / `join(sep)` | `&[&[T]]`, `&[&str]` | flatten / join | `["a","b"].join(",")` → `"a,b"` |

```rust
println!("{:?}", v.windows(3).map(|w| w.iter().sum::<i32>()).collect::<Vec<i32>>());
println!("{:?}", v.chunks_exact(2).remainder());
println!("{:?}", v.iter().as_slice());
```

Output:

```
[6, 9, 12]
[5]
[1, 2, 3, 4, 5]
```

Note `v.windows(3)` and `v.chunks(2)` *are* iterators once created — you can
chain adapters onto them. They just can't be created from an arbitrary
iterator.

## Where iterators come from

Standard types each offer several. Knowing these saves a `collect`.

| Type | Method | Yields |
|---|---|---|
| `Vec<T>` / `&[T]` | `iter` / `iter_mut` / `into_iter` | `&T` / `&mut T` / `T` |
| `HashMap<K,V>` / `BTreeMap` | `iter`, `iter_mut`, `into_iter` | `(&K,&V)`, `(&K,&mut V)`, `(K,V)` |
| | `keys`, `values`, `values_mut`, `into_keys`, `into_values` | the one half |
| `HashSet<T>` / `BTreeSet` | `iter`, `into_iter`, `union`, `intersection`, `difference` | `&T` / `T` / `&T` |
| `Option<T>` | `iter`, `into_iter` | 0 or 1 item |
| `Result<T,E>` | `iter`, `into_iter`, `iter_mut` | 0 or 1 `T` |
| `str` | `chars`, `char_indices`, `bytes`, `lines`, `split`, `splitn`, `split_whitespace`, `matches` | see lesson 6.4 |
| `BinaryHeap<T>` | `iter` (arbitrary order), `into_sorted_vec` | `&T` |

```rust
use std::collections::BTreeMap;

let m: BTreeMap<i32, i32> = vec![3, 1, 2].iter().map(|x| (*x, x * 10)).collect();
println!("{:?}", m.keys().collect::<Vec<_>>());
println!("{}", m.values().sum::<i32>());
println!("{:?}", m.iter().rev().collect::<Vec<_>>());
println!("{:?}", m.into_values().collect::<Vec<i32>>());
```

Output:

```
[1, 2, 3]
60
[(3, 30), (2, 20), (1, 10)]
[10, 20, 30]
```

`BTreeMap` is double-ended and ordered, so `rev` works and the order is
meaningful. `HashMap` gives neither.

```rust
let mut m2: BTreeMap<&str, i32> = BTreeMap::from([("a", 1), ("b", 2)]);
for val in m2.values_mut() { *val *= 10; }
println!("{m2:?}");
```

Output:

```
{"a": 10, "b": 20}
```

## Marker traits that unlock methods

| Trait | Required method | Unlocks |
|---|---|---|
| `Iterator` | `next` | everything in the tables above |
| `DoubleEndedIterator` | `next_back` | `rev`, `rfind`, `rfold`, `rposition`, `try_rfold` |
| `ExactSizeIterator` | `len` (or a tight `size_hint`) | `len`, `rposition` |
| `FusedIterator` | — (marker) | lets adapters skip redundant `None` checks |
| `Extend` | `extend` | on the *collection*, the target of `extend` |
| `FromIterator` | `from_iter` | makes a type a `collect` target |
| `IntoIterator` | `into_iter` | usable in a `for` loop |

`next_back` lets you consume from both ends at once:

```rust
let mut it = v.iter();
println!("{:?} {:?} {:?}", it.next(), it.next_back(), it.next());
```

Output:

```
Some(1) Some(3) Some(2)
```

Which iterators are double-ended: slices, `Vec`, arrays, ranges, `VecDeque`,
`BTreeMap`/`BTreeSet`, `chars`. Which are not: `HashMap`/`HashSet` iterators,
`Lines`, `successors`, `from_fn`, anything wrapped in `flat_map` over a
non-double-ended inner iterator.

## Which one do I want?

| Goal | Use |
|---|---|
| Transform every item | `map` |
| Keep some items | `filter` |
| Transform and drop failures | `filter_map` |
| Transform and stop at first failure | `map_while` |
| One item → many | `flat_map` |
| Unnest one level | `flatten` |
| Index alongside the item | `enumerate`, or `zip(1..)` for 1-based |
| Walk two sequences together | `zip` |
| Concatenate two sequences | `chain` |
| First n / all but first n | `take` / `skip` |
| Stop at a condition | `take_while` |
| Start at a condition | `skip_while` |
| Every nth | `step_by` |
| Backwards | `rev` |
| Look ahead one item | `peekable` |
| Running total | `scan` |
| Debug a chain | `inspect` |
| Build a `Vec`/`String`/map/set | `collect` |
| Single value from all items | `fold`, or `sum`/`product`/`max`/`count` |
| Single value, no identity available | `reduce` |
| Reduction that can fail | `try_fold` |
| Does any / do all match | `any` / `all` |
| First match: the item | `find` |
| First match: a transformation | `find_map` |
| First match: the index | `position` |
| Largest / smallest | `max` / `min`, `_by_key` for one field, `_by` for floats |
| Split into two groups | `partition` |
| Split tuples into two collections | `unzip` |
| Side effects only | `for_each`, or just a `for` loop |
| Append onto an existing collection | `collection.extend(iter)` |
| Sliding pairs/triples | `v.windows(n)`, or `zip(skip(1))` for pairs |
| Fixed-size batches | `v.chunks(n)` |
| Remove adjacent duplicates | `v.dedup()` |
| Remove all duplicates | `collect::<HashSet<_>>()`, or `itertools::unique` |
| Sort | `collect` then `sort`, or `itertools::sorted` |
| Group by a key | `fold` into a `HashMap<K, Vec<V>>`, or `itertools::chunk_by` |
| Join into a string | `collect::<Vec<_>>().join(sep)`, or `itertools::join` |
| Run in parallel | `rayon`'s `par_iter` (lesson 10.8) |

## Laziness summary

| Category | Lazy | Short-circuits |
|---|---|---|
| `map`, `filter`, `filter_map`, `flat_map`, `flatten`, `inspect`, `cloned`, `copied`, `scan`, `enumerate`, `zip`, `chain`, `take`, `skip`, `step_by`, `rev`, `cycle`, `peekable`, `fuse`, `by_ref`, `map_while`, `take_while`, `skip_while` | yes | n/a |
| `any`, `all`, `find`, `rfind`, `find_map`, `position`, `rposition`, `try_fold`, `try_for_each` | no | yes |
| `collect`, `sum`, `product`, `count`, `fold`, `rfold`, `reduce`, `for_each`, `max*`, `min*`, `partition`, `unzip`, `last`, `is_sorted*`, `eq`/`cmp`/`lt` | no | no (`eq`/`cmp` stop at the first difference) |
| `nth`, `size_hint`, `len` | no | `nth` stops at `n` |

## Common Mistakes

**Looking for `group_by` or `windows` on `Iterator`.** They are slice methods
or `itertools`; collect first.

**Using `nth`/`last` on a `Vec` iterator.** Both are O(n); index or `.last()`
on the slice instead.

**Calling `max()` on floats.** `f64` is not `Ord`; use
`max_by(f64::total_cmp)`.

**Calling `rev()` on a `HashMap` iterator.** Not `DoubleEndedIterator`, and
there is no meaningful order anyway.

**Assuming `dedup` removes all duplicates.** It only removes *adjacent* ones;
sort first, or collect into a set.

## Exercises

1. For `vec![4, 8, 15, 16, 23, 42]`, compute with one chain each: the sum of
   odd-indexed elements, the largest value under 20, and the running maximum.
2. Write the same "every third element" three ways: `step_by`, `zip(0..)` with
   a filter, and `enumerate` with a filter. Compare readability.
3. Use `iter::successors` to generate powers of two under 1000 without a
   mutable counter.
4. For `vec![1, 2, 3, 4, 5]`, compute the differences between adjacent elements
   using `windows`, then using `zip(skip(1))`.
5. Pick five methods from the tables you have never used and write one line
   each demonstrating the output.
6. Given `Vec<&str>` of names, produce a `BTreeMap<char, Vec<&str>>` grouped by
   first letter, using `fold`.

---

[← 10.5 Consumers: collect, fold, and Friends](05-consumers.md) · [Course Index](../README.md) · [10.7 Writing Your Own Iterator →](07-custom-iterator.md)
