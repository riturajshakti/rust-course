[← 11.2 Rc and Shared Ownership](02-rc.md) · [Course Index](../README.md) · [11.4 Arc and Mutex →](04-arc-mutex.md)

# 11.3 — RefCell and Interior Mutability

## What You'll Learn

- Interior mutability: mutating through a `&T`, checked at runtime
- `Cell<T>` for `Copy` values, `RefCell<T>` for everything else
- The real panic when two `borrow_mut()` overlap, and `try_borrow` to avoid it
- `Rc<RefCell<T>>` as the shared-mutable pattern — and when it's a smell

## Coming From Another Language

| Other | Rust |
|---|---|
| JS/Python: any reference can mutate | `&T` cannot — unless `T` is a `Cell`/`RefCell` |
| Go: mutate through any pointer | borrow rules enforced at compile time, or runtime with `RefCell` |
| C++ `mutable` member | `Cell<T>` / `RefCell<T>` field |

## The concept

Rust's rule is: one `&mut T` **or** any number of `&T`, never both. The compiler
proves it statically, which rules out correct programs it cannot see through —
a shared cache, a counter on an `&self` method, a graph where two parents point
at one child.

Interior mutability moves that check to runtime. `RefCell<T>` holds a borrow
*counter* alongside the value: `borrow()` increments the shared count,
`borrow_mut()` takes the exclusive slot, and each handle releases on drop. The
rule is unchanged — only the enforcement time, and the failure mode, change.
Break it and you get a panic instead of a compile error.

That is the whole trade: you buy flexibility with a runtime check and the risk
of a panic. Use it where the compiler genuinely cannot see your invariant, not
to silence it.

## Cell\<T\>: get/set for Copy types

No borrowing at all — `Cell` copies values in and out, so there is nothing to
track and nothing to panic about.

```rust
use std::cell::Cell;

fn main() {
    let c = Cell::new(5);
    println!("{}", c.get());
    c.set(10);
    println!("{}", c.get());
    println!("{}", c.replace(20));
    println!("{}", c.get());
    println!("{}", c.take());
    println!("{}", c.get());
}
```

Output:

```
5
10
10
20
20
0
```

`replace` returns the old value, `take` returns it and leaves `Default::default()`.
Note `c` is not `mut` — that is the point.

## Cell in an immutable struct

A hit counter on a `&self` method, with no `&mut self` anywhere.

```rust
use std::cell::Cell;

struct Counter {
    hits: Cell<u32>,
}

impl Counter {
    fn record(&self) {
        self.hits.set(self.hits.get() + 1);
    }
}

fn main() {
    let c = Counter { hits: Cell::new(0) };
    c.record();
    c.record();
    c.record();
    println!("{}", c.hits.get());
}
```

Output:

```
3
```

## Cell needs Copy

```rust
use std::cell::Cell;

fn main() {
    let c = Cell::new(String::from("hi"));
    println!("{}", c.get());
}
```

Does not compile — intentional:

```
error[E0599]: the method `get` exists for struct `Cell<String>`, but its trait bounds were not satisfied
   --> src/main.rs:5:22
    |
  5 |     println!("{}", c.get());
    |                      ^^^
    |
353 | pub struct String {
    | ----------------- doesn't satisfy `String: Copy`
    |
    = note: the following trait bounds were not satisfied:
            `String: Copy`
```

`Cell<String>` still works with `set`, `replace` and `take` — only `get` needs
`Copy`. For anything you want to read in place, reach for `RefCell`.

## RefCell: borrow and borrow_mut

```rust
use std::cell::RefCell;

fn main() {
    let r = RefCell::new(vec![1, 2, 3]);
    println!("{:?}", r.borrow());
    r.borrow_mut().push(4);
    println!("{:?}", r.borrow());
    println!("{}", r.borrow().len());
    let owned = r.into_inner();
    println!("{:?}", owned);
}
```

Output:

```
[1, 2, 3]
[1, 2, 3, 4]
4
[1, 2, 3, 4]
```

`borrow()` returns `Ref<T>`, `borrow_mut()` returns `RefMut<T>`; both deref to
`T`, so you use them like references. Each statement here drops its guard at the
semicolon, which is why nothing conflicts. `into_inner` consumes the cell.

