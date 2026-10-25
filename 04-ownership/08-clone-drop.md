[← Course Index](../README.md)

# 4.8 — Clone, Copy, and Drop

## What You'll Learn

- `Clone` vs `Copy`: explicit deep copy vs implicit bitwise duplicate
- What `derive(Clone, Copy)` requires, and the exact error when it can't
- `Drop`: drop order for variables and for struct fields, printed
- `std::mem::drop`, why `Drop` and `Copy` are mutually exclusive, and RAII

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `structuredClone(x)` | `x.clone()` — explicit, always |
| Python `copy.deepcopy` | `.clone()` is deep for `String`/`Vec`/nested |
| Go: assignment copies, shallow | assignment moves; `.clone()` is deep |
| C++ destructor | `impl Drop`, but no copy constructor |

Three traits that between them decide what `=` does and what happens at the
closing brace. `Copy` is opt-in and free; `Clone` is opt-in and explicit;
`Drop` is the cleanup hook, and it is incompatible with `Copy`.

## Copy: assignment duplicates, both stay valid

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Point { x: i32, y: i32 }

fn main() {
    let p = Point { x: 1, y: 2 };
    let q = p;
    println!("{:?} {:?} {}", p, q, p == q);

    let mut r = p;
    r.x = 99;
    println!("{:?} {:?}", p, r);
}
```

Output:

```
Point { x: 1, y: 2 } Point { x: 1, y: 2 } true
Point { x: 1, y: 2 } Point { x: 99, y: 2 }
```

They are genuinely independent values — mutating `r` left `p` alone. `Copy`
requires `Clone` as a supertrait, so you always derive both.

## Clone: explicit, and deep

```rust
#[derive(Debug, Clone)]
struct Config { name: String, tags: Vec<String> }

fn main() {
    let c = Config { name: String::from("db"), tags: vec![String::from("a")] };
    let mut d = c.clone();
    d.tags.push(String::from("b"));
    println!("{:?}", c);
    println!("{:?}", d);
}
```

Output:

```
Config { name: "db", tags: ["a"] }
Config { name: "db", tags: ["a", "b"] }
```

The derived `Clone` clones every field recursively, so `d.tags` is a separate
allocation. No aliasing surprises.

## Deep vs shallow, demonstrated by pointer

`clone()` on a `String` allocates a new buffer — the pointers differ, the
contents compare equal.

```rust
fn main() {
    let s = String::from("hi");
    let t = s.clone();
    println!("{s} {t}");
    println!("{} {}", s.as_ptr() == t.as_ptr(), s == t);

    let v = vec![1, 2, 3];
    let w = v.clone();
    println!("{:?} {:?} {}", v, w, v == w);
}
```

Output:

```
hi hi
false true
[1, 2, 3] [1, 2, 3] true
```

A **shallow** copy — same pointer, two owners — is exactly what Rust forbids,
because both would free it. That is why `String` is not `Copy` (lesson 4.2).

## Clone costs real work

`Copy` on `Point` is 8 bytes of register shuffling. `clone()` on a large `Vec`
is an allocation plus a memcpy.

```rust
fn main() {
    let big = vec![0u8; 1_000_000];
    println!("{} {}", big.len(), big.capacity());
    let copy = big.clone();
    println!("{}", copy.len());
    println!("{}", std::mem::size_of_val(&big));
}
```

Output:

```
1000000 1000000
1000000
24
```

The handle is 24 bytes; the clone moved a megabyte. This is the asymmetry
`.clone()` hides, and the reason it is a method call and not implicit. It is
still the right call while you are learning — correctness first.

## Copy vs Clone, side by side

| | `Copy` | `Clone` |
|---|---|---|
| Triggered by | `let b = a;`, passing by value | explicit `a.clone()` |
| Implementation | bitwise memcpy, no user code | `fn clone(&self) -> Self` |
| Cost | always trivial | arbitrary; may allocate |
| Source after | still valid | still valid |
| Allowed with heap data | no | yes |
| Allowed with `Drop` | **no** | yes |
| Derivable | only if all fields are `Copy` | if all fields are `Clone` |

## When derive(Copy) is refused

Does not compile (intentionally):

```rust
#[derive(Copy, Clone)]
struct Wrong { name: String }
```

```
error[E0204]: the trait `Copy` cannot be implemented for this type
 --> src/main.rs:2:8
  |
