[← 14.3 Shared State with Mutex and RwLock](03-shared-state.md) · [Course Index](../README.md) · [14.5 Atomics →](05-atomics.md)

# 14.4 — Send, Sync, and Thread Safety

## What You'll Learn

- What `Send` and `Sync` mean, and why you never implement them
- The `&T: Send` ⇔ `T: Sync` rule that explains every confusing error message
- What is and is not thread-safe, in one table
- The `'static` bound on `spawn`, opting out with `PhantomData`, and `unsafe impl`

## Coming From Another Language

There is no equivalent. Go, Python, Java and JS all let you share anything with
any thread and hope. Rust encodes thread safety in two marker traits that the
compiler derives for you, so "this type cannot cross a thread boundary" is a
compile error with a line number.

## The two traits

Both are **auto traits**: the compiler implements them for your type
automatically if every field implements them. They have no methods — they are
pure facts about a type.

- **`Send`** — a value of this type can be *moved* to another thread.
- **`Sync`** — a `&T` can be *shared* with another thread, i.e. `&T: Send`.

That second definition is the whole trick: `Sync` is defined in terms of `Send`
on references. From it follows the one rule to remember:

> To share `&T` across threads, `T` must be `Sync`.
> To move `T` across threads, `T` must be `Send`.

Almost everything is both. The interesting types are the few that are not, and
they are not for exactly one reason each: a non-atomic refcount (`Rc`), a
non-atomic borrow flag (`RefCell`, `Cell`), a thread-bound OS resource
(`MutexGuard`), or no safety claim at all (raw pointers).

## Checking a type

Two helper functions turn a trait bound into a yes/no question.

```rust
use std::sync::{Arc, Mutex, RwLock};

fn is_send<T: Send>() -> &'static str { "Send" }
fn is_sync<T: Sync>() -> &'static str { "Sync" }

fn main() {
    println!("i32:         {} {}", is_send::<i32>(), is_sync::<i32>());
    println!("String:      {} {}", is_send::<String>(), is_sync::<String>());
    println!("Arc<i32>:    {} {}", is_send::<Arc<i32>>(), is_sync::<Arc<i32>>());
    println!("Mutex<i32>:  {} {}", is_send::<Mutex<i32>>(), is_sync::<Mutex<i32>>());
    println!("RwLock<i32>: {} {}", is_send::<RwLock<i32>>(), is_sync::<RwLock<i32>>());
    println!("&i32:        {} {}", is_send::<&i32>(), is_sync::<&i32>());
    println!("Cell<i32>:   {}", is_send::<std::cell::Cell<i32>>());
}
```

Output:

```
i32:         Send Sync
String:      Send Sync
Arc<i32>:    Send Sync
Mutex<i32>:  Send Sync
RwLock<i32>: Send Sync
&i32:        Send Sync
Cell<i32>:   Send
```

`Cell<i32>` is `Send` (moving the whole cell is fine — one thread at a time owns
it) but not `Sync` (two threads sharing `&Cell` could both write). Asking for
`is_sync::<Cell<i32>>()` is a compile error, which is why that line is missing.

Try a type that is not `Send` at all:

```rust
use std::sync::MutexGuard;
fn is_send<T: Send>() -> &'static str { "Send" }

fn main() {
    println!("{}", is_send::<MutexGuard<'static, i32>>());
}
```

Intentionally broken:

```
error[E0277]: `std::sync::MutexGuard<'static, i32>` cannot be sent between threads safely
 --> src/main.rs:4:30
  |
