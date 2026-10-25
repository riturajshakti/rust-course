[← Course Index](../README.md)

# 4.4 — Mutable Borrows and the Borrow Checker

## What You'll Learn

- `&mut T`: one writer, no readers at the same time
- `E0499` and `E0502` verbatim, and the reordering that fixes them
- Non-lexical lifetimes: a borrow ends at its **last use**, not the brace
- Split borrows on distinct fields, reborrowing, and `&mut` in loops

## Coming From Another Language

| Other | Rust |
|---|---|
| Go: any number of `*T` can write | exactly one `&mut T` at a time |
| JS: aliasing a mutable array is normal | aliasing + mutation is a compile error |
| C: `T* restrict` as a hint | `&mut T` *is* `restrict`, enforced |

The full borrowing rule: at any point, a value has **either** any number of
`&T`, **or** exactly one `&mut T`, never both. This is what makes iterator
invalidation and data races impossible rather than merely discouraged.

## &mut T and explicit assignment

```rust
fn main() {
    let mut s = String::from("hello");
    let r = &mut s;
    r.push_str(", world");
    println!("{r}");
    println!("{s}");

    let mut n = 1;
    let rn = &mut n;
    *rn += 10;
    println!("{n}");
}
```

Output:

```
hello, world
hello, world
11
```

The owner needs `let mut`; the borrow needs `&mut`. Method calls auto-deref
(`r.push_str`), but assigning to the pointee needs `*rn`.

## Mutating a collection through &mut

```rust
let mut v = vec![1, 2, 3];
let rv = &mut v;
rv.push(4);
rv[0] = 99;
println!("{:?}", v);
```

Output:

```
[99, 2, 3, 4]
```

## error[E0499]: two mutable borrows

Does not compile (intentionally):

```rust
let mut s = String::from("hello");
let a = &mut s;
let b = &mut s;
println!("{a} {b}");
```

```
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:4:13
  |
3 |     let a = &mut s;
  |             ------ first mutable borrow occurs here
4 |     let b = &mut s;
  |             ^^^^^^ second mutable borrow occurs here
5 |     println!("{a} {b}");
  |                - first borrow later used here
```

The *last* line is the one that makes it an error: `a` is used after `b` is
created, so the two borrows genuinely overlap.

## error[E0502]: mutable plus immutable

Does not compile:

```rust
let mut s = String::from("hello");
let r = &s;
let w = &mut s;
println!("{r} {w}");
```

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:13
  |
3 |     let r = &s;
  |             -- immutable borrow occurs here
4 |     let w = &mut s;
  |             ^^^^^^ mutable borrow occurs here
5 |     println!("{r} {w}");
  |                - immutable borrow later used here
```

## The classic: holding a reference across a push

This is the error that explains the rule. Does not compile:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4);
println!("{first}");
```

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(4);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{first}");
  |                ----- immutable borrow later used here
```

`push` may exceed capacity, reallocate, and copy the elements to a new address
(lesson 4.1). `first` would then point at freed memory. In Go or JS this
compiles and you get a stale value or a crash; here it is a build failure.

## Non-lexical lifetimes: reordering fixes it

A borrow lasts from its creation to its **last use**, not to the end of the
block. Move the `println!` above the `push` and the same code compiles.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];
    println!("{first}");
    v.push(4);
    println!("{:?}", v);
}
```

Output:

```
1
[1, 2, 3, 4]
```

Nothing else changed. `first` is dead by line 4, so the mutable borrow on
line 5 is unopposed. The same trick ends a mutable borrow:

```rust
let mut s = String::from("hi");
let a = &mut s;
a.push('!');
println!("{a}");
let b = &mut s;
b.push('?');
println!("{b}");
```

Output:

```
hi!
hi!?
```

Two `&mut` to the same value in one block, legal because their live ranges do
not overlap. And immutable-then-mutable:

```rust
let mut t = String::from("x");
let r = &t;
println!("{r}");
let w = &mut t;
w.push('y');
println!("{t}");
```

Output:

```
x
xy
```

**The practical rule:** when you hit `E0499` or `E0502`, look at the
"later used here" annotation first. Often you can just move that line up.

## Split borrows: distinct fields are independent

The checker tracks fields separately, so two `&mut` into different fields
coexist.

```rust
#[derive(Debug)]
struct Point { x: i32, y: i32 }

fn main() {
    let mut p = Point { x: 1, y: 2 };
    let rx = &mut p.x;
    let ry = &mut p.y;
    *rx += 10;
    *ry += 20;
    println!("{:?}", p);
}
```

Output:

```
Point { x: 11, y: 22 }
```

Borrowing the **whole** struct while a field is borrowed does not work. Does
not compile:

```rust
struct Point { x: i32, y: i32 }

fn sum(p: &Point) -> i32 { p.x + p.y }

fn main() {
    let mut p = Point { x: 1, y: 2 };
    let rx = &mut p.x;
    let total = sum(&p);
    *rx += 1;
    println!("{total}");
}
```

```
error[E0502]: cannot borrow `p` as immutable because it is also borrowed as mutable
 --> src/main.rs:8:21
  |
7 |     let rx = &mut p.x;
  |              -------- mutable borrow occurs here
8 |     let total = sum(&p);
  |                     ^^ immutable borrow occurs here
9 |     *rx += 1;
  |     -------- mutable borrow later used here
```

