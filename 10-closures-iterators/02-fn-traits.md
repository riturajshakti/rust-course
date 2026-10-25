[← 10.1 Closures and Capture Modes](01-closures.md) · [Course Index](../README.md) · [10.3 The Iterator Trait →](03-iterator-trait.md)

# 10.2 — Fn, FnMut, FnOnce

## What You'll Learn

- The three callable traits and which closures implement which
- Taking closures as parameters: `impl Fn` vs `&dyn Fn` vs `Box<dyn Fn>`
- Returning closures from functions
- Function pointers, and storing callbacks in a struct

## Coming From Another Language

In JS, Python, and Go a callback is just a value and any callback fits any
signature. Rust splits callables three ways by what they do to their captures:
read them (`Fn`), mutate them (`FnMut`), or consume them (`FnOnce`). Picking the
loosest bound your function actually needs is the whole skill here.

## The three traits

| Trait | Call signature | Closure can be called |
|---|---|---|
| `FnOnce` | `self` | once |
| `FnMut` | `&mut self` | many times, needs a `mut` binding |
| `Fn` | `&self` | many times, from a shared reference |

They nest: every `Fn` is also `FnMut`, and every `FnMut` is also `FnOnce`. So
`FnOnce` is the *loosest* bound (accepts the most closures) and `Fn` is the
tightest.

```rust
fn call_fn(f: impl Fn() -> String) { println!("{} {}", f(), f()); }
fn call_fnmut(mut f: impl FnMut() -> i32) { println!("{} {}", f(), f()); }
fn call_fnonce(f: impl FnOnce() -> String) { println!("{}", f()); }

fn main() {
    let greeting = String::from("hi");
    call_fn(|| greeting.clone());

    let mut total = 0;
    call_fnmut(|| { total += 1; total });

    let owned = String::from("bye");
    call_fnonce(move || owned);
}
```

Output:

```
hi hi
1 2
bye
```

Note `mut f` in `call_fnmut`: the *parameter* must be mutable, which is on the
callee, not the caller.

## Which closure gets which trait

The capture mode decides, not the `move` keyword.

| Closure | Implements |
|---|---|
| `\|\| println!("{list:?}")` (reads) | `Fn + FnMut + FnOnce` |
| `\|\| count += 1` (mutates) | `FnMut + FnOnce` |
| `move \|\| drop(v)` (moves out) | `FnOnce` only |
| `move \|\| n` where `n: i32` (`Copy`) | `Fn + FnMut + FnOnce` |

A `move` closure that only *reads* its owned captures is still `Fn`. `move`
affects where the data lives, not how the body touches it.

## FnOnce consumes its captures

```rust
let v = vec![1, 2, 3];
let consume = move || { let owned = v; owned.len() };
println!("{}", consume());
println!("{}", consume());
```

Does not compile:

```
error[E0382]: use of moved value: `consume`
 --> src/main.rs:5:20
  |
4 |     println!("{}", consume());
  |                    --------- `consume` moved due to this call
5 |     println!("{}", consume());
  |                    ^^^^^^^ value used here after move
  |
note: closure cannot be invoked more than once because it moves the variable `v` out of its environment
 --> src/main.rs:3:41
  |
3 |     let consume = move || { let owned = v; owned.len() };
  |                                         ^
note: this value implements `FnOnce`, which causes it to be moved when called
```

Calling an `FnOnce` *moves the closure*. That is the whole mechanism — nothing
is reference-counted or flagged at runtime.

## Picking too tight a bound

```rust
fn call_fn(f: impl Fn() -> i32) { println!("{}", f()); }

let mut n = 0;
call_fn(|| { n += 1; n });
```

Does not compile:

```
error[E0594]: cannot assign to `n`, as it is a captured variable in a `Fn` closure
 --> src/main.rs:4:18
  |
1 | fn call_fn(f: impl Fn() -> i32) { println!("{}", f()); }
  |               ---------------- change this to accept `FnMut` instead of `Fn`
...
4 |     call_fn(|| { n += 1; n });
  |     ------- --   ^^^^^^ cannot assign
  |     |       |
  |     |       in this closure
  |     expects `Fn` instead of `FnMut`
```

The compiler names the fix. Rule of thumb: call it once → `FnOnce`; call it
repeatedly → `FnMut`; need it callable from `&self` or shared across threads →
`Fn`.

## impl Fn vs Box<dyn Fn> vs &dyn Fn

All three accept a closure; they differ in dispatch and where it lives.