1 | #[derive(Copy, Clone)]
  |          ---- in this derive macro expansion
2 | struct Wrong { name: String }
  |        ^^^^^   ------------ this field does not implement `Copy`
```

`#[derive(Clone)]` alone is fine for the same struct.

## A hand-written Clone

Useful when you want to see or control the copy.

```rust
#[derive(Debug)]
struct Counter { n: u32 }

impl Clone for Counter {
    fn clone(&self) -> Counter {
        println!("cloning at n={}", self.n);
        Counter { n: self.n }
    }
}

fn main() {
    let c = Counter { n: 7 };
    let d = c.clone();
    println!("{:?} {:?}", c, d);
}
```

Output:

```
cloning at n=7
Counter { n: 7 } Counter { n: 7 }
```

Nothing stops a `Clone` impl from being shallow or lazy — that is how `Rc`
works (lesson 11.2): `Rc::clone` bumps a counter instead of copying data.

## Cloning something that is already Copy

Legal but pointless. Clippy says so.

```rust
let n = 5i32;
let m = n.clone();
```

```
warning: using `clone` on type `i32` which implements the `Copy` trait
 --> src/main.rs:3:13
  |
3 |     let m = n.clone();
  |             ^^^^^^^^^ help: try removing the `clone` call: `n`
  |
  = note: `#[warn(clippy::clone_on_copy)]` on by default
