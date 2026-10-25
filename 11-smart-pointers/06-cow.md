[← Course Index](../README.md)

# 11.6 — Cow: Clone on Write

## What You'll Learn

- `Cow::Borrowed` vs `Cow::Owned`, and why one type holds both
- Returning `Cow<str>` from a function that only sometimes modifies its input
- `into_owned` and `to_mut`
- `Cow` in struct fields, and when it isn't worth it

## Coming From Another Language

In JS or Python every string operation allocates and you never think about it.
In Rust you have to choose a return type: `&str` (no allocation, but you cannot
modify) or `String` (can modify, but allocates even when nothing changed).
`Cow<'a, str>` is the third option — it is *either*, decided at runtime.

```rust
fn sanitize(s: &str) -> String      // allocates always
fn sanitize(s: &str) -> &str        // cannot modify
fn sanitize(s: &str) -> Cow<'_, str> // allocates only when modified
```

`Cow` stands for "clone on write." It is a plain enum over a borrow and an owned
value, with `Deref` to the borrowed form so it reads like the borrowed type.

## The two variants

```rust
use std::borrow::Cow;

fn main() {
    let b: Cow<str> = Cow::Borrowed("borrowed");
    let o: Cow<str> = Cow::Owned(String::from("owned"));
    println!("{} {}", b, o);
    println!("{} {}", b.len(), o.to_uppercase());

    for c in [&b, &o] {
        match c {
            Cow::Borrowed(s) => println!("borrowed: {}", s),
            Cow::Owned(s) => println!("owned: {}", s),
        }
    }

    let from_str: Cow<str> = "implicit".into();
    let from_string: Cow<str> = String::from("also implicit").into();
    println!("{:?}", from_str);
    println!("{:?}", from_string);
    println!("{} {}", size_of::<Cow<str>>(), size_of::<String>());
}
```

Output:

```
borrowed owned
8 OWNED
borrowed: borrowed
owned: owned
"implicit"
"also implicit"
24 24
```

Read methods work on both variants without matching, because `Cow<str>` derefs
to `str`. `.into()` picks the right variant from a `&str` or a `String`.

And `Cow<str>` is the same 24 bytes as `String` — the discriminant fits in the
existing padding, so there is no size penalty for the flexibility.

## The motivating case: sanitize

HTML-escape a string, but only allocate when there is something to escape.

```rust
use std::borrow::Cow;

fn sanitize(input: &str) -> Cow<'_, str> {
    if input.contains(['<', '>', '&']) {
        let mut out = String::with_capacity(input.len());
        for c in input.chars() {
            match c {
                '<' => out.push_str("&lt;"),
                '>' => out.push_str("&gt;"),
                '&' => out.push_str("&amp;"),
                _ => out.push(c),
            }
        }
        Cow::Owned(out)
    } else {
        Cow::Borrowed(input)
    }
}

fn main() {
    for s in ["plain text", "a <b> & c", ""] {
        let r = sanitize(s);
        let kind = match r {
            Cow::Borrowed(_) => "borrowed",
            Cow::Owned(_) => "owned",
        };
        println!("{:10} -> {:?} ({})", s, r, kind);
    }
}
```

Output:

```
plain text -> "plain text" (borrowed)
a <b> & c  -> "a &lt;b&gt; &amp; c" (owned)
           -> "" (borrowed)
```

Two of three inputs allocate nothing. The caller treats all three identically —
it can print them, measure them, or call `into_owned()` if it needs a `String`.

The structure is always the same: a cheap scan to decide, then either
`Cow::Borrowed(input)` on the common path or `Cow::Owned(built)` on the rare
one. If the scan is as expensive as the transform, `Cow` is not buying anything.

## into_owned

Collapses either variant into an owned value — cloning the borrowed case, moving
the owned one.

```rust
use std::borrow::Cow;

fn main() {
    let b: Cow<str> = Cow::Borrowed("copy me");
    let owned: String = b.into_owned();
    println!("{}", owned);

    let o: Cow<str> = Cow::Owned(String::from("already owned"));
    let s: String = o.into_owned();
    println!("{}", s);
}
```