```rust
fn s(f: impl Fn(i32) -> i32) -> i32 { f(1) }
fn b(f: Box<dyn Fn(i32) -> i32>) -> i32 { f(2) }
fn r(f: &dyn Fn(i32) -> i32) -> i32 { f(3) }

fn main() {
    println!("{}", s(|x| x * 10));
    println!("{}", b(Box::new(|x| x * 10)));
    println!("{}", r(&|x| x * 10));
}
```

Output:

```
10
20
30
```

| Form | Dispatch | Allocates | Use when |
|---|---|---|---|
| `impl Fn(..)` / `<F: Fn(..)>` | static, inlinable | no | default for parameters |
| `&dyn Fn(..)` | dynamic | no | borrow a callback for the call's duration |
| `Box<dyn Fn(..)>` | dynamic | yes | store it, or return different closures |

`impl Fn` in argument position is shorthand for a generic parameter, so each
call site gets its own monomorphised copy.

## Returning closures: impl Fn

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn counter() -> impl FnMut() -> u32 {
    let mut n = 0;
    move || { n += 1; n }
}

fn main() {
    let add5 = make_adder(5);
    println!("{} {}", add5(1), add5(10));

    let mut c = counter();
    println!("{} {} {}", c(), c(), c());
}
```

Output:

```
6 15
1 2 3
```

`move` is mandatory in both: `n` is a local, and a borrow of it would dangle.
`counter` is a closure holding mutable state — a generator without a struct.

## Returning closures: Box<dyn Fn>

`impl Fn` returns one concrete type, so branches returning different closures
fail.

```rust
fn pick(n: i32, flag: bool) -> impl Fn(i32) -> i32 {
    if flag { move |x| x + n } else { move |x| x * n }
}
```

Does not compile:

```
error[E0308]: `if` and `else` have incompatible types
 --> src/main.rs:2:39
  |
2 |     if flag { move |x| x + n } else { move |x| x * n }
  |               --------------          ^^^^^^^^^^^^^^ expected closure, found a different closure
  |
  = note: no two closures, even if identical, have the same type
  = help: consider boxing your closure and/or using it as a trait object
help: you could change the return type to be a boxed trait object
  |