4 |     println!("{}", is_send::<MutexGuard<'static, i32>>());
  |                              ^^^^^^^^^^^^^^^^^^^^^^^^ `std::sync::MutexGuard<'static, i32>` cannot be sent between threads safely
  |
  = help: the trait `Send` is not implemented for `std::sync::MutexGuard<'static, i32>`
note: required by a bound in `is_send`
```

A `MutexGuard` is `!Send` because POSIX requires the thread that locked a mutex
to be the one that unlocks it. It *is* `Sync` when `T: Sync`, so you may lend
`&T` out of a guard to a scoped thread.

## The table

| Type | Send | Sync | Why |
|---|---|---|---|
| `i32`, `String`, `Vec<T>`, `Box<T>` | yes | yes | if `T` is |
| `&T` | if `T: Sync` | if `T: Sync` | sharing a reference *is* `Sync` |
| `&mut T` | if `T: Send` | if `T: Sync` | exclusive, so moving it is a move of `T` |
| `Rc<T>` | **no** | **no** | non-atomic refcount; two threads cloning would corrupt it |
| `Arc<T>` | if `T: Send + Sync` | if `T: Send + Sync` | atomic refcount |
| `Cell<T>` | if `T: Send` | **no** | no synchronization on writes |
| `RefCell<T>` | if `T: Send` | **no** | non-atomic borrow counter |
| `Mutex<T>` | if `T: Send` | if `T: Send` | lock provides the synchronization |
| `RwLock<T>` | if `T: Send` | if `T: Send + Sync` | hands out `&T` to many threads |
| `MutexGuard<'_, T>` | **no** | if `T: Sync` | must unlock on the locking thread |
| `AtomicUsize` etc. | yes | yes | that is the point |
| `*const T`, `*mut T` | **no** | **no** | no safety claim is being made |
| `std::thread::Thread` | yes | yes | handle is just an id |

Note the `Arc<T>: Send` requirement: `T` must be `Sync` too, because `Arc`
hands `&T` to every clone. That is why `Arc<RefCell<T>>` is useless across
threads and `Arc<Mutex<T>>` is not.

## Sending an Rc fails

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let r = Rc::new(5);
    let r2 = Rc::clone(&r);
    thread::spawn(move || {
        println!("{}", *r2);
    }).join().unwrap();
}
```

Intentionally broken:

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> src/main.rs:7:19
    |
  7 |       thread::spawn(move || {
    |       ------------- ^------
    |       |             |
    |  _____|_____________within this `{closure@src/main.rs:7:19: 7:26}`
    | |     |
    | |     required by a bound introduced by this call
  8 | |         println!("{}", *r2);
  9 | |     }).join().unwrap();
    | |_____^ `Rc<i32>` cannot be sent between threads safely
    |
    = help: within `{closure@src/main.rs:7:19: 7:26}`, the trait `Send` is not implemented for `Rc<i32>`
note: required because it's used within this closure
note: required by a bound in `spawn`
    |
128 |     F: Send + 'static,
    |        ^^^^ required by this bound in `spawn`
```

Read it backwards: `spawn` needs `F: Send`; the closure captured an `Rc`;
`Rc` is not `Send`; therefore the closure is not `Send`. **The error is always
reported on the closure**, and the `within` line names the real culprit.

Swap in `Arc` and it compiles. That is the entire difference between the two
types — `Rc` uses `+= 1` on the refcount, `Arc` uses `fetch_add`.

## Why &T needs T: Sync

```rust
use std::cell::RefCell;
use std::sync::Arc;
use std::thread;

fn main() {
    let c = Arc::new(RefCell::new(0));
    let c2 = Arc::clone(&c);
    thread::spawn(move || {
        *c2.borrow_mut() += 1;
    }).join().unwrap();
}
```

Intentionally broken:

```
error[E0277]: `RefCell<i32>` cannot be shared between threads safely
   --> src/main.rs:8:19
    |
  8 |       thread::spawn(move || {
    |  _____-------------_^
    | |     |
    | |     required by a bound introduced by this call
  9 | |         *c2.borrow_mut() += 1;
 10 | |     }).join().unwrap();
    | |_____^ `RefCell<i32>` cannot be shared between threads safely
    |
    = help: the trait `Sync` is not implemented for `RefCell<i32>`
    = note: if you want to do aliasing and mutation between multiple threads, use `std::sync::RwLock` instead
    = note: required for `Arc<RefCell<i32>>` to implement `Send`
```

The two `note` lines are the rule in action: `Arc<T>` is `Send` only if
`T: Sync`, and `RefCell` is not `Sync`. Nothing was moved wrongly — the problem
is that cloning the `Arc` would let two threads call `borrow_mut()` at once and
tear the borrow counter. Use `Arc<Mutex<i32>>` or `Arc<AtomicI32>`.

## The 'static bound

`spawn` requires `F: Send + 'static`. `Send` is about *what* you captured;
`'static` is about *how long* it lives.

