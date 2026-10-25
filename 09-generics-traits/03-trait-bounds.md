[← 9.2 Defining and Implementing Traits](02-traits.md) · [Course Index](../README.md) · [9.4 Default Methods and Supertraits →](04-default-supertraits.md)

# 9.3 — Trait Bounds and where Clauses

## What You'll Learn

- `<T: Trait>`, `+` for several bounds, and `where` clauses
- Bounds on `impl` blocks and conditional impls
- `impl Trait` in argument and return position, including closures
- Blanket impls, and the exact error when a bound is missing

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `<T extends Comparable>` | `<T: Ord>` |
| Go `[T constraints.Ordered]` | `<T: PartialOrd>` |
| Python `TypeVar(bound=...)` | `<T: Trait>` |

Same idea as Go type constraints. The difference: Rust checks the bound at the
**definition**, so a generic body only compiles if every operation it performs
is covered by a declared bound. There is no "it works for the types I tested".

## A single bound

```rust
use std::fmt::Display;

fn show<T: Display>(x: T) { println!("{}", x); }

fn main() {
    show(42);
    show("hi");
}
```

Output:

```
42
hi
```

## Multiple bounds with `+`

```rust
use std::fmt::{Debug, Display};

fn show_both<T: Display + Debug>(x: T) { println!("{} / {:?}", x, x); }

show_both(3.5);
```

Output:

```
3.5 / 3.5
```

Order of the `+` terms is irrelevant. A bound can be a trait, `Sized`, or a
lifetime (`T: 'static`).

## `where` clauses

Identical meaning, better readable when bounds pile up.

```rust
use std::collections::HashMap;
use std::hash::Hash;

fn tally<T>(items: &[T]) -> HashMap<T, usize>
where
    T: Eq + Hash + Clone,
{
    let mut m = HashMap::new();
    for i in items { *m.entry(i.clone()).or_insert(0) += 1; }
    m
}

let mut counts: Vec<_> = tally(&["a", "b", "a", "c", "a"]).into_iter().collect();
counts.sort();
println!("{:?}", counts);
```

Output:

```
[("a", 3), ("b", 1), ("c", 1)]
```

Idiomatic: inline `<T: Display>` for one short bound, `where` for two or more.

## Several bounded parameters

```rust
fn longest_name<T, U>(a: T, b: U) -> String
where
    T: Display,
    U: Display,
{
    let (a, b) = (a.to_string(), b.to_string());
    if a.len() >= b.len() { a } else { b }
}

println!("{}", longest_name("abc", 123456));
println!("{}", longest_name(1, 2));
```

Output:

```
123456
1
```

Each parameter has its own bounds; the types need not match.

## The error when a bound is not satisfied

Intentionally broken — `Vec` has no `Display`:

```rust
fn show<T: Display>(x: T) { println!("{}", x); }

show(vec![1, 2, 3]);
```

```
error[E0277]: `Vec<{integer}>` doesn't implement `std::fmt::Display`
 --> src/main.rs:4:10
  |
4 |     show(vec![1, 2, 3]);
  |     ---- ^^^^^^^^^^^^^ the trait `std::fmt::Display` is not implemented for `Vec<{integer}>`
  |     |
  |     required by a bound introduced by this call
  |
note: required by a bound in `show`
 --> src/main.rs:2:12
  |
2 | fn show<T: Display>(x: T) { println!("{}", x); }
  |            ^^^^^^^ required by this bound in `show`
```

`E0277` always names the type, the missing trait, and the bound that demanded
it. Read the `note:` line — it points at the signature to change.

## Bounds on impl blocks

A bounded `impl` block adds methods only for types meeting the bound.

