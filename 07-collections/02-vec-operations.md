[← Course Index](../README.md)

# 7.2 — Vec Operations Reference

## What You'll Learn

- Every `Vec` and slice method worth knowing, grouped by what it does
- What each one returns, and whether it panics
- The O(n) versus O(1) traps: `remove` vs `swap_remove`, `insert` vs `push`
- Capacity control, slicing, and conversion

## Coming From Another Language

This lesson is the lookup table you'd otherwise keep open in a browser tab.
Two things to internalize before the tables:

- Many "Vec methods" are actually **slice** methods. `Vec<T>` derefs to
  `[T]`, so `v.sort()`, `v.chunks()`, `v.first()` all live on `[T]` and work
  on arrays and slices too.
- Methods that could fail come in pairs: a panicking one (`v[i]`, `remove`)
  and an `Option`-returning one (`get`, `pop`). Pick deliberately.

## Group 1: Create

| Method | Signature | Returns | Note |
|---|---|---|---|
| `Vec::new` | `() -> Vec<T>` | empty vec | capacity 0, no allocation |
| `Vec::with_capacity` | `(usize) -> Vec<T>` | empty vec | reserves room, `len == 0` |
| `vec![]` | macro | `Vec<T>` | `vec![a,b]`, `vec![x; n]`, `vec![]` |
| `Vec::from` | `([T; N]) -> Vec<T>` | vec | also from `&[T]`, `&str` chars etc. |
| `Vec::from_iter` | `(I: IntoIterator) -> Vec<T>` | vec | same as `.collect()` |
| `<[T]>::to_vec` | `(&self) -> Vec<T>` | owned copy | `T: Clone` |
| `<[T]>::repeat` | `(&self, usize) -> Vec<T>` | tiled copy | `T: Copy` |
| `.collect()` | `(I) -> Vec<T>` | vec | needs a target annotation |
| `Vec::default` | `() -> Vec<T>` | empty vec | same as `new` |
| `.clone()` | `(&self) -> Vec<T>` | deep copy | clones every element |

```rust
let a: Vec<i32> = Vec::new();
let b: Vec<i32> = Vec::with_capacity(4);
println!("{:?} cap={}", a, a.capacity());
println!("{:?} cap={}", b, b.capacity());

println!("{:?}", vec![1, 2, 3]);
println!("{:?}", vec!["x"; 3]);
println!("{:?}", Vec::from([1, 2, 3]));
println!("{:?}", Vec::<i32>::from_iter(1..4));
println!("{:?}", [1, 2].repeat(3));
println!("{:?}", (1..4).collect::<Vec<i32>>());
println!("{:?}", Vec::<i32>::default());
println!("{:?}", [1, 2, 3].as_slice().to_vec());
```

Output:

```
[] cap=0
[] cap=4
[1, 2, 3]
["x", "x", "x"]
[1, 2, 3]
[1, 2, 3]
[1, 2, 1, 2, 1, 2]
[1, 2, 3]
[]
[1, 2, 3]
```

`Vec::new()` does not allocate at all until the first push. That makes an
empty `Vec` free to hold in a struct.

## Group 2: Read

| Method | Signature | Returns | Note |
|---|---|---|---|
| `get` | `(usize) -> Option<&T>` | `None` out of bounds | also takes a range |
| `get_mut` | `(usize) -> Option<&mut T>` | mutable borrow | |
| `first` / `last` | `() -> Option<&T>` | `None` if empty | |
| `first_mut` / `last_mut` | `() -> Option<&mut T>` | | |
| `v[i]` (`Index`) | `(usize) -> &T` | **panics** out of bounds | `i` must be `usize` |
| `v[a..b]` | `(Range) -> &[T]` | **panics** if out of range | |
| `len` | `() -> usize` | element count | |
| `is_empty` | `() -> bool` | `len() == 0` | |
| `contains` | `(&T) -> bool` | linear scan | `T: PartialEq` |
| `iter` | `() -> Iter<'_, T>` | yields `&T` | |
| `iter_mut` | `() -> IterMut<'_, T>` | yields `&mut T` | |
| `as_ptr` | `() -> *const T` | raw pointer | invalidated by any realloc |

