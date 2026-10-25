[← Course Index](../README.md)

# 11.5 — Deref, AsRef, and Borrow

## What You'll Learn

- `Deref` and the coercion that makes `&String` work where `&str` is wanted
- Method resolution through `Deref`, and `DerefMut`
- Why `Deref` is for smart pointers, not inheritance
- `AsRef`/`AsMut` for flexible parameters, `Borrow` for `HashMap` lookups

## Coming From Another Language

| Other | Rust |
|---|---|
| JS: prototype chain delegates method lookup | `Deref` delegates method lookup, but not trait impls |
| Python `__getattr__` fallback | `Deref` is compile-time and typed |
| Go: embedding promotes methods | `Deref` promotes methods but not interface satisfaction |

Three traits that all look like "give me a reference to the inside," with three
different jobs. This lesson is mostly about telling them apart.

## The Deref trait

One associated type and one method.

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> { MyBox(x) }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &T { &self.0 }
}

fn main() {
    let b = MyBox::new(5);
    println!("{}", *b);
}
```

Output:

```
5
```

`*b` is sugar for `*(b.deref())`. The extra `*` is there because `deref` returns
a *reference* — the compiler inserts it.

## Method resolution through Deref

With `Deref` implemented, every method of `Target` is callable on `MyBox`.

```rust
fn hello(name: &str) {
    println!("hello, {}", name);
}

fn main() {
    let s = MyBox::new(String::from("rust"));
    println!("{}", *s);
    println!("{}", s.len());
    println!("{}", s.to_uppercase());
    hello(&s);
}
```

Output:

```
rust
4
RUST
hello, rust
```

`s.len()` has no `len` on `MyBox`, so the compiler derefs to `String` and finds
it there. `hello(&s)` is the coercion: `&MyBox<String>` -> `&String` -> `&str`,
chained as many steps as needed, all at compile time with zero runtime cost.

Without coercion you would write `hello(&(*s)[..])`.

## DerefMut

The `&mut` counterpart, needed for mutating methods and index assignment.

```rust
use std::ops::DerefMut;

impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut T { &mut self.0 }
}

fn main() {
    let mut v = MyBox::new(vec![1, 2, 3]);
    v.push(4);
    println!("{:?}", *v);
    v[0] = 99;
    println!("{:?}", *v);
}
```

Output:

```
[1, 2, 3, 4]
[99, 2, 3, 4]
```

`DerefMut` requires `Deref`; `Target` comes from `Deref`.

## The built-in coercions you already rely on

```rust
fn takes_str(s: &str) -> usize { s.len() }
fn takes_slice(v: &[i32]) -> i32 { v.iter().sum() }

fn main() {
    let owned = String::from("hello");
    println!("{}", takes_str(&owned));
    println!("{}", takes_str(&&owned));
    let boxed: Box<String> = Box::new(String::from("boxed"));
    println!("{}", takes_str(&boxed));
    let rc = std::rc::Rc::new(String::from("rc value"));
    println!("{}", takes_str(&rc));

    let v = vec![1, 2, 3];
    println!("{}", takes_slice(&v));
    println!("{}", takes_slice(&v[1..]));
    let bv: Box<Vec<i32>> = Box::new(vec![10, 20]);
    println!("{}", takes_slice(&bv));
}
```

Output:

```
5
5
5
8
6
5
30
```

`&String -> &str`, `&Vec<T> -> &[T]`, and through any number of `Box`/`Rc`
layers. `&&owned` works too — reference-to-reference also derefs. This is why
you take `&str` and `&[T]` in function signatures: callers with `String` and
`Vec` need no conversion.

Coercion applies to function arguments, method receivers, and `let` with an
explicit type. It does **not** apply to generic type parameters — that's what
`AsRef` is for.

## Deref is not inheritance

Methods are promoted. Trait implementations are not.

```rust
use std::ops::Deref;

struct Base;
impl Base {
    fn shared(&self) { println!("base method"); }
}

struct Derived { base: Base }
impl Deref for Derived {
    type Target = Base;
    fn deref(&self) -> &Base { &self.base }
}

