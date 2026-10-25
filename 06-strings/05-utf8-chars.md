[← Course Index](../README.md)

# 6.5 — UTF-8, Chars, Bytes, and Grapheme Safety

## What You'll Learn

- Why `s[0]` does not compile, and what to write instead
- Byte vs `char` vs grapheme cluster, with real counts
- Slicing off a char boundary panics — the exact message
- Reversing, truncating, and case-folding text without corrupting it

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `s[0]`, `s.charAt(0)` | `s.chars().next()` — no indexing at all |
| JS `s.length` (UTF-16 units) | `s.len()` (UTF-8 bytes) or `s.chars().count()` |
| Python `s[0]` (a code point) | `s.chars().nth(0)` — O(n), not O(1) |
| Go `s[0]` (a byte) | `s.as_bytes()[0]` |
| Go `for i, r := range s` | `for (i, c) in s.char_indices()` |

A Rust `String` is **always valid UTF-8**, enforced by the type system. UTF-8 is
variable-width, so there is no O(1) "nth character" — and rather than let you
write code that silently means "nth byte", Rust removes integer indexing
entirely.

## `s[0]` is a compile error

```rust
let s = String::from("hello");
let c = s[0];
```

Does not compile:

```
error[E0277]: the type `str` cannot be indexed by `{integer}`
   --> src/main.rs:3:15
    |
  3 |     let c = s[0];
    |               ^ string indices are ranges of `usize`
    |
    = help: the trait `SliceIndex<str>` is not implemented for `{integer}`
    = note: you can use `.chars().nth()` or `.bytes().nth()`
```

The compiler tells you the two answers. There is no `Index<usize>` impl for
`str` — only `Index<Range<usize>>`, which is why `&s[0..2]` *is* allowed.

```rust
let s = "hello";
println!("{:?}", s.chars().nth(0));
println!("{:?}", s.bytes().nth(0));
println!("{:?}", s.as_bytes()[0]);
println!("{:?}", &s[0..1]);
```

Output:

```
Some('h')
Some(104)
104
"h"
```

`chars().nth(i)` is O(i) — it decodes from the start. If you index in a loop,
collect to `Vec<char>` once instead.

## Byte, char, grapheme

- **byte** — one `u8`. 1–4 of them encode one `char`.
- **`char`** — one Unicode scalar value, always 4 bytes in memory, 1–4 bytes in
  UTF-8.
- **grapheme cluster** — what a human calls a character. Can be many `char`s.

```rust
for s in ["a", "é", "€", "🦀", "héllo", "日本語", "🇮🇳"] {
    println!("{:<8} len={} chars={} bytes={:?}", s, s.len(), s.chars().count(), s.as_bytes());
}
```

Output:

```
a        len=1 chars=1 bytes=[97]
é        len=2 chars=1 bytes=[195, 169]
€        len=3 chars=1 bytes=[226, 130, 172]
🦀        len=4 chars=1 bytes=[240, 159, 166, 128]
héllo    len=6 chars=5 bytes=[104, 195, 169, 108, 108, 111]
日本語      len=9 chars=3 bytes=[230, 151, 165, 230, 156, 172, 232, 170, 158]
🇮🇳       len=8 chars=2 bytes=[240, 159, 135, 174, 240, 159, 135, 179]
```

`"🦀".len()` is 4. The Indian flag is **one** user-visible character, **two**
`char`s (two regional-indicator scalars), and **eight** bytes. ASCII is the only
case where all three counts agree.

An accent can also arrive as a separate combining character:

```rust
let combining = "e\u{301}";
println!("{} len={} chars={}", combining, combining.len(), combining.chars().count());
println!("{}", combining == "é");
println!("{:?}", combining.chars().collect::<Vec<char>>());
```

Output:

```
é len=3 chars=2
false
['e', '\u{301}']
```

It renders identically to `"é"` but is **not equal** to it — different bytes.
Comparing user-entered text needs Unicode normalization (the
`unicode-normalization` crate); `==` is byte equality.

## `len()` is bytes, `chars().count()` is characters

```rust
let s = "héllo";
println!("{}", s.len());
println!("{}", s.chars().count());
```

Output:

```
6
5
```

`len()` is O(1) — it reads a field. `chars().count()` is O(n) — it decodes
everything. Use `len()` for buffer sizes and `is_empty()`, `chars().count()`
only when you genuinely mean characters.

## Slicing off a char boundary panics

Byte ranges are allowed, but the ends must fall between characters.

```rust
let s = "héllo";
println!("{}", &s[0..2]);
```

Panics at runtime (this example is intentionally broken):