```rust
let mut v = vec![10, 20, 30, 40];
println!("{:?}", v.get(1));
println!("{:?}", v.get(9));
println!("{:?}", v.get(1..3));
println!("{:?}", v.first());
println!("{:?}", v.last());
println!("{}", v[2]);
println!("{} {} {}", v.len(), v.is_empty(), v.contains(&30));

if let Some(x) = v.get_mut(0) { *x = 11; }
if let Some(x) = v.first_mut() { *x += 1; }
if let Some(x) = v.last_mut() { *x = 99; }
println!("{:?}", v);
println!("{:?}", v.iter().rev().collect::<Vec<_>>());
println!("{:?} {:?}", v.iter().min(), v.iter().max());
println!("{}", v.iter().sum::<i32>());

let e: Vec<i32> = vec![];
println!("{:?} {:?}", e.first(), e.last());
```

Output:

```
Some(20)
None
Some([20, 30])
Some(10)
Some(40)
30
4 false true
[12, 20, 30, 99]
[99, 30, 20, 12]
Some(12) Some(99)
161
None None
```

Note `get` with a range returns `Option<&[T]>`, which is the non-panicking
form of `v[a..b]`.

## Group 3: Add

| Method | Signature | Returns | Note |
|---|---|---|---|
| `push` | `(T)` | `()` | amortized O(1) at the end |
| `insert` | `(usize, T)` | `()` | **O(n)**, shifts right; panics if `i > len` |
| `extend` | `(I: IntoIterator<Item=T>)` | `()` | takes items by value |
| `extend_from_slice` | `(&[T])` | `()` | `T: Clone` |
| `extend_from_within` | `(R: RangeBounds)` | `()` | duplicates part of itself |
| `append` | `(&mut Vec<T>)` | `()` | **empties** the other vec |
| `resize` | `(usize, T)` | `()` | grows with clones of `T`, or truncates |
| `resize_with` | `(usize, F)` | `()` | grows by calling `F` per new slot |
| `splice` | `(R, I) -> Splice` | removed items | replaces a range with an iterator |

```rust
let mut v = vec![1, 2, 3];
v.push(4);
println!("{:?}", v);
v.insert(0, 0);
println!("{:?}", v);
v.insert(v.len(), 5);
println!("{:?}", v);
v.extend([6, 7]);
println!("{:?}", v);
v.extend_from_slice(&[8, 9]);
println!("{:?}", v);

let mut other = vec![10, 11];
v.append(&mut other);
println!("{:?} other={:?}", v, other);
```

Output:

```
[1, 2, 3, 4]
[0, 1, 2, 3, 4]
[0, 1, 2, 3, 4, 5]
[0, 1, 2, 3, 4, 5, 6, 7]
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11] other=[]
```

`insert(v.len(), x)` is legal and equals `push(x)`; `insert(v.len()+1, x)`
panics.

Resizing and splicing:

```rust
let mut v = vec![0, 1, 2];
v.resize(6, 9);
println!("{:?}", v);
v.resize(3, 9);
println!("{:?}", v);
let mut n = 100;
v.resize_with(6, || { n += 1; n });
println!("{:?}", v);

let mut w = vec![1, 2, 3];
w.extend_from_within(..2);
println!("{:?}", w);

let mut x = vec![1, 2, 3];
x.splice(1..2, [7, 8, 9]);
println!("{:?}", x);
```

Output:

```
[0, 1, 2, 9, 9, 9]
[0, 1, 2]
[0, 1, 2, 101, 102, 103]
[1, 2, 3, 1, 2]
[1, 7, 8, 9, 3]
```

`resize` to a smaller length is a truncate — it drops the tail.

## Group 4: Remove