## The runtime panic

Two live `borrow_mut()` guards:

```rust
use std::cell::RefCell;

fn main() {
    let r = RefCell::new(5);
    let a = r.borrow_mut();
    let b = r.borrow_mut();
    println!("{} {}", a, b);
}
```

Compiles fine. At runtime — intentional:

```
thread 'main' (13857223) panicked at src/main.rs:6:15:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The number in parentheses is the thread id; yours will differ. The other
direction — a live `borrow()` plus a `borrow_mut()` — panics at the same place:

```rust
let a = r.borrow();
let b = r.borrow_mut();
```

```
thread 'main' (13858057) panicked at src/main.rs:6:15:
RefCell already borrowed
```

Many simultaneous `borrow()` calls are fine; it's the exclusive one that fails.

## The realistic version of that mistake

Nobody writes two `borrow_mut()` in a row. This is how it actually happens — an
iterator guard held across the loop body:

```rust
use std::cell::RefCell;

fn main() {
    let log = RefCell::new(vec![String::from("start")]);
    for entry in log.borrow().iter() {
        log.borrow_mut().push(format!("seen {}", entry));
    }
    println!("{:?}", log.borrow());
}
```

Runtime panic — intentional:

```
thread 'main' (13860581) panicked at src/main.rs:6:13:
RefCell already borrowed
```

The `for` loop keeps the `Ref` alive for the whole loop. Fix: collect what you
need first, drop the guard, then mutate.

```rust
let entries: Vec<String> = log.borrow().clone();
for entry in entries {
    log.borrow_mut().push(format!("seen {}", entry));
}
```

## try_borrow and try_borrow_mut

The non-panicking forms, returning `Result`.

```rust
use std::cell::RefCell;

fn main() {
    let r = RefCell::new(5);
    let a = r.borrow_mut();
    println!("{:?}", r.try_borrow());
    println!("{:?}", r.try_borrow_mut().is_err());
    if let Err(e) = r.try_borrow() {
        println!("{}", e);
    }
    drop(a);
    println!("{:?}", r.try_borrow().is_ok());
    println!("{:?}", r.try_borrow_mut().is_ok());
}
```

Output:

```
Err(BorrowError)
true
RefCell already mutably borrowed
true
true
```

`try_borrow` fails with `BorrowError`, `try_borrow_mut` with `BorrowMutError`.
Use them in a `Drop` impl or a callback where a panic would be unrecoverable;
otherwise `borrow_mut()` and a tight guard scope is clearer.

## Rc\<RefCell\<T\>\>: the shared-mutable pattern

`Rc` gives many owners, `RefCell` gives mutation. Together they are Rust's
answer to "two things need to write to the same object."

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Account {
    owner: String,
    balance: i64,
}

type Shared = Rc<RefCell<Account>>;

fn deposit(acc: &Shared, amount: i64) {
    acc.borrow_mut().balance += amount;
}

fn main() {
    let acc: Shared = Rc::new(RefCell::new(Account { owner: "raj".into(), balance: 100 }));
    let teller = Rc::clone(&acc);
    let auditor = Rc::clone(&acc);

    deposit(&teller, 50);
    deposit(&auditor, -30);
    println!("count = {}", Rc::strong_count(&acc));
    println!("{:?}", acc.borrow());
    println!("{} has {}", acc.borrow().owner, acc.borrow().balance);
}
```

Output:

```
count = 3
Account { owner: "raj", balance: 120 }
raj has 120
```

The order is always `Rc<RefCell<T>>`, not `RefCell<Rc<T>>`: you want one shared
cell, not a swappable pointer. A `type` alias is worth it — the nested form
appears in every signature. Across threads the same pattern is
`Arc<Mutex<T>>` (lesson 11.4).

## When this is right, and when it's a smell

Right:

- A graph or tree where nodes are genuinely shared and mutated.
- A cache or memo table behind an `&self` API.
- An observer list, where callbacks need to touch shared state.
- A field whose mutation is an implementation detail the caller shouldn't see.

A smell:

- `Rc<RefCell<T>>` on every field of every struct — you are writing Java or JS
  in Rust and have disabled the borrow checker wholesale.
