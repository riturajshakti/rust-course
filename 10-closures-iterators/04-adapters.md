[← Course Index](../README.md)

# 10.4 — Adapters: map, filter, and Friends

## What You'll Learn

- Every common adapter, with what it yields
- `filter_map`, `flat_map`, `scan`, `peekable`, `by_ref` — the ones worth knowing
- Chaining several, and why order changes performance by 10x
- The `&&T` deref noise `filter` produces, and how to kill it

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `.map().filter()` (eager, allocates each step) | lazy, allocates once at `collect` |
| Python `map()` / `itertools` | same laziness, built into every iterator |
| Go hand-written loops | adapters, same speed |

An **adapter** takes an iterator and returns a new iterator. It never runs
anything on its own (lesson 10.3); a consumer (lesson 10.5) drives the chain.

## map

```rust
let v = vec![1, 2, 3, 4, 5];
println!("{:?}", v.iter().map(|x| x * x).collect::<Vec<i32>>());
println!("{:?}", v.iter().map(|x| x.to_string()).collect::<Vec<String>>());
```

Output:

```
[1, 4, 9, 16, 25]
["1", "2", "3", "4", "5"]
```

One in, one out, always the same count. The output item type is whatever the
closure returns.

## filter

The closure receives a **reference to the item**, so on `v.iter()` it gets
`&&i32`.

```rust
println!("{:?}", v.iter().filter(|x| **x % 2 == 0).collect::<Vec<&i32>>());
println!("{:?}", v.iter().copied().filter(|x| x % 2 == 0).collect::<Vec<i32>>());
```

Output:

```
[2, 4]
[2, 4]
```

Forget a `*` and you get this:

```rust
let out: Vec<&i32> = v.iter().filter(|x| x % 2 == 0).collect();
```

Does not compile:

```
error[E0369]: cannot calculate the remainder of `&&{integer}` divided by `{integer}`
 --> src/main.rs:3:48
  |
3 |     let out: Vec<&i32> = v.iter().filter(|x| x % 2 == 0).collect();
  |                                              - ^ - {integer}
  |                                              |
  |                                              &&{integer}
  |
help: `%` can be used on `&{integer}` if you dereference the left-hand side
  |
3 |     let out: Vec<&i32> = v.iter().filter(|x| *x % 2 == 0).collect();
  |                                              +
```

Idiomatic fix for `Copy` items: insert `.copied()` before the `filter` and work
with plain values. For pattern-matching the reference away, `|&&x| x % 2 == 0`
also works.

## filter_map

Map and filter in one: return `Option`, `None` drops the item.

```rust
let raw = vec!["1", "x", "3", "", "5"];
println!("{:?}", raw.iter().filter_map(|s| s.parse::<i32>().ok()).collect::<Vec<i32>>());
println!("{:?}", raw.iter().map(|s| s.parse::<i32>()).filter(|r| r.is_ok()).collect::<Vec<_>>());
```

Output:

```
[1, 3, 5]
[Ok(1), Ok(3), Ok(5)]
```

The second form leaves the `Result` wrappers on. `filter_map` with `.ok()` is
the standard "parse what you can, discard the rest" idiom.

## flat_map and flatten

`flat_map` maps each item to an iterator and concatenates.

```rust
let words = vec!["hello world", "foo bar"];
println!("{:?}", words.iter().flat_map(|s| s.split(' ')).collect::<Vec<&str>>());
```

Output:

```
["hello", "world", "foo", "bar"]
```

`flatten` is `flat_map(|x| x)` — one level of nesting removed.

```rust
let nested = vec![vec![1, 2], vec![3], vec![]];
println!("{:?}", nested.into_iter().flatten().collect::<Vec<i32>>());

let opts = vec![Some(1), None, Some(3)];
println!("{:?}", opts.into_iter().flatten().collect::<Vec<i32>>());
```

Output:

```
[1, 2, 3]
[1, 3]
```