Output:

```
copy me
already owned
```

Call it at the boundary where you need `String` — storing in a struct that
outlives the input, or sending across a thread. `to_string()` works too but
always copies, even on `Cow::Owned`.

## to_mut

Returns `&mut` to the owned form, upgrading `Borrowed` to `Owned` in place if
needed. This is the "write" in clone-on-write.

```rust
use std::borrow::Cow;

fn main() {
    let mut c: Cow<str> = Cow::Borrowed("grow");
    println!("{}", matches!(c, Cow::Borrowed(_)));
    c.to_mut().push_str("ing");
    println!("{} {}", c, matches!(c, Cow::Owned(_)));
    c.to_mut().push('!');
    println!("{}", c);
}
```

Output:

```
true
growing true
growing!
```

The first `to_mut()` clones; the second is free, since it is already owned. That
makes it right for a pipeline of maybe-mutations: the first one that fires pays
for the allocation and the rest are ordinary mutations.

## Cow\<[T]\>

Same thing for slices.

```rust
use std::borrow::Cow;

fn main() {
    let mut v: Cow<[i32]> = Cow::Borrowed(&[1, 2, 3]);
    println!("{:?} {}", v, matches!(v, Cow::Borrowed(_)));
    v.to_mut().push(4);
    println!("{:?} {}", v, matches!(v, Cow::Owned(_)));
    println!("{} {}", v.len(), v.iter().sum::<i32>());
    println!("{:?}", v.into_owned());
}
```

Output:

```
[1, 2, 3] true
[1, 2, 3, 4] true
4 10
[1, 2, 3, 4]
```

`Cow<'a, [T]>` borrows `&'a [T]` and owns `Vec<T>`. It works for any `T: Clone`,
and the pattern generalizes to any type implementing `ToOwned`.

## Cow in the standard library

You have been using it already.

```rust
use std::borrow::Cow;

fn main() {
    let good = String::from_utf8_lossy(b"valid ascii");
    let bad = String::from_utf8_lossy(&[0x66, 0x6f, 0xff, 0x6f]);
    println!("{:?} {}", good, matches!(good, Cow::Borrowed(_)));
    println!("{:?} {}", bad, matches!(bad, Cow::Owned(_)));

    let p = std::path::Path::new("/tmp/file.txt");
    println!("{:?}", p.to_string_lossy());
}
```

Output:

```
"valid ascii" true
"fo�o" true
"/tmp/file.txt"
```

Valid UTF-8 is borrowed as-is; invalid bytes force an owned copy with the
replacement character. `Path::to_string_lossy` and
`percent_encoding`'s decode functions do the same. Note `str::replace` returns
`String`, not `Cow` — a deliberate simplification in the standard library, and a
hint that `Cow` is not always worth the API cost.

## Cow in struct fields

Lets one struct serve both a zero-copy parse and an owned, long-lived value.

```rust
use std::borrow::Cow;

#[derive(Debug)]
struct Header<'a> {
    name: Cow<'a, str>,
    value: Cow<'a, str>,
}

impl<'a> Header<'a> {
    fn parse(line: &'a str) -> Option<Header<'a>> {
        let (n, v) = line.split_once(':')?;
        Some(Header {
            name: if n.chars().any(|c| c.is_uppercase()) {
                Cow::Owned(n.to_lowercase())
            } else {
                Cow::Borrowed(n)
            },
            value: Cow::Borrowed(v.trim()),
        })
    }
    fn into_static(self) -> Header<'static> {
        Header {
            name: Cow::Owned(self.name.into_owned()),
            value: Cow::Owned(self.value.into_owned()),
        }
    }
}

fn main() {
    let h1 = Header::parse("content-type: text/html").unwrap();
    let h2 = Header::parse("Accept-Encoding: gzip").unwrap();
    println!("{:?}", h1);
    println!("{:?}", h2);
    println!("{}", Header::parse("no colon here").is_none());
    let owned = h1.into_static();
    println!("{} = {}", owned.name, owned.value);
}
```

