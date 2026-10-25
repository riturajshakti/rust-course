[← 5.5 Option: No More Null](05-option.md) · [Course Index](../README.md) · [5.7 Builder Pattern and Default →](07-builder-default.md)

# 5.6 — Deriving Common Traits

## What You'll Learn

- What `#[derive(...)]` actually generates, and its one hard rule
- Each of the nine standard derives: what it enables and what it requires
- `Eq` vs `PartialEq`, floats, and `Ord`'s lexicographic field order
- When to write the impl by hand instead

## Coming From Another Language

| Other | Rust |
|---|---|
| Python `@dataclass(eq=True, order=True)` | `#[derive(PartialEq, Eq, PartialOrd, Ord)]` |
| Go: write `String()`, `Equal()` by hand | derive `Debug`, `PartialEq` |
| JS: objects compare by reference | `==` is `PartialEq`, structural and opt-in |

Rust has no inherited behaviour at all. A struct can be printed, compared,
copied, or hashed only if that trait is implemented — and `derive` writes the
obvious implementation for you.

## What derive does

`#[derive(Trait)]` is a macro that generates an `impl` block by recursing into
the fields. This:

```rust
#[derive(PartialEq)]
struct Point { x: i32, y: i32 }
```

expands to roughly:

```rust
impl PartialEq for Point {
    fn eq(&self, other: &Point) -> bool {
        self.x == other.x && self.y == other.y
    }
}
```

**The hard rule: every field must already implement the trait.** Derive is
field-wise, with no exceptions.

```rust
struct Inner;

#[derive(Debug)]
struct Outer { inner: Inner, n: u32 }
```

Does not compile:

```
error[E0277]: `Inner` doesn't implement `Debug`
 --> src/main.rs:4:16
  |
3 | #[derive(Debug)]
  |          ----- in this derive macro expansion
4 | struct Outer { inner: Inner, n: u32 }
  |                ^^^^^^^^^^^^ the trait `Debug` is not implemented for `Inner`
  |
help: consider annotating `Inner` with `#[derive(Debug)]`
  |
1 + #[derive(Debug)]
2 | struct Inner;
  |