| Method | Signature | Returns | Note |
|---|---|---|---|
| `pop` | `() -> Option<T>` | last element | O(1), `None` if empty |
| `pop_if` | `(F) -> Option<T>` | last element if `F` says so | O(1) |
| `remove` | `(usize) -> T` | the element | **O(n)**, shifts left, preserves order, **panics** if out of bounds |
| `swap_remove` | `(usize) -> T` | the element | **O(1)**, moves the last element into the hole, **destroys order**, panics if out of bounds |
| `truncate` | `(usize)` | `()` | drops everything past `n`; no-op if `n >= len` |
| `clear` | `()` | `()` | `len = 0`, **capacity kept** |
| `drain` | `(R) -> Drain<T>` | iterator of removed items | removes the range even if you drop the iterator |
| `retain` | `(F: Fn(&T) -> bool)` | `()` | keeps where `F` is true, order preserved, O(n) |
| `retain_mut` | `(F: FnMut(&mut T) -> bool)` | `()` | can mutate while deciding |
| `extract_if` | `(R, F) -> ExtractIf` | iterator of removed items | the inverse of `retain`, lazily |
| `dedup` | `()` | `()` | removes **consecutive** duplicates only |

```rust
let mut v = vec![1, 2, 3, 4, 5];
println!("{:?}", v.pop());
println!("{}", v.remove(0));
println!("{:?}", v);
println!("{}", v.swap_remove(0));
println!("{:?}", v);

let mut v = vec![1, 2, 3, 4, 5, 6];
v.truncate(4);
println!("{:?}", v);
let drained: Vec<i32> = v.drain(1..3).collect();
println!("drained={:?} left={:?}", drained, v);

let mut v = vec![1, 2, 3, 4, 5, 6];
v.retain(|&x| x % 2 == 0);
println!("{:?}", v);

let mut v = vec![1, 2, 3, 4];
v.retain_mut(|x| { *x *= 10; *x < 35 });
println!("{:?}", v);

let mut v = vec![1, 1, 2, 2, 2, 3, 1];
v.dedup();
println!("{:?}", v);

let mut v = vec![1, 2, 3];
v.clear();
println!("{:?} len={} cap={}", v, v.len(), v.capacity());

let mut v = vec![1, 2, 3, 4, 5];
println!("{:?}", v.pop_if(|x| *x == 5));
println!("{:?}", v);

let mut v = vec![1, 2, 3, 4, 5, 6];
let odd: Vec<i32> = v.extract_if(.., |x| *x % 2 == 1).collect();
println!("odd={:?} v={:?}", odd, v);
```

Output:

```
Some(5)
1
[2, 3, 4]
2
[4, 3]
[1, 2, 3, 4]
drained=[2, 3] left=[1, 4]
[2, 4, 6]
[10, 20, 30]
[1, 2, 3, 1]
[] len=0 cap=3
Some(5)
[1, 2, 3, 4]
odd=[1, 3, 5] v=[2, 4, 6]
```

Two things to note. `dedup` left the trailing `1` because it was not
*adjacent* to the earlier ones — sort first for unique values (lesson 7.3).
And `clear` keeps the capacity, which is what you want when reusing a buffer
in a loop.

### `remove` versus `swap_remove`

Both take an index and return the element. `remove` shifts every later element
down by one; `swap_remove` just moves the last element into the gap.

```rust
let mut v = vec!['a', 'b', 'c', 'd'];
println!("{}", v.remove(1));
println!("{:?}", v);

let mut v = vec!['a', 'b', 'c', 'd'];
println!("{}", v.swap_remove(1));
println!("{:?}", v);
```

Output:

```
b
['a', 'c', 'd']
b
['a', 'd', 'c']
```

The cost difference is not academic. Removing from the front of a 200,000
element vec, one at a time, in release mode:

```rust
use std::time::Instant;

let n = 200_000;
let mut a: Vec<usize> = (0..n).collect();
let t = Instant::now();
while !a.is_empty() { a.remove(0); }
let d1 = t.elapsed();

let mut b: Vec<usize> = (0..n).collect();
let t = Instant::now();
while !b.is_empty() { b.swap_remove(0); }
let d2 = t.elapsed();

println!("remove(0) x{n}:      {:?}", d1);
println!("swap_remove(0) x{n}: {:?}", d2);
```

Output:

```
remove(0) x200000:      2.249181833s
swap_remove(0) x200000: 44.416µs
```

O(n²) versus O(n): a 50,000x difference. If order does not matter, use
`swap_remove`. If you are removing many elements by a predicate, use `retain`
(one pass) instead of `remove` in a loop. If you need ordered removal from the
front, use a `VecDeque` (lesson 7.8).

## Group 5: Transform and rearrange

| Method | Signature | Returns | Note |
|---|---|---|---|
| `iter().map(f).collect()` | | new `Vec<U>` | the general transform |
| `iter().filter(p).collect()` | | `Vec<&T>` | add `.cloned()` for `Vec<T>` |
| `iter().filter_map(f).collect()` | | `Vec<U>` | map + filter in one pass |
| `split_off` | `(usize) -> Vec<T>` | the tail | `self` keeps the head |
| `concat` | `(&self) -> Vec<T>` | flattened | on `Vec<Vec<T>>` / `Vec<&str>` |
| `join` | `(&self, sep) -> Vec<T>`/`String` | flattened with separator | |
| `chunks` | `(usize) -> Chunks<T>` | `&[T]` slices | last chunk may be short |
| `chunks_exact` | `(usize) -> ChunksExact<T>` | equal slices | remainder via `.remainder()` |
| `rchunks` | `(usize) -> RChunks<T>` | chunks from the end | |
| `windows` | `(usize) -> Windows<T>` | overlapping slices | empty if `n > len` |
| `split_at` | `(usize) -> (&[T], &[T])` | two halves | panics if `i > len` |
| `split` | `(P) -> Split<T>` | slices between matches | |
| `rotate_left` / `rotate_right` | `(usize)` | `()` | in place, O(n) |
| `swap` | `(usize, usize)` | `()` | panics out of bounds |
| `reverse` | `()` | `()` | in place |
| `fill` | `(T)` | `()` | `T: Clone` |
| `fill_with` | `(F)` | `()` | one call per slot |
| `copy_from_slice` | `(&[T])` | `()` | **panics** on length mismatch; `T: Copy` |

```rust
let v = vec![1, 2, 3, 4];
println!("{:?}", v.iter().map(|x| x * 2).collect::<Vec<i32>>());
println!("{:?}", v.iter().filter(|x| **x > 2).collect::<Vec<&i32>>());
println!("{:?}", v.iter()
    .filter_map(|x| if x % 2 == 0 { Some(x * 10) } else { None })
    .collect::<Vec<i32>>());

let mut v = vec![1, 2, 3, 4, 5];
let tail = v.split_off(2);
println!("head={:?} tail={:?}", v, tail);

let nested = vec![vec![1, 2], vec![3], vec![4, 5]];
println!("{:?}", nested.concat());
println!("{}", vec!["a", "b", "c"].join("-"));
println!("{:?}", vec![vec![1, 2], vec![3, 4]].join(&0));
```

Output:

```
[2, 4, 6, 8]
[3, 4]
[20, 40]
head=[1, 2] tail=[3, 4, 5]
[1, 2, 3, 4, 5]
a-b-c
[1, 2, 0, 3, 4]
```

`filter` yields `&T`, so the collected type is `Vec<&i32>`. Insert `.copied()`
or `.cloned()` before `collect` to get `Vec<i32>`.

Slicing into pieces:

```rust
let v = vec![1, 2, 3, 4, 5];
println!("{:?}", v.chunks(2).collect::<Vec<_>>());
println!("{:?}", v.chunks_exact(2).collect::<Vec<_>>());
println!("{:?}", v.rchunks(2).collect::<Vec<_>>());
println!("{:?}", v.windows(3).collect::<Vec<_>>());
println!("{:?}", v.split_at(2));
println!("{:?}", v.split(|x| *x == 3).collect::<Vec<_>>());
```

Output:

```
[[1, 2], [3, 4], [5]]
[[1, 2], [3, 4]]
[[4, 5], [2, 3], [1]]
[[1, 2, 3], [2, 3, 4], [3, 4, 5]]
([1, 2], [3, 4, 5])
[[1, 2], [4, 5]]
```

`chunks` is non-overlapping and includes a short tail; `chunks_exact` drops
the tail; `windows` overlaps by `n-1`.

In-place rearranging:

```rust
let mut v = vec![1, 2, 3, 4, 5];
v.rotate_left(2);
println!("{:?}", v);
v.rotate_right(2);
println!("{:?}", v);
v.swap(0, 4);
println!("{:?}", v);
v.reverse();
println!("{:?}", v);

let mut f = vec![0; 5];
f.fill(7);
println!("{:?}", f);
let mut n = 0;
let mut g = vec![0; 5];
g.fill_with(|| { n += 1; n });
println!("{:?}", g);
let mut h = vec![1, 2, 3, 4];
h.copy_from_slice(&[9, 9, 9, 9]);
println!("{:?}", h);
```

Output:

```
[3, 4, 5, 1, 2]
[1, 2, 3, 4, 5]
[5, 2, 3, 4, 1]
[1, 4, 3, 2, 5]
[7, 7, 7, 7, 7]
[1, 2, 3, 4, 5]
[9, 9, 9, 9]
```

## Group 6: Convert

| Method | Signature | Returns | Note |
|---|---|---|---|
| `as_slice` | `(&self) -> &[T]` | borrowed view | usually implicit via deref |
| `as_mut_slice` | `(&mut self) -> &mut [T]` | mutable view | |
| `into_boxed_slice` | `(self) -> Box<[T]>` | 2 words, no capacity | drops spare capacity |
| `Box<[T]>::into_vec` | `(self) -> Vec<T>` | back to a vec | O(1), no copy |
| `to_vec` | `(&[T]) -> Vec<T>` | owned copy | on a slice |
| `into_iter` | `(self) -> IntoIter<T>` | owned items | consumes the vec |
| `try_into` | `(self) -> Result<[T; N], Vec<T>>` | fixed array | `Err` hands the vec back |
| `leak` | `(self) -> &'static mut [T]` | leaked slice | intentionally never freed |

```rust
let v = vec![1, 2, 3];
let s: &[i32] = v.as_slice();
println!("{:?} {}", s, s.len());

let mut m = vec![1, 2, 3];
m.as_mut_slice()[0] = 9;
println!("{:?}", m);

let boxed: Box<[i32]> = vec![1, 2, 3].into_boxed_slice();
println!("{:?} {}", boxed, std::mem::size_of_val(&boxed));
println!("{:?}", boxed.into_vec());

let arr: [i32; 3] = vec![1, 2, 3].try_into().unwrap();
let bad: Result<[i32; 2], _> = vec![1, 2, 3].try_into();
println!("{:?}", arr);
println!("{:?}", bad);
```

Output:

```
[1, 2, 3] 3
[9, 2, 3]
[1, 2, 3] 16
[1, 2, 3]
[1, 2, 3]
Err([1, 2, 3])
```

`Box<[T]>` is two words instead of three — worth it when you build a list once,
never resize it, and store millions of them.

## Group 7: Capacity

| Method | Signature | Returns | Note |
|---|---|---|---|
| `capacity` | `() -> usize` | slots allocated | always `>= len()` |
| `reserve` | `(usize)` | `()` | room for `n` **more**; may over-allocate |
| `reserve_exact` | `(usize)` | `()` | room for exactly `n` more |
| `try_reserve` | `(usize) -> Result<(), TryReserveError>` | fallible reserve | for huge allocations |
| `shrink_to_fit` | `()` | `()` | capacity down to `len` |
| `shrink_to` | `(usize)` | `()` | capacity down to at least `n` |

