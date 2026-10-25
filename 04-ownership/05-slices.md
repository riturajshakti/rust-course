[← Course Index](../README.md)

# 4.5 — Slices

## What You'll Learn

- `&[T]` and `&str` as borrowed views: ptr + len, no ownership
- Every range form, including the empty and inclusive ones
- Why `&[T]` beats `&Vec<T>` and `&str` beats `&String` in signatures
- Panics vs `.get()`, mutable slices, `split_at_mut`, `chunks`, `windows`

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `s[1:3]` — a slice header with cap | `&s[1..3]` — ptr + len, no cap, no append |
| Python `xs[1:3]` — copies | borrows; zero allocation |
| JS `arr.slice(1,3)` — copies | borrows |

A slice is two words: a pointer into existing data and a length. It allocates
nothing and owns nothing, so it is subject to every borrowing rule from 4.3
and 4.4.

## Every range form

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];
    println!("{:?}", &a[..]);
    println!("{:?}", &a[1..3]);
    println!("{:?}", &a[2..]);
    println!("{:?}", &a[..2]);
    println!("{:?}", &a[1..=3]);
    println!("{:?}", &a[3..3]);
    println!("{}", a[1..3].len());
    println!("{}", a[3..3].is_empty());
}
```

Output:

```
[10, 20, 30, 40, 50]
[20, 30]
[30, 40, 50]
[10, 20]
[20, 30, 40]
[]
2
true
```

| Form | Elements |
|---|---|
| `[..]` | all |
| `[a..b]` | `a` up to but excluding `b` |
| `[a..]` | `a` to the end |
| `[..b]` | start up to but excluding `b` |
| `[a..=b]` | `a` through `b` inclusive |
| `[..=b]` | start through `b` inclusive |
| `[a..a]` | empty, legal |

## Slicing arrays, Vecs, and Strings

The same syntax, three source types.

```rust
fn main() {
    let arr = [1, 2, 3, 4];
    let v = vec![1, 2, 3, 4];
    let text = String::from("hello world");

    let s1: &[i32] = &arr[1..];
    let s2: &[i32] = &v[1..];
    let s3: &str = &text[6..];

    println!("{:?}", s1);
    println!("{:?}", s2);
    println!("{}", s3);
    println!("{}", std::mem::size_of::<&[i32]>());
    println!("{}", std::mem::size_of::<&str>());
}
```

Output:

```
[2, 3, 4]
[2, 3, 4]
world
16
16
```

Both are 16 bytes: a pointer plus a `usize`. `&str` is exactly `&[u8]` with a
UTF-8 guarantee.

## Out of bounds panics

```rust
let a = [1, 2, 3];
println!("{:?}", &a[1..5]);
```

Output (a panic, not a compile error):

```
thread 'main' panicked at src/main.rs:3:24:
range end index 5 out of range for slice of length 3
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Indexing a `String` at a non-character boundary panics too:

```rust
let s = String::from("héllo");
println!("{}", &s[0..2]);
```

Output:

```
thread 'main' panicked at src/main.rs:3:22:
end byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)
```

String ranges are **byte** offsets. Lesson 6.5 covers UTF-8 safely.

## .get() returns Option instead of panicking

```rust
fn main() {
    let v = vec![1, 2, 3, 4];
    println!("{:?}", v.get(1));
    println!("{:?}", v.get(99));
    println!("{:?}", v.get(1..3));
    println!("{:?}", v.get(1..99));
    println!("{:?}", v.first());
    println!("{:?}", v.last());
}
```

Output:

```
Some(2)
None
Some([2, 3])
None
Some(1)
Some(4)
```

`get` accepts a range as well as an index. Use it on any index that comes from
outside your code.

## Prefer &[T] over &Vec<T>

A `&Vec<i32>` parameter accepts only a `Vec`. A `&[i32]` accepts a `Vec`, an
array, and any slice of either.

```rust
fn sum(xs: &[i32]) -> i32 { xs.iter().sum() }

fn main() {
    let v = vec![1, 2, 3, 4];
    let a = [5, 6, 7];
    println!("{}", sum(&v));
    println!("{}", sum(&a));
    println!("{}", sum(&v[1..3]));
    println!("{}", sum(&[]));
}
```

