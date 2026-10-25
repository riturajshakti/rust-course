[← 9.3 Trait Bounds and where Clauses](03-trait-bounds.md) · [Course Index](../README.md) · [9.5 Trait Objects and Dynamic Dispatch →](05-trait-objects.md)

# 9.4 — Default Methods and Supertraits

## What You'll Learn

- Default method bodies, overriding them, defaults calling required methods
- Supertraits (`trait Pet: Animal`) and calling supertrait methods
- Fully qualified syntax `<T as Trait>::method` for disambiguation
- Associated consts in traits, and traits with generic methods

## Coming From Another Language

| Other | Rust |
|---|---|
| TS abstract class with concrete methods | trait with default bodies |
| Go embedded interface | supertrait `trait A: B` |
| Python mixin | default methods + supertraits |

A default body gives you a mixin without inheritance. Note what Rust does *not*
have: a trait cannot have fields and cannot store state. Defaults can only call
the trait's own methods.

## Default method bodies

Give a body in the trait and implementors inherit it.

```rust
trait Greet {
    fn name(&self) -> String;
    fn greet(&self) -> String { format!("Hello, {}!", self.name()) }
}

struct En { who: String }

impl Greet for En {
    fn name(&self) -> String { self.who.clone() }
}

fn main() {
    println!("{}", En { who: "Raj".into() }.greet());
}
```

Output:

```
Hello, Raj!
```

`name` is required; `greet` is free. The `impl` block only lists what it must.

## Defaults calling required methods

This is the point: write the algorithm once against a minimal interface.

```rust
trait Greet {
    fn name(&self) -> String;
    fn greet(&self) -> String { format!("Hello, {}!", self.name()) }
    fn shout(&self) -> String { self.greet().to_uppercase() }
}
```

A default can call other defaults, so `shout` composes with whatever `greet`
each type ended up with.

## Overriding a default

```rust
struct En { who: String }
struct Fr { who: String }

impl Greet for En {
    fn name(&self) -> String { self.who.clone() }
}

impl Greet for Fr {
    fn name(&self) -> String { self.who.clone() }
    fn greet(&self) -> String { format!("Bonjour, {} !", self.who) }
}

let e = En { who: "Raj".into() };
let f = Fr { who: "Ana".into() };
println!("{}", e.greet());
println!("{}", f.greet());
println!("{}", e.shout());
println!("{}", f.shout());
```

Output:

```
Hello, Raj!
Bonjour, Ana !
HELLO, RAJ!
BONJOUR, ANA !
```

`shout` was never overridden, yet `f.shout()` picked up the French `greet` —
dispatch inside a default body goes through the implementing type.

There is no `super::greet()`. If an override needs the default's behaviour, move
the default into a free function and call it from both.

## Associated consts

A trait can require a constant, with or without a default.

```rust
trait Bounded {
    const MIN: i32;
    const MAX: i32;
    fn span() -> i32 { Self::MAX - Self::MIN }
    fn clamp_to(v: i32) -> i32 { v.max(Self::MIN).min(Self::MAX) }
}

struct Percent;
struct Signed;

impl Bounded for Percent { const MIN: i32 = 0; const MAX: i32 = 100; }
impl Bounded for Signed { const MIN: i32 = -50; const MAX: i32 = 50; }

println!("{} {}", Percent::span(), Signed::span());
println!("{} {}", Percent::clamp_to(150), Signed::clamp_to(-200));
println!("{} {}", Percent::MIN, Signed::MIN);
```

Output:

```
100 100
100 -50
0 -50
```

Default methods can read `Self::MIN` even though the trait does not know its
value. Consts resolve at compile time, so this is free.

A const with a default, used from a default method:

```rust
trait Greet {
    const TAG: &'static str = "greet";
    fn name(&self) -> String;
    fn greet(&self) -> String { format!("Hello, {}!", self.name()) }
    fn log(&self) { println!("[{}] {}", Self::TAG, self.greet()); }
}

impl Greet for Fr {
    fn name(&self) -> String { self.who.clone() }
    fn greet(&self) -> String { format!("Bonjour, {} !", self.who) }
    const TAG: &'static str = "fr";
}

e.log();
f.log();
println!("{} {}", En::TAG, Fr::TAG);
println!("{}", <Fr as Greet>::TAG);
```

