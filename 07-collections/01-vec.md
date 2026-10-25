[← 6.7 Parsing and Converting Text](../06-strings/07-parsing.md) · [Course Index](../README.md) · [7.2 Vec Operations Reference →](02-vec-operations.md)

# 7.1 — Vec: The Workhorse

## What You'll Learn

- Every way to construct a `Vec`, and when each one is right
- Indexing (panics) versus `.get()` (returns `Option`)
- How growth, capacity, and reallocation actually behave
- Iterating by value, by `&`, and by `&mut`

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `[1, 2, 3]` | `vec![1, 2, 3]` |
| Python `list` | `Vec<T>` |
| Go `[]int` / `make([]int, 0, 10)` | `Vec<i32>` / `Vec::with_capacity(10)` |

A `Vec<T>` is a growable, heap-allocated array of a **single** type. Closest to
a Go slice with its own backing store. Two differences: it is homogeneous (no
`Vec` of mixed types without an enum or `Box<dyn Trait>`), and `v[i]` panics
rather than returning a zero value or `undefined`.

## Constructing

```rust
let a: Vec<i32> = Vec::new();
let b = vec![1, 2, 3];
let c = vec![0u8; 5];
let d = Vec::from([10, 20, 30]);
let e: Vec<char> = "abc".chars().collect();
let f = vec![vec![1, 2], vec![3, 4]];

println!("{:?}", a);
println!("{:?}", b);
println!("{:?}", c);
println!("{:?}", d);
println!("{:?}", e);
println!("{:?}", f);
```

Output:

```
[]
[1, 2, 3]
[0, 0, 0, 0, 0]
[10, 20, 30]
['a', 'b', 'c']
[[1, 2], [3, 4]]
```

`Vec::new()` on an empty vec needs a type annotation — there is nothing to
infer from. `vec![x; n]` repeats `x` `n` times and requires `T: Clone`.

## `with_capacity`

Allocates room for `n` elements but starts at length 0.

```rust
let g: Vec<i32> = Vec::with_capacity(10);
println!("len={} cap={}", g.len(), g.capacity());
```

Output:

```
len=0 cap=10
```

Use it whenever you know roughly how many elements are coming; it skips the
reallocations.

## push and pop

```rust
let mut v = vec![1, 2, 3];
v.push(4);
println!("{:?}", v);
println!("{:?}", v.pop());
println!("{:?}", v);

let mut empty: Vec<i32> = Vec::new();
println!("{:?}", empty.pop());
println!("{}", empty.is_empty());
empty.push(9);
println!("{} {}", empty.len(), empty.is_empty());
```

Output:

```
[1, 2, 3, 4]
Some(4)
[1, 2, 3]
None
true
1 false
```

`pop()` returns `Option<T>` — `None` on an empty vec, never a panic. Both
`push` and `pop` require `mut`.

## Indexing versus `.get()`

```rust
let v = vec![10, 20, 30];
println!("{}", v[0]);
println!("{:?}", v.get(1));
println!("{:?}", v.get(99));
println!("{:?}", v.first());
println!("{:?}", v.last());
println!("{}", v.get(99).copied().unwrap_or(-1));
```

Output:

```
10
Some(20)
None
Some(10)
Some(30)
-1
```

The index must be `usize`. `v[i as usize]` or `v[i]` where `i: i32` is a type
error, not a silent cast.

## Out of bounds panics

```rust
let v = vec![10, 20, 30];
println!("{}", v[5]);
```

This compiles and then panics at runtime (deliberately broken):

```
thread 'main' panicked at src/main.rs:3:21:
index out of bounds: the len is 3 but the index is 5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Idiomatic choice: index when the bound is already proven (a loop over
`0..v.len()`), `.get()` when the index comes from outside.

## len, capacity, is_empty

`len()` is how many elements exist; `capacity()` is how many fit before a
reallocation. They are independent.

```rust
let mut v = Vec::with_capacity(100);
v.extend_from_slice(&[1, 2, 3]);
println!("len={} cap={}", v.len(), v.capacity());
v.shrink_to_fit();
println!("len={} cap={}", v.len(), v.capacity());
```

Output:

```
len=3 cap=100
len=3 cap=3
```

## Growth and reallocation

Capacity roughly doubles when it runs out. Here is every reallocation while
pushing 20 elements:

```rust
let mut g: Vec<i32> = Vec::new();
let mut last = usize::MAX;
for i in 0..20 {
    g.push(i);
    if g.capacity() != last {
        println!("len {:2} -> cap {}", g.len(), g.capacity());
        last = g.capacity();
    }
}
```

Output:

```
len  1 -> cap 4
len  5 -> cap 8
len  9 -> cap 16
len 17 -> cap 32
```

Each step copies every existing element to a new allocation. That is why
`push` is *amortized* O(1), not O(1), and why any reference into the vec is
invalidated by a push — the borrow checker enforces that for you.

## Iteration: by `&`, by `&mut`, by value

```rust
let v = vec![1, 2, 3];
for x in &v { print!("{x} "); }
println!();

let mut m = vec![1, 2, 3];
for x in &mut m { *x *= 10; }
println!("{:?}", m);

