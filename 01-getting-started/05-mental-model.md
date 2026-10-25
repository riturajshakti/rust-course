[← 1.4 rustfmt, Clippy, and rust-analyzer](04-tooling.md) · [Course Index](../README.md) · [2.1 Variables, Mutability, and Shadowing →](../02-syntax-basics/01-variables.md)

# 1.5 — How Rust Thinks: A Mental Model

## What You'll Learn

- Ownership as the replacement for garbage collection
- Mutability XOR sharing — the one rule behind most borrow errors
- Errors and absence as ordinary values, not control flow
- How to read a rustc error, and what to do when you're stuck

## Coming From Another Language

| Other | Rust |
|---|---|
| GC frees memory, eventually | Scope ends, value drops, exactly then |
| Any reference can mutate anything | One mutable borrow XOR many shared ones |
| `throw` / `raise` / `panic` | `Result<T, E>` returned as a value |
| `null` / `None` / `nil` | `Option<T>`, and it must be unwrapped |
| Abstractions cost at runtime | Iterators compile to the same code as loops |

This lesson is the one to come back to in week three. The syntax you can look
up; these four ideas are what make Rust code look the way it does.

## Ownership replaces the garbage collector

Every value has exactly one owner. When the owner goes out of scope the value is
freed — at a point you can see in the source, not whenever a collector decides.

```rust
struct Conn(&'static str);

impl Drop for Conn {
    fn drop(&mut self) {
        println!("closing {}", self.0);
    }
}

fn main() {
    let _a = Conn("db");
    {
        let _b = Conn("cache");
        println!("inner scope");
    }
    println!("outer scope");
}
```

Output:

```
inner scope
closing cache
outer scope
closing db
```

Nobody called `close`. The `}` did it, at that exact line, in reverse
declaration order. This is the whole trick, and it buys three things at once:

- **No GC.** No runtime, no pauses, no heap scanning.
- **No manual free.** The compiler inserts the `drop` call.
- **No use-after-free or double-free.** Ownership is tracked at compile time.

The same mechanism closes files, releases locks, flushes buffers, and returns
connections to a pool. Rust has no `finally`, no `defer`, no `with`, and no
`using` because `Drop` covers all of them.

The cost is that passing a value **moves** it:

```rust
fn consume(s: String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("hello");
    let n = consume(s);
    println!("{n} {s}");
}
```

Does not compile:

```
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:8:20
  |
6 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
7 |     let n = consume(s);
  |                     - value moved here
8 |     println!("{n} {s}");
  |                    ^ value borrowed here after move
  |
note: consider changing this parameter type in function `consume` to borrow instead if owning the value isn't necessary
 --> src/main.rs:1:15
  |
1 | fn consume(s: String) -> usize {
  |    -------    ^^^^^^ this parameter takes ownership of the value
  |    |
  |    in this function
help: consider cloning the value if the performance cost is acceptable
  |
7 |     let n = consume(s.clone());
  |                      ++++++++
```

Read that error closely, because it is the template for the whole category. It
tells you *why* (`String` isn't `Copy`), *where it moved*, *where you used it
after*, and **two** different fixes — borrow instead of own, or clone.

Take the first one. `&String` borrows without moving:

```rust
fn consume(s: &String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("hello");
    let n = consume(&s);
    println!("{n} {s}");
}
```

Output:

```
5 hello
```

The habit to build: a function that only *reads* should take `&T`. Taking `T`
means "I am keeping this". Module 04 covers moves and borrows in full.

## Mutability XOR sharing

The rule the borrow checker enforces, in one line:

> At any moment, a value may have **either** one mutable reference **or** any
> number of immutable ones — never both.

Not a style guide. A compile-time guarantee, and the reason data races cannot
happen in safe Rust.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];
    v.push(4);
    println!("{first}");
}
```

Does not compile:

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

Three annotated lines: where the shared borrow started, where the exclusive one
conflicted, and where the shared one was still needed. If any one of those three
moved, the program would be fine.

This is not pedantry. `push` may reallocate the vector's buffer, which would
leave `first` pointing at freed memory. In C++ that is a dangling pointer; in
Python and JS it is a silent correctness bug when you mutate a list while
iterating it. Here it is a compile error.

Shorten the borrow and it compiles:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = v[0];
    v.push(4);
    println!("{first} {v:?}");
}
```