Output:

```
[greet] Hello, Raj!
[fr] Bonjour, Ana !
greet fr
fr
```

## Supertraits

`trait Pet: Animal` means "to implement `Pet` you must also implement
`Animal`", and `Pet`'s methods may call `Animal`'s.

```rust
trait Animal {
    fn species(&self) -> &str;
}

trait Pet: Animal {
    fn pet_name(&self) -> &str;
    fn tag_line(&self) -> String {
        format!("{} the {}", self.pet_name(), self.species())
    }
}

struct Dog { n: String }

impl Animal for Dog {
    fn species(&self) -> &str { "dog" }
}

impl Pet for Dog {
    fn pet_name(&self) -> &str { &self.n }
}

let d = Dog { n: "Rex".into() };
println!("{}", d.tag_line());
```

Output:

```
Rex the dog
```

`self.species()` inside `tag_line` is the supertrait call — available because
the `Animal` bound is guaranteed. This is a bound, not inheritance: `Dog` needs
two separate `impl` blocks.

One `T: Pet` bound implies `Animal` too:

```rust
fn describe<T: Pet>(p: &T) { println!("{} ({})", p.tag_line(), p.species()); }

describe(&d);
```

Output:

```
Rex the dog (dog)
```

## Forgetting the supertrait impl

Intentionally broken:

```rust
trait Animal { fn species(&self) -> &str; }
trait Pet: Animal { fn pet_name(&self) -> &str; }

struct Cat;

impl Pet for Cat {
    fn pet_name(&self) -> &str { "Mia" }
}
```

```
error[E0277]: the trait bound `Cat: Animal` is not satisfied
 --> src/main.rs:4:14
  |
4 | impl Pet for Cat {
  |              ^^^ unsatisfied trait bound
  |
help: the trait `Animal` is not implemented for `Cat`
...
note: required by a bound in `Pet`
 --> src/main.rs:2:12
  |
2 | trait Pet: Animal { fn pet_name(&self) -> &str; }
```

Multiple supertraits: `trait Pet: Animal + Clone + Debug`.

## Two traits with the same method name

The method call is ambiguous. Intentionally broken:

```rust
trait Pilot { fn fly(&self) -> String; }
trait Wizard { fn fly(&self) -> String; }

struct Human;
impl Pilot for Human { fn fly(&self) -> String { "mayday".into() } }
impl Wizard for Human { fn fly(&self) -> String { "up!".into() } }

println!("{}", Human.fly());
```

```
error[E0034]: multiple applicable items in scope
 --> src/main.rs:7:26
  |
7 |     println!("{}", Human.fly());
  |                          ^^^ multiple `fly` found
  |
note: candidate #1 is defined in an impl of the trait `Pilot` for the type `Human`
note: candidate #2 is defined in an impl of the trait `Wizard` for the type `Human`
help: disambiguate the method for candidate #1
  |
7 +     println!("{}", Pilot::fly(&Human));
  |
help: disambiguate the method for candidate #2
  |
7 +     println!("{}", Wizard::fly(&Human));
  |
```

## Disambiguating with the trait name

`Trait::method(&value)` names which trait. An **inherent** method always wins
over any trait method, so `h.fly()` resolves without error when one exists:

```rust
impl Human {
    fn fly(&self) -> String { "arms waving".into() }
}

let h = Human;
println!("{}", h.fly());
println!("{}", Pilot::fly(&h));
println!("{}", Wizard::fly(&h));
println!("{}", Human::fly(&h));
```

Output:

```
arms waving
mayday
up!
arms waving
```

## Fully qualified syntax `<T as Trait>::method`

Required when there is no `self` to infer from — associated functions and
consts. Intentionally broken:

```rust
trait Pilot { fn label() -> String; }
trait Wizard { fn label() -> String; }

struct Human;
impl Pilot for Human { fn label() -> String { "pilot".into() } }
impl Wizard for Human { fn label() -> String { "wizard".into() } }

println!("{}", Human::label());
```

