[← 4.8 Clone, Copy, and Drop](../04-ownership/08-clone-drop.md) · [Course Index](../README.md) · [5.2 Methods and Associated Functions →](02-methods.md)

# 5.1 — Defining and Instantiating Structs

## What You'll Learn

- Every way to define a struct and build an instance
- Field init shorthand and struct update syntax
- How mutability, ownership, and moves work at field level
- Debug printing, nesting, generics, and visibility

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `interface User { name: string }` | `struct User { name: String }` |
| Python `@dataclass` | `struct` + `#[derive(...)]` |
| Go `type User struct { Name string }` | `struct User { name: String }` |

Closest to a Go struct. Two differences that bite: **mutability is per-binding,
not per-field**, and there is no zero value — you must initialize every field.

## Defining and instantiating

```rust
#[derive(Debug)]
struct User {
    username: String,
    email: String,
    active: bool,
    sign_in_count: u64,
}

fn main() {
    let user1 = User {
        username: String::from("raj"),
        email: String::from("raj@example.com"),
        active: true,
        sign_in_count: 1,
    };

    println!("{}", user1.username);
    println!("{}", user1.email);
    println!("{} {}", user1.active, user1.sign_in_count);
}
```

Output:

```
raj
raj@example.com
true 1
```

Field order in the literal doesn't matter, but every field must be present.

## Debug printing

`{:?}` for one line, `{:#?}` for pretty. Both need `#[derive(Debug)]`.

```rust
println!("{:?}", user1);
println!("{:#?}", user1);
```

Output:

```
User { username: "raj", email: "raj@example.com", active: true, sign_in_count: 1 }
User {
    username: "raj",
    email: "raj@example.com",
    active: true,
    sign_in_count: 1,
}
```

Without the derive you get `error[E0277]: User doesn't implement Debug`.

## Mutability is per-binding

```rust
struct Point { x: i32, y: i32 }

let p = Point { x: 1, y: 2 };
p.x = 5;
```

Does not compile:

```
error[E0594]: cannot assign to `p.x`, as `p` is not declared as mutable
 --> src/main.rs:4:5
  |
4 |     p.x = 5;
  |     ^^^^^^^ cannot assign
  |
help: consider changing this to be mutable
  |
3 |     let mut p = Point { x: 1, y: 2 };
  |         +++
```

`mut` on the binding makes **every** field writable. There is no per-field
`mut`, and no `readonly` on a single field.

```rust
let mut u = User {
    username: String::from("raj"),
    email: String::from("a@b.com"),
    active: true,
    sign_in_count: 0,
};
u.email = String::from("new@b.com");
u.sign_in_count += 1;
println!("{:?}", u);
```

Output:

```
User { username: "raj", email: "new@b.com", active: true, sign_in_count: 1 }
```

## Field init shorthand

When a variable has the same name as the field, write it once.

```rust
fn build_user(username: String, email: String) -> User {
    User { username, email, active: true, sign_in_count: 0 }
}
```

The long form `username: username` is identical; nobody writes it.

## Struct update syntax

`..other` fills the remaining fields from another instance.

```rust
let u2 = User { email: String::from("second@b.com"), ..u };
println!("{:?}", u2);
```

Output:

```
User { username: "raj", email: "second@b.com", active: true, sign_in_count: 1 }
```

`..base` must come last, and it is not spread-into-a-new-object like JS: it
**moves** non-`Copy` fields out of `base`.

```rust
let u = User { /* ... */ };
let u2 = User { active: false, ..u };
println!("{:?}", u);
```

```
error[E0382]: borrow of partially moved value: `u`
 --> src/main.rs:6:22
  |
5 |     let u2 = User { active: false, ..u };
  |              --------------------------- value partially moved here
6 |     println!("{:?}", u);
  |                      ^ value borrowed here after partial move
  |
  = note: partial move occurs because `u.username` has type `String`, which
    does not implement the `Copy` trait
```

Individual `Copy` fields of a partially moved value still work:

```rust
let u2 = User { email: String::from("x@y.com"), ..u };
println!("{}", u.active);
```

Output:

```
true
```

Use `..base.clone()` when you need `base` afterwards.

## Nested structs

```rust
#[derive(Debug)]
struct Address { city: String, zip: String }

#[derive(Debug)]
struct Company { name: String, address: Address, employees: Vec<String> }

let c = Company {
    name: "Acme".to_string(),
    address: Address { city: "Pune".to_string(), zip: "411001".to_string() },
    employees: vec!["raj".to_string(), "sam".to_string()],
};

println!("{}", c.address.city);
println!("{}", c.employees[0]);
println!("{} employees", c.employees.len());
println!("{:#?}", c.address);
```

Output:

```
Pune
raj
2 employees
Address {
    city: "Pune",
    zip: "411001",
}
```

Dotted access nests arbitrarily deep. A struct **owns** its fields — dropping
`c` drops the `Address` and the `Vec` with it.

## Generic fields

