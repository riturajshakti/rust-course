[← 3.6 if let, while let, and let else](../03-control-flow/06-if-let-chains.md) · [Course Index](../README.md) · [4.2 Move Semantics and Copy Types →](02-moves-copy.md)

# 4.1 — Stack, Heap, and Why Ownership Exists

## What You'll Learn

- What actually lives on the stack vs the heap in Rust
- The memory layout of `String` and `&str`, field by field
- The three ownership rules and what "scope" means for them
- When a value is freed, demonstrated with printed output

## Coming From Another Language

| Other | Rust |
|---|---|
| JS/Python/Go: GC frees whenever | Freed at the closing brace of the owner's scope |
| C: `malloc` / `free` by hand | `String::from` / automatic `drop` |
| Go: escape analysis decides stack vs heap | The type decides; `String`/`Vec`/`Box` are heap |

There is no garbage collector and no `free` call. Each heap value has exactly
one owning variable, and when that variable goes out of scope the memory is
released. That is the whole mechanism; the rest of this module is the rules
that make it safe.

## Stack values have a size known at compile time

```rust
let n = 42i32;
let arr = [1u8, 2, 3];
let flag = true;
let c = 'x';
let t = (1i32, 2.5f64);
```

Fixed size, pushed and popped with the function frame, no allocation. Copying
one is a byte copy.

## Heap values are a stack handle plus an allocation

`String` and `Vec` are three words on the stack: a pointer, a length, and a
capacity. The characters themselves are elsewhere.

```rust
fn main() {
    let n = 42;
    let arr = [1u8, 2, 3];
    let s = String::from("heap text");
    let v: Vec<u8> = vec![7, 8];
    let lit = "static text";

    println!("i32            {:>3} bytes on stack", std::mem::size_of_val(&n));
    println!("[u8; 3]        {:>3} bytes on stack", std::mem::size_of_val(&arr));
    println!("String         {:>3} bytes on stack, {} on heap", std::mem::size_of_val(&s), s.capacity());
    println!("Vec<u8>        {:>3} bytes on stack, {} on heap", std::mem::size_of_val(&v), v.capacity());
    println!("&str           {:>3} bytes on stack, 0 on heap", std::mem::size_of_val(&lit));
}
```

Output:

```
i32              4 bytes on stack
[u8; 3]          3 bytes on stack
String          24 bytes on stack, 9 on heap
Vec<u8>         24 bytes on stack, 2 on heap
&str            16 bytes on stack, 0 on heap
```

## String: ptr, len, capacity

Three distinct numbers. `len` is bytes currently used, `capacity` is bytes
allocated.

```rust
let mut s = String::new();
println!("{} {}", s.len(), s.capacity());
for c in "abcdefghi".chars() {
    s.push(c);
    println!("{} {} {}", s, s.len(), s.capacity());
}
```

Output:

```
0 0
a 1 8
ab 2 8
abc 3 8
abcd 4 8
abcde 5 8
abcdef 6 8
abcdefg 7 8
abcdefgh 8 8
abcdefghi 9 16
```

An empty `String` allocates nothing. The first `push` grabs 8 bytes; exceeding
capacity reallocates and copies to a larger block. That reallocation is exactly
why a reference into a `String` cannot survive a `push` — the old pointer would
dangle. Lesson 4.4 shows the compiler blocking it.

## &str: ptr, len — two words, no capacity

A `&str` is a *view*: a pointer and a length, with no ownership and no
capacity field. It cannot grow, which is why there is no `push` on `&str`.

```rust
let owned = String::from("hello world");
let view: &str = &owned[0..5];
println!("{} {}", view, view.len());
println!("{}", std::mem::size_of::<String>());
println!("{}", std::mem::size_of::<&str>());
println!("{}", std::mem::size_of::<&String>());
```

Output:

```
hello 5
24
16
8
```

`&String` is one word — a plain pointer to the 24-byte handle. `&str` is two,
because it carries the length with it. A string literal is a `&'static str`
pointing into the compiled binary.

## Vec grows the same way

