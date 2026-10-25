[← 9.8 Associated Types and Generic Traits](../09-generics-traits/08-associated-types.md) · [Course Index](../README.md) · [10.2 Fn, FnMut, FnOnce →](02-fn-traits.md)

# 10.1 — Closures and Capture Modes

## What You'll Learn

- Every syntactic form a closure can take
- How a closure's parameter types are inferred, and why they lock after first use
- The three capture modes and which one the compiler picks
- `move`, closure size, and per-field capture

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `x => x + 1` | `\|x\| x + 1` |
| Python `lambda x: x + 1` | `\|x\| x + 1` |
| Go `func(x int) int { return x + 1 }` | `\|x: i32\| -> i32 { x + 1 }` |

Same idea, one big difference: a Rust closure's captures are governed by the
borrow checker. The closure's *type* records whether it borrowed, mutably
borrowed, or took ownership — and that determines what you can do with it.

## The syntax forms

```rust
fn main() {
    let add_one = |x| x + 1;
    let add_one_typed = |x: i32| -> i32 { x + 1 };
    let five = || 5;
    let shout = |s: &str| s.to_uppercase();
    println!("{} {} {} {}", add_one(1), add_one_typed(2), five(), shout("hi"));
}
```

Output:

```
2 3 5 HI
```

Rules: no parameters means `||`. A return type annotation requires braces, so
`|x: i32| -> i32 x + 1` is not legal. Annotations are optional whenever
inference can see a call site.

## Capturing the environment

A closure reads surrounding variables with no syntax at all.

```rust
let n = 10;
let add_n = |x: i32| x + n;
println!("{}", add_n(5));
```

Output:

```
15
```

This is the one thing `fn` cannot do — a plain function body cannot see `n`.

## Types lock after the first call

Inference picks a single concrete signature; it is not generic.

```rust
let f = |x| x;
let a = f(5);
let b = f("hello");
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:4:15
  |
4 |     let b = f("hello");
  |             - ^^^^^^^ expected integer, found `&str`
  |
note: expected because the closure was earlier called with an argument of type `{integer}`
 --> src/main.rs:3:15
  |
3 |     let a = f(5);
  |             - ^ expected because this argument is of type `{integer}`
```

For a genuinely generic callable, write a generic `fn`.

## Capture by immutable reference

If the body only reads, the closure holds `&T` and the original stays usable.

```rust
let list = vec![1, 2, 3];
let print_list = || println!("{:?}", list);
print_list();
print_list();
println!("{:?}", list);
```

Output:

```
[1, 2, 3]
[1, 2, 3]
[1, 2, 3]
```

Many immutable borrows coexist, so this closure can be called any number of
times and `list` can be read alongside it.

## Capture by mutable reference

Writing to a captured variable upgrades the capture to `&mut T`.

```rust
let mut count = 0;
let mut inc = || { count += 1; };
inc();
inc();
println!("{}", count);
```

Output:

```
2
```

Note `let mut inc` — calling a closure that holds a `&mut` requires a mutable
binding, because the call mutably borrows the closure itself.

```rust
let mut count = 0;
let inc = || { count += 1; };
inc();
```

Does not compile:

```
error[E0596]: cannot borrow `inc` as mutable, as it is not declared as mutable
 --> src/main.rs:4:5
  |
3 |     let inc = || { count += 1; };
  |                    ----- calling `inc` requires mutable binding due to mutable borrow of `count`
4 |     inc();
  |     ^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
3 |     let mut inc = || { count += 1; };
  |         +++
```

While a `&mut` capture is alive you cannot touch the original — the usual
exclusivity rule, nothing closure-specific.

## Capture by move

`move` forces every capture to be taken by value.

```rust
let data = vec![1, 2, 3];
let owns = move || println!("{:?}", data);
owns();
```

Output:

```
[1, 2, 3]
```

`data` is gone from the enclosing scope after that line. For `Copy` types
`move` copies instead, so the original remains usable.

## What the compiler picks

The compiler chooses the **least** invasive mode that makes the body compile:
`&T`, then `&mut T`, then by value. `move` overrides this and jumps straight to
by value. You never annotate per-variable capture modes; to force a move of one
value only, rebind it first (`let x = x.clone();`) and `move` that.

| Body does | Capture |
|---|---|
| Only reads | `&T` |
| Mutates | `&mut T` |
| Moves the value out, or needs ownership | `T` |
| Anything, with `move` | `T` |

## When `move` is required

Two situations force it. First, a thread:

```rust
use std::thread;
let data = vec![1, 2, 3];
let h = thread::spawn(|| println!("{:?}", data));
h.join().unwrap();
```

Does not compile:

```
error[E0373]: closure may outlive the current function, but it borrows `data`, which is owned by the current function
 --> src/main.rs:4:27
  |
4 |     let h = thread::spawn(|| println!("{:?}", data));
  |                           ^^                  ---- `data` is borrowed here
  |                           |
  |                           may outlive borrowed value `data`
  |
help: to force the closure to take ownership of `data` (and any other referenced variables), use the `move` keyword
  |
4 |     let h = thread::spawn(move || println!("{:?}", data));
  |                           ++++
```

Second, returning a closure from a function — a borrow of a local would dangle.
Lesson 10.2 covers the return forms.

## Capturing is per-field, not per-struct

A closure borrows only the places it actually names.

