[← 7.7 HashSet and BTreeSet](07-sets.md) · [Course Index](../README.md) · [7.9 Choosing the Right Collection →](09-choosing.md)

# 7.8 — VecDeque, BinaryHeap, LinkedList

## What You'll Learn

- `VecDeque` for O(1) push and pop at both ends
- `BinaryHeap` as a max-heap, and `Reverse` to flip it
- A top-k selection in six lines
- Why `LinkedList` exists and why you will not use it

## Coming From Another Language

| Other | Rust |
|---|---|
| Python `collections.deque` | `VecDeque<T>` |
| Python `heapq` (min-heap) | `BinaryHeap<Reverse<T>>` |
| JS `arr.shift()` | `deque.pop_front()` |
| Go `container/heap` | `BinaryHeap<T>` |
| Java `ArrayDeque` / `PriorityQueue` | `VecDeque` / `BinaryHeap` |

```rust
use std::collections::{VecDeque, BinaryHeap, LinkedList};
```

## VecDeque: a ring buffer

Same ring-buffer idea as a `Vec`, but with a head offset, so both ends are
O(1). It is still one contiguous allocation — not a linked list.

```rust
let mut d: VecDeque<i32> = VecDeque::new();
d.push_back(1);
d.push_back(2);
d.push_front(0);
println!("{:?}", d);
println!("{:?} {:?}", d.pop_front(), d.pop_back());
println!("{:?}", d);

let e = VecDeque::from([1, 2, 3]);
let f: VecDeque<i32> = (1..4).collect();
let g: VecDeque<i32> = VecDeque::with_capacity(8);
println!("{:?} {:?} {}", e, f, g.capacity() >= 8);
```

Output:

```
[0, 1, 2]
Some(0) Some(2)
[1]
[1, 2, 3] [1, 2, 3] true
```

All four push/pop methods: `push_front`, `push_back`, `pop_front`, `pop_back`.
The pops return `Option<T>`.

## Why not just use a Vec

`Vec::remove(0)` shifts every element. Removing 200,000 elements from the
front, one at a time, release build:

```rust
use std::time::Instant;

let n = 200_000usize;
let mut sum = 0usize;

let mut v: Vec<usize> = (0..n).collect();
let t = Instant::now();
while let Some(x) = v.first().copied() { sum += x; v.remove(0); }
let a = t.elapsed();

let mut d: VecDeque<usize> = (0..n).collect();
let t = Instant::now();
while let Some(x) = d.pop_front() { sum += x; }
let b = t.elapsed();

println!("Vec::remove(0)      x{n}: {:?}", a);
println!("VecDeque::pop_front x{n}: {:?}", b);
```

Output:

```
Vec::remove(0)      x200000: 2.272394458s
VecDeque::pop_front x200000: 185.625µs
```

O(n²) versus O(n), 12,000x apart. Any time you find yourself writing
`remove(0)`, this is the type you wanted.

## VecDeque indexed access and middle operations

```rust
let mut d = VecDeque::from([10, 20, 30]);
println!("{:?} {:?}", d.front(), d.back());
println!("{:?}", d.get(1));
println!("{:?}", d.get(9));
println!("{}", d[0]);

if let Some(x) = d.front_mut() { *x = 11; }
if let Some(x) = d.back_mut() { *x = 31; }
println!("{:?}", d);

d.insert(1, 99);
println!("{:?}", d);
println!("{:?}", d.remove(1));
println!("{:?}", d.swap_remove_front(1));
println!("{:?}", d);
```

Output:

```
Some(10) Some(30)
Some(20)
None
10
[11, 20, 31]
[11, 99, 20, 31]
Some(99)
Some(20)
[11, 31]
```

Indexing works and is O(1), but `d[i]` is an offset calculation plus a
wraparound check, so it is marginally slower than `Vec`. `insert` and `remove`
in the middle are O(n) — but only O(min(i, len-i)), since it shifts from the
nearer end. `swap_remove_front(i)` and `swap_remove_back(i)` are the O(1),
order-destroying variants.

The rest of the `Vec` vocabulary carries over:

