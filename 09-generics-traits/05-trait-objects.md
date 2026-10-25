[← Course Index](../README.md)

# 9.5 — Trait Objects and Dynamic Dispatch

## What You'll Learn

- `dyn Trait`, `&dyn Trait`, `Box<dyn Trait>`, `Vec<Box<dyn Trait>>`
- Static vs dynamic dispatch, and what each costs
- Dyn compatibility (object safety) and the real `E0038`
- `dyn Trait + 'a` lifetimes, and downcasting with `Any`

## Coming From Another Language

| Other | Rust |
|---|---|
| TS variable typed as an interface | `&dyn Trait` / `Box<dyn Trait>` |
| Go `var x Shape` (interface value) | `Box<dyn Shape>` |
| Python a list of mixed objects | `Vec<Box<dyn Trait>>` |

A Go interface value *is* a trait object: a data pointer plus a method table.
The difference is that in Rust this is opt-in — generics are the default and
`dyn` is what you write when you need a heterogeneous collection or runtime
choice.

## The problem `dyn` solves

`Vec<T>` holds one type. These three widgets are three types.

```rust
trait Draw {
    fn draw(&self) -> String;
    fn area(&self) -> f64 { 0.0 }
}

struct Button { label: String }
struct Checkbox { on: bool }
struct Circle { r: f64 }

impl Draw for Button {
    fn draw(&self) -> String { format!("[{}]", self.label) }
}
impl Draw for Checkbox {
    fn draw(&self) -> String { if self.on { "[x]".into() } else { "[ ]".into() } }
}
impl Draw for Circle {
    fn draw(&self) -> String { format!("O r={}", self.r) }
    fn area(&self) -> f64 { 3.14159 * self.r * self.r }
}

fn main() {
    let widgets: Vec<Box<dyn Draw>> = vec![
        Box::new(Button { label: "OK".into() }),
        Box::new(Checkbox { on: false }),
        Box::new(Circle { r: 1.5 }),
    ];
    for w in &widgets {
        println!("{} area={:.2}", w.draw(), w.area());
    }
    println!("{}", widgets.len());
    println!("{:.2}", widgets.iter().map(|w| w.area()).sum::<f64>());
}
```

Output:

```
[OK] area=0.00
[ ] area=0.00
O r=1.5 area=7.07
3
7.07
```

`Box::new(...)` coerces to `Box<dyn Draw>` automatically. The annotation on the
`let` is what triggers it; without it the `vec!` would try to unify three types.

## Four ways to take a `Draw`

```rust
fn render_static<T: Draw>(d: &T) -> String { d.draw() }
fn render_impl(d: &impl Draw) -> String { d.draw() }
fn render_dyn(d: &dyn Draw) -> String { d.draw() }
fn render_boxed(d: Box<dyn Draw>) -> String { d.draw() }

let b = Button { label: "Save".into() };
println!("{}", render_static(&b));
println!("{}", render_dyn(&b));
println!("{}", render_impl(&b));
println!("{}", render_boxed(Box::new(b)));
```

Output:

```
[Save]
[Save]
[Save]
[Save]
```

The first two are monomorphized; the last two dispatch at runtime. Idiomatic
choice: `&dyn Trait` for a borrowed parameter, `Box<dyn Trait>` only when you
must own or store it.

## `&dyn Trait` in a collection

No allocation, but the referents must outlive the vec.

```rust
let btn = Button { label: "A".into() };
let cir = Circle { r: 1.0 };
let refs: Vec<&dyn Draw> = vec![&btn, &cir];
for r in &refs { println!("{}", r.draw()); }
```

Output:

```
[A]
O r=1
```

## Returning a runtime-chosen type

The classic reason to reach for `Box<dyn Trait>` — `impl Trait` cannot do this
(lesson 9.3).

```rust
fn make(kind: &str) -> Box<dyn Draw> {
    match kind {
        "button" => Box::new(Button { label: "OK".into() }),
        "circle" => Box::new(Circle { r: 2.0 }),
        _ => Box::new(Checkbox { on: true }),
    }
}

println!("{}", make("button").draw());
println!("{}", make("circle").draw());
println!("{}", make("other").draw());
```

Output:

```
[OK]
O r=2
[x]
```

## A trait object is a fat pointer

Two words: data pointer + vtable pointer.

```rust
println!("{} {}", std::mem::size_of::<&Button>(), std::mem::size_of::<&dyn Draw>());
println!("{} {}", std::mem::size_of::<Box<Button>>(), std::mem::size_of::<Box<dyn Draw>>());
```

Output:

```
8 16
8 16
```

`dyn Draw` itself has no compile-time size — it is `!Sized`, which is why it
only appears behind `&`, `Box`, `Rc`, or `Arc`.

## Static vs dynamic dispatch