Output:

```
1 [1, 2, 3, 4]
```

Copying the `i32` out ends the borrow immediately. The general fixes, in the
order to try them: **copy the value out**, **narrow the scope of the borrow**, or
**restructure so the two never overlap**.

Mutation also requires `mut` on the binding:

```rust
fn main() {
    let v = vec![1, 2, 3];
    v.push(4);
    println!("{v:?}");
}
```

Does not compile:

```
error[E0596]: cannot borrow `v` as mutable, as it is not declared as mutable
 --> src/main.rs:3:5
  |
3 |     v.push(4);
  |     ^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
2 |     let mut v = vec![1, 2, 3];
  |         +++
```

Immutable by default, in a language where that actually holds — `let x` means
the value cannot change through any path, not merely that the binding is
reassigned elsewhere (contrast `const` in JS, which freezes the binding and not
the object).

## Errors and absence are values

There is no `throw`, no exceptions, and no invisible control flow. A function
that can fail says so in its return type.

```rust
fn main() {
    let names = vec!["raj", "sam"];
    println!("{:?}", names.first());
    println!("{:?}", names.get(9));

    let ok: Result<i32, _> = "42".parse::<i32>();
    let bad: Result<i32, _> = "4x".parse::<i32>();
    println!("{ok:?}");
    println!("{bad:?}");
}
```

Output:

```
Some("raj")
None
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

`names.get(9)` does not throw and does not return `undefined` — it returns
`None`, which you cannot accidentally use as a string. The type system forces
the check.

```rust
fn main() {
    match "4x".parse::<i32>() {
        Ok(n) => println!("parsed {n}"),
        Err(e) => println!("failed: {e}"),
    }

    let names = vec!["raj", "sam"];
    println!("{}", names.get(9).copied().unwrap_or("nobody"));
    println!("{}", "4x".parse::<i32>().unwrap_or(-1));

    if let Some(n) = names.first() {
        println!("first is {n}");
    }
}
```

Output:

```
failed: invalid digit found in string
nobody
-1
first is raj
```

Two consequences worth internalizing:

- **Every failure point is visible in the signature.** You cannot be surprised
  by an exception from four frames down, and you never need to guess what a
  function might throw.
- **`Option<T>` is not `T`.** There is no implicit null in any type, so
  "cannot read property of undefined" has no analogue. A nullable value is a
  different type that you must open.

`panic!` exists, for genuinely unrecoverable states — a violated invariant, a
failed assertion. It aborts the thread rather than being caught. `.unwrap()`
panics on `None`/`Err`; it is fine in examples and tests, and a code smell in
anything that must stay up. Module 08 covers the `?` operator, which makes
propagating `Result` as terse as exceptions without hiding it.

## Zero-cost abstractions

High-level code is supposed to compile to the same machine code as the low-level
version. Not approximately — measurably.

```rust
#[inline(never)]
#[unsafe(no_mangle)]
pub fn sum_loop(v: &[u64]) -> u64 {
    let mut t = 0u64;
    let mut i = 0;
    while i < v.len() {
        t += v[i];
        i += 1;
    }
    t
}

#[inline(never)]
#[unsafe(no_mangle)]
pub fn sum_iter(v: &[u64]) -> u64 {
    v.iter().sum()
}