```

## The nine standard derives

| Derive | Enables | Requires | Field constraint |
|---|---|---|---|
| `Debug` | `{:?}`, `{:#?}` | — | all fields `Debug` |
| `Clone` | `.clone()` | — | all fields `Clone` |
| `Copy` | implicit bitwise copy instead of move | `Clone` | all fields `Copy` (no `String`, `Vec`, `Box`) |
| `PartialEq` | `==`, `!=` | — | all fields `PartialEq` |
| `Eq` | `HashMap` keys, `BTreeMap` keys, `sort` | `PartialEq` | all fields `Eq` (no `f32`/`f64`) |
| `PartialOrd` | `<`, `>`, `<=`, `>=`, `partial_cmp` | `PartialEq` | all fields `PartialOrd` |
| `Ord` | `.sort()`, `.max()`, `BTreeMap` | `Eq` + `PartialOrd` | all fields `Ord` |
| `Hash` | `HashMap`/`HashSet` keys | pair with `Eq` | all fields `Hash` (no `f32`/`f64`) |
| `Default` | `T::default()`, `..Default::default()` | — | all fields `Default` |

The stacking requirements are not optional: `Ord` without `Eq` and `PartialOrd`
does not compile. The usual full set is
`#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]`.

## All nine at once

```rust
use std::collections::{HashMap, HashSet};

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
struct Version { major: u32, minor: u32, patch: u32 }

fn main() {
    let v = Version { major: 1, minor: 99, patch: 1 };
    println!("{:?}", v);
    let w = v;
    println!("{:?} {:?}", v, w);
    println!("{}", v == w);
    println!("{}", v < Version { major: 2, minor: 0, patch: 0 });
    println!("{:?}", Version::default());

    let mut set = HashSet::new();
    set.insert(v);
    set.insert(w);
    println!("{}", set.len());

    let mut counts: HashMap<Version, u32> = HashMap::new();
    *counts.entry(v).or_insert(0) += 1;
    println!("{:?}", counts);
}
```

Output:

```
Version { major: 1, minor: 99, patch: 1 }
Version { major: 1, minor: 99, patch: 1 } Version { major: 1, minor: 99, patch: 1 }
true
true
Version { major: 0, minor: 0, patch: 0 }
1
{Version { major: 1, minor: 99, patch: 1 }: 1}
```

`let w = v` copies instead of moving, so `v` is still usable — that is `Copy`.
`set.len()` is 1 because the two are equal and hash alike.

## Copy needs every field to be Copy

```rust
#[derive(Clone, Copy)]
struct Tag { name: String }
```

Does not compile:

```
error[E0204]: the trait `Copy` cannot be implemented for this type
 --> src/main.rs:2:8
  |
1 | #[derive(Clone, Copy)]
  |                 ---- in this derive macro expansion
2 | struct Tag { name: String }
  |        ^^^   ------------ this field does not implement `Copy`
```

Derive `Copy` only on small all-`Copy` types: coordinates, IDs, flags, fieldless
enums. Anything holding a `String`, `Vec`, or `Box` gets `Clone` alone, and you
clone explicitly.

## Eq vs PartialEq, and floats

`PartialEq` means "comparison may be undefined for some values". `Eq` is the
extra promise that `a == a` is always true. Floats break that promise because
`NaN != NaN`:

```rust
let n = f64::NAN;
println!("{}", n == n);
println!("{:?}", 1.0_f64.partial_cmp(&2.0));
println!("{:?}", n.partial_cmp(&1.0));
```

Output:

```
false
Some(Less)
None
```

So `f64` implements `PartialEq` and `PartialOrd` but **not** `Eq` or `Ord`.
Derive `Eq` on a struct with a float field and the derive fails:

```rust
#[derive(PartialEq, Eq)]
struct Point { x: f64, y: f64 }
```

Does not compile:

```
error[E0277]: the trait bound `f64: Eq` is not satisfied
 --> src/main.rs:2:16
  |
1 | #[derive(PartialEq, Eq)]
  |                     -- in this derive macro expansion
2 | struct Point { x: f64, y: f64 }
  |                ^^^^^^ the trait `Eq` is not implemented for `f64`
  |
note: required by a bound in `std::cmp::AssertParamIsEq`
```

The practical consequence: a float-bearing struct cannot be a `HashMap` key and
cannot use `.sort()`. Sort with `sort_by` and `partial_cmp`, or `total_cmp`:

```rust
let mut fs = vec![3.0_f64, 1.0, 2.0];
fs.sort_by(|a, b| a.partial_cmp(b).unwrap());
println!("{:?}", fs);

let mut v = vec![3.0_f64, 1.0, 2.0];
v.sort_by(f64::total_cmp);
println!("{:?}", v);
```

Output:

```
[1.0, 2.0, 3.0]
[1.0, 2.0, 3.0]
```

`total_cmp` never panics; `partial_cmp(...).unwrap()` panics on a `NaN`.

## Ord compares fields in declaration order

Derived `Ord` is lexicographic: compare field one, and only on a tie move to
field two. **Field order in the struct is therefore the sort order.**

```rust
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
struct Employee { dept: String, age: u32, name: String }

let mut staff = vec![
    Employee { dept: "eng".into(), age: 40, name: "zoe".into() },
    Employee { dept: "eng".into(), age: 30, name: "amy".into() },
    Employee { dept: "art".into(), age: 50, name: "bob".into() },
];
staff.sort();
for e in &staff { println!("{} {} {}", e.dept, e.age, e.name); }
println!("{:?}", staff.iter().max());
```

Output:

```
art 50 bob
eng 30 amy
eng 40 zoe
Some(Employee { dept: "eng", age: 40, name: "zoe" })
```

Sorted by `dept`, then `age`, then `name`. Reorder the fields and the sort order
silently changes — a real source of bugs. If the sort order matters, write `Ord`
by hand rather than depending on field layout.

## Derives on enums

Same rules. Derived `Ord` on an enum orders by **variant declaration order**,
then by payload.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
enum Priority { Low, Medium, High }

let mut ps = vec![Priority::High, Priority::Low, Priority::Medium];
ps.sort();
println!("{:?}", ps);
println!("{} {}", Priority::High > Priority::Low, Priority::Low == Priority::Low);
println!("{:?}", ps.iter().max());
```

Output:

```
[Low, Medium, High]
true true
Some(High)
```

Declare the variants in ascending severity and `>` means what you want for free.

`#[derive(Default)]` on an enum needs `#[default]` on one variant — lesson 5.7.

## Hash is required for HashMap keys

```rust
use std::collections::HashMap;

#[derive(Debug)]
struct Key { a: u32 }

let mut m: HashMap<Key, u32> = HashMap::new();
m.insert(Key { a: 1 }, 10);
```

Does not compile:

```
error[E0599]: the method `insert` exists for struct `HashMap<Key, u32>`, but its trait bounds were not satisfied
 --> src/main.rs:6:7
  |
3 | struct Key { a: u32 }
  | ---------- doesn't satisfy `Key: Eq` or `Key: Hash`
...
6 |     m.insert(Key { a: 1 }, 10);
  |       ^^^^^^
  |
  = note: the following trait bounds were not satisfied:
          `Key: Eq`
          `Key: Hash`
help: consider annotating `Key` with `#[derive(Eq, Hash, PartialEq)]`
```

`Hash` and `Eq` must agree: equal values must hash the same. Derive both
together, or implement both together. `BTreeMap` wants `Ord` instead.

## Generic types get an implicit bound

Deriving on a generic struct adds `T: Trait` to the impl, even when the struct
would not strictly need it.

```rust
#[derive(Debug, Clone)]
struct Holder<T> { value: T }

struct NoClone;

let h = Holder { value: NoClone };
let _ = h.clone();
```

Does not compile:

```
error[E0599]: the method `clone` exists for struct `Holder<NoClone>`, but its trait bounds were not satisfied
 --> src/main.rs:6:15
  |
2 | struct Holder<T> { value: T }
  | ---------------- method `clone` not found for this struct because it doesn't satisfy `Holder<NoClone>: Clone`
3 | struct NoClone;
  | -------------- doesn't satisfy `NoClone: Clone`
...
note: trait bound `NoClone: Clone` was not satisfied
  = help: consider manually implementing the trait to avoid undesired bounds
```

`Holder<i32>` still clones fine. Only the `NoClone` instantiation loses the
method. Module 09 covers generic bounds.

## Manual impls

Write it by hand when the derived behaviour is wrong.

**`Debug` that hides a secret:**

```rust
use std::fmt;

struct Secret { user: String, token: String }

impl fmt::Debug for Secret {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Secret")
            .field("user", &self.user)
            .field("token", &"***")
            .finish()
    }
}