Same three words, same doubling, and `with_capacity` to skip the reallocations
when you know the size.

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    println!("{} {}", v.len(), v.capacity());
    for i in 0..9 {
        v.push(i);
        println!("{} {}", v.len(), v.capacity());
    }
    let pre: Vec<i32> = Vec::with_capacity(100);
    println!("{} {}", pre.len(), pre.capacity());
}
```

Output:

```
0 0
1 4
2 4
3 4
4 4
5 8
6 8
7 8
8 8
9 16
0 100
```

`Vec<i32>` starts at 4 because the growth policy is in bytes, not elements.
`with_capacity` allocates up front with `len` still 0.

## Box: one heap value, one stack word

`Box<T>` is the minimal heap allocation — a single pointer, no length or
capacity.

```rust
fn main() {
    let b = Box::new(5i32);
    println!("{} {}", *b, std::mem::size_of_val(&b));
    let boxed: Box<String> = Box::new(String::from("hi"));
    println!("{} {}", boxed.len(), std::mem::size_of_val(&boxed));
}
```

Output:

```
5 8
2 8
```

Note the second one: the `Box` is 8 bytes on the stack, pointing at the 24-byte
`String` handle on the heap, which points at the 2 bytes of text. Module 11
covers `Box` and the other smart pointers.

## The three ownership rules

1. Each value has exactly one owner.
2. There can be only one owner at a time.
3. When the owner goes out of scope, the value is dropped.

```rust
fn main() {
    let s = String::from("owned by s");
    println!("{s}");
}
```

Output:

```
owned by s
```

The allocation is released at the `}`. No call needed, and no way to forget it.

## Drop at the end of scope, demonstrated

Implementing `Drop` lets you see the moment it happens. Lesson 4.8 covers the
trait itself.

```rust
struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("dropping {}", self.0);
    }
}