```rust
struct Wrapper<T> { value: T }

impl<T> Wrapper<T> {
    fn new(value: T) -> Self { Wrapper { value } }
}

impl<T: Display> Wrapper<T> {
    fn print(&self) { println!("[{}]", self.value); }
}

impl<T: Display + PartialOrd> Wrapper<T> {
    fn cmp_print(&self, other: &T) {
        if self.value > *other { println!("{} > {}", self.value, other); }
        else { println!("{} <= {}", self.value, other); }
    }
}

let w = Wrapper::new(5);
w.print();
w.cmp_print(&3);

let n = Wrapper::new(vec![1, 2]);
println!("{:?}", n.value);
```

Output:

```
[5]
5 > 3
[1, 2]
```

`Wrapper<Vec<i32>>` exists and `new` works, but `print` does not:

```rust
let n = Wrapper { value: vec![1, 2] };
n.print();
```

```
error[E0599]: the method `print` exists for struct `Wrapper<Vec<{integer}>>`, but its trait bounds were not satisfied
  --> src/main.rs:8:7
   |
 8 |     n.print();
   |       ^^^^^ method cannot be called on `Wrapper<Vec<{integer}>>` due to unsatisfied trait bounds
   |
   = note: doesn't satisfy `Vec<{integer}>: std::fmt::Display`
note: trait bound `Vec<{integer}>: std::fmt::Display` was not satisfied
  --> src/main.rs:3:9
   |
 3 | impl<T: Display> Wrapper<T> {
   |         ^^^^^^^  ----------
```

Note the wording: "exists ... but its trait bounds were not satisfied", distinct
from the trait-not-imported `E0599` in lesson 9.2.

## Conditional impls

Same mechanism, implementing a trait instead of adding methods.

```rust
use std::fmt;

struct Wrapper<T> { value: T }

impl<T: fmt::Display> fmt::Display for Wrapper<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "<{}>", self.value)
    }
}

println!("{}", Wrapper { value: 7 });
println!("{}", Wrapper { value: "x" });
```

Output:

```
<7>
<x>
```

This is exactly how std works: `Vec<T>` is `Clone` only when `T: Clone`,
`Option<T>` is `Copy` only when `T: Copy`.

## Blanket impls

An `impl` over all `T` matching a bound. One block, every eligible type.

```rust
trait Loud { fn loud(&self) -> String; }

impl<T: Display> Loud for T {
    fn loud(&self) -> String { format!("{}!!!", self).to_uppercase() }
}

println!("{}", 42.loud());
println!("{}", "hey".loud());
```

Output:

```
42!!!
HEY!!!
```

Std's `impl<T: Display> ToString for T` is why `.to_string()` works on anything
printable. Caveat: a blanket impl claims the whole space, so you cannot then add
a manual `impl Loud for String` — that would be a conflicting implementation.

## `impl Trait` in argument position

Shorthand for an anonymous generic parameter.

```rust
fn notify(item: &impl Display) { println!("news: {}", item); }

fn combine(a: impl Display, b: impl Display) -> String { format!("{}-{}", a, b) }

notify(&"breaking");
println!("{}", combine(1, "two"));
```

Output:

```
news: breaking
1-two
```

`fn notify(item: &impl Display)` and `fn notify<T: Display>(item: &T)` compile
to the same thing. The difference: with `impl Trait` you have no name for the
type, so you cannot turbofish it or reuse it in a second argument. Use it when
the parameter appears once.

## `impl Trait` in return position

Says "some single concrete type implementing this", without naming it. Essential
for iterators and closures, which have unnameable types.

```rust
fn make_range(n: u32) -> impl Iterator<Item = u32> { (0..n).map(|x| x * x) }

println!("{:?}", make_range(5).collect::<Vec<_>>());
```

Output:

```
[0, 1, 4, 9, 16]
```

The caller sees only `Iterator<Item = u32>` — you can change the internals
freely. Still statically dispatched and zero-cost.

## Returning a closure