| | Static (`<T: Trait>`, `impl Trait`) | Dynamic (`dyn Trait`) |
|---|---|---|
| Resolution | compile time | runtime vtable lookup |
| Call cost | direct call, inlinable | one indirect call, no inlining |
| Binary size | one copy per type used | one copy total |
| Compile time | grows with instantiations | flat |
| Pointer size | thin (1 word) | fat (2 words) |
| Heterogeneous collections | no | yes |
| Runtime-chosen return type | no | yes |
| Generic methods / `Self` returns | yes | no (not dyn compatible) |
| Allocation | none | usually `Box` |

The call overhead is a few nanoseconds; the real cost is the lost inlining.
Default to generics, switch to `dyn` when the shape of the problem requires it
or when monomorphization bloat becomes measurable.

## Dyn compatibility (object safety)

A trait can be a `dyn Trait` only if a vtable can be built for it. The rules
that bite:

- no method without a `self` receiver (associated functions)
- no method returning `Self`
- no method with its own generic type parameters
- the trait must not require `Sized`

A `self`-less method. Intentionally broken:

```rust
trait Container {
    fn get(&self) -> i32;
    fn make() -> Self;
}

fn main() {
    let _c: Box<dyn Container>;
}
```

```
error[E0038]: the trait `Container` is not dyn compatible
 --> src/main.rs:7:21
  |
7 |     let _c: Box<dyn Container>;
  |                     ^^^^^^^^^ `Container` is not dyn compatible
  |
note: for a trait to be dyn compatible it needs to allow building a vtable
      for more information, visit <https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>
 --> src/main.rs:3:8
  |
1 | trait Container {
  |       --------- this trait is not dyn compatible...
2 |     fn get(&self) -> i32;
3 |     fn make() -> Self;
  |        ^^^^ ...because associated function `make` has no `self` parameter
help: consider turning `make` into a method by giving it a `&self` argument
  |
3 |     fn make(&self) -> Self;
  |             +++++
help: alternatively, consider constraining `make` so it does not apply to trait objects
  |
3 |     fn make() -> Self where Self: Sized;
  |                       +++++++++++++++++
```

The second suggestion is the usual fix: `where Self: Sized` excludes that one
method from the vtable while keeping it callable on concrete types.

A `Self` return:

```rust
trait Cloneish { fn dup(&self) -> Self; }
```

```
error[E0038]: the trait `Cloneish` is not dyn compatible
  |
1 | trait Cloneish { fn dup(&self) -> Self; }
  |       --------                    ^^^^ ...because method `dup` references the `Self` type in its return type
```

A generic method:

```rust
trait Store { fn put<T: std::fmt::Debug>(&self, v: T); }
```

```
error[E0038]: the trait `Store` is not dyn compatible
  |
1 | trait Store { fn put<T: std::fmt::Debug>(&self, v: T); }
  |       -----      ^^^ ...because method `put` has generic type parameters
  |       |
  |       this trait is not dyn compatible...
  = help: consider moving `put` to another trait
```

A vtable has one slot per method; an unbounded family of `T` has no finite slot
count. `Clone` is not dyn compatible for the same reason — hence
`Box<dyn Draw>` cannot be cloned without a `fn box_clone(&self) -> Box<dyn Draw>`
helper method.

## `dyn Trait` and lifetimes

`Box<dyn Trait>` silently means `Box<dyn Trait + 'static>`. Intentionally
broken:

```rust
trait Draw { fn draw(&self) -> String; }

struct Label<'a> { text: &'a str }
impl<'a> Draw for Label<'a> {
    fn draw(&self) -> String { format!("<{}>", self.text) }
}

fn keep(d: Box<dyn Draw>) -> String { d.draw() }

fn main() {
    let s = String::from("tmp");
    println!("{}", keep(Box::new(Label { text: &s })));
}
```

```
error[E0597]: `s` does not live long enough
  --> src/main.rs:9:48
   |
 8 |     let s = String::from("tmp");
   |         - binding `s` declared here
 9 |     println!("{}", keep(Box::new(Label { text: &s })));
   |                         -----------------------^^---
   |                         |                      |
   |                         |                      borrowed value does not live long enough
   |                         coercion requires that `s` is borrowed for `'static`
   |
   = note: due to object lifetime defaults, `Box<dyn Draw>` actually means `Box<(dyn Draw + 'static)>`
```

Widen the bound to accept borrowed data:

```rust
struct Static;
impl Draw for Static { fn draw(&self) -> String { "static".into() } }

fn store_forever(d: Box<dyn Draw + 'static>) -> String { d.draw() }
fn store_short<'a>(d: Box<dyn Draw + 'a>) -> String { d.draw() }

let owned = String::from("owned");
println!("{}", store_short(Box::new(Label { text: &owned })));
println!("{}", store_forever(Box::new(Static)));

let s = String::from("tmp");
let b: Box<dyn Draw + '_> = Box::new(Label { text: &s });
println!("{}", b.draw());
```

Output:

```
<owned>
static
<tmp>
```

Auto traits stack the same way — `Box<dyn Draw + Send>` is what you need to move
a trait object across threads (Module 14).

```rust
fn send_it(d: Box<dyn Draw + Send>) -> String { d.draw() }
println!("{}", send_it(Box::new(Static)));
```

Output:

```
static
```

## Downcasting with `Any`

A trait object forgets its concrete type. `std::any::Any` gives it back, for
`'static` types only.