trait Greet { fn greet(&self); }
impl Greet for Base {
    fn greet(&self) { println!("base greet"); }
}

fn main() {
    let d = Derived { base: Base };
    d.shared();
    d.greet();
}
```

Output:

```
base method
base greet
```

Both calls resolve. But a generic bound does not:

```rust
fn needs_greet<T: Greet>(_t: &T) {}

needs_greet(&d);
```

Does not compile — intentional:

```
error[E0277]: the trait bound `Derived: Greet` is not satisfied
  --> src/main.rs:25:17
   |
25 |     needs_greet(&d);
   |     ----------- ^^ unsatisfied trait bound
   |
help: the trait `Greet` is not implemented for `Derived`
help: the trait `Greet` is implemented for `Base`
note: required by a bound in `needs_greet`
   |
19 | fn needs_greet<T: Greet>(_t: &T) {}
   |                   ^^^^^ required by this bound in `needs_greet`
```

So `Deref`-as-inheritance breaks the moment anything generic is involved, and it
makes method resolution confusing to read. **Implement `Deref` only when your
type is a smart pointer** — something whose whole purpose is to stand in for the
value inside. For code reuse, use traits with default methods or plain
delegation.

## AsRef and AsMut

`AsRef<T>` is an explicit, cheap reference conversion, used as a *generic bound*
so one function accepts several argument types.

```rust
use std::path::{Path, PathBuf};

fn describe<P: AsRef<Path>>(p: P) {
    println!("{:?}", p.as_ref().extension());
}

fn shout<S: AsRef<str>>(s: S) -> String {
    s.as_ref().to_uppercase()
}

fn fill<B: AsMut<[u8]>>(mut b: B, v: u8) {
    for byte in b.as_mut() { *byte = v; }
}

fn main() {
    describe("file.txt");
    describe(String::from("archive.tar.gz"));
    describe(PathBuf::from("noext"));

    println!("{}", shout("str"));
    println!("{}", shout(String::from("string")));
    println!("{}", shout(&String::from("ref string")));

    let mut arr = [0u8; 4];
    fill(&mut arr, 7);
    println!("{:?}", arr);

    let mut v = vec![1u8, 2, 3];
    fill(&mut v, 9);
    println!("{:?}", v);
}
```

Output:

```
Some("txt")
Some("gz")
None
STR
STRING
REF STRING
[7, 7, 7, 7]
[9, 9, 9]
```

`AsRef<Path>` is the standard signature for any function taking a file path —
it's what `File::open` uses. `AsRef<str>` is the same idea for text. The cost is
one `.as_ref()` call inside and a slightly noisier signature.

Unlike `Deref`, a type can implement `AsRef` for many targets (`String`
implements `AsRef<str>`, `AsRef<[u8]>`, `AsRef<Path>` and `AsRef<OsStr>`), and
nothing is inserted implicitly.

## Borrow and the HashMap lookup

`Borrow<T>` looks like `AsRef<T>` but carries an extra promise: the borrowed
form must hash and compare **identically** to the owned one. That is exactly
what a `HashMap` needs to let you look up a `String` key with a `&str`.

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, u32> = HashMap::new();
    m.insert(String::from("alpha"), 1);
    m.insert(String::from("beta"), 2);

    println!("{:?}", m.get("alpha"));
    println!("{:?}", m.get(&String::from("beta")));
    println!("{}", m.contains_key("gamma"));
    println!("{:?}", m.remove("alpha"));
}
```

Output:

```
Some(1)
Some(2)
false
Some(1)
```

`get` is declared roughly as:

```rust
fn get<Q>(&self, k: &Q) -> Option<&V>
where K: Borrow<Q>, Q: Hash + Eq + ?Sized
```

Because `String: Borrow<str>`, passing `&str` works — no `String` allocated for
a read. The same trick gives you `HashMap<Vec<T>, _>.get(&[T])` and
`BTreeMap<String, _>.range("a".."b")`.

Using it directly, and in a generic:

```rust
use std::borrow::Borrow;

fn count_a<K: Borrow<str>>(keys: &[K]) -> usize {
    keys.iter().filter(|k| { let s: &str = (*k).borrow(); s.starts_with('a') }).count()
}

fn main() {
    let owned = vec![String::from("ant"), String::from("bee"), String::from("ape")];
    println!("{}", count_a(&owned));

    let s = String::from("text");
    let b: &str = s.borrow();
    println!("{}", b);
}
```

Output:

```
2
text
```

The `let s: &str` annotation is needed: `borrow()` is generic in its return
type, so inference needs a target. `AsRef` has no such problem — another reason
to prefer `AsRef` unless you specifically need the hash/eq guarantee.

## Deref vs AsRef vs Borrow

| | `Deref` | `AsRef<T>` | `Borrow<T>` |
|---|---|---|---|
| Invoked | implicitly, by coercion | explicitly, `.as_ref()` | explicitly, `.borrow()` |
| Targets per type | exactly one | many | many |
| Extra guarantee | none | cheap conversion | `Hash`/`Eq`/`Ord` must match |
| Used for | smart pointers (`Box`, `Rc`, `String`, `Vec`) | flexible parameters (`AsRef<Path>`) | keyed lookup (`HashMap::get`) |
| `&mut` version | `DerefMut` | `AsMut<T>` | `BorrowMut<T>` |
| Blanket impl | — | `&T: AsRef<U>` if `T: AsRef<U>` | `T: Borrow<T>`, `&T: Borrow<T>` |
| Implement it when | your type *is* a pointer to `T` | a caller might hold `T` in several forms | your owned type has a canonical borrowed form |

Rule of thumb: implementing a smart pointer -> `Deref`. Writing a convenient
function signature -> `AsRef`. Writing a collection keyed by your type ->
`Borrow`.

## Quick Reference

| Item | Signature |
|---|---|
| `Deref` | `type Target; fn deref(&self) -> &Self::Target` |
| `DerefMut` | `fn deref_mut(&mut self) -> &mut Self::Target` |
| `AsRef<T>` | `fn as_ref(&self) -> &T` |
| `AsMut<T>` | `fn as_mut(&mut self) -> &mut T` |
| `Borrow<T>` | `fn borrow(&self) -> &T` |
| `BorrowMut<T>` | `fn borrow_mut(&mut self) -> &mut T` |
| `&String -> &str` | deref coercion |
| `&Vec<T> -> &[T]` | deref coercion |
| `&Box<T> -> &T` | deref coercion |
| `AsRef<Path>` | idiomatic path parameter |
| `K: Borrow<Q>` | `HashMap::get(&Q)` with owned `K` |

## Common Mistakes

**Implementing `Deref` on a business struct to fake inheritance.** Trait bounds
won't follow and readers can't tell where a method came from. Use traits.

**Taking `&String` or `&Vec<T>` in a signature.** Take `&str` and `&[T]`;
coercion means callers lose nothing and `&str` callers gain.

**Expecting deref coercion at a generic bound.** `fn f<T: AsRef<str>>` works;
`fn f<T: Deref<Target = str>>` is unusual and awkward. Reach for `AsRef`.

**Implementing `Borrow<T>` when hash/eq differ between the forms.** It silently
breaks `HashMap` lookups. Use `AsRef` instead.

**Calling `.borrow()` and hitting an inference error.** Annotate the target
type, or use `.as_ref()`.

## Exercises

1. Write `struct Meters(f64)` with `Deref<Target = f64>`, then call `.sqrt()`
   and `.floor()` on a `Meters` value directly.
2. Implement `Deref` and `DerefMut` for `struct Stack<T>(Vec<T>)` and show
   `push`, `pop`, `len` and indexing all working through it.
3. Write `fn read_first_line<P: AsRef<Path>>(p: P) -> io::Result<String>` and
   call it with a `&str`, a `String` and a `PathBuf`.
4. Build a `HashMap<String, Vec<u32>>`, then write a lookup function taking
   `&str` that returns the sum, with no allocation in the lookup path.
5. Define `struct Id(String)` with `Borrow<str>`, put it in a `HashSet<Id>`, and
   check membership with a plain `&str`.

---

[← Course Index](../README.md)