```rust
fn adder(n: i32) -> impl Fn(i32) -> i32 { move |x| x + n }

fn counter() -> impl FnMut() -> i32 {
    let mut c = 0;
    move || { c += 1; c }
}

let add5 = adder(5);
println!("{} {}", add5(1), add5(10));

let mut c = counter();
println!("{} {} {}", c(), c(), c());
```

Output:

```
6 15
1 2 3
```

`move` is needed so the closure owns `n` / `c` rather than borrowing locals that
are about to die. Module 10 covers `Fn` / `FnMut` / `FnOnce`.

## When `impl Trait` return won't do

`impl Trait` means **one** type, chosen at compile time. Two branches returning
two closures are two types. Intentionally broken:

```rust
fn pick(flag: bool, n: i32) -> impl Fn(i32) -> i32 {
    if flag { move |x| x + n } else { move |x| x * n }
}
```

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
1 - fn pick(flag: bool, n: i32) -> impl Fn(i32) -> i32 {
1 + fn pick(flag: bool, n: i32) -> Box<dyn Fn(i32) -> i32> {
```

Take the suggestion — `Box<dyn Trait>` erases the type at the cost of one
allocation and an indirect call.

```rust
fn boxed_op(kind: &str) -> Box<dyn Fn(i32) -> i32> {
    match kind {
        "double" => Box::new(|x| x * 2),
        _ => Box::new(|x| x + 1),
    }
}

println!("{} {}", boxed_op("double")(21), boxed_op("inc")(21));
```

Output:

```
42 22
```

Lesson 9.5 covers `dyn Trait` in full.

## Quick Reference

| Form | Syntax |
|---|---|
| One bound | `fn f<T: Display>(x: T)` |
| Several bounds | `fn f<T: Display + Clone>(x: T)` |
| `where` clause | `fn f<T>(x: T) where T: Display` |
| Lifetime bound | `fn f<T: 'static>(x: T)` |
| Bounded impl | `impl<T: Display> Wrapper<T> { ... }` |
| Conditional impl | `impl<T: Display> Display for Wrapper<T>` |
| Blanket impl | `impl<T: Display> MyTrait for T` |
| Argument position | `fn f(x: impl Display)` |
| Return position | `fn f() -> impl Iterator<Item = u32>` |
| Return a closure | `-> impl Fn(i32) -> i32` with `move` |
| Erase the type | `-> Box<dyn Fn(i32) -> i32>` |

## Common Mistakes

**Adding the bound at the call site instead of the signature.** The `E0277`
`note:` line points at the definition; that is the line to change.

**Using `impl Trait` in argument position and then needing the type name.** You
cannot turbofish it or tie two parameters to it. Switch to `<T: Trait>`.

**Returning `impl Trait` from a function with two branches.** Two closures are
two types. Use `Box<dyn Trait>`.

**Returning a closure without `move`.** It borrows a local that dies at the end
of the function; `E0373`.

**Writing a blanket impl then a specific one.** The blanket already covers that
type — `E0119` conflicting implementations.

## Exercises

1. Write `fn print_all<T: Display>(items: &[T])`. Call it with `&[1, 2, 3]`,
   then with `&[vec![1]]` and read the error.
2. Convert `fn pair<T: Display + Clone, U: Debug + Default>(a: T, b: U)` to use
   a `where` clause.
3. Define `struct Cache<K, V> { map: HashMap<K, V> }` with `insert`/`get`, with
   the bounds on the `impl` block, not the struct.
4. Write a blanket `impl<T: Debug> Loggable for T` with `fn log(&self)`, then try
   adding `impl Loggable for String` and read `E0119`.
5. Write `fn evens(limit: u32) -> impl Iterator<Item = u32>`, then rewrite it to
   return `Box<dyn Iterator<Item = u32>>` and switch between two different
   iterators based on a `bool` argument.

---

[← 9.2 Defining and Implementing Traits](02-traits.md) · [Course Index](../README.md) · [9.4 Default Methods and Supertraits →](04-default-supertraits.md)