```

## Drop: the cleanup hook

`drop(&mut self)` runs when the value goes out of scope. You do not call it and
you cannot forget it.

```rust
struct Res(&'static str);

impl Drop for Res {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let _x = Res("x");
    let _y = Res("y");
    let _z = Res("z");
    println!("end of main");
}
```

Output:

```
end of main
drop z
drop y
drop x
```

**Variables drop in reverse declaration order** — a stack, so later values can
rely on earlier ones still being alive.

## Struct fields drop in declaration order

The opposite of variables. Worth knowing because it is counterintuitive.

```rust
struct Res(&'static str);
impl Drop for Res {
    fn drop(&mut self) { println!("drop {}", self.0); }
}

struct Holder { a: Res, b: Res }

fn main() {
    let _h = Holder { a: Res("field a"), b: Res("field b") };
    println!("holder built");
}
```

Output:

```
holder built
drop field a
drop field b
```

`a` before `b`, matching the declaration. Tuples, arrays, and `Vec` elements
also drop front to back.

## std::mem::drop

`drop(x)` is a one-line function in the prelude that takes `x` by value and
lets it fall out of scope immediately.

```rust
struct Res(&'static str);
impl Drop for Res {
    fn drop(&mut self) { println!("drop {}", self.0); }
}

fn main() {
    let early = Res("early");
    drop(early);
    println!("after explicit drop");
    let _late = Res("late");
    println!("end of main");
}
```

Output:

```
drop early
after explicit drop
end of main
drop late
```

Calling the trait method directly is forbidden — it would run the destructor
and then run it again at scope end. Does not compile:

```rust
let r = Res;
r.drop();
```

```
error[E0040]: explicit use of destructor method
 --> src/main.rs:8:7
  |
8 |     r.drop();
  |       ^^^^ explicit destructor calls not allowed
  |
help: consider using `drop` function
  |
8 -     r.drop();
8 +     drop(r);
  |
```

## Why Drop and Copy are mutually exclusive

If a type were both, a bitwise duplicate would produce two values and the
destructor would run twice on the same resource — a double free or a
double close. Does not compile:

```rust
#[derive(Copy, Clone)]
struct Guard;

impl Drop for Guard {
    fn drop(&mut self) {}
}
```

```
error[E0184]: the trait `Copy` cannot be implemented for this type; the type has a destructor
 --> src/main.rs:2:8
  |
1 | #[derive(Copy, Clone)]
  |          ---- in this derive macro expansion
2 | struct Guard;
  |        ^^^^^ `Copy` not allowed on types with destructors
  |
note: destructor declared here
 --> src/main.rs:5:5
  |
5 |     fn drop(&mut self) {}
  |     ^^^^^^^^^^^^^^^^^^
```

`Clone` + `Drop` is fine — `Clone` runs your code, which can acquire a second
resource properly.

## Drop for RAII

This is the real use: a value whose existence *is* the held resource. Acquire in
the constructor, release in `drop`, and it is impossible to leak on any exit
path — including an early `return`, a `?`, or a panic.

```rust
struct FileHandle { path: String }

impl FileHandle {
    fn open(path: &str) -> FileHandle {
        println!("open {path}");
        FileHandle { path: path.to_string() }
    }
    fn write(&self, data: &str) {
        println!("write {data} to {}", self.path);
    }
}

impl Drop for FileHandle {
    fn drop(&mut self) {
        println!("close {}", self.path);
    }
}

fn process() -> Result<(), String> {
    let f = FileHandle::open("/tmp/a.log");
    f.write("line 1");
    Err(String::from("disk full"))
}

fn main() {
    println!("{:?}", process());
}
```

Output:

```
open /tmp/a.log
write line 1 to /tmp/a.log
close /tmp/a.log
Err("disk full")
```

The close happened on the error path with no `defer`, no `finally`, and no
cleanup code at the return site. Nested scopes release in the right order
automatically:

```rust
fn main() {
    let _outer = FileHandle::open("outer.log");
    {
        let _inner = FileHandle::open("inner.log");
    }
    println!("inner is closed already");
}
```

Output:

```
open outer.log
open inner.log
close inner.log
inner is closed already
close outer.log
```

The standard library uses this everywhere: `File` closes its descriptor,
`MutexGuard` unlocks on drop (lesson 14.3), `Vec` frees its buffer. You rarely
write `impl Drop` yourself — you get it from the types you hold.

## Quick Reference

| Trait | Signature | Derive requires |
|---|---|---|
| `Copy` | marker, no methods | every field `Copy`, no `Drop` |
| `Clone` | `fn clone(&self) -> Self` | every field `Clone` |
| `Drop` | `fn drop(&mut self)` | hand-written only |

| Drop order | Direction |
|---|---|
| local variables | reverse declaration |
| struct fields | declaration order |
| tuple elements | left to right |
| `Vec` / array elements | front to back |
| nested scopes | innermost first |

| Operation | Effect |
|---|---|
| `let b = a;` with `Copy` | duplicate, `a` valid |
| `a.clone()` | deep copy via your impl |
| `drop(a)` | moves `a`, runs destructor now |
| `a.drop()` | `E0040`, not allowed |
| `mem::forget(a)` | moves `a`, skips the destructor |

| Error | Cause |
|---|---|
| `E0204` | `derive(Copy)` with a non-`Copy` field |
| `E0184` | `Copy` on a type with a `Drop` impl |
| `E0040` | calling `.drop()` directly |

## Common Mistakes

**Deriving `Copy` on anything with a `String` or `Vec`.** `E0204`. Derive only
`Clone`.

**Expecting `.clone()` to be cheap.** It can allocate megabytes. Fine while
learning; profile before it ships.

**Assuming struct fields drop in reverse like variables.** Fields go in
declaration order.

**Calling `x.drop()`.** Use `drop(x)`; the method form is `E0040`.

**Writing `impl Drop` to "free" memory.** Memory is already handled. `Drop` is
for things the allocator does not know about: flushes, unlocks, logs, handles.

## Exercises

1. Derive `Clone, Copy` on a struct of three numeric fields, assign it twice,
   mutate one copy, and prove the other is unchanged.
2. Add a `String` field to that struct, read `E0204`, then fix it with the
   minimum change.
3. Write `Noisy` with `Drop` and produce this exact output order using only
   nested blocks and `drop()`: `b`, `a`, `c`.
4. Build `struct Tree { label: String, children: Vec<Tree> }` with a `Drop` that
   prints the label. Build a three-level tree and predict the drop order before
   running it.
5. Write a `Timer` that records `Instant::now()` on construction and prints the
   elapsed time in `drop`. Use it to time a loop with no explicit stop call.

---

[← Course Index](../README.md)