```rust
use std::any::Any;

trait Plugin: Any {
    fn run(&self) -> String;
    fn as_any(&self) -> &dyn Any;
}

struct Counter { n: u32 }
struct Echo { msg: String }

impl Plugin for Counter {
    fn run(&self) -> String { format!("count {}", self.n) }
    fn as_any(&self) -> &dyn Any { self }
}
impl Plugin for Echo {
    fn run(&self) -> String { format!("echo {}", self.msg) }
    fn as_any(&self) -> &dyn Any { self }
}

let plugins: Vec<Box<dyn Plugin>> = vec![
    Box::new(Counter { n: 7 }),
    Box::new(Echo { msg: "hi".into() }),
];
for p in &plugins {
    println!("{}", p.run());
    if let Some(c) = p.as_any().downcast_ref::<Counter>() {
        println!("  downcast Counter n={}", c.n);
    } else if let Some(e) = p.as_any().downcast_ref::<Echo>() {
        println!("  downcast Echo len={}", e.msg.len());
    }
    println!("  is Counter: {}", p.as_any().is::<Counter>());
}
```

Output:

```
count 7
  downcast Counter n=7
  is Counter: true
echo hi
  downcast Echo len=2
  is Counter: false
```

The `as_any` method is boilerplate you must write — `&dyn Plugin` does not
coerce to `&dyn Any` on its own.

Owned downcast returns a `Result` giving the box back on failure:

```rust
let boxed: Box<dyn Any> = Box::new(42i32);
match boxed.downcast::<i32>() {
    Ok(v) => println!("got i32 {}", v),
    Err(_) => println!("not an i32"),
}
let boxed2: Box<dyn Any> = Box::new("s");
println!("{:?}", boxed2.downcast::<i32>().is_err());
```

Output:

```
got i32 42
true
```

`downcast_mut` is the `&mut` variant. If you reach for `Any` often, an enum is
usually the better design — `match` on variants is exhaustive and free.

## Generics vs trait objects

Reach for **generics** when:

- the type is known at the call site
- the call is hot and should inline
- the trait has generic methods, `Self` returns, or associated types you need

Reach for **trait objects** when:

- one collection must hold several types
- the type is chosen at runtime (plugins, config, parsed input)
- the generic parameter would otherwise propagate through many signatures
- monomorphization bloat is measured and real

`Box<dyn Error>` in Module 08 is this pattern: any error type, one return type.

## Quick Reference

| Form | Meaning |
|---|---|
| `dyn Trait` | unsized trait object type |
| `&dyn Trait` | borrowed, 2 words, no allocation |
| `&mut dyn Trait` | mutable borrow |
| `Box<dyn Trait>` | owned, heap, implies `+ 'static` |
| `Rc<dyn Trait>` / `Arc<dyn Trait>` | shared ownership |
| `Vec<Box<dyn Trait>>` | heterogeneous collection |
| `Box<dyn Trait + 'a>` | may borrow data living `'a` |
| `Box<dyn Trait + Send + Sync>` | cross-thread trait object |
| `where Self: Sized` on a method | keeps a dyn-hostile method out of the vtable |
| `fn as_any(&self) -> &dyn Any` | enables `downcast_ref::<T>()` |
| `x.downcast_ref::<T>()` | `Option<&T>` |
| `b.downcast::<T>()` | `Result<Box<T>, Box<dyn Any>>` |
| `x.is::<T>()` | `bool` type test |

## Common Mistakes

**Writing `let x: dyn Trait = ...`.** `dyn Trait` is unsized; put it behind
`&`, `Box`, `Rc`, or `Arc`.

**Expecting `Box<dyn Trait>` to be cloneable.** `Clone` is not dyn compatible.
Add a `fn box_clone(&self) -> Box<dyn Trait>` method.

**Putting a `Self`-returning or generic method in a trait you then use as `dyn`.**
`E0038`. Add `where Self: Sized` or move the method to a separate trait.

**Forgetting that `Box<dyn Trait>` means `+ 'static`.** Write
`Box<dyn Trait + 'a>` when the object borrows.

**Using `dyn` everywhere out of habit.** Generics are the default in Rust;
`dyn` is for heterogeneity and runtime choice.

## Exercises

1. Define `trait Command { fn execute(&self) -> String; }`, implement it for
   three structs, and drive a `Vec<Box<dyn Command>>` in a loop.
2. Write `fn parse_command(s: &str) -> Box<dyn Command>` dispatching on a string,
   with a fallback.
3. Add `fn clone_box(&self) -> Box<dyn Command>` to the trait so you can clone a
   `Vec<Box<dyn Command>>`. First try `trait Command: Clone` and read `E0038`.
4. Add `fn describe() -> &'static str;` to `Command`, observe the dyn
   compatibility error, then fix it with `where Self: Sized`.
5. Add `as_any` and write a function that counts how many commands in a
   `Vec<Box<dyn Command>>` are a specific concrete type.

---

[← Course Index](../README.md)
