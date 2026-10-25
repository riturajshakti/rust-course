[← 9.7 From, Into, TryFrom, and Conversions](07-conversions.md) · [Course Index](../README.md) · [10.1 Closures and Capture Modes →](../10-closures-iterators/01-closures.md)

# 9.8 — Associated Types and Generic Traits

## What You'll Learn

- Associated types (`type Item;`) vs generic trait parameters (`trait T<U>`)
- Why `Iterator` uses `Item` and `From` uses `<T>`
- Associated type bounds, `C::Item`, and `Trait<Item = T>` constraints
- Generic associated types, briefly

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `interface C<T>` only | both `trait C<T>` and `type Item` |
| Go interface with type param | generic trait `trait C<T>` |
| Python `Iterator[T]` | `Iterator` with `type Item = T` |

Rust gives you two ways to put a type in a trait, and the choice is not
cosmetic. The deciding question: **can one type implement this trait more than
once?** If yes, use a generic parameter. If no, use an associated type.

## The motivating example: Iterator

`Iterator` has one associated type, not a parameter.

```rust
struct Counter { n: u32 }

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.n >= 3 { None } else { self.n += 1; Some(self.n) }
    }
}

fn main() {
    println!("{:?}", Counter { n: 0 }.collect::<Vec<u32>>());
    println!("{:?}", Counter { n: 0 }.map(|x| x * 10).collect::<Vec<_>>());
}
```

Output:

```
[1, 2, 3]
[10, 20, 30]
```

A `Counter` yields `u32`s and nothing else — iterating it twice as two different
types is meaningless. With `type Item`, the signature is `fn next(&mut self) ->
Option<Self::Item>` and callers never annotate. Had it been `Iterator<T>`, every
use would need `Iterator<u32>` and `.collect()` could not infer.

## Defining a trait with an associated type

```rust
use std::fmt::Debug;

trait Container {
    type Item;
    fn get(&self, i: usize) -> Option<&Self::Item>;
    fn first(&self) -> Option<&Self::Item> { self.get(0) }
    fn len(&self) -> usize;
}

struct Ints(Vec<i32>);
struct Names(Vec<String>);

impl Container for Ints {
    type Item = i32;
    fn get(&self, i: usize) -> Option<&i32> { self.0.get(i) }
    fn len(&self) -> usize { self.0.len() }
}

impl Container for Names {
    type Item = String;
    fn get(&self, i: usize) -> Option<&String> { self.0.get(i) }
    fn len(&self) -> usize { self.0.len() }
}

let ints = Ints(vec![10, 20, 30]);
let names = Names(vec!["raj".into(), "ana".into()]);
println!("{:?} {:?}", ints.get(5), names.first());
```

Output:

```
None Some("raj")
```

`Self::Item` inside the trait; in the impl you may write the concrete type
directly. Note the default method `first` works for both without knowing the
item type.

Forgetting the `type` line is `E0046`:

```rust
impl Container for A { fn get(&self) -> i32 { 1 } }
```

```
error[E0046]: not all trait items implemented, missing: `Item`
 --> src/main.rs:3:1
  |
1 | trait Container { type Item; fn get(&self) -> Self::Item; }
  |                   --------- `Item` from trait
3 | impl Container for A { fn get(&self) -> i32 { 1 } }
  | ^^^^^^^^^^^^^^^^^^^^ missing `Item` in implementation
```

## Referring to the associated type in a bound

Write `C::Item` to constrain it.

```rust
fn describe<C: Container>(c: &C) -> String
where
    C::Item: Debug,
{
    format!("{} items, first={:?}", c.len(), c.first())
}

println!("{}", describe(&ints));
println!("{}", describe(&names));
```

Output:

```
3 items, first=Some(10)
2 items, first=Some("raj")
```

`C::Item: Debug` is an **associated type bound** — it says nothing about what
`Item` is, only what it can do. The long form is `<C as Container>::Item`.

## Pinning the associated type

`Container<Item = i32>` means "any container whose items are `i32`".

```rust
fn sum_ints<C: Container<Item = i32>>(c: &C) -> i32 {
    (0..c.len()).filter_map(|i| c.get(i)).sum()
}

println!("{}", sum_ints(&ints));
```

Output:

```
60
```