```rust
let mut d = VecDeque::from([1, 2, 3, 4]);
println!("{:?}", d.swap_remove_back(0));
println!("{:?}", d);
d.rotate_left(1);
println!("{:?}", d);
println!("{}", d.contains(&2));
println!("{:?}", d.iter().rev().collect::<Vec<_>>());

let mut d = VecDeque::from([1, 2, 3]);
d.extend([4, 5]);
let mut o = VecDeque::from([6]);
d.append(&mut o);
println!("{:?} {:?}", d, o);
d.retain(|x| x % 2 == 0);
println!("{:?}", d);

let mut d = VecDeque::from([1, 2, 3, 4, 5]);
let dr: Vec<i32> = d.drain(1..3).collect();
println!("{:?} {:?}", dr, d);
let hi = d.split_off(1);
println!("{:?} {:?}", d, hi);
```

Output:

```
Some(1)
[4, 2, 3]
[2, 3, 4]
true
[4, 3, 2]
[1, 2, 3, 4, 5, 6] []
[2, 4, 6]
[2, 3] [1, 4, 5]
[1] [4, 5]
```

## VecDeque as a queue

```rust
let mut q = VecDeque::new();
for job in ["a", "b", "c"] { q.push_back(job); }
while let Some(job) = q.pop_front() { println!("processing {job}"); }
```

Output:

```
processing a
processing b
processing c
```

That `while let Some(..) = q.pop_front()` is the canonical FIFO drain. Swap
`pop_front` for `pop_back` and you have a LIFO stack — though for a pure stack
a `Vec` with `push`/`pop` is simpler and faster.

## VecDeque as a fixed-size ring buffer

Keep only the last N items:

```rust
let mut ring: VecDeque<i32> = VecDeque::with_capacity(3);
for i in 1..=6 {
    if ring.len() == 3 { ring.pop_front(); }
    ring.push_back(i);
    println!("{:?}", ring);
}
```

Output:

```
[1]
[1, 2]
[1, 2, 3]
[2, 3, 4]
[3, 4, 5]
[4, 5, 6]
```

Perfect for a rolling window, a last-N-log-lines buffer, or a moving average.

## make_contiguous

Because the data wraps around the allocation, a `VecDeque` is **two** slices,
not one.

```rust
let mut d: VecDeque<i32> = VecDeque::new();
d.push_back(2);
d.push_back(3);
d.push_front(1);
println!("{:?}", d.as_slices());
println!("{:?}", d.make_contiguous());
println!("{:?}", d.as_slices());
```

Output:

```
([1], [2, 3])
[1, 2, 3]
([1, 2, 3], [])
```

`make_contiguous` rotates the elements so they are one run, and returns
`&mut [T]`. That is what you need to pass a deque to anything expecting a
slice, or to use a slice method:

```rust
let mut d2 = VecDeque::from([3, 1, 2]);
d2.push_front(0);
d2.make_contiguous().sort();
println!("{:?}", d2);
println!("{}", std::mem::size_of::<VecDeque<i32>>());
```

Output:

```
[0, 1, 2, 3]
32
```

Conversion between `Vec` and `VecDeque` is `From` in both directions, and
`VecDeque -> Vec` is O(1) when it is already contiguous:

```rust
let v: Vec<i32> = VecDeque::from([1, 2, 3]).into();
let dq: VecDeque<i32> = vec![1, 2, 3].into();
println!("{:?} {:?}", v, dq);
```

Output:

```
[1, 2, 3] [1, 2, 3]
```

## BinaryHeap: a max-heap

`push` and `pop` are O(log n); `peek` is O(1). `pop` always yields the
**largest** remaining element.

```rust
let mut h = BinaryHeap::new();
h.push(3); h.push(1); h.push(4); h.push(1); h.push(5);
println!("{:?}", h.peek());
println!("{}", h.len());

let mut out = vec![];
while let Some(x) = h.pop() { out.push(x); }
println!("{:?}", out);
```

Output:

```
Some(5)
5
[5, 4, 3, 1, 1]
```

Construction and the rest of the surface:

```rust
let h2 = BinaryHeap::from([3, 1, 4, 1, 5]);
println!("{:?}", h2.peek());
println!("{:?}", h2.clone().into_sorted_vec());
println!("{:?}", h2.clone().into_vec());

let h3: BinaryHeap<i32> = vec![9, 2, 7].into_iter().collect();
println!("{:?}", h3.peek());

let mut h = BinaryHeap::from([1, 2, 3]);
h.extend([10, 0]);
println!("{:?}", h.peek());

let mut a = BinaryHeap::from([1, 2]);
let mut b = BinaryHeap::from([9]);
a.append(&mut b);
println!("{:?} {}", a.peek(), b.len());

let mut h = BinaryHeap::from([5, 1]);
if let Some(mut top) = h.peek_mut() { *top = 0; }
println!("{:?}", h.into_sorted_vec());
```

Output:

```
Some(5)
[1, 1, 3, 4, 5]
[5, 3, 4, 1, 1]
Some(9)
Some(10)
Some(9) 0
[0, 1]
```

Two things there. `into_sorted_vec()` gives you an ascending `Vec` in
O(n log n) — the easy way to drain a heap in order. `into_vec()` gives you the
raw backing array in **heap order**, which is not sorted; use it only to
recycle the allocation. And `peek_mut` returns a guard that re-sifts the heap
when dropped, which is why setting the top to 0 still produced a valid heap.

Iteration order is arbitrary — the heap only guarantees the root:

```rust
let h = BinaryHeap::from([3, 1, 4, 1, 5]);
println!("{:?}", h.iter().collect::<Vec<_>>());
```

Output:

```
[5, 3, 4, 1, 1]
```

Only `peek()` and the `pop()` sequence are meaningful. `BinaryHeap` has no
`get`, no `contains`, and no `remove(x)` — it is a priority queue, not a
container you search.

## Reverse for a min-heap

There is no `MinHeap`. Wrap values in `std::cmp::Reverse`, which inverts their
`Ord`.

```rust
use std::cmp::Reverse;

let mut mh = BinaryHeap::new();
for x in [3, 1, 4, 1, 5] { mh.push(Reverse(x)); }
println!("{:?}", mh.peek());

let mut out = vec![];
while let Some(Reverse(x)) = mh.pop() { out.push(x); }
println!("{:?}", out);
```

Output:

```
Some(Reverse(1))
[1, 1, 3, 4, 5]
```

The `Reverse(x)` destructuring in the `while let` pattern unwraps it on the
way out, so the noise is confined to push and pop.

## Top-k with a bounded min-heap

The classic use: keep the k largest items while streaming, in O(n log k) time
and O(k) space. Hold a **min**-heap of size k and evict its smallest whenever
it overflows.

```rust
use std::cmp::Reverse;

let scores = [("raj", 88), ("sam", 95), ("ana", 72), ("kim", 99), ("leo", 81)];
let mut heap = BinaryHeap::new();
for (n, s) in scores {
    heap.push(Reverse((s, n)));
    if heap.len() > 3 { heap.pop(); }
}

let mut top: Vec<(i32, &str)> = heap.into_iter().map(|Reverse(p)| p).collect();
top.sort_by(|a, b| b.0.cmp(&a.0));
println!("{:?}", top);
```

Output:

```
[(99, "kim"), (95, "sam"), (88, "raj")]
```

Tuples get a derived `Ord` that compares left to right, so `(score, name)`
sorts by score first — no comparator needed. For small k over a large stream
this beats sorting everything outright.

## LinkedList

A doubly-linked list. It exists, it works, and you almost never want it:
every element is a separate heap allocation, so iteration thrashes the cache
and is many times slower than a `Vec` or `VecDeque` for the same work. Its one
theoretical advantage — O(1) splice at a known position — needs the `Cursor`
API, and the `Vec`/`VecDeque` alternative usually wins anyway on real data.

```rust
let mut l: LinkedList<i32> = LinkedList::new();
l.push_back(2);
l.push_back(3);
l.push_front(1);
println!("{:?}", l);
println!("{:?} {:?}", l.front(), l.back());
println!("{:?} {:?}", l.pop_front(), l.pop_back());
println!("{} {}", l.len(), l.contains(&2));

let mut a = LinkedList::from([1, 2]);
let mut b = LinkedList::from([3]);
a.append(&mut b);
println!("{:?} {}", a, b.len());
let tail = a.split_off(1);
println!("{:?} {:?}", a, tail);
```