```
error[E0034]: multiple applicable items in scope
 --> src/main.rs:7:27
  |
7 |     println!("{}", Human::label());
  |                           ^^^^^ multiple `label` found
  |
help: use fully-qualified syntax to disambiguate
  |
7 +     println!("{}", <Human as Pilot>::label());
  |
7 +     println!("{}", <Human as Wizard>::label());
  |
```

The fix:

```rust
println!("{}", <Human as Pilot>::label());
println!("{}", <Human as Wizard>::label());
println!("{}", <Human as Pilot>::fly(&h));
```

Output:

```
pilot
wizard
mayday
```

`<Type as Trait>::item` is the fully general form — it works for methods,
associated functions, consts, and associated types (lesson 9.8). `Trait::m(&x)`
is the shorthand when `Self` can be inferred from the receiver.

## Traits with generic methods

A method can have its own type parameters, independent of the trait.

```rust
use std::fmt::{Debug, Display};

trait Store {
    fn put<T: Debug>(&self, key: &str, value: T);
    fn describe_all<T: Display>(&self, items: &[T]) -> String {
        items.iter().map(|i| i.to_string()).collect::<Vec<_>>().join(",")
    }
}

struct Mem;
impl Store for Mem {
    fn put<T: Debug>(&self, key: &str, value: T) { println!("{} = {:?}", key, value); }
}

let m = Mem;
m.put("a", 1);
m.put("b", vec!["x"]);
println!("{}", m.describe_all(&[1, 2, 3]));
```

Output:

```
a = 1
b = ["x"]
1,2,3
```

The `impl` must repeat the parameter and its bounds exactly. A generic method
makes the trait unusable as `dyn Trait` — there is no single vtable entry for an
unbounded family of types (lesson 9.5).

## Quick Reference

| Form | Syntax |
|---|---|
| Required method | `fn m(&self) -> R;` |
| Default method | `fn m(&self) -> R { ... }` |
| Override | restate `fn m` in the `impl` |
| Associated const | `const K: T;` or `const K: T = v;` |
| Read it in a default | `Self::K` |
| Supertrait | `trait Pet: Animal { ... }` |
| Several supertraits | `trait P: A + Clone + Debug` |
| Call supertrait method | `self.species()` inside `Pet` |
| Disambiguate method | `Trait::m(&value)` |
| Disambiguate anything | `<Type as Trait>::item` |
| Generic trait method | `fn put<T: Debug>(&self, v: T);` |

## Common Mistakes

**Expecting a trait to hold state.** Traits have no fields. Add a required
getter and let implementors store it.

**Looking for `super::method()` in an override.** There is none. Factor the
shared logic into a free function.

**Assuming a supertrait is inherited.** `impl Pet for Cat` does not supply
`Animal`; write both impls or you get `E0277`.

**Being surprised an inherent method shadows a trait method.** Inherent wins
silently. Use `Trait::m(&x)` when you meant the trait's.

**Adding a generic method or an associated const default that uses `Self`, then
trying `dyn Trait`.** Both break dyn compatibility (lesson 9.5).

## Exercises

1. Write `trait Logger` with required `fn prefix(&self) -> String` and defaults
   `info`, `warn`, `error` that all call `prefix`. Implement for two types, one
   overriding `warn`.
2. Add `const LEVEL: u8 = 1;` to `Logger` and a default `fn enabled(min: u8) -> bool`
   using `Self::LEVEL`. Override it in one impl.
3. Define `trait Named { fn name(&self) -> String; }` and
   `trait Employee: Named { fn id(&self) -> u32; fn badge(&self) -> String }`
   where `badge` uses both. Then try implementing only `Employee` and read the
   error.
4. Give one struct two traits with a `fn reset(&mut self)` each, and call both
   from `main` using disambiguation.
5. Write `trait Collector { fn collect_into<T: Clone>(&self, src: &[T]) -> Vec<T>; }`
   and implement it for a type that reverses the slice.

---

[← 9.3 Trait Bounds and where Clauses](03-trait-bounds.md) · [Course Index](../README.md) · [9.5 Trait Objects and Dynamic Dispatch →](05-trait-objects.md)