Calling `sum_ints(&names)` is rejected:

```
error[E0271]: type mismatch resolving `<Names as Container>::Item == i32`
   |
16 |     println!("{}", sum_ints(&Names(vec![])));
   |                    -------- ^^^^^^^^^^^^^^ type mismatch resolving `<Names as Container>::Item == i32`
   |
note: expected this to be `i32`
   |
 8 |     type Item = String;
   |                 ^^^^^^
```

You see this syntax constantly in iterator signatures:
`impl Iterator<Item = u32>`, `T: IntoIterator<Item = String>`.

A `dyn` trait object must pin it too:

```rust
let _x: Box<dyn Container>;
```

```
error[E0191]: the value of the associated type `Item` in `Container` must be specified
 --> src/main.rs:2:29
  |
1 | trait Container { type Item; fn len(&self) -> usize; }
  |                   --------- `Item` defined here
2 | fn main() { let _x: Box<dyn Container>; }
  |                             ^^^^^^^^^
  |
help: specify the associated type
  |
2 | fn main() { let _x: Box<dyn Container<Item = /* Type */>>; }
  |                                      +++++++++++++++++++
```

## One type can implement it only once

Intentionally broken:

```rust
trait Container {
    type Item;
    fn get(&self) -> Self::Item;
}

struct Both;
impl Container for Both { type Item = i32; fn get(&self) -> i32 { 1 } }
impl Container for Both { type Item = String; fn get(&self) -> String { "x".into() } }
```

```
error[E0119]: conflicting implementations of trait `Container` for type `Both`
 --> src/main.rs:7:1
  |
6 | impl Container for Both { type Item = i32; fn get(&self) -> i32 { 1 } }
  | ----------------------- first implementation here
7 | impl Container for Both { type Item = String; fn get(&self) -> String { "x".into() } }
  | ^^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `Both`
```

`impl Container for Both` names the full impl — the associated type is an
*output*, not part of the identity. This is the entire trade.

## Generic traits: many impls per type

A trait with a type parameter can be implemented once per parameter value. This
is what `From<T>` does (lesson 9.7).

```rust
trait Adder<Rhs> {
    type Sum;
    fn add_to(&self, rhs: Rhs) -> Self::Sum;
}

#[derive(Debug)]
struct Cents(i64);

impl Adder<Cents> for Cents {
    type Sum = Cents;
    fn add_to(&self, rhs: Cents) -> Cents { Cents(self.0 + rhs.0) }
}
impl Adder<i64> for Cents {
    type Sum = Cents;
    fn add_to(&self, rhs: i64) -> Cents { Cents(self.0 + rhs) }
}
impl Adder<&str> for Cents {
    type Sum = String;
    fn add_to(&self, rhs: &str) -> String { format!("{}{}", self.0, rhs) }
}

println!("{:?}", Cents(100).add_to(Cents(50)));
println!("{:?}", Cents(100).add_to(5i64));
println!("{:?}", Cents(100).add_to(" cents"));
```

Output:

```
Cents(150)
Cents(105)
"100 cents"
```

`Rhs` is an input, so the three impls are distinct. `Sum` is an output, fixed
once `Rhs` is chosen — hence `Adder<&str>` can produce a `String` while
`Adder<i64>` produces `Cents`. `std::ops::Add` has exactly this shape:
`Add<Rhs = Self>` with `type Output`.

## The cost of the generic parameter

The parameter has to be inferred or named at every call. Intentionally broken:

```rust
trait Conv<T> { fn conv(&self) -> T; }

struct S;
impl Conv<i32> for S { fn conv(&self) -> i32 { 1 } }
impl Conv<String> for S { fn conv(&self) -> String { "x".into() } }

println!("{:?}", S.conv());
```

```
error[E0283]: type annotations needed
 --> src/main.rs:6:24
  |
6 |     println!("{:?}", S.conv());
  |                        ^^^^
  |
note: multiple `impl`s satisfying `S: Conv<_>` found
 --> src/main.rs:3:1
  |
3 | impl Conv<i32> for S { fn conv(&self) -> i32 { 1 } }
  | ^^^^^^^^^^^^^^^^^^^^
4 | impl Conv<String> for S { fn conv(&self) -> String { "x".into() } }
  | ^^^^^^^^^^^^^^^^^^^^^^^
help: try using a fully qualified path to specify the expected types
  |
6 +     println!("{:?}", <S as Conv<T>>::conv(&S));
  |
```