fn main() {
    let v: Vec<u64> = (1..=1000).collect();
    println!("{} {}", sum_loop(&v), sum_iter(&v));
}
```

Output:

```
500500 500500
```

Built with `--release` and disassembled, both functions are **39 instructions**,
with identical bodies — the iterator chain, the `sum()` trait method, and the
closure machinery all evaporate. The `#[inline(never)]` is only there to stop
the optimizer from deleting both functions into the caller so they can be
compared at all.

So prefer the clear version:

```rust
fn main() {
    let total: i32 = (1..=10).filter(|n| n % 2 == 0).map(|n| n * n).sum();
    println!("{total}");
}
```

Output:

```
220
```

That compiles to a single loop with no intermediate allocation. Generics behave
the same way: they are monomorphized — the compiler stamps out a specialized
copy per concrete type, so there is no boxing and no dynamic dispatch unless you
explicitly ask for `dyn Trait`.

What this means in practice: **write the readable version first.** Iterator
chains, `Option` combinators, small wrapper structs, and traits are not
performance compromises. The things that genuinely cost are allocation (`clone`,
`to_string`, collecting into a `Vec`), `dyn` dispatch, and `Arc<Mutex<_>>`
contention — and those are all visible in the source.

## How to read a rustc error

Rust's errors are verbose because they are trying to teach. Learn the anatomy
once and they become fast to scan.

```
error[E0308]: mismatched types
 --> src/main.rs:2:18
  |
2 |     let x: i32 = "5";
  |            ---   ^^^ expected `i32`, found `&str`
  |            |
  |            expected due to this

For more information about this error, try `rustc --explain E0308`.
```

| Part | Here | Means |
|---|---|---|
| `error[E####]` | `E0308` | Stable error code; `rustc --explain E0308` |
| Summary | `mismatched types` | The category |
| `-->` | `src/main.rs:2:18` | File, line, column |
| `^^^` | under `"5"` | The code at fault |
| `---` | under `i32` | Related span — the *cause* |
| Label | `expected due to this` | Why the compiler expected that |
| `= note:` | — | Extra context, lint names, trait info |
| `= help:` | — | What to try |
| `help:` + diff | — | A concrete patch, often auto-applicable |

Note what is **not** here: no suggestion to call `.parse()`. The compiler points
at the conflict and the reason for it; it does not guess your intent. Read the
`---` span as often as the `^^^` one, since the secondary span usually holds the
actual mistake.

The reading order that works:

1. The summary line — which category is this?
2. The `-->` location and the `^^^` span.
3. The `---` spans and their labels — this is where the *why* lives.
4. `help:` last. If it shows a diff, it is usually right.

Two commands that do the rest:

```bash
rustc --explain E0308
cargo clippy --fix --allow-dirty
```

`--explain` prints a page with a minimal example and the fix. `--fix` applies
machine-applicable suggestions for you.

Error codes worth recognizing on sight:

| Code | Means | Usual fix |
|---|---|---|
| `E0308` | Mismatched types | Convert: `.to_string()`, `as`, `.into()` |
| `E0382` | Use of moved value | Borrow with `&`, or `.clone()` |
| `E0502` | Mutable + immutable borrow | Narrow the borrow's scope |
| `E0499` | Two mutable borrows | Split the scopes, or use indices |
| `E0596` | Mutating a non-`mut` binding | Add `mut` |
| `E0106` | Missing lifetime specifier | Own the data, or add `<'a>` |
| `E0277` | Trait not implemented | Add a `#[derive]`, or the trait bound |
| `E0599` | No method found | Import the trait that provides it |

## The learning curve, honestly

The curve is real and it is front-loaded. What to expect:

- **Week 1** — syntax is easy; you already know what a struct is. Fine.
- **Weeks 2-4** — you fight the borrow checker daily. This is the hard part.
  Everyone goes through it; it is not a sign you're missing something.
- **Weeks 5-8** — borrow errors mostly stop happening, because you start
  structuring code so they can't arise. The compiler is quiet more often.
- **After** — the compiler becomes a pair programmer. "If it compiles it works"
  is overstated, but refactoring without fear is not.