```rust
use std::thread;

fn process(data: &[i32]) {
    thread::spawn(move || {
        println!("{}", data.len());
    }).join().unwrap();
}

fn main() {
    process(&[1, 2, 3]);
}
```

Intentionally broken:

```
error[E0521]: borrowed data escapes outside of function
 --> src/main.rs:4:5
  |
3 |   fn process(data: &[i32]) {
  |              ----  - let's call the lifetime of this reference `'1`
  |              |
  |              `data` is a reference that is only valid in the function body
4 | /     thread::spawn(move || {
5 | |         println!("{}", data.len());
6 | |     }).join().unwrap();
  | |      ^
  | |      |
  | |______`data` escapes the function body here
  |        argument requires that `'1` must outlive `'static`
```

The `join()` on the next line proves the thread cannot outlive `process` — but
the *type signature* of `spawn` does not know that. `spawn` returns a
`JoinHandle` you are free to drop, detaching the thread, so it must assume the
worst. `'static` means "contains no borrow the compiler cannot prove eternal",
not "lives forever".

Three ways out: own the data (`data.to_vec()`), use an
`Arc<[i32]>`, or use `thread::scope`, which ties the thread to a block and
therefore *can* accept borrows — lesson 14.6.

## Opting out with PhantomData

To make your own type `!Send`, give it a zero-sized field of a `!Send` type.
`PhantomData<*const ()>` is the conventional marker.

```rust
use std::marker::PhantomData;
use std::thread;

struct Handle {
    id: u32,
    _not_send: PhantomData<*const ()>,
}

fn consume(h: Handle) -> u32 { h.id }

fn main() {
    let h = Handle { id: 7, _not_send: PhantomData };
    thread::spawn(move || println!("{}", consume(h))).join().unwrap();
}
```

Intentionally broken:

```
error[E0277]: `*const ()` cannot be sent between threads safely
   --> src/main.rs:13:19
    |
 13 |     thread::spawn(move || println!("{}", consume(h))).join().unwrap();
    |     ------------- -------^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |     |             |
    |     |             `*const ()` cannot be sent between threads safely
    |     |             within this `{closure@src/main.rs:13:19: 13:26}`
    |     required by a bound introduced by this call
    |
    = help: within `{closure@src/main.rs:13:19: 13:26}`, the trait `Send` is not implemented for `*const ()`
note: required because it appears within the type `PhantomData<*const ()>`
note: required because it appears within the type `Handle`
```

`PhantomData<T>` is zero-sized, so the marker costs nothing at runtime. Use this
for handles tied to one thread — a GPU context, a GUI widget, an FFI object with
thread-local state.

A subtlety worth knowing: closure capture is **per field** since edition 2021.
`move || println!("{}", h.id)` captures only `h.id: u32` and compiles fine; the
version above compiles the whole `Handle` into the closure by passing it to
`consume`, which is what triggers the error. If you need the guarantee to be
airtight, keep the fields private.

## unsafe impl Send and the contract

When you wrap a raw pointer, the compiler cannot know your type is safe to move,
so you must assert it. `unsafe impl` is you taking responsibility.

```rust
use std::thread;

struct RawBuf {
    ptr: *mut u8,
    len: usize,
}

unsafe impl Send for RawBuf {}

impl RawBuf {
    fn new(len: usize) -> Self {
        let mut v = vec![0u8; len];
        let ptr = v.as_mut_ptr();
        std::mem::forget(v);
        RawBuf { ptr, len }
    }
    fn fill(&mut self, b: u8) {
        unsafe { std::ptr::write_bytes(self.ptr, b, self.len) }
    }
    fn first(&self) -> u8 {
        unsafe { *self.ptr }
    }
}

impl Drop for RawBuf {
    fn drop(&mut self) {
        unsafe { drop(Vec::from_raw_parts(self.ptr, self.len, self.len)) }
    }
}

fn main() {
    let mut buf = RawBuf::new(4);
    let h = thread::spawn(move || {
        buf.fill(0xAB);
        buf.first()
    });
    println!("{:#x}", h.join().unwrap());
}
```

Output:

```
0xab
```