```rust
#[derive(Debug)]
struct Pair<T> { first: T, second: T }

let p = Pair { first: 1, second: 2 };
let q = Pair { first: "a", second: "b" };
println!("{:?} {:?}", p, q);
```

Output:

```
Pair { first: 1, second: 2 } Pair { first: "a", second: "b" }
```

Module 09 covers generics properly.

## Reference fields need a lifetime

```rust
struct Excerpt { part: &str }
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:24
  |
1 | struct Excerpt { part: &str }
  |                        ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 | struct Excerpt<'a> { part: &'a str }
  |               ++++          ++
```

Take the compiler's suggestion, or store an owned `String`. Lesson 4.7 explains
why. Owned fields are the default choice until you have a reason otherwise.

## Deriving more than Debug

```rust
#[derive(Debug, Clone, PartialEq, Default)]
struct Config { host: String, port: u16, tls: bool }

let d = Config::default();
println!("{:?}", d);

let a = Config { host: "localhost".into(), port: 8080, tls: false };
let b = a.clone();
println!("{}", a == b);

let c = Config { port: 443, tls: true, ..a.clone() };
println!("{:?}", c);
```

Output:

```
Config { host: "", port: 0, tls: false }
true
Config { host: "localhost", port: 443, tls: true }
```

`Default` gives `""`, `0`, `false` per field. Lesson 5.6 covers each derive,
5.7 covers `Default` and builders.

## Structs in collections and Options

```rust
#[derive(Debug)]
struct Wrapper { inner: Option<Box<Config>> }

println!("{:?}", Wrapper { inner: Some(Box::new(c)) });
println!("{:?}", Wrapper { inner: None });

let list = vec![
    Config { host: "a".into(), port: 1, tls: false },
    Config { host: "b".into(), port: 2, tls: true },
];
for cfg in &list {
    println!("{}:{} tls={}", cfg.host, cfg.port, cfg.tls);
}
println!("{}", list.iter().filter(|c| c.tls).count());
```

Output:

```
Wrapper { inner: Some(Config { host: "localhost", port: 443, tls: true }) }
Wrapper { inner: None }
a:1 tls=false
b:2 tls=true
1
```

## Size and layout

```rust
struct Empty;
println!("{}", std::mem::size_of::<Address>());
println!("{}", std::mem::size_of::<Empty>());
```

Output:

```
48
0
```

Two `String`s are 24 bytes each. A fieldless struct is zero-sized and costs
nothing at runtime. Field order in memory is not guaranteed — the compiler
reorders to minimize padding unless you add `#[repr(C)]` (lesson 16.5).

## Visibility

Private by default, including fields.

```rust
pub struct Account {
    pub id: u64,
    balance: i64,
}
```

`id` is readable outside the module; `balance` is not. Note `pub struct` alone
does **not** make fields public — a common surprise. Module 12 covers modules.

## Quick Reference

| Form | Syntax |
|---|---|
| Define | `struct S { a: T, b: U }` |
| Instantiate | `S { a: x, b: y }` |
| Field init shorthand | `S { a, b }` |
| Struct update | `S { a: x, ..other }` |
| Field access | `s.a`, `s.inner.deep` |
| Mutate | `let mut s = ...; s.a = v;` |
| Debug print | `{:?}` / `{:#?}` with `#[derive(Debug)]` |
| Generic | `struct S<T> { a: T }` |
| Reference field | `struct S<'a> { a: &'a str }` |
| Public | `pub struct S { pub a: T }` |
| Size | `std::mem::size_of::<S>()` |

## Common Mistakes

**Expecting a zero value.** Every field must be given a value; there is no
implicit `0`/`""`/`nil`. Use `#[derive(Default)]` and `Config::default()`.

**Trying to mark one field `mut`.** Mutability is on the binding. Split the type
or use interior mutability (Module 11) if you truly need per-field control.

**Using `..base` then reusing `base`.** It moves non-`Copy` fields. Use
`..base.clone()`.

**Making a struct `pub` and expecting fields to follow.** Each field needs its
own `pub`.

**Putting `&str` in a struct to avoid allocation.** It drags a lifetime through
every signature that touches the type. Store `String` until profiling says
otherwise.

## Exercises

1. Define `Book { title: String, pages: u32, available: bool }`, build one,
   print it with `{:#?}`.
2. Write `fn new_book(title: String, pages: u32) -> Book` using field init
   shorthand, defaulting `available` to `true`.
3. From one `Book`, make a second that differs only in `pages`, using struct
   update syntax. Then print the original and fix whatever error appears.
4. Build a `Library { name: String, books: Vec<Book> }`. Print the count of
   available books.
5. Add `#[derive(Default, PartialEq, Clone)]` to `Book`. Print
   `Book::default()`, clone one, and compare two instances with `==`.

---

[← 4.8 Clone, Copy, and Drop](../04-ownership/08-clone-drop.md) · [Course Index](../README.md) · [5.2 Methods and Associated Functions →](02-methods.md)