fn main() {
    println!("start of main");
    {
        let _a = Noisy("inner a");
        let _b = Noisy("inner b");
        println!("end of inner block");
    }
    println!("after inner block");
    let _c = Noisy("outer c");
    println!("end of main");
}
```

Output:

```
start of main
end of inner block
dropping inner b
dropping inner a
after inner block
end of main
dropping outer c
```

Two things to read off that output: drops happen at the closing brace, not at
the last use, and they run in **reverse declaration order** — `b` before `a`.

## Scope is lexical, not based on last use

```rust
fn main() {
    let s = String::from("hello");
    println!("{s}");
    let other = String::from("world");
    println!("{other}");
}
```

Both strings stay alive until the end of `main` even though `s` is unused after
line 3. (References are the exception — their *borrow* ends at last use; see
lesson 4.4.)

## Moving a value out of a scope keeps it alive

The value outlives the inner block because ownership transferred to `outer`.

```rust
fn main() {
    let outer;
    {
        let s = String::from("moved out");
        outer = s;
        println!("inner end");
    }
    println!("{outer}");
}
```

Output:

```
inner end
moved out
```

## Shadowing does not drop early

A shadowed value is not freed at the shadowing point — it is still owned by a
live (now unreachable) binding, and drops at end of scope.

```rust
struct Noisy(&'static str);
impl Drop for Noisy {
    fn drop(&mut self) { println!("dropping {}", self.0); }
}

fn main() {
    let x = Noisy("first");
    let x = Noisy("shadowed");
    println!("body done");
    let _ = x;
}
```

Output:

```
body done
dropping shadowed
dropping first
```

Reverse declaration order again. If you need the old value gone immediately,
call `drop(x)` explicitly (lesson 4.8).

## Ownership crosses function boundaries

Passing by value hands the drop responsibility to the callee — the value is
freed before the call even returns. Passing a reference does not.

```rust
struct Noisy(&'static str);
impl Drop for Noisy {
    fn drop(&mut self) { println!("dropping {}", self.0); }
}

fn takes(n: Noisy) { println!("inside takes, got {}", n.0); }
fn borrows(n: &Noisy) { println!("inside borrows, saw {}", n.0); }

fn main() {
    let a = Noisy("a");
    borrows(&a);
    println!("back in main, a still alive");
    takes(a);
    println!("back in main, a is gone");
}
```

Output:

```
inside borrows, saw a
back in main, a still alive
inside takes, got a
dropping a
back in main, a is gone
```

`dropping a` prints *before* the last line — `takes` became the owner and its
scope ended first. Lesson 4.2 covers what that does to the caller's binding.

## Nested owners drop their contents

Dropping a struct drops its fields; dropping a `Vec` drops every element.

```rust
struct Noisy(&'static str);
impl Drop for Noisy {
    fn drop(&mut self) { println!("dropping {}", self.0); }
}

fn main() {
    let v = vec![Noisy("v[0]"), Noisy("v[1]")];
    println!("vec built with {} items", v.len());
}
```

Output:

```
vec built with 2 items
dropping v[0]
dropping v[1]
```

## How this compares to GC and to malloc/free

| | GC (JS, Python, Go) | `malloc`/`free` (C) | Rust ownership |
|---|---|---|---|
| When is memory freed | Whenever the collector runs | When you call `free` | At the owner's end of scope |
| Runtime cost | Collector pauses, tracing | None | None |
| Use after free | Impossible | Possible | Compile error |
| Double free | Impossible | Possible | Compile error |
| Memory leak | Hard (cycles only) | Easy | Hard (needs `Rc` cycles or `mem::forget`) |
| Who enforces it | Runtime | You | Compiler |
| Where you pay | Latency at runtime | Debugging at runtime | Fighting the compiler at build time |

Rust buys C's performance with GC's safety, and pays for it in compile-time
friction. That friction is this module.

## What this costs you up front

Three familiar patterns stop compiling. The next lessons cover each:

```rust
let a = String::from("x");
let b = a;
```

`a` is no longer usable — lesson 4.2.

```rust
let mut v = vec![1, 2];
let first = &v[0];
v.push(3);
```

Cannot mutate while borrowed — lesson 4.4.

```rust
fn f() -> &String { &String::from("x") }
```

Nothing for the return value to borrow from — lesson 4.3.

None of them are hard once you know the rule being enforced. While learning,
`.clone()` is a perfectly good escape hatch — reach for it and keep moving.

## Quick Reference

| Thing | Stack | Heap |
|---|---|---|
| `i32`, `bool`, `char`, `f64` | all of it | — |
| `[T; N]` | all of it | — |
| `&T`, `&mut T` | 1 word | — |
| `&str`, `&[T]` | 2 words (ptr + len) | — |
| `String` | 3 words (ptr + len + cap) | the bytes |
| `Vec<T>` | 3 words (ptr + len + cap) | the elements |
| `Box<T>` | 1 word (ptr) | the `T` |

| Concept | Rule |
|---|---|
| Owner | Exactly one variable per value |
| Drop point | Closing brace of the owner's scope |
| Drop order | Reverse declaration order |
| Move out of scope | Value survives, new owner's scope applies |
| Shadowing | Old value drops at end of scope, not at shadow |
| `size_of::<T>()` | Stack size of `T` |
| `size_of_val(&x)` | Stack size of `x`'s type |

## Common Mistakes

**Assuming `String` and `&str` are the same type.** One owns an allocation and
is 24 bytes; the other is a 16-byte borrowed view. Lesson 6.1 goes deeper.

**Expecting a value to be freed at its last use.** It is freed at the closing
brace. Use an inner `{ }` block or `drop(x)` if you need it sooner.

**Thinking the heap is slower so you should avoid `String`.** One allocation is
nanoseconds. Use `String` until profiling says otherwise.

**Reading "ownership" as "immutability".** Owning a value says nothing about
mutating it; `let mut` controls that, independently.

**Fighting the borrow checker instead of cloning.** During the first week,
`.clone()` is the right answer. Correct first, cheap later.

## Exercises

1. Print `size_of::<T>()` for `i32`, `[i32; 10]`, `String`, `&str`, `Vec<f64>`,
   `Box<i32>`, and `Option<Box<i32>>`. Explain the last one.
2. Build a `String` with `push` in a loop of 40 characters, printing `len` and
   `capacity` each step. Note every capacity change.
3. Write a `Noisy` struct with `Drop` and arrange for exactly this output:
   `a`, `c`, `b` in that drop order. Use nested blocks.
4. Create a `String` inside an inner block and move it out so it is still
   printable after the block closes.
5. Add a `Vec<Noisy>` field to a struct that also implements `Drop`. Predict
   the output order before running it, then check.

---

[← 3.6 if let, while let, and let else](../03-control-flow/06-if-let-chains.md) · [Course Index](../README.md) · [4.2 Move Semantics and Copy Types →](02-moves-copy.md)