println!("{:?}", Secret { user: "raj".into(), token: "hunter2".into() });
```

Output:

```
Secret { user: "raj", token: "***" }
```

**`Ord` that is not field order** — highest priority first, ties by name:

```rust
use std::cmp::Ordering;

#[derive(Debug, PartialEq, Eq)]
struct Task { name: String, priority: u8 }

impl Ord for Task {
    fn cmp(&self, other: &Self) -> Ordering {
        other.priority.cmp(&self.priority)
            .then_with(|| self.name.cmp(&other.name))
    }
}
impl PartialOrd for Task {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> { Some(self.cmp(other)) }
}

let mut tasks = vec![
    Task { name: "b".into(), priority: 1 },
    Task { name: "a".into(), priority: 9 },
    Task { name: "c".into(), priority: 9 },
];
tasks.sort();
for t in &tasks { println!("{} {}", t.priority, t.name); }
```

Output:

```
9 a
9 c
1 b
```

When you implement `Ord` manually, `PartialOrd` must delegate to it — otherwise
`<` and `.sort()` disagree, which is a logic bug the compiler cannot catch.

**`Default` with a non-zero value** — lesson 5.7 covers this in full:

```rust
#[derive(Debug)]
struct Session { id: u32, retries: u32 }

impl Default for Session {
    fn default() -> Self { Session { id: 0, retries: 3 } }
}