for x in m { print!("{x} "); }
println!();
```

Output:

```
1 2 3 
[10, 20, 30]
10 20 30 
```

`&v` yields `&T` and leaves `v` usable. `&mut v` yields `&mut T`, so you
dereference to assign. Bare `v` **consumes** the vec — it is gone after the
loop. `for x in v.iter()` is the same as `&v`; `.into_iter()` is the same as
bare `v`.

With indices:

```rust
let v = vec![1, 2, 3];
for (i, x) in v.iter().enumerate() { println!("{i}: {x}"); }
```

Output:

```
0: 1
1: 2
2: 3
```

## `Vec<String>` versus `Vec<&str>`

```rust
let owned: Vec<String> = vec!["a".to_string(), "b".to_string()];
let borrowed: Vec<&str> = vec!["a", "b"];
println!("{:?} {:?}", owned, borrowed);
println!("{} {}", std::mem::size_of::<String>(), std::mem::size_of::<&str>());

let back: Vec<String> = borrowed.iter().map(|s| s.to_string()).collect();
println!("{:?}", back);
```

Output:

```
["a", "b"] ["a", "b"]
24 16
["a", "b"]
```

`Vec<String>` owns its data and can outlive anything. `Vec<&str>` borrows, so
it drags a lifetime through every type that holds it. Default to
`Vec<String>` for anything stored or returned; use `Vec<&str>` for a transient
view, like the result of `split`.

## Nested Vec

```rust
let grid = vec![vec![1, 2, 3], vec![4, 5, 6]];
println!("{}", grid[1][2]);
for row in &grid {
    for c in row { print!("{c} "); }
    println!();
}

let mut grid2 = vec![vec![0; 3]; 2];
grid2[0][1] = 7;
println!("{:?}", grid2);
```

Output:

```
6
1 2 3 
4 5 6 
[[0, 7, 0], [0, 0, 0]]
```

`vec![vec![0; 3]; 2]` builds two **independent** rows — each inner vec is
cloned, not shared. Rows need not be the same length.

## Collecting into a Vec

```rust
let sq: Vec<i32> = (1..=5).map(|x| x * x).collect();
println!("{:?}", sq);

let evens: Vec<i32> = (1..=10).filter(|x| x % 2 == 0).collect();
println!("{:?}", evens);

let words: Vec<&str> = "a,b,c".split(',').collect();
println!("{:?}", words);

let nums: Result<Vec<i32>, _> =
    vec!["1", "2", "x"].iter().map(|s| s.parse::<i32>()).collect();
println!("{:?}", nums);
```

Output:

```
[1, 4, 9, 16, 25]
[2, 4, 6, 8, 10]
["a", "b", "c"]
Err(ParseIntError { kind: InvalidDigit })
```

`collect()` needs to know the target type, from the annotation or the return
type. The last one is the trick worth remembering: collecting an iterator of
`Result` into `Result<Vec<_>, E>` short-circuits on the first error.

## Memory layout

A `Vec<T>` is exactly three words on the stack — pointer, length, capacity —
pointing at a heap buffer.

```rust
println!("{}", std::mem::size_of::<Vec<i32>>());
println!("{}", std::mem::size_of::<usize>() * 3);
```

Output:

```
24
24
```

The size is the same for any `T`; only the heap buffer scales. Moving a `Vec`
copies those 24 bytes and nothing else — no element is touched.

## Quick Reference

| Operation | Form | Notes |
|---|---|---|
| Empty | `Vec::new()` | needs type annotation |
| Preallocated | `Vec::with_capacity(n)` | `len == 0` |
| Literal | `vec![a, b, c]` | |
| Repeat | `vec![x; n]` | `T: Clone` |
| From array | `Vec::from([..])` | |
| From iterator | `iter.collect()` | annotate the target |
| Append | `v.push(x)` | amortized O(1) |
| Remove last | `v.pop()` | `Option<T>` |
| Index | `v[i]` | panics out of bounds, `i: usize` |
| Safe index | `v.get(i)` | `Option<&T>` |
| Ends | `v.first()`, `v.last()` | `Option<&T>` |
| Size | `v.len()`, `v.is_empty()` | |
| Capacity | `v.capacity()`, `v.shrink_to_fit()` | |
| Borrow each | `for x in &v` | `&T` |
| Mutate each | `for x in &mut v` | `&mut T`, deref to assign |
| Consume | `for x in v` | `v` unusable after |

## Common Mistakes

**Using `v[i]` on an index from user input.** It panics. Use `v.get(i)` and
handle `None`.

**Forgetting `mut`.** `push`, `pop`, and `v[i] = x` all need
`let mut v = ...`.

**Iterating with bare `v` and then using `v`.** That loop moved the vec. Write
`for x in &v`.

**Indexing with an `i32`.** Indices are `usize`; there is no implicit
conversion.

**Reaching for `Vec<&str>` to avoid allocating.** You inherit a lifetime
parameter on every struct and signature that touches it. Start with
`Vec<String>`.

## Exercises

1. Build a `Vec<i32>` of the first 10 squares two ways: a `push` loop and
   `collect()`. Print both.
2. Write `fn nth(v: &Vec<i32>, i: usize) -> i32` that returns `-1` instead of
   panicking when `i` is out of range.
3. Push 1000 elements into a fresh `Vec`, printing length and capacity each
   time capacity changes. Then do the same with `Vec::with_capacity(1000)` and
   compare the number of reallocations.
4. Given `vec!["10", "20", "abc", "30"]`, produce a `Vec<i32>` of only the
   values that parse, then produce a `Result<Vec<i32>, _>` over all of them and
   print both.
5. Build a 3x3 multiplication table as a `Vec<Vec<u32>>`, mutate the center
   cell to 0 in place, and print it row by row.

---

[← 6.7 Parsing and Converting Text](../06-strings/07-parsing.md) · [Course Index](../README.md) · [7.2 Vec Operations Reference →](02-vec-operations.md)