Output:

```
10
18
5
0
```

`sum(&v)` works by **deref coercion**: `&Vec<i32>` auto-converts to `&[i32]`.
The reverse does not. Does not compile:

```rust
fn sum(xs: &Vec<i32>) -> i32 { xs.iter().sum() }

fn main() {
    let a = [1, 2, 3];
    println!("{}", sum(&a));
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:5:24
  |
5 |     println!("{}", sum(&a));
  |                    --- ^^ expected `&Vec<i32>`, found `&[{integer}; 3]`
  |                    |
  |                    arguments to this function are incorrect
  |
  = note: expected reference `&Vec<i32>`
             found reference `&[{integer}; 3]`
note: function defined here
 --> src/main.rs:1:4
  |
1 | fn sum(xs: &Vec<i32>) -> i32 { xs.iter().sum() }
  |    ^^^ -------------
```

Clippy flags this as `ptr_arg`.

## Prefer &str over &String

Same asymmetry.

```rust
fn shout(s: &str) -> String { s.to_uppercase() }

fn main() {
    let owned = String::from("hello");
    println!("{}", shout(&owned));
    println!("{}", shout("literal"));
    println!("{}", shout(&owned[1..]));
}
```

Output:

```
HELLO
LITERAL
ELLO
```

With `&String` the literal is rejected. Does not compile:

```rust
fn shout(s: &String) -> String { s.to_uppercase() }

fn main() {
    println!("{}", shout("literal"));
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:4:26
  |
4 |     println!("{}", shout("literal"));
  |                    ----- ^^^^^^^^^ expected `&String`, found `&str`
  |                    |
  |                    arguments to this function are incorrect
  |
  = note: expected reference `&String`
             found reference `&'static str`
```

**The rule for every signature you write:** take `&str`, not `&String`; take
`&[T]`, not `&Vec<T>`.

## first_word: the motivating example

Returning a slice of the input avoids both an allocation and an index that
could go stale.

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &b) in bytes.iter().enumerate() {
        if b == b' ' {
            return &s[..i];
        }
    }
    s
}

fn main() {
    println!("{:?}", first_word("hello world"));
    println!("{:?}", first_word("single"));
    println!("{:?}", first_word(""));
    println!("{:?}", first_word(" leading"));
}
```

Output:

```
"hello"
"single"
""
""
```

No lifetime annotation needed: one input reference, so the output borrows from
it (lesson 4.7). Returning `usize` instead would let the caller mutate the
string and keep a meaningless index — the slice version makes that a compile
error.

## Mutable slices