This is why a method taking `&mut self` cannot be called while any field is
separately borrowed — a frequent surprise. The usual fixes: finish with `rx`
first, or have the method take the two fields as parameters.

## Passing &mut to a function

```rust
fn bump(n: &mut i32) { *n += 1; }
fn extend(v: &mut Vec<i32>, n: i32) { v.push(n); }

fn main() {
    let mut n = 5;
    bump(&mut n);
    bump(&mut n);
    println!("{n}");

    let mut v = vec![1];
    extend(&mut v, 2);
    extend(&mut v, 3);
    println!("{:?}", v);
}
```

Output:

```
7
[1, 2, 3]
```

Each call's borrow ends when the call returns, so consecutive calls are fine.

## Reborrowing through a &mut

Assigning a `&mut T` to another `&mut T` does **not** move it — the compiler
inserts a reborrow (`&mut *outer`). The original is usable again once the
reborrow ends.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let outer = &mut v;
    {
        let inner: &mut Vec<i32> = outer;
        inner.push(4);
    }
    outer.push(5);
    println!("{:?}", v);
}
```

Output:

```
[1, 2, 3, 4, 5]
```

This is why `&mut T` is not `Copy` but still feels like it: passing one to a
function reborrows rather than consumes.

## &mut in a loop

`for x in &mut collection` yields `&mut T` per element.

```rust
let mut nums = vec![1, 2, 3];
for n in &mut nums {
    *n *= 10;
}
println!("{:?}", nums);
```

Output:

```
[10, 20, 30]
```

`iter_mut()` is the same thing written out. For index-based mutation:

```rust
for i in 0..nums.len() {
    nums[i] += 1;
}
println!("{:?}", nums);
```

Output:

```
[11, 21, 31]
```

## Mutating the collection you are iterating

Does not compile — and the compiler names both fixes:

```rust
let mut v = vec![1, 2, 3];
for n in &v {
    if *n == 2 {
        v.push(99);
    }
}
```

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:5:13
  |
3 |       for n in &v {
  |       -        --
  |       |        |
  |       |        immutable borrow occurs here
  |  _____|        immutable borrow later used here
  | |
4 | |         if *n == 2 {
5 | |             v.push(99);
  | |             ^^^^^^^^^^ mutable borrow occurs here
6 | |         }
7 | |     }
  | |_____- this for loop borrows `v` immutably, preventing mutation within its body
  |
  = help: consider using an index-based loop instead, or collecting modifications into a separate collection
```

The collect-then-apply version:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let extra: Vec<i32> = v.iter().filter(|n| **n == 2).map(|n| n + 97).collect();
    v.extend(extra);
    println!("{:?}", v);
}
```

Output:

```
[1, 2, 3, 99]
```

## Quick Reference

| Form | Meaning |
|---|---|
| `&mut x` | borrow `x` exclusively |
| `&mut T` | type: exclusive reference |
| `*r = v` | write through `r` |
| `*r += 1` | read-modify-write through `r` |
| `let r2 = r;` (`r: &mut T`) | reborrow, not a move |
| `&mut s.field` | borrow one field exclusively |
| `for x in &mut v` | yields `&mut T` |
| `v.iter_mut()` | same, explicit |
| `fn f(x: &mut T)` | caller writes `f(&mut x)` |

| Rule | Allowed |
|---|---|
| N × `&T` | yes |
| 1 × `&mut T` | yes |
| `&T` + `&mut T` overlapping | no — `E0502` |
| 2 × `&mut T` overlapping | no — `E0499` |
| Non-overlapping live ranges | yes, any mix |
| `&mut a.x` + `&mut a.y` | yes |
| `&mut a.x` + `&a` | no — `E0502` |

## Common Mistakes

**Forgetting `let mut` on the owner.** `&mut s` on a non-`mut` binding is
`E0596: cannot borrow ... as mutable, as it is not declared as mutable`, with a
`help` suggesting the `mut`.

**Reading the error top-down.** Start at "later used here" — that line is
usually the one to move or delete.

**Assuming the borrow lasts to the closing brace.** It ends at last use.
Reordering two lines fixes a large share of borrow errors.

**Calling `&mut self` methods while holding a field borrow.** The method borrows
the whole struct. Narrow the borrow or pass fields explicitly.

**Pushing to a `Vec` you are iterating.** Collect the additions, then `extend`
after the loop.

## Exercises

1. Write `fn double_all(v: &mut Vec<i32>)` and call it twice on the same vector.
2. Produce `E0499`, then fix it by reordering only — no new scopes, no clones.
3. Produce `E0502` with `&v[0]` and `v.push`, then fix it three ways:
   reordering, an inner block, and cloning the element.
4. Given `struct Stats { hits: u32, misses: u32 }`, hold `&mut` to both fields
   at once and increment each. Then add a `fn ratio(&self)` call between them
   and read the error.
5. Remove every even number from a `Vec<i32>` while iterating. Find two
   approaches that compile (`retain` is one).

---

[← Course Index](../README.md)