```rust
let mut v: Vec<i32> = Vec::new();
println!("cap={}", v.capacity());
v.reserve(10);
println!("cap>=10: {}", v.capacity() >= 10);
v.push(1);
v.shrink_to_fit();
println!("len={} cap={}", v.len(), v.capacity());

let mut w: Vec<i32> = Vec::with_capacity(100);
w.extend([1, 2, 3]);
w.shrink_to(50);
println!("cap={}", w.capacity());
let mut z: Vec<i32> = Vec::new();
z.reserve_exact(7);
println!("cap={}", z.capacity());
println!("{}", z.try_reserve(5).is_ok());
```

Output:

```
cap=0
cap>=10: true
len=1 cap=1
cap=50
cap=7
true
```

`reserve(n)` means "n more than I currently have", not "total n". The exact
resulting capacity is not guaranteed — only that it is at least enough.

## Quick Reference

| Need | Method | Cost |
|---|---|---|
| Add at end | `push` | O(1) amortized |
| Add at index | `insert` | O(n) |
| Add many | `extend`, `extend_from_slice`, `append` | O(k) |
| Remove at end | `pop` | O(1) |
| Remove at index, keep order | `remove` | O(n) |
| Remove at index, order free | `swap_remove` | O(1) |
| Remove a range | `drain(a..b)` | O(n) |
| Remove by predicate | `retain`, `extract_if` | O(n) one pass |
| Remove all | `clear` (keeps cap), `truncate(0)` | O(n) |
| Safe read | `get`, `first`, `last` | O(1) |
| Panicking read | `v[i]` | O(1) |
| Membership | `contains` | O(n) |
| Split in two | `split_at` (borrow), `split_off` (own) | O(1) / O(n) |
| Flatten | `concat`, `join` | O(n) |
| Fixed-size views | `chunks`, `chunks_exact`, `windows` | lazy |
| Reorder | `rotate_left/right`, `swap`, `reverse` | O(n) |
| Overwrite | `fill`, `fill_with`, `copy_from_slice` | O(n) |
| Tighten memory | `shrink_to_fit`, `into_boxed_slice` | O(n) |

## Common Mistakes

**`remove(0)` in a loop.** That is O(n²). Use `retain`, `drain`, or a
`VecDeque`.

**Expecting `clear()` to free memory.** It only sets `len = 0`. Follow it with
`shrink_to_fit()`, or drop the vec.

**Expecting `dedup()` to remove all duplicates.** It only collapses adjacent
ones. Sort first.

**`reserve(n)` as a total capacity.** It is `n` *additional* slots on top of
`len()`.

**Collecting `filter` straight into `Vec<T>`.** `filter` yields `&T`; add
`.copied()` or `.cloned()`.

**`append(&mut other)` and then using `other`.** It is left empty, not copied.

## Exercises

1. Write `fn remove_all(v: &mut Vec<i32>, target: i32)` two ways: with a
   `remove` loop and with `retain`. Time both on 100,000 elements.
2. Given `vec![1,2,3,4,5,6,7,8,9]`, use `drain` to pull out the middle three
   into a new vec, and print both before and after.
3. Use `chunks(3)` to print a 9-element vec as a 3x3 grid, then use
   `windows(2)` to print each adjacent pair's difference.
4. Build a `Vec<String>` from `vec!["a","b","c"]`, append `vec!["d","e"]` to
   it with `append`, then split it at index 2 with `split_off` and print both
   halves.
5. Start with `Vec::with_capacity(1000)`, push 10 elements, print the
   capacity, call `shrink_to_fit()`, print again, and then convert it to a
   `Box<[i32]>` and print `size_of_val` of both.

---

[← Course Index](../README.md)