`Option` and `Result` are iterators of 0 or 1 items, which is why `flatten`
unwraps them. Only one level — nested `Vec<Vec<Vec<T>>>` needs two calls.

## enumerate

```rust
println!("{:?}", v.iter().enumerate().collect::<Vec<(usize, &i32)>>());
for (i, x) in v.iter().enumerate().take(2) { println!("{i}:{x}"); }
```

Output:

```
[(0, 1), (1, 2), (2, 3), (3, 4), (4, 5)]
0:1
1:2
```

Index is always `usize` and always starts at 0. For a different start, use
`.zip(1..)`.

## zip

Stops at the shorter side.

```rust
let names = vec!["a", "b", "c"];
let ages = vec![30, 40];
println!("{:?}", names.iter().zip(ages.iter()).collect::<Vec<_>>());
println!("{:?}", names.iter().zip(1..).collect::<Vec<_>>());
```

Output:

```
[("a", 30), ("b", 40)]
[("a", 1), ("b", 2), ("c", 3)]
```

Zipping against an unbounded range `1..` is the idiomatic 1-based enumerate.

## unzip is a consumer, not an adapter

It is the inverse of `zip` but it must run the whole iterator, so it returns
collections, not an iterator.

```rust
let pairs = vec![(1, 'a'), (2, 'b')];
let (ns, cs): (Vec<i32>, Vec<char>) = pairs.into_iter().unzip();
println!("{ns:?} {cs:?}");
```

Output:

```
[1, 2] ['a', 'b']
```

Lesson 10.5 covers it with the other consumers.

## chain

```rust
println!("{:?}", (1..3).chain(10..12).collect::<Vec<i32>>());
```

Output:

```
[1, 2, 10, 11]
```

Both sides must have the same `Item` type.

## take, take_while, skip, skip_while

```rust
println!("{:?}", v.iter().take(2).collect::<Vec<_>>());
println!("{}", v.iter().take(99).count());
println!("{:?}", v.iter().take_while(|x| **x < 4).collect::<Vec<_>>());
println!("{:?}", v.iter().skip(3).collect::<Vec<_>>());
println!("{:?}", v.iter().skip_while(|x| **x < 4).collect::<Vec<_>>());
```

Output:

```
[1, 2]
5
[1, 2, 3]
[4, 5]
[4, 5]
```

`take(n)` past the end is not an error — it just yields fewer. The `_while`
variants stop at the **first** failure and never look again, so on
`[1, 9, 2]`, `take_while(|x| **x < 4)` gives `[1]`, not `[1, 2]`.

## step_by and rev

```rust
println!("{:?}", (0..20).step_by(5).collect::<Vec<i32>>());
println!("{:?}", v.iter().rev().collect::<Vec<_>>());
println!("{:?}", v.iter().rev().skip(1).take(2).collect::<Vec<_>>());
```

Output:

```
[0, 5, 10, 15]
[5, 4, 3, 2, 1]
[4, 3]
```

`step_by` always yields the first element then every nth after. `rev` requires
a `DoubleEndedIterator` — `Vec`, slices, ranges, `VecDeque` qualify;
`HashMap` and `Lines` do not.

## peekable

`peek()` looks at the next item without consuming it. Needs a `mut` binding.

```rust
let mut it = v.iter().peekable();
while let Some(x) = it.next() {
    match it.peek() {
        Some(nxt) => println!("{x} -> {nxt}"),
        None => println!("{x} -> end"),
    }
}
```

Output:

```
1 -> 2
2 -> 3
3 -> 4
4 -> 5
5 -> end
```

Real use: grouping consecutive runs, which no plain adapter does.

```rust
let data = vec![1, 1, 2, 2, 2, 3];
let mut it = data.iter().peekable();
let mut runs = Vec::new();
while let Some(x) = it.next() {
    let mut n = 1;
    while it.peek() == Some(&x) { it.next(); n += 1; }
    runs.push((*x, n));
}
println!("{runs:?}");
```