Output:

```
Header { name: "content-type", value: "text/html" }
Header { name: "accept-encoding", value: "gzip" }
true
content-type = text/html
```

The already-lowercase name borrows, the mixed-case one allocates. `into_static`
is the standard escape hatch when the header must outlive the buffer it was
parsed from — real HTTP and serde crates expose exactly this method.

The cost is the lifetime parameter `'a`, which propagates into every type and
signature that holds a `Header`. That is the real price of `Cow` in a field.

## When Cow pays, and when it's premature

It pays when:

- The common case genuinely does not modify — escaping, normalizing,
  trimming, decoding — and that case dominates.
- The function is hot, or runs per-request / per-line over a large input.
- You are writing a library and want callers to avoid an allocation they would
  otherwise be forced into.

It's premature when:

- You cannot say which branch is common. `String` is simpler and the allocator
  is fast.
- The input is small and called rarely. One `String` per call is noise.
- It would add a lifetime to a struct that currently has none. Lifetime
  plumbing through a dozen signatures costs more developer time than the
  allocations cost the machine.
- Callers always need an owned value anyway — then you have added a match for
  nothing.

Default to `String`. Switch to `Cow` when a profile or a library boundary asks
for it.

## Quick Reference

| Item | Meaning |
|---|---|
| `Cow<'a, str>` | `Borrowed(&'a str)` or `Owned(String)` |
| `Cow<'a, [T]>` | `Borrowed(&'a [T])` or `Owned(Vec<T>)` |
| `Cow::Borrowed(x)` | no allocation |
| `Cow::Owned(x)` | holds the owned value |
| `"s".into()` | makes a `Borrowed` |
| `String::from("s").into()` | makes an `Owned` |
| `.into_owned()` | `String`/`Vec<T>`; clones if borrowed |
| `.to_mut()` | `&mut` owned form; clones once if borrowed |
| `matches!(c, Cow::Borrowed(_))` | check which variant |
| deref | read methods of the borrowed type work on both |
| `size_of::<Cow<str>>()` | 24 — same as `String` |
| std users | `String::from_utf8_lossy`, `Path::to_string_lossy` |

## Common Mistakes

**Returning `Cow` from a function that always allocates.** You added a match for
no benefit. Return `String`.

**Calling `.to_string()` instead of `.into_owned()`.** `to_string` copies even
when the `Cow` already owns its data.

**Calling `to_mut()` in a loop before checking.** Each call is cheap after the
first, but the first clones — make sure you actually need to write.

**Putting `Cow<'a, str>` in a struct "for performance."** The lifetime spreads
through your whole API. Measure first; `into_static()` is a hint you didn't need
the borrow.

**Expecting `Cow` to deduplicate or refcount.** It is just an enum over borrow
and owned. Sharing is `Rc`/`Arc` (lessons 11.2 and 11.4).

## Exercises

1. Write `fn trim_cow(s: &str) -> Cow<'_, str>` that returns `Borrowed` when the
   input has no surrounding whitespace and `Owned` otherwise. Print the variant
   for three inputs.
2. Write `fn normalize(s: &str) -> Cow<'_, str>` that lowercases only if the
   input contains an uppercase letter. Confirm the all-lowercase path borrows.
3. Write `fn dedup(v: &[i32]) -> Cow<'_, [i32]>` that borrows when the slice is
   already sorted and deduplicated, and allocates otherwise.
4. Chain two `Cow`-returning functions so that the second takes `Cow<str>` and
   uses `to_mut()`. Verify only one allocation happens.
5. Define `struct Record<'a> { id: Cow<'a, str>, tags: Cow<'a, [u32]> }`, parse
   one from a borrowed line, then write `into_owned()` producing a
   `Record<'static>`.

---

[← Course Index](../README.md)