- Reaching for it the first time the compiler complains. Usually the fix is to
  restructure: pass `&mut` down instead of sharing, split the struct so the
  mutated part is separate, or return new values instead of mutating in place.
- Tree traversal with parent pointers — indices into a `Vec` arena are simpler
  and faster. Lesson 11.7 revisits this.

If you cannot name the invariant that makes your runtime borrows non-overlapping,
you have a latent panic, not a design.

## OnceCell and LazyCell

`OnceCell<T>` is write-once. `LazyCell<T>` runs a closure the first time it is
read.

```rust
use std::cell::{LazyCell, OnceCell};

fn main() {
    let once: OnceCell<String> = OnceCell::new();
    println!("{:?}", once.get());
    println!("{:?}", once.set(String::from("first")));
    println!("{:?}", once.set(String::from("second")));
    println!("{:?}", once.get());
    println!("{}", once.get_or_init(|| String::from("never runs")));

    let lazy: LazyCell<Vec<u32>> = LazyCell::new(|| {
        println!("computing");
        (1..=5).collect()
    });
    println!("declared");
    println!("{:?}", *lazy);
    println!("{}", lazy.len());
}
```

Output:

```
None
Ok(())
Err("second")
Some("first")
first
declared
computing
[1, 2, 3, 4, 5]
5
```

The second `set` returns `Err` with your value back. Note `computing` prints
after `declared` — that's the laziness. For a `static` or anything shared across
threads, use the `std::sync` versions, `OnceLock` and `LazyLock`.

## Quick Reference

| Type / call | Returns | Panics? |
|---|---|---|
| `Cell::new(v)` / `.get()` | `T` (needs `Copy`) | no |
| `.set(v)` / `.replace(v)` / `.take()` | `()` / `T` / `T` | no |
| `RefCell::new(v)` | `RefCell<T>` | no |
| `.borrow()` | `Ref<T>` | if mutably borrowed |
| `.borrow_mut()` | `RefMut<T>` | if borrowed at all |
| `.try_borrow()` | `Result<Ref<T>, BorrowError>` | no |
| `.try_borrow_mut()` | `Result<RefMut<T>, BorrowMutError>` | no |
| `.into_inner()` | `T` | no (consumes) |
| `.get_mut()` | `&mut T` | no (needs `&mut self`) |
| `OnceCell::set` / `get_or_init` | `Result<(), T>` / `&T` | no |
| `LazyCell::new(f)` | `LazyCell<T>` | on deref if `f` panics |
| `Rc<RefCell<T>>` | shared + mutable | yes, on overlap |

## Common Mistakes

**Holding a guard across a loop or a call.** `for x in c.borrow().iter()` then
mutating inside panics. Clone or collect first, or bind the guard in a tighter
scope.

**Writing `let v = c.borrow_mut();` at the top of a long function.** The guard
lives to the end of the scope. Keep it to one statement, or `drop(v)` explicitly.

**Using `RefCell` to dodge a borrow error you could restructure away.** You
replaced a compile error with a panic in production.

**Picking `Cell` for a `String` or `Vec`.** `get` needs `Copy`. Use `RefCell`.

**Using `RefCell` across threads.** It is not `Sync`. Use `Mutex` — lesson 11.4.

## Exercises

1. Write `struct Stats { count: Cell<u32>, total: Cell<i64> }` with
   `fn add(&self, v: i64)` taking `&self`. Call it four times and print both
   fields.
2. Build a `RefCell<Vec<i32>>`, then deliberately trigger the borrow panic.
   Record the exact message, then fix it with a tighter guard scope.
3. Implement `struct Memo { cache: RefCell<HashMap<u32, u64>> }` with
   `fn fib(&self, n: u32) -> u64` taking `&self`. Print `fib(40)` and the cache
   size.
4. Model two `Rc<RefCell<Node>>` children sharing one parent `Vec`, push to it
   from both handles, and print the result plus `strong_count`.
5. Rewrite exercise 4 without `Rc<RefCell<>>`, storing nodes in a
   `Vec<Node>` and referring to them by index. Compare the two for readability.

---

[← 11.2 Rc and Shared Ownership](02-rc.md) · [Course Index](../README.md) · [11.4 Arc and Mutex →](04-arc-mutex.md)