```
thread 'main' panicked at src/main.rs:3:22:
end byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The message names the offending character and its byte span. With a 4-byte
character:

```rust
let s = "🦀rust";
println!("{}", &s[0..2]);
```

```
thread 'main' panicked at src/main.rs:3:22:
end byte index 2 is not a char boundary; it is inside '🦀' (bytes 0..4 of string)
```

Out of range is a different message:

```rust
let s = "abc";
println!("{}", &s[0..10]);
```

```
thread 'main' panicked at src/main.rs:3:22:
end byte index 10 is out of bounds for string of length 3
```

The same rule applies to `insert`, `insert_str`, `remove`, `truncate`, `drain`,
and `replace_range`:

```rust
let mut s = String::from("héllo");
s.remove(2);
```

```
thread 'main' panicked at src/main.rs:3:7:
start byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)
```

A `find`/`char_indices` index is always a valid boundary. Indices you compute
yourself are not.

## `is_char_boundary` and `get`

```rust
let s = "héllo";
for i in 0..=s.len() {
    println!("boundary({}) = {}", i, s.is_char_boundary(i));
}
```

Output:

```
boundary(0) = true
boundary(1) = true
boundary(2) = false
boundary(3) = true
boundary(4) = true
boundary(5) = true
boundary(6) = true
```

`0` and `len()` are always boundaries. `get` returns `None` instead of panicking:

```rust
let s = "héllo";
println!("{:?}", s.get(0..2));
println!("{:?}", s.get(0..3));
```

Output:

```
None
Some("hé")
```

To snap an arbitrary offset forward to a legal boundary:

```rust
let s = "日本語abc";
let mut i = 2;
while !s.is_char_boundary(i) {
    i += 1;
}
println!("snapped to {} -> {:?}", i, &s[..i]);
```

Output:

```
snapped to 3 -> "日"
```

## `char_indices` is the safe way to slice

It gives you byte offsets that are boundaries by construction.

```rust
let s = "日本語abc";
println!("{:?}", s.char_indices().collect::<Vec<_>>());
println!("{:?}", s.char_indices().nth(2));
println!("{:?}", s.char_indices().nth(3).map(|(i, _)| &s[i..]));
```

Output:

```
[(0, '日'), (3, '本'), (6, '語'), (9, 'a'), (10, 'b'), (11, 'c')]
Some((6, '語'))
Some("abc")
```

Offsets jump 0, 3, 6, 9 — three bytes per CJK character. For "first n
characters", skip the index arithmetic entirely:

```rust
let s = "日本語abc";
println!("{:?}", s.chars().take(3).collect::<String>());
println!("{:?}", s.chars().skip(3).collect::<String>());
```

Output:

```
"日本語"
"abc"
```

This allocates, but it cannot panic. Idiomatic for user-facing truncation.

## Reversing: naive vs correct

Reversing `chars` works for simple text:

```rust
let s = "héllo";
println!("{}", s.chars().rev().collect::<String>());
```

Output:

```
olléh
```

Reversing bytes corrupts it:

```rust
let s = "héllo";
println!("{}", s.bytes().rev().map(|b| b as char).collect::<String>());

let bytes: Vec<u8> = s.bytes().rev().collect();
println!("{:?}", String::from_utf8(bytes).is_err());
```

Output:

```
oll©Ãh
true
```

The reversed bytes are not valid UTF-8 at all — `from_utf8` rejects them. This
is why there is no `s.reverse()`.

But `chars().rev()` is also wrong once grapheme clusters appear. Add:

```
cargo add unicode-segmentation
```

```rust
use unicode_segmentation::UnicodeSegmentation;

let flag = "a🇮🇳b";
println!("{}", flag.chars().rev().collect::<String>());
println!("{}", flag.graphemes(true).rev().collect::<String>());
println!("{:?}", flag.graphemes(true).collect::<Vec<_>>());
println!("chars={} graphemes={}", flag.chars().count(), flag.graphemes(true).count());
```

Output:

```
b🇳🇮a
b🇮🇳a
["a", "🇮🇳", "b"]
chars=4 graphemes=3
```

`chars().rev()` reversed the two regional indicators and produced a *different
flag*. Grapheme reversal keeps it intact.

Combining accents break the same way:

```rust
use unicode_segmentation::UnicodeSegmentation;