Disambiguate with the qualified path or an annotation:

```rust
println!("{:?}", <S as Conv<i32>>::conv(&S));
println!("{:?}", <S as Conv<String>>::conv(&S));
let n: i32 = S.conv();
println!("{}", n);
```

Output:

```
1
"x"
1
```

Had `Conv` used an associated type, `S.conv()` would resolve with no annotation
— but `S` could then convert to only one type.

## Several associated types

A trait can have as many as it needs, and they can carry bounds in the
declaration.

```rust
use std::collections::HashMap;

trait Pipeline {
    type In;
    type Out;
    fn run(&self, input: Self::In) -> Self::Out;
}

struct Lengths;
impl Pipeline for Lengths {
    type In = Vec<String>;
    type Out = Vec<usize>;
    fn run(&self, input: Vec<String>) -> Vec<usize> {
        input.iter().map(|s| s.len()).collect()
    }
}

struct Tally;
impl Pipeline for Tally {
    type In = Vec<char>;
    type Out = HashMap<char, usize>;
    fn run(&self, input: Vec<char>) -> HashMap<char, usize> {
        let mut m = HashMap::new();
        for c in input { *m.entry(c).or_insert(0) += 1; }
        m
    }
}

fn run_both<P: Pipeline>(p: &P, i: P::In) -> P::Out { p.run(i) }

println!("{:?}", run_both(&Lengths, vec!["a".into(), "bcd".into()]));
let mut t: Vec<_> = run_both(&Tally, vec!['a', 'b', 'a']).into_iter().collect();
t.sort();
println!("{:?}", t);
```

Output:

```
[1, 3]
[('a', 2), ('b', 1)]
```

`P::In` and `P::Out` in a signature — no type parameters at the call site at
all. Doing this with generics would mean `trait Pipeline<In, Out>` and three
parameters in every signature that touches it.

Bounds declared on the associated type itself apply in every impl and are
available in every generic use:

```rust
trait Graph {
    type Node: Debug + Clone;
    type Edge: Debug;
    fn nodes(&self) -> Vec<Self::Node>;
    fn edges(&self) -> Vec<Self::Edge>;
}

struct Flat;
impl Graph for Flat {
    type Node = u32;
    type Edge = (u32, u32);
    fn nodes(&self) -> Vec<u32> { vec![1, 2, 3] }
    fn edges(&self) -> Vec<(u32, u32)> { vec![(1, 2), (2, 3)] }
}

fn summarize<G: Graph>(g: &G) { println!("{:?} / {:?}", g.nodes(), g.edges()); }

summarize(&Flat);
```

Output:

```
[1, 2, 3] / [(1, 2), (2, 3)]
```

`summarize` can `{:?}` both without a `where` clause — the bounds came with the
trait.

## Associated types in std

`IntoIterator` has two: the item and the iterator type.

```rust
struct Bag { items: Vec<String> }

impl IntoIterator for Bag {
    type Item = String;
    type IntoIter = std::vec::IntoIter<String>;
    fn into_iter(self) -> Self::IntoIter { self.items.into_iter() }
}
```

Others worth knowing: `Add::Output`, `Index::Output`, `Deref::Target`,
`TryFrom::Error`, `FromStr::Err`, `ToOwned::Owned`.

Name one explicitly with `<Type as Trait>::Assoc`:

```rust
let v: Vec<i32> = Vec::new();
println!("{:?}", <Vec<i32> as IntoIterator>::into_iter(v).next());
println!("{:?}", <Ints as Container>::Item::default());
```

Output:

```
None
0
```

## Generic associated types

An associated type can take its own lifetime or type parameter. This is what
lets a trait return a borrow of `self` whose lifetime is chosen per call.

```rust
trait Lender {
    type Borrowed<'a> where Self: 'a;
    fn lend(&self) -> Self::Borrowed<'_>;
}

struct Holder { data: Vec<i32> }

impl Lender for Holder {
    type Borrowed<'a> = &'a [i32];
    fn lend(&self) -> &[i32] { &self.data }
}

let h = Holder { data: vec![1, 2, 3] };
println!("{:?}", h.lend());
```