Two reframings that shorten it:

**Compiler errors are not failures.** In a GC'd language the equivalent bugs
exist too — they just show up in production instead of in your terminal. A
borrow error is a data race or a use-after-free that you are being shown early.

**Don't fight the error; fix the design.** A stubborn borrow error usually means
the data layout is wrong, not that you need a cleverer workaround. If you are
reaching for `Rc<RefCell<T>>` in week two, you're probably porting a
linked-structure design that wants to be a `Vec` plus indices.

## When you're stuck

In order:

1. **Read the whole error, including the `---` spans.** The answer is in it more
   often than feels plausible.
2. **`rustc --explain E####`.**
3. **`.clone()` and move on.** A clone that unblocks you is worth more than an
   hour of fighting. Leave a TODO; profile later. Clones are cheap to remove and
   almost never the actual bottleneck.
4. **Let the compiler tell you the type.** Annotate deliberately wrong and read
   the `E0308`:

```rust
let x: () = some_confusing_expression();
```

5. **Simplify until it compiles, then add back.** Delete half the function.
6. **Check the signature.** `&T` vs `T` vs `&mut T` is the usual answer.
7. **Ask with a minimal example.** The Rust community is unusually good at this;
   `users.rust-lang.org` answers borrow-checker questions well.

Escape hatches to know exist, and to use sparingly: `.clone()`, `Rc`/`Arc` for
shared ownership (11.2, 11.4), `RefCell` to move a borrow check to runtime
(11.3), and `unsafe` (16.4). Reach for the first one; the rest have real costs.

## Quick Reference

| Idea | In one line |
|---|---|
| Ownership | One owner; dropped at end of scope, deterministically |
| Move | Passing a non-`Copy` value transfers it |
| Borrow | `&T` read, `&mut T` write, without transferring |
| Mutability XOR sharing | One `&mut`, or many `&`, never both |
| `Drop` | Replaces `finally`, `defer`, `with`, destructors |
| `Option<T>` | Absence as a type; no implicit null anywhere |
| `Result<T, E>` | Failure as a return value; no exceptions |
| `panic!` | Unrecoverable only; aborts the thread |
| Zero-cost | Iterators and generics compile to hand-written loops |
| Monomorphization | Generics specialized per type; no boxing |

## Common Mistakes

**Treating a borrow error as the compiler being difficult.** It is reporting a
real aliasing bug. Change the design, not the syntax.

**Refusing to `.clone()` while learning.** Clone freely at first; optimize when
a profiler says to. Premature borrow-golfing is the main reason people stall.

**Reaching for `Rc<RefCell<T>>` early.** It is almost always a ported design
that wants a `Vec` and indices instead.

**Taking `T` when you only need to read.** Default to `&T` in parameters;
take ownership only when you will store or consume the value.

**`.unwrap()` everywhere in real code.** Fine in tests and examples. In a
service it turns a recoverable error into a crash.

**Reading only the `^^^` span.** The `---` secondary spans carry the cause and
the two suggested fixes.

## Exercises

1. Write a struct with a `Drop` impl that prints. Create three in nested scopes
   and predict the output order before running it.
2. Write a function taking `String` by value, call it twice with the same
   variable, and read the `E0382`. Fix it once with `&` and once with `.clone()`.
3. Reproduce the `E0502` from this lesson. Then fix it three different ways:
   copy the value out, add a scope, and reorder the statements.
4. Write `let x: () = "hello".chars().rev();` and use the resulting `E0308` to
   discover the real type of that expression.
5. Trigger `E0599` by calling a trait method without importing the trait (try
   `std::io::Write`). Then run `rustc --explain E0599` and compare its
   explanation to what the error already told you.

---

[← 1.4 rustfmt, Clippy, and rust-analyzer](04-tooling.md) · [Course Index](../README.md) · [2.1 Variables, Mutability, and Shadowing →](../02-syntax-basics/01-variables.md)