1 - fn pick(n: i32, flag: bool) -> impl Fn(i32) -> i32 {
1 + fn pick(n: i32, flag: bool) -> Box<dyn Fn(i32) -> i32> {
```

Take the suggestion:

```rust
fn pick(n: i32, flag: bool) -> Box<dyn Fn(i32) -> i32> {
    if flag { Box::new(move |x| x + n) } else { Box::new(move |x| x * n) }
}

println!("{} {}", pick(3, true)(10), pick(3, false)(10));
```

Output:

```
13 30
```

Non-capturing closures are the exception — they all coerce to the same `fn`
pointer type, so an `impl Fn` return with `if flag { |x| x + 1 } else { |x| x * 2 }`
does compile. Don't rely on it; box when branching.

## Function pointers

`fn(A) -> B` is a real type. A closure with **no** captures coerces to it.

```rust
fn takes_fn_ptr(f: fn(i32) -> i32) -> i32 { f(10) }
fn triple(x: i32) -> i32 { x * 3 }

println!("{} {}", takes_fn_ptr(triple), takes_fn_ptr(|x| x * 3));
```

Output:

```
30 30
```

Add a capture and the coercion is gone:

```rust
let k = 3;
println!("{}", takes_fn_ptr(|x| x * k));
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:4:33
  |
4 |     println!("{}", takes_fn_ptr(|x| x * k));
  |                    ------------ ^^^^^^^^^ expected fn pointer, found closure
  |
  = note: expected fn pointer `fn(i32) -> i32`
                found closure `{closure@src/main.rs:4:33: 4:36}`
note: closures can only be coerced to `fn` types if they do not capture any variables
 --> src/main.rs:4:41
  |
4 |     println!("{}", takes_fn_ptr(|x| x * k));
  |                                         ^ `k` captured here
```

Prefer `impl Fn` in your own APIs; `fn` pointers matter mainly for FFI and for
storing a callable in a `Copy` struct.

## A function name where a closure is expected

Any `fn` whose signature matches works as the argument to `map`, `filter`, etc.

```rust
let v = vec!["1", "2", "3"];
let up: Vec<String> = v.iter().copied().map(str::to_uppercase).collect();
let owned: Vec<String> = v.iter().map(|s| s.to_string()).collect();
println!("{up:?} {owned:?}");
```

Output:

```
["1", "2", "3"] ["1", "2", "3"]
```

`str::to_uppercase` is terser than `|s| s.to_uppercase()` when the shapes line
up exactly. They don't always — hence `.copied()` above, to turn `&&str` into
`&str`.

## Storing callbacks in a struct

Field type decides the trait. `Box<dyn Fn>` for read-only handlers,
`Box<dyn FnMut>` when a handler accumulates state.

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct EventBus {
    handlers: Vec<Box<dyn Fn(&str)>>,
}

impl EventBus {
    fn new() -> Self { EventBus { handlers: Vec::new() } }
    fn on(&mut self, f: impl Fn(&str) + 'static) { self.handlers.push(Box::new(f)); }
    fn emit(&self, msg: &str) { for h in &self.handlers { h(msg); } }
}

fn main() {
    let log = Rc::new(RefCell::new(Vec::new()));
    let l = Rc::clone(&log);

    let mut bus = EventBus::new();
    bus.on(|m| println!("print: {m}"));
    bus.on(move |m| l.borrow_mut().push(m.to_string()));

    bus.emit("save");
    bus.emit("quit");
    println!("{:?}", log.borrow());
}
```

Output:

```
print: save
print: quit
["save", "quit"]
```

Two details. `+ 'static` on `on` says the closure may not borrow anything
shorter-lived than the bus — required because it is stored. And `emit` takes
`&self`, so the handlers must be `Fn`, not `FnMut`; the second handler gets
around that with `Rc<RefCell<..>>` (Module 11).

## Generic field instead of boxing

If a struct holds exactly one closure and you don't need heterogeneity, a
generic parameter avoids the allocation and the vtable.

```rust
struct Retry<F> { attempts: u32, op: F }

impl<F: Fn(u32) -> bool> Retry<F> {
    fn run(&self) -> bool {
        (0..self.attempts).any(|i| (self.op)(i))
    }
}

let r = Retry { attempts: 4, op: |i| i == 2 };
println!("{}", r.run());
```

Output:

```
true
```

## Comparison table

| | `Fn` | `FnMut` | `FnOnce` | `fn` pointer |
|---|---|---|---|---|
| Receiver | `&self` | `&mut self` | `self` | — |
| Callable repeatedly | yes | yes | no | yes |
| May read captures | yes | yes | yes | no captures |
| May mutate captures | no | yes | yes | no |
| May move captures out | no | no | yes | — |
| Caller needs `mut` binding | no | yes | no | no |
| Implements the ones to its right | — | `FnOnce` | — | all three |
| Is `Copy`/`Clone` | if captures are | if captures are | if captures are | always |

## Quick Reference

| Need | Write |
|---|---|
| Parameter, called many times, read-only | `f: impl Fn(A) -> B` |
| Parameter, called many times, stateful | `mut f: impl FnMut(A) -> B` |
| Parameter, called once | `f: impl FnOnce(A) -> B` |
| Borrowed callback | `f: &dyn Fn(A) -> B` |
| Stored / returned, one shape | `-> impl Fn(A) -> B` |
| Stored / returned, several shapes | `Box<dyn Fn(A) -> B>` |
| Stored in a struct field | `Box<dyn Fn(A)>` or a generic `F` |
| Must outlive the struct | `impl Fn(A) + 'static` |
| Plain function value | `f: fn(A) -> B` |

## Common Mistakes

**Demanding `Fn` when `FnOnce` would do.** A tighter bound rejects valid
closures; take the loosest one your body needs.

**Forgetting `mut` on an `FnMut` parameter.** `fn go(f: impl FnMut())` cannot
call `f()`; write `mut f`.

**Returning `impl Fn` from a branching function.** Each closure has its own
type; return `Box<dyn Fn ...>`.

**Omitting `move` when returning a closure.** The captures are locals; without
`move` the closure borrows them and the borrow dangles.

**Expecting a capturing closure to be a `fn` pointer.** Only non-capturing
closures coerce; otherwise use `impl Fn` or `Box<dyn Fn>`.

## Exercises

1. Write `fn twice(f: impl Fn(i32) -> i32, x: i32) -> i32` returning
   `f(f(x))`. Then write the `FnMut` version and explain why it needs `mut`.
2. Write `fn apply_once(f: impl FnOnce() -> String) -> String`. Pass it a
   `move` closure that returns a captured `String`, and show the error from
   calling the closure twice yourself.
3. Write `fn make_multiplier(n: i32) -> impl Fn(i32) -> i32` and then
   `fn make_op(name: &str) -> Box<dyn Fn(i32) -> i32>` handling `"inc"`,
   `"double"`, and anything else as identity.
4. Build `struct Middleware { stack: Vec<Box<dyn Fn(String) -> String>> }` with
   a `run` that threads a string through every layer.
5. Write a function taking `fn(i32) -> i32`, call it with a named `fn` and with
   a non-capturing closure, then with a capturing one. Record the error.

---

[← 10.1 Closures and Capture Modes](01-closures.md) · [Course Index](../README.md) · [10.3 The Iterator Trait →](03-iterator-trait.md)