`&mut [T]` lets you write elements but never change the length.

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    let s = &mut v[..3];
    s[0] = 100;
    s.sort();
    println!("{:?}", v);
}
```

Output:

```
[2, 3, 100, 4, 5, 6]
```

The sort applied to the first three elements only. There is no `push` on a
slice — length is fixed by construction.

## split_at and split_at_mut

```rust
let v = vec![2, 3, 100, 4, 5, 6];
let (l, r) = v.split_at(3);
println!("{:?} {:?}", l, r);
```

Output:

```
[2, 3, 100] [4, 5, 6]
```

`split_at_mut` is the interesting one: it hands out **two** `&mut` into the
same `Vec`, which the plain borrow checker would reject. It is sound because
the halves are provably disjoint.

```rust
fn main() {
    let mut v = vec![2, 3, 100, 4, 5, 6];
    let (l, r) = v.split_at_mut(3);
    l[0] = 0;
    r[0] = 0;
    println!("{:?}", v);
}
```

Output:

```
[0, 3, 100, 0, 5, 6]
```

## chunks and windows

```rust
fn main() {
    let data = [1, 2, 3, 4, 5];
    for c in data.chunks(2) { print!("{:?} ", c); }
    println!();
    for w in data.windows(3) { print!("{:?} ", w); }
    println!();
    for c in data.chunks_exact(2) { print!("{:?} ", c); }
    println!();
    println!("{}", data.chunks(2).count());
    println!("{}", data.windows(3).count());
}
```

Output:

```
[1, 2] [3, 4] [5] 
[1, 2, 3] [2, 3, 4] [3, 4, 5] 
[1, 2] [3, 4] 
3
3
```

`chunks` are non-overlapping and the last one may be short; `chunks_exact`
drops the short tail; `windows` overlap and are always full width.
`chunks_mut` and `chunks_exact_mut` are the writing versions.

## In-place slice operations

```rust
fn main() {
    let mut m = [3, 1, 2];
    m.reverse();
    println!("{:?}", m);
    m.swap(0, 2);
    println!("{:?}", m);
    m.fill(9);
    println!("{:?}", m);

    let data = [1, 2, 3, 4, 5];
    println!("{:?}", data.iter().rev().collect::<Vec<_>>());
    println!("{}", data.contains(&3));
}
```

Output:

```
[2, 1, 3]
[3, 1, 2]
[9, 9, 9]
[5, 4, 3, 2, 1]
true
```

Lesson 7.3 covers `sort`, `binary_search`, and `dedup` in full.

## Quick Reference

| Form | Type | Notes |
|---|---|---|
| `&v[..]` | `&[T]` | whole thing |
| `&v[a..b]` | `&[T]` | `b` excluded |
| `&v[a..=b]` | `&[T]` | `b` included |
| `&s[a..b]` | `&str` | **byte** offsets, panics mid-char |
| `&mut v[a..b]` | `&mut [T]` | write elements, length fixed |

| Method | Returns |
|---|---|
| `.len()` / `.is_empty()` | `usize` / `bool` |
| `.get(i)` | `Option<&T>` |
| `.get(a..b)` | `Option<&[T]>` |
| `.first()` / `.last()` | `Option<&T>` |
| `.split_at(n)` | `(&[T], &[T])` |
| `.split_at_mut(n)` | `(&mut [T], &mut [T])` |
| `.chunks(n)` | iterator, last may be short |
| `.chunks_exact(n)` | iterator, short tail dropped |
| `.windows(n)` | overlapping, always width `n` |
| `.iter()` / `.iter_mut()` | `&T` / `&mut T` |
| `.reverse()` / `.swap(i,j)` / `.fill(v)` | `()`, in place |
| `.contains(&x)` | `bool` |

| Parameter type | Accepts |
|---|---|
| `&[T]` | `Vec<T>`, `[T; N]`, any slice |
| `&Vec<T>` | only `Vec<T>` |
| `&str` | `String`, literal, any `&str` slice |
| `&String` | only `String` |

## Common Mistakes

**Using `&Vec<T>` or `&String` in a signature.** Halves the callers you accept
for no benefit. Clippy's `ptr_arg` will tell you.

**Indexing a `String` by character position.** Ranges are bytes; a non-ASCII
boundary panics. Use `.chars()` (lesson 6.5).

**Expecting `.push()` on a slice.** A slice has no capacity. Take `&mut Vec<T>`
if you need to grow.

**Using `[i]` on untrusted input.** Out of bounds panics. Use `.get(i)` and
match the `Option`.

**Returning an index where a slice would do.** The index can go stale; the
slice keeps the borrow alive and the compiler checks it.

## Exercises

1. Print every range form over `[1,2,3,4,5]` including an empty one. Then
   trigger the out-of-bounds panic and read its message.
2. Write `fn last_word(s: &str) -> &str` as the mirror of `first_word`. Test
   with `""`, `"one"`, `"trailing "`.
3. Write `fn mean(xs: &[f64]) -> Option<f64>` returning `None` for an empty
   slice. Call it with a `Vec`, an array, and a sub-slice.
4. Use `split_at_mut` to reverse the first half of a `Vec` and sort the second,
   in one pass without cloning.
5. Given `&[u8]`, use `windows(2)` to count how many adjacent pairs are
   increasing. Then do it with `chunks(2)` and explain the different answer.

---

[← Course Index](../README.md)