Output:

```
[1, 2, 3]
```

Without the `<'a>` you could not name a type borrowing from `self` — a plain
`type Borrowed;` needs a lifetime from somewhere, and the trait has none. GATs
show up in lending iterators and in async trait desugaring (lesson 15.7). You
rarely write them; you do read them in error messages.

## Associated type vs generic parameter

| | Associated type `type Item;` | Generic parameter `trait T<U>` |
|---|---|---|
| Impls per type | exactly one | one per parameter value |
| Role of the type | output, determined by `Self` | input, chosen by the caller |
| At the call site | inferred, nothing to write | must be inferable or named |
| Refer to it | `C::Item`, `<C as T>::Item` | `U` is already in scope |
| Constrain it | `C: T<Item = i32>` or `C::Item: Debug` | `U: Debug` |
| Signature noise | low — one parameter `C` | grows with each extra type |
| Ambiguity risk | none | `E0283` with multiple impls |
| std examples | `Iterator::Item`, `Deref::Target`, `Add::Output` | `From<T>`, `TryFrom<T>`, `AsRef<T>`, `PartialEq<Rhs>` |
| Dyn compatible | yes, with `dyn T<Item = X>` | yes, with `dyn T<U>` |

Deciding rule: if the type is a **consequence** of `Self`, make it associated.
If the caller should **pick** it, make it a parameter. Many std traits use both
at once — `Add<Rhs>` takes the right-hand type as a parameter and reports the
result as `Output`.

## Quick Reference

| Form | Syntax |
|---|---|
| Declare | `trait C { type Item; }` |
| Declare with bounds | `type Item: Debug + Clone;` |
| Use inside the trait | `Self::Item` |
| Supply in an impl | `impl C for S { type Item = i32; ... }` |
| Bound it generically | `where C::Item: Debug` |
| Pin it | `C: Container<Item = i32>` |
| Fully qualified | `<S as Container>::Item` |
| Generic trait | `trait Adder<Rhs> { ... }` |
| Many impls | `impl Adder<i64> for Cents`, `impl Adder<&str> for Cents` |
| Disambiguate a call | `<S as Conv<i32>>::conv(&S)` |
| Several associated types | `type In; type Out;` |
| GAT | `type Borrowed<'a> where Self: 'a;` |

## Common Mistakes

**Omitting the `type` line in an impl.** `E0046`, missing `Item` — the compiler
lists it by name.

**Implementing a trait twice for one type to vary the associated type.**
`E0119`. The associated type is not part of the impl's identity; switch to a
generic parameter.

**Using a generic parameter for something determined by `Self`.** Every call
site then needs an annotation, and multiple impls give `E0283`.

**Writing `C: Container, C::Item: Debug` when you meant `Container<Item = i32>`.**
The first says "items are printable", the second says "items *are* `i32`".

**Expecting `dyn Trait` to work without pinning the associated type.** A trait
object needs `dyn Container<Item = i32>` — a vtable cannot leave `Item` open.

## Exercises

1. Define `trait Stack { type Elem; fn push(&mut self, e: Self::Elem); fn pop(&mut self) -> Option<Self::Elem>; }`
   and implement it for a wrapper over `Vec<i32>` and one over `Vec<String>`.
2. Write `fn drain_all<S: Stack>(s: &mut S) -> Vec<S::Elem>` and
   `fn total<S: Stack<Elem = i32>>(s: &mut S) -> i32`.
3. Add a second impl of `Stack` for your `Vec<i32>` wrapper with a different
   `Elem`. Read `E0119`, then convert the trait to `trait Stack<E>` and make both
   impls compile.
4. Implement `Iterator` for `struct Chunks { data: Vec<i32>, size: usize }` with
   `type Item = Vec<i32>`, and `.collect()` the results.
5. Define `trait Codec { type Encoded; type Error; fn encode(&self, s: &str) -> Self::Encoded; fn decode(&self, e: Self::Encoded) -> Result<String, Self::Error>; }`
   and implement it for a type that encodes to `Vec<u8>`.

---

[← 9.7 From, Into, TryFrom, and Conversions](07-conversions.md) · [Course Index](../README.md) · [10.1 Closures and Capture Modes →](../10-closures-iterators/01-closures.md)