println!("{:?}", Session::default());
```

Output:

```
Session { id: 0, retries: 3 }
```

## Other derives you will meet

`Hash`, `Clone`, and friends are in `std`, but derive is an open mechanism — any
crate can ship one. `serde`'s `#[derive(Serialize, Deserialize)]` is the one you
will reach for most; lesson 17.1 covers it. `thiserror`'s `#[derive(Error)]`
appears in Module 08.

## Quick Reference

| Need | Derive |
|---|---|
| `println!("{:?}")` | `Debug` |
| `.clone()` | `Clone` |
| assignment copies instead of moving | `Clone, Copy` |
| `==` | `PartialEq` |
| `HashMap`/`HashSet` key | `PartialEq, Eq, Hash` |
| `BTreeMap`/`BTreeSet` key | `PartialEq, Eq, PartialOrd, Ord` |
| `<`, `>` | `PartialEq, PartialOrd` |
| `.sort()`, `.max()`, `.min()` | `PartialEq, Eq, PartialOrd, Ord` |
| `T::default()`, `..Default::default()` | `Default` |
| float fields | `PartialEq, PartialOrd` only — never `Eq`/`Ord`/`Hash` |
| custom print, custom sort, non-zero default | manual `impl` |

## Common Mistakes

**Deriving `Eq` or `Hash` on a struct with an `f64`.** Not possible. Either store
the number as a scaled integer or implement the traits by hand.

**Deriving `Copy` on anything holding a `String` or `Vec`.** `E0204`. Use
`Clone` and clone deliberately; the move semantics are usually what you want.

**Reordering struct fields on a type that derives `Ord`.** The sort order changes
silently. Write `Ord` manually when the order is load-bearing.

**Implementing `Ord` but deriving `PartialOrd`.** They can disagree. Implement
`partial_cmp` as `Some(self.cmp(other))`.

**Deriving `Hash` without `Eq`.** `HashMap` needs both and will tell you so, but
the deeper rule is that they must stay consistent — change one, change the other.

**Adding `Clone` to a generic struct and expecting it to always clone.** The
derive adds `T: Clone`, so `Holder<NoClone>` has no `.clone()`.

## Exercises

1. Define `Point3 { x: i32, y: i32, z: i32 }` with all nine derives. Print it,
   copy it, compare it, put it in a `HashSet`, and print `Point3::default()`.
2. Change `x` to `f64` and try to keep `Eq`. Capture the error, then pick the
   smallest derive set that still compiles.
3. Build `Vec<Version>` with five versions, `sort()` it, and print the max. Then
   swap `major` and `patch` in the declaration and explain the new order.
4. Write `enum LogLevel { Trace, Debug, Info, Warn, Error }` with `Ord` derived,
   and filter a vec down to `>= LogLevel::Warn`.
5. Implement `Debug` manually for `struct Card { rank: u8, suit: char }` so it
   prints `Card(10♠)`. Use `write!`.
6. Implement `Ord` manually for `struct Interval { start: u32, end: u32 }`
   ordering by length, then by start. Sort four intervals.

---

[← 5.5 Option: No More Null](05-option.md) · [Course Index](../README.md) · [5.7 Builder Pattern and Default →](07-builder-default.md)