let combo = "e\u{301}x";
println!("{}", combo.chars().rev().collect::<String>());
println!("{}", combo.graphemes(true).rev().collect::<String>());
println!("{:?}", combo.graphemes(true).collect::<Vec<_>>());
```

Output:

```
x́e
xé
["e\u{301}", "x"]
```

The naive reversal moved the accent onto the `x`.

`unicode-segmentation` also gives you word boundaries:

```rust
use unicode_segmentation::UnicodeSegmentation;
println!("{:?}", "the quick fox".unicode_words().collect::<Vec<_>>());
```

Output:

```
["the", "quick", "fox"]
```

The standard library deliberately stops at `char`: grapheme rules are large,
versioned Unicode tables that don't belong in `core`. Reach for the crate
whenever you count, truncate, or reverse text a human will look at.

## Case folding is not reversible

```rust
println!("{:?}", "ß".to_uppercase());
println!("{:?}", "ß".to_uppercase().to_lowercase());
println!("{}", "ß".len());
println!("{}", "ß".to_uppercase().len());
println!("{:?}", "ﬁ".to_uppercase());
```

Output:

```
"SS"
"ss"
2
2
"FI"
```

`ß` round-trips to `ss`, not back to `ß`. The ligature `ﬁ` becomes two letters.
Never assume `s.to_uppercase().to_lowercase() == s.to_lowercase()`.

Turkish `İ` lowercases to **two** `char`s:

```rust
println!("{:?}", "İ".to_lowercase());
println!("{}", "İ".to_lowercase().chars().count());
println!("{:?}", "ı".to_uppercase());
```

Output:

```
"i\u{307}"
2
"I"
```

And `ı` (dotless i) uppercases to plain `I`, which lowercases to plain `i` — a
different letter in Turkish. Rust's `to_lowercase` is locale-independent by
design; if you need Turkish rules, you need a locale-aware library. For
case-insensitive *matching*, prefer `eq_ignore_ascii_case` on identifiers you
control and full lowercasing on everything else — and never case-fold a
password, username, or security token.

## Quick Reference

| Want | Call | Cost |
|---|---|---|
| Byte length | `s.len()` | O(1) |
| Character count | `s.chars().count()` | O(n) |
| Grapheme count | `s.graphemes(true).count()` | O(n), needs crate |
| First char | `s.chars().next()` | O(1) |
| Last char | `s.chars().last()` | O(n) |
| i-th char | `s.chars().nth(i)` | O(i) |
| i-th byte | `s.as_bytes()[i]` | O(1) |
| Char + byte offset | `s.char_indices()` | O(n) |
| First n chars | `s.chars().take(n).collect::<String>()` | O(n), allocates |
| Safe slice | `s.get(a..b) -> Option<&str>` | O(1) |
| Boundary check | `s.is_char_boundary(i)` | O(1) |
| Reverse (simple) | `s.chars().rev().collect::<String>()` | O(n) |
| Reverse (correct) | `s.graphemes(true).rev().collect::<String>()` | O(n), needs crate |

| Operation | Panics when |
|---|---|
| `&s[a..b]` | `a`/`b` off a boundary or out of range |
| `insert`, `insert_str`, `remove`, `truncate` | index off a boundary |
| `drain`, `replace_range` | range end off a boundary |
| `get`, `chars().nth`, `pop` | never — they return `Option` |

## Common Mistakes

**Writing `s[0]`.** There is no integer indexing. Use `s.chars().next()` or
`s.as_bytes()[0]`.

**Slicing with a computed offset.** `&s[..n]` panics mid-character. Use
`s.get(..n)` or `s.chars().take(n)`.

**Using `len()` to limit display width.** It is bytes. Truncate by graphemes.

**Reversing with `bytes().rev()`.** The result isn't valid UTF-8. Reverse
`chars()` at minimum, graphemes for real text.

**Comparing accented input with `==`.** `"e\u{301}"` != `"é"` despite looking
identical. Normalize first.

**Assuming case conversion preserves length.** `ß` → `SS`, `İ` → two chars.

## Exercises

1. Write `fn describe(s: &str)` printing byte length, char count, and the byte
   length of each char. Run it on `"a"`, `"é"`, `"€"`, `"🦀"`, and `"🇮🇳"`.
2. Write `fn truncate_chars(s: &str, n: usize) -> String` that never panics.
   Test it with `n = 1` on `"🦀🦀🦀"`.
3. Deliberately slice `"日本語"` at byte 1 and record the exact panic message.
   Then rewrite the same slice safely with `get`.
4. Write `fn safe_prefix(s: &str, max_bytes: usize) -> &str` that returns the
   longest prefix no longer than `max_bytes` ending on a char boundary, using
   `is_char_boundary`.
5. Add `unicode-segmentation` and write `fn grapheme_reverse(s: &str) -> String`.
   Compare it against a `chars().rev()` version on `"a🇮🇳a"` and on
   `"xe\u{301}"`, and show the two disagree.

---

[← Course Index](../README.md)