Output:

```
[1, 2, 3]
Some(1) Some(3)
Some(1) Some(3)
1 true
[1, 2, 3] 0
[1] [2, 3]
```

Note there is no indexing at all — no `l[0]`, no `get(i)`. Use `VecDeque`.

## Quick Reference

### VecDeque

| Operation | Method | Cost |
|---|---|---|
| Add front / back | `push_front`, `push_back` | O(1) amortized |
| Remove front / back | `pop_front`, `pop_back` → `Option<T>` | O(1) |
| Peek ends | `front`, `back`, `front_mut`, `back_mut` | O(1) |
| Index | `d[i]`, `get(i)`, `get_mut(i)` | O(1) |
| Insert / remove middle | `insert(i, x)`, `remove(i)` | O(min(i, n-i)) |
| Fast middle remove | `swap_remove_front(i)`, `swap_remove_back(i)` | O(1) |
| Bulk | `extend`, `append`, `drain(r)`, `retain`, `split_off(i)`, `truncate` | |
| Rearrange | `rotate_left`, `rotate_right`, `make_contiguous` | O(n) |
| Slices | `as_slices()` → two slices, `make_contiguous()` → one | |
| Convert | `Vec::from(d)`, `VecDeque::from(v)` | O(1) if contiguous |

### BinaryHeap

| Operation | Method | Cost |
|---|---|---|
| Add | `push(x)` | O(log n) |
| Remove largest | `pop()` → `Option<T>` | O(log n) |
| Inspect largest | `peek()` → `Option<&T>` | O(1) |
| Mutate largest | `peek_mut()` → guard, re-sifts on drop | O(log n) |
| Build | `from([..])`, `.collect()`, `with_capacity(n)` | O(n) |
| Bulk | `extend`, `append(&mut other)`, `drain()`, `clear()` | |
| Drain sorted | `into_sorted_vec()` ascending | O(n log n) |
| Raw buffer | `into_vec()` — heap order, not sorted | O(1) |
| Min-heap | `BinaryHeap<Reverse<T>>` | |
| Not available | `get`, `contains`, `remove(x)`, meaningful `iter` order | |

### LinkedList

| Operation | Method |
|---|---|
| Ends | `push_front`, `push_back`, `pop_front`, `pop_back`, `front`, `back` |
| Bulk | `append`, `split_off(i)`, `contains`, `len`, `clear` |
| Not available | indexing, `get(i)` |

## Common Mistakes

**`Vec::remove(0)` in a loop.** Use `VecDeque::pop_front`.

**Iterating a `BinaryHeap` and expecting sorted order.** Only the root is
ordered. Use `into_sorted_vec()` or repeated `pop()`.

**Looking for `MinHeap`.** Push `Reverse(x)`, or negate the key for numbers.

**Passing a `VecDeque` where a `&[T]` is wanted.** Call `make_contiguous()`
first; `as_slices()` returns two.

**Using `into_vec()` thinking it sorts.** That is `into_sorted_vec()`.

**Choosing `LinkedList` for "fast insertion".** The allocation per node and
cache misses cost more than the shifting you avoided.

## Exercises

1. Implement a breadth-first traversal of a tree represented as
   `HashMap<u32, Vec<u32>>` using a `VecDeque` as the frontier.
2. Write a `moving_average(xs: &[f64], window: usize) -> Vec<f64>` using a
   `VecDeque` ring buffer and a running sum.
3. Merge three sorted `Vec<i32>` into one sorted vec using a
   `BinaryHeap<Reverse<(i32, usize)>>` where the `usize` is the source index.
4. Given 1,000,000 random `u32`s, print the 10 largest using a bounded
   min-heap, and compare the time to sorting the whole thing.
5. Build a `VecDeque<char>` from a word, then check whether the word is a
   palindrome by popping from both ends until fewer than two elements remain.

---

[← 7.7 HashSet and BTreeSet](07-sets.md) · [Course Index](../README.md) · [7.9 Choosing the Right Collection →](09-choosing.md)