```rust
struct Rect { w: f64, h: f64 }

let mut r = Rect { w: 3.0, h: 4.0 };
let mut grow = || r.w += 1.0;
grow();
println!("{}", r.h);
println!("{}", r.w);
```

Output:

```
4
4
```

`grow` holds `&mut r.w`, so reading `r.h` in between is fine. Name both fields
and the borrow covers both:

```rust
let mut r = Rect { w: 3.0, h: 4.0 };
let show = || println!("{:?}", (r.w, r.h));
r.h = 5.0;
show();
```

Does not compile:

```
error[E0506]: cannot assign to `r.h` because it is borrowed
 --> src/main.rs:5:5
  |
4 |     let show = || println!("{:?}", (r.w, r.h));
  |                -- `r.h` is borrowed here --- borrow occurs due to use in closure
5 |     r.h = 5.0;
  |     ^^^^^^^^^ `r.h` is assigned to here but it was already borrowed
6 |     show();
  |     ---- borrow later used here
```

Disjoint field capture landed in edition 2021; in edition 2018 the first
example above failed too, because the closure captured all of `r`. Edition 2024
behaves as shown.

## Closures in struct fields

A closure has an anonymous, unnameable type, so a struct field needs
`Box<dyn Fn...>` (or a generic parameter).

```rust
struct Button {
    label: String,
    on_click: Box<dyn Fn(&str)>,
}

let b = Button {
    label: "OK".to_string(),
    on_click: Box::new(|l| println!("clicked {l}")),
};
(b.on_click)(&b.label);
```

Output:

```
clicked OK
```

The parentheses around `b.on_click` are required — without them Rust looks for
a method named `on_click`.

## Closures vs `fn`

A non-capturing closure coerces to a function pointer, so both work as an
argument to the same generic parameter.

```rust
fn apply(f: impl Fn(i32) -> i32, v: i32) -> i32 { f(v) }
fn double(x: i32) -> i32 { x * 2 }

println!("{} {}", apply(double, 7), apply(|x| x * 2, 7));
```

Output:

```
14 14
```

| | `fn` | closure |
|---|---|---|
| Captures environment | no | yes |
| Has a nameable type | yes (`fn(i32) -> i32`) | no (anonymous) |
| Type inference on params | no | yes |
| Size | one pointer | size of its captures |

## Closure size

A closure is a struct holding its captures. No captures means zero bytes.

```rust
let n = 10;
let more = vec![1, 2, 3];

let a = || ();
let b = |x: i32| x;
let c = move || n;
let big = move || (n, more.len());
println!("{} {} {} {}",
    std::mem::size_of_val(&a),
    std::mem::size_of_val(&b),
    std::mem::size_of_val(&c),
    std::mem::size_of_val(&big));
```

Output:

```
0 0 4 32
```

`c` holds one `i32`. `big` holds an `i32` plus a `Vec` (24 bytes) with padding.
There is no heap allocation and no vtable unless you box it — which is why
`iter().map(|x| ...)` costs nothing over a hand-written loop.

## Multi-statement bodies

Braces make a block; the last expression is the return value.

```rust
let mut acc = String::new();
let mut push = |s: &str| acc.push_str(s);
push("a");
push("b");
println!("{acc}");

let classify = |n: i32| {
    if n < 0 { "neg" }
    else if n == 0 { "zero" }
    else { "pos" }
};
println!("{} {} {}", classify(-1), classify(0), classify(3));
```

Output:

```
ab
neg zero pos
```

## Quick Reference

| Form | Meaning |
|---|---|
| `\|\| expr` | no parameters |
| `\|x\| expr` | one inferred parameter |
| `\|x: i32\| expr` | annotated parameter |
| `\|x: i32\| -> i32 { expr }` | annotated return; braces required |
| `\|x, y\| expr` | two parameters |
| `move \|\| expr` | capture everything by value |
| `let mut c = ...; c()` | needed when the closure captures `&mut` |
| `Box<dyn Fn(A) -> B>` | storing a closure in a field |
| `std::mem::size_of_val(&c)` | size of its captures |

## Common Mistakes

**Expecting a closure to be generic.** `|x| x` locks to the first type it is
called with; use a generic `fn` instead.

**Forgetting `mut` on the binding.** A closure that mutates a capture needs
`let mut`, even though you never reassign it.

**Reaching for `move` by default.** Let the compiler pick; add `move` only for
threads, `'static` bounds, or returning a closure.

**Writing `|x: i32| -> i32 x + 1`.** A return annotation requires a block body.

**Calling `b.on_click(...)` on a field.** Write `(b.on_click)(...)`; method
syntax looks for a method, not a field.

## Exercises

1. Write three closures that add 1: one fully inferred, one with a parameter
   annotation, one with both annotations and a block body.
2. Capture a `String` immutably in one closure and mutably in another. Make the
   mutable one compile, then try calling both and explain the error.
3. Take `let v = vec![1, 2, 3];` and spawn a thread that prints it. Fix the
   error the compiler reports.
4. Build `struct Repeat { times: u32, action: Box<dyn Fn(u32)> }` with a method
   that calls `action` once per index.
5. Print `size_of_val` for closures capturing nothing, one `u8`, one `u64`, and
   a `String`. Explain each number.

---

[← 9.8 Associated Types and Generic Traits](../09-generics-traits/08-associated-types.md) · [Course Index](../README.md) · [10.2 Fn, FnMut, FnOnce →](02-fn-traits.md)