Output:

```
[(1, 2), (2, 3), (3, 1)]
```

## inspect

A `map` that returns nothing — pure debugging. It sees only what reaches it.

```rust
let total: i32 = v.iter().inspect(|x| println!("saw {x}")).filter(|x| **x > 3).sum();
println!("{total}");
```

Output:

```
saw 1
saw 2
saw 3
saw 4
saw 5
9
```

Move the `inspect` after the `filter` and you would see only 4 and 5. That is
how you find where a chain drops elements.

## scan

`fold` that yields every intermediate. The closure gets `&mut state` and
returns `Option` — `None` ends the iterator.

```rust
println!("{:?}", v.iter().scan(0, |acc, x| { *acc += x; Some(*acc) }).collect::<Vec<i32>>());
println!("{:?}", v.iter().scan(0, |acc, x| { *acc += x; if *acc > 6 { None } else { Some(*acc) } }).collect::<Vec<i32>>());
```

Output:

```
[1, 3, 6, 10, 15]
[1, 3, 6]
```

Running totals, cumulative products, stateful parsers.

## cycle

Infinite repeat. Always pair it with `take` or a short-circuiting consumer.

```rust
println!("{:?}", [1, 2, 3].iter().cycle().take(7).collect::<Vec<_>>());
```

Output:

```
[1, 2, 3, 1, 2, 3, 1]
```

`cycle` on an empty iterator yields nothing; it does not hang.

## map_while

`filter_map` that stops at the first `None` instead of skipping it.

```rust
let raw = vec!["1", "2", "x", "4"];
println!("{:?}", raw.iter().map_while(|s| s.parse::<i32>().ok()).collect::<Vec<i32>>());
println!("{:?}", raw.iter().filter_map(|s| s.parse::<i32>().ok()).collect::<Vec<i32>>());
```

Output:

```
[1, 2]
[1, 2, 4]
```

## cloned and copied

Turn `&T` into `T`. `copied` requires `Copy` and is the one to prefer when it
applies — it cannot accidentally deep-clone.

```rust
let strs = vec![String::from("a"), String::from("b")];
println!("{:?}", strs.iter().cloned().collect::<Vec<String>>());
println!("{:?}", v.iter().copied().collect::<Vec<i32>>());
```

Output:

```
["a", "b"]
[1, 2, 3, 4, 5]
```

## by_ref

Adapters take `self` by value, so `it.take(2)` normally eats `it`. `by_ref`
hands them a `&mut` instead, leaving the original usable.

```rust
let mut it = v.iter();
let first2: Vec<_> = it.by_ref().take(2).collect();
let rest: Vec<_> = it.collect();
println!("{first2:?} {rest:?}");
```

Output:

```
[1, 2] [3, 4, 5]
```

This is how you split a stream — read a header, then the body.

## Chaining several

```rust
let log = "\
200 /home
404 /missing
200 /about
500 /api
bad line
200 /home";

let codes: Vec<(u16, &str)> = log
    .lines()
    .filter_map(|l| l.split_once(' '))
    .filter_map(|(c, p)| c.parse::<u16>().ok().map(|c| (c, p)))
    .filter(|(c, _)| *c < 400)
    .collect();
println!("{codes:?}");
```

Output:

```
[(200, "/home"), (200, "/about"), (200, "/home")]
```

Each stage is independent and nothing is allocated until `collect`. Another
chain over the same input:

```rust
let report: Vec<String> = log
    .lines()
    .enumerate()
    .filter_map(|(i, l)| l.split_once(' ').map(|(c, p)| (i + 1, c, p)))
    .skip_while(|(_, c, _)| *c == "200")
    .take(3)
    .map(|(n, c, p)| format!("line {n}: {c} {p}"))
    .collect();
for r in &report { println!("{r}"); }
```

Output:

```
line 2: 404 /missing
line 3: 200 /about
line 4: 500 /api
```