What the `unsafe impl Send` promises: the pointer is uniquely owned by this
struct, nothing else aliases it, and it is valid on any thread. It is true here
because `RawBuf::new` is the only constructor and the allocation is exclusive.

Note what was **not** written: `unsafe impl Sync`. That would additionally claim
that two threads holding `&RawBuf` is fine — false, since `first()` could race
with another thread's `fill()` via interior mutability patterns. `Send` without
`Sync` is a common and correct combination. Module 16 covers `unsafe` properly.

Getting these wrong is undefined behaviour, not a panic. If a `Mutex` or an
`Arc` can express it, use those instead.

## How this makes data races a compile error

A data race needs three things at once: two threads, the same memory, and at
least one write with no synchronization. Rust blocks each path to it:

- Moving data in requires `Send`, so `Rc`'s non-atomic counter cannot be shared.
- Sharing `&T` requires `Sync`, so `RefCell`'s non-atomic flag cannot be shared.
- Getting `&mut T` out of a shared handle requires a `Mutex`/`RwLock`/atomic,
  which is precisely the synchronization.
- `'static` (or a `scope`) guarantees the memory outlives the thread.

Nothing here is a runtime check. `Send` and `Sync` have no methods and no code —
they are erased entirely at compile time, so the safety costs nothing.

Deadlocks, livelocks and logic races are still yours to get right (14.3).

## Quick Reference

| Item | Meaning |
|---|---|
| `T: Send` | a `T` may be moved to another thread |
| `T: Sync` | `&T` may be shared with another thread (`&T: Send`) |
| auto trait | derived structurally; implemented iff all fields are |
| `spawn` bound | `F: FnOnce() -> T + Send + 'static`, `T: Send` |
| `Rc` → `Arc` | the `Send`/`Sync` fix for shared ownership |
| `RefCell`/`Cell` → `Mutex`/`RwLock`/atomic | the `Sync` fix for interior mutability |
| `PhantomData<*const ()>` | makes a type `!Send` and `!Sync`, zero cost |
| `PhantomData<Cell<()>>` | `Send` but `!Sync` |
| `unsafe impl Send for T {}` | you assert moving it between threads is sound |
| `fn f<T: Send>()` | the trick for asking the compiler a yes/no question |

## Common Mistakes

**Reading the error as being about the closure.** The closure is only guilty by
association. Look for the `within ...` / `required because` line — it names the
field or capture that is actually `!Send`.

**Reaching for `unsafe impl Send` to silence an error.** That turns a compile
error into potential undefined behaviour. Nearly always the answer is `Arc`,
`Mutex`, or an atomic.

**Expecting `Arc<T>` alone to make `T` shareable.** `Arc<T>: Send` requires
`T: Sync`. `Arc<RefCell<T>>` is rejected; `Arc<Mutex<T>>` is not.

**Thinking `'static` means "leaked".** It means no unprovable borrows. An owned
`String` is `'static`; a `&String` borrowed from a local is not.

**Trying to `impl Send for MyType {}` without `unsafe`.** These are unsafe
traits; the compiler rejects a safe impl outright.

## Exercises

1. Write `fn assert_send_sync<T: Send + Sync>() {}` and call it with
   `Arc<Mutex<Vec<String>>>`, then with `Arc<RefCell<i32>>`. Read the second
   error and name the exact offending bound.
2. Take the failing `Rc` example, convert it to compile with four threads
   incrementing a shared total, and print 4.
3. Define `struct GpuContext { handle: u64, _m: PhantomData<*const ()> }` and a
   `fn use_ctx(c: GpuContext)`. Show it cannot be sent, then make a sendable
   `SendableCtx` wrapper with `unsafe impl Send` and a comment-free justification
   in your reply.
4. Fix `fn process(data: &[i32])` from the `'static` section three ways: owning,
   `Arc<[i32]>`, and `thread::scope`.
5. Predict `Send`/`Sync` for `Vec<Rc<i32>>`, `Mutex<Rc<i32>>`,
   `Arc<Mutex<Rc<i32>>>`, and `RwLock<Cell<i32>>`, then verify each with
   `assert_send_sync`.

---

[← 14.3 Shared State with Mutex and RwLock](03-shared-state.md) · [Course Index](../README.md) · [14.5 Atomics →](05-atomics.md)