## Order matters

Put cheap filters before expensive maps. Same result, very different cost.

```rust
use std::hint::black_box;
use std::time::Instant;

fn expensive(s: &str) -> String { s.to_uppercase() }

let words: Vec<String> = (0..1_000_000).map(|i| format!("word{i}")).collect();

let t = Instant::now();
let a: usize = words.iter().map(|s| expensive(s)).filter(|s| s.ends_with('7')).count();
let map_first = t.elapsed();

let t = Instant::now();
let b: usize = words.iter().filter(|s| s.ends_with('7')).map(|s| expensive(s)).count();
let filter_first = t.elapsed();

println!("map then filter: {} in {map_first:?}", black_box(a));
println!("filter then map: {} in {filter_first:?}", black_box(b));
```

Output (`cargo run --release`; numbers are machine-specific):

```
map then filter: 100000 in 28.217584ms
filter then map: 100000 in 2.32025ms
```

12x, because the first version allocates a `String` for all million inputs and
throws 900,000 away. Also put `take` as early as correctness allows.

## Quick Reference

| Adapter | Yields | Note |
|---|---|---|
| `map(f)` | `f(item)` | same count |
| `filter(p)` | items where `p(&item)` | closure gets a reference |
| `filter_map(f)` | inner of each `Some` | map + filter |
| `flat_map(f)` | items of each `f(item)` | |
| `flatten()` | items of each item | one level only |
| `enumerate()` | `(usize, item)` | always from 0 |
| `zip(other)` | `(a, b)` | stops at the shorter |
| `chain(other)` | all of self, then all of other | same `Item` |
| `take(n)` | first `n` | fewer if shorter |
| `take_while(p)` | until `p` first fails | |
| `skip(n)` | all but the first `n` | |
| `skip_while(p)` | from the first failure of `p` on | |
| `step_by(n)` | every nth, starting at the first | |
| `rev()` | back to front | needs `DoubleEndedIterator` |
| `peekable()` | same items, adds `.peek()` | needs `mut` |
| `inspect(f)` | same items, calls `f(&item)` | debugging |
| `scan(init, f)` | each `Some` from `f(&mut state, item)` | `None` ends it |
| `cycle()` | repeats forever | needs `Clone` |
| `map_while(f)` | inner of each `Some`, stops at first `None` | |
| `cloned()` / `copied()` | `T` from `&T` | `copied` needs `Copy` |
| `by_ref()` | `&mut` the iterator | keeps the original usable |

## Common Mistakes

**Forgetting `*` in a `filter` closure.** The closure gets `&Item`, so on
`.iter()` it is `&&T`. Add `.copied()` or pattern-match with `|&&x|`.

**Using `take_while` when you meant `filter`.** `take_while` stops at the first
failure; `filter` keeps scanning.

**Building a chain with no consumer.** Nothing runs — see the `must_use`
warning in lesson 10.3.

**Mapping before filtering.** Do the cheap test first; it can be an order of
magnitude.

**Expecting `flatten` to go all the way down.** It removes exactly one level.

## Exercises

1. From `vec!["10", "abc", "30"]`, produce `Vec<i32>` of the parseable values
   with `filter_map`, then the `map` + `filter` + `map` equivalent.
2. Given `Vec<Vec<i32>>`, get a sorted `Vec<i32>` of every element greater
   than 5, using `flatten`.
3. Zip `vec!["a","b","c"]` against `1..` and print `1. a` style lines.
4. Use `peekable` to collapse `vec![1,1,1,2,3,3]` into `vec![(1,3),(2,1),(3,2)]`.
5. Use `scan` to turn `vec![10, -3, 5]` into the running balance
   `[10, 7, 12]`, then make it stop as soon as the balance exceeds 11.
6. Read the first two items of an iterator with `by_ref().take(2)` and the rest
   separately. Then remove `by_ref` and record the error.

---

[← Course Index](../README.md)
