[← Course Index](../README.md)

# 6.3 — String Operations Reference

## What You'll Learn

- Every `String`/`str` method you will actually reach for, grouped by purpose
- What each one returns — `()`, `Option`, `Result`, or a new `String`
- Which methods mutate in place and which allocate
- Which ones panic, and on what

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `s.length` | `s.len()` — **bytes**, not characters |
| JS `s.slice(a, b)` | `&s[a..b]` — byte range, panics off-boundary |
| JS `s.at(0)` | `s.chars().next()` — there is no indexing |
| Python `s.strip()` | `s.trim()` |
| Python `s.upper()` | `s.to_uppercase()` (new `String`) |
| Go `len(s)` | `s.len()` — identical semantics (bytes) |

Two rules carry across every method here: indices are **byte** offsets, and
anything that produces different text returns a **new** `String` rather than
mutating. The mutating methods are the small set that takes `&mut self`.

## Inspection

| Method | Signature | Returns | Note |
|---|---|---|---|
| `len` | `&self -> usize` | byte count | Not character count |
| `is_empty` | `&self -> bool` | `len() == 0` | Cheaper than comparing to `""` |
| `capacity` | `&self -> usize` | allocated bytes | `String` only |
| `chars().count()` | — | `usize` | O(n) character count |
| `bytes().len()` | — | `usize` | Same as `len()` |
| `lines().count()` | — | `usize` | Line count |

```rust
let s = String::from("Hello, Rust!");
println!("{}", s.len());
println!("{}", s.is_empty());
println!("{}", s.capacity());
println!("{}", "🦀".len());
println!("{}", "héllo".len());
println!("{}", "héllo".chars().count());
println!("{}", "a\nb\nc".lines().count());
```

Output:

```
12
false
12
4
6
5
3
```

`"🦀".len()` is 4 and `"héllo".len()` is 6 for 5 characters. `len()` is bytes,
always. Lesson 6.5 goes into why.

## Access

| Method | Signature | Returns | Note |
|---|---|---|---|
| `chars` | `&self -> Chars` | iterator of `char` | Decodes UTF-8 |
| `bytes` | `&self -> Bytes` | iterator of `u8` | Raw bytes |
| `char_indices` | `&self -> CharIndices` | `(usize, char)` pairs | Index is a **byte** offset |
| `lines` | `&self -> Lines` | iterator of `&str` | Strips `\n` and `\r\n` |
| `get` | `&self, range -> Option<&str>` | `Some`/`None` | Non-panicking slice |
| `as_str` | `&String -> &str` | borrowed view | `String` only |
| `as_bytes` | `&self -> &[u8]` | borrowed bytes | Zero cost |
| `chars().next()` | — | `Option<char>` | First char |
| `chars().last()` | — | `Option<char>` | Last char, O(n) |
| `chars().nth(i)` | — | `Option<char>` | i-th char, O(n) |
| `&s[a..b]` | — | `&str` | **Panics** off-boundary |

```rust
let s = "héllo";
println!("{:?}", s.chars().collect::<Vec<char>>());
println!("{:?}", s.char_indices().collect::<Vec<_>>());
println!("{:?}", s.as_bytes());
```

Output:

```
['h', 'é', 'l', 'l', 'o']
[(0, 'h'), (1, 'é'), (3, 'l'), (4, 'l'), (5, 'o')]
[104, 195, 169, 108, 108, 111]
```

`char_indices` skips index 2 — `é` occupies bytes 1 and 2. `bytes()` yields the
same values as `as_bytes()`, one at a time.

Element access returns `Option` — there is no panicking variant. `get` is the
safe form of slicing:

```rust
let s = "héllo";
println!("{:?}", s.chars().next());
println!("{:?}", s.chars().last());
println!("{:?}", s.chars().nth(1));
println!("{:?}", s.chars().nth(99));
println!("{:?}", s.get(0..1));
println!("{:?}", s.get(0..2));
println!("{:?}", s.get(0..100));
```

Output:

```
Some('h')
Some('o')
Some('é')
None
Some("h")
None
None
```

`get(0..2)` splits `é` in half, so it is `None` rather than a panic. Use `get`
whenever the range comes from user input.

```rust
println!("{:?}", "".lines().collect::<Vec<_>>());
println!("{:?}", "a\r\nb".lines().collect::<Vec<_>>());
println!("{:?}", "a\nb\n".lines().collect::<Vec<_>>());
```

Output:

```
[]
["a", "b"]
["a", "b"]
```

`lines()` handles `\r\n` and ignores a single trailing newline. An empty string
yields zero lines, not one. On a `String`, `as_str()` and `as_bytes()` give the
borrowed `&str` and `&[u8]` views for free.

## Modification

All of these need `&mut String`. Indices are byte offsets.

| Method | Signature | Returns | Note |
|---|---|---|---|
| `push` | `&mut self, char` | `()` | Append one char |
| `push_str` | `&mut self, &str` | `()` | Append a slice |
| `insert` | `&mut self, usize, char` | `()` | O(n); panics off-boundary |
| `insert_str` | `&mut self, usize, &str` | `()` | O(n); panics off-boundary |
| `remove` | `&mut self, usize -> char` | the removed char | **Panics** if out of range/off-boundary |
| `pop` | `&mut self -> Option<char>` | last char | `None` when empty |
| `truncate` | `&mut self, usize` | `()` | Keeps first n **bytes**; panics off-boundary |
| `clear` | `&mut self` | `()` | len → 0, capacity kept |
| `retain` | `&mut self, FnMut(char)->bool` | `()` | In-place filter |
| `drain` | `&mut self, range -> Drain` | iterator of removed chars | Removes the range |
| `replace_range` | `&mut self, range, &str` | `()` | Replacement may differ in length |
| `extend` | `&mut self, IntoIterator` | `()` | Append from an iterator |

```rust
let mut s = String::from("hello world");
println!("{:?}", s.remove(0));
println!("{:?}", s.pop());
s.truncate(4);
println!("{} len={} cap={}", s, s.len(), s.capacity());
```

Output:

```
'h'
Some('d')
ello len=4 cap=11
```

`remove` returns the char it took out; `pop` returns an `Option` because the
string may be empty (`String::new().pop()` is `None`). `truncate` keeps the
capacity.

`retain` filters in place with a predicate, and `drain` removes a range and
hands you back what it took:

```rust
let mut r = String::from("a1b2c3");
r.retain(|c| c.is_alphabetic());
println!("{}", r);

let mut r2 = String::from("a1b2c3");
r2.retain(char::is_numeric);
println!("{}", r2);

let mut d = String::from("hello world");
let taken: String = d.drain(0..6).collect();
println!("taken={:?} left={:?}", taken, d);

let mut d2 = String::from("abcdef");
d2.drain(..2);
println!("{}", d2);
```

Output:

```
abc
123
taken="hello " left="world"
cdef
```

Dropping the `Drain` without collecting still removes the range, so `drain(..)`
is `clear()` with a return value.

`replace_range` substitutes text of any length; an empty range inserts without
removing:

```rust
let mut rr = String::from("hello world");
rr.replace_range(0..5, "goodbye");
println!("{}", rr);

let mut rr2 = String::from("abcdef");
rr2.replace_range(2.., "XY");
rr2.replace_range(..0, ">>");
println!("{}", rr2);
```

Output:

```
goodbye world
>>abXY
```

`clear` resets the length and keeps the buffer; the appending methods compose
in the obvious way:

```rust
let mut c = String::from("stuff");
c.clear();
println!("[{}] len={} cap={}", c, c.len(), c.capacity());

let mut p = String::from("ab");
p.push('c');
p.push_str("de");
p.insert(0, '!');
p.insert_str(0, "<<");
p.extend(['>', '>']);
println!("{}", p);

let mut t = String::from("é-x");
t.truncate(2);
println!("{:?}", t);
```

Output:

```
[] len=0 cap=5
<<!abcde>>
"é"
```

`truncate` cuts at a **byte** index: byte 2 is the boundary after the two-byte
`é`, and `truncate(1)` would panic.

## Trimming

All return a borrowed `&str` — nothing is allocated.

| Method | Signature | Returns | Note |
|---|---|---|---|
| `trim` | `&self -> &str` | both ends | Unicode whitespace |
| `trim_start` | `&self -> &str` | leading | |
| `trim_end` | `&self -> &str` | trailing | |
| `trim_matches` | `&self, P -> &str` | both ends | **Repeats**; pattern must be char/closure/char-array |
| `trim_start_matches` | `&self, P -> &str` | leading | Repeats; accepts `&str` too |
| `trim_end_matches` | `&self, P -> &str` | trailing | Repeats; accepts `&str` too |
| `strip_prefix` | `&self, P -> Option<&str>` | `Some` if matched | Removes **once** |
| `strip_suffix` | `&self, P -> Option<&str>` | `Some` if matched | Removes **once** |

```rust
let s = "  \t hello \n ";
println!("{:?}", s.trim());
println!("{:?}", s.trim_start());
println!("{:?}", s.trim_end());
```

Output:

```
"hello"
"hello \n "
"  \t hello"
```

Pattern-based trimming repeats until the pattern no longer matches:

```rust
println!("{:?}", "xxhixx".trim_matches('x'));
println!("{:?}", "xxhixx".trim_start_matches('x'));
println!("{:?}", "xxhixx".trim_end_matches('x'));
println!("{:?}", "--1,2--".trim_matches(|c: char| !c.is_alphanumeric()));
println!("{:?}", "xyzhixyz".trim_matches(['x', 'y', 'z']));
```

Output:

```
"hi"
"hixx"
"xxhi"
"1,2"
"hi"
```

A `&str` pattern works on the one-sided versions only; `trim_matches` with a
`&str` does not compile, because trimming from both ends needs a reversible
searcher:

```rust
println!("{:?}", "ababhiabab".trim_start_matches("ab"));
println!("{:?}", "www.rust.org".trim_start_matches("www."));
println!("{:?}", "file.txt".trim_end_matches(".txt"));
println!("{:?}", "a.txt.txt".trim_end_matches(".txt"));
```

Output:

```
"hiabab"
"rust.org"
"file"
"a"
```

That last line is the trap: `trim_end_matches` strips **every** repetition, so
`"a.txt.txt"` becomes `"a"`. Use `strip_suffix` when you mean "remove one".

```rust
println!("{:?}", "pre-body".strip_prefix("pre-"));
println!("{:?}", "pre-body".strip_prefix("nope"));
println!("{:?}", "body.txt".strip_suffix(".md"));
println!("{:?}", "a.txt.txt".strip_suffix(".txt"));
println!("{:?}", "xabc".strip_prefix('x'));
```

Output:

```
Some("body")
None
None
Some("a.txt")
Some("abc")
```

`strip_*` returning `Option` lets you branch on whether the affix was there —
something `trim_*` cannot tell you. Idiomatic: `strip_prefix` for parsing,
`trim` for cleaning input.

## Case

| Method | Signature | Returns | Note |
|---|---|---|---|
| `to_uppercase` | `&self -> String` | new | Full Unicode; may change length |
| `to_lowercase` | `&self -> String` | new | Full Unicode |
| `to_ascii_uppercase` | `&self -> String` | new | Only `a-z`; non-ASCII untouched |
| `to_ascii_lowercase` | `&self -> String` | new | Only `A-Z` |
| `make_ascii_uppercase` | `&mut self` | `()` | In place, no allocation |
| `make_ascii_lowercase` | `&mut self` | `()` | In place |
| `eq_ignore_ascii_case` | `&self, &str -> bool` | comparison | ASCII only |

```rust
let s = "Hello, Straße! ÅÉÎ";
println!("{}", s.to_uppercase());
println!("{}", s.to_lowercase());
println!("{}", s.to_ascii_uppercase());
println!("{}", s.to_ascii_lowercase());
```

Output:

```
HELLO, STRASSE! ÅÉÎ
hello, straße! åéî
HELLO, STRAßE! ÅÉÎ
hello, straße! ÅÉÎ
```

The Unicode versions map `ß → SS` and `Å → å`; the ASCII versions leave both
alone. Case conversion is not length-preserving — `"ß".to_uppercase()` is `"SS"`,
one character becoming two — which is why these return a `String` rather than
mutating in place.

```rust
println!("{}", "HELLO".eq_ignore_ascii_case("hello"));
println!("{}", "Straße".eq_ignore_ascii_case("STRASSE"));
println!("{}", "ÉCOLE".eq_ignore_ascii_case("école"));
println!("{}", "ÉCOLE".to_lowercase() == "école");
```

Output:

```
true
false
false
true
```

`eq_ignore_ascii_case` is fast and allocation-free but blind to non-ASCII. For
anything international, lowercase both sides and compare.

The `make_ascii_*` pair mutates in place with no allocation, which is only
possible because ASCII case mapping never changes the byte count.

## Comparison

| Operation | Returns | Note |
|---|---|---|
| `==` / `!=` | `bool` | Byte-exact; works across `String`/`&str` |
| `<` `>` `<=` `>=` | `bool` | Lexicographic by bytes |
| `cmp` | `Ordering` | `Less`/`Equal`/`Greater` |
| `partial_cmp` | `Option<Ordering>` | Always `Some` for strings |
| `starts_with` | `bool` | Pattern |
| `ends_with` | `bool` | Pattern |
| `contains` | `bool` | Pattern |

```rust
let a = String::from("apple");
println!("{}", a == "apple");
println!("{}", a == String::from("apple"));
println!("{}", "apple" < "banana");
println!("{}", "Apple" < "apple");
println!("{}", "abc" < "abcd");
println!("{:?}", "a".cmp("b"));
println!("{:?}", "b".cmp("a"));
println!("{:?}", "a".cmp("a"));
println!("{:?}", "a".partial_cmp("b"));
println!("{:?}", "10".cmp("9"));
```

Output:

```
true
true
true
true
true
Less
Greater
Equal
Some(Less)
Less
```

`"Apple" < "apple"` because uppercase bytes are numerically smaller, and
`"10" < "9"` because ordering is by bytes, not dictionary or numeric rules.
Parse first if you want numbers.

```rust
let a = String::from("apple");
println!("{}", a.starts_with("app"));
println!("{}", a.starts_with(char::is_alphabetic));
println!("{}", a.starts_with(['x', 'a']));
println!("{}", a.ends_with("le"));
println!("{}", a.contains("ppl"));
println!("{}", a.contains('z'));
println!("{}", a.contains(""));
```

Output:

```
true
true
true
true
true
false
true
```

All three accept the same four pattern kinds (`char`, `&str`, char array,
closure) — lesson 6.4 covers patterns in full. Every string contains `""`.

Sorting a `Vec<&str>` with `.sort()` uses the same byte ordering, so uppercase
names sort before lowercase ones; `.sort_by_key(|s| s.to_lowercase())` is the
case-insensitive form.

## Padding and alignment

There is no `pad_left`/`ljust` method — padding is a `format!` feature. The spec
is `{:[fill][align][width]}` with align one of `<` `^` `>`.

```rust
println!("[{:>8}]", "hi");
println!("[{:<8}]", "hi");
println!("[{:^8}]", "hi");
println!("[{:*^8}]", "hi");
println!("[{:08}]", 42);
println!("[{:>3}]", "longer text");

let w = 10;
println!("[{:>w$}]", "dyn");

for (name, qty) in [("apple", 3), ("kiwi", 12)] {
    println!("{:<10}{:>5}", name, qty);
}
```

Output:

```
[      hi]
[hi      ]
[   hi   ]
[***hi***]
[00000042]
[longer text]
[       dyn]
apple         3
kiwi         12
```

The fill character comes **before** the alignment arrow, and width is a minimum
— longer input is never truncated. Width counts `char`s, not display columns, so
emoji and CJK text will not line up in a terminal. Lesson 6.6 covers the rest of
the format spec.

## Conversion

| From → To | Call | Returns |
|---|---|---|
| `&str` → number | `s.parse::<T>()` | `Result<T, _>` |
| `&str` → `&[u8]` | `s.as_bytes()` | borrowed, zero cost |
| `String` → `Vec<u8>` | `s.into_bytes()` | consumes, zero copy |
| `Vec<u8>` → `String` | `String::from_utf8(v)` | `Result` |
| `&str` → `Vec<char>` | `s.chars().collect()` | allocates |
| `&str` → `Vec<u8>` | `s.bytes().collect()` | allocates |
| anything `Display` → `String` | `x.to_string()` | allocates |
| `String` → `Box<str>` | `s.into_boxed_str()` | drops spare capacity |
| anything `Debug` → `String` | `format!("{x:?}")` | allocates |

```rust
println!("{:?}", "42".parse::<i32>());
println!("{:?}", "3.5".parse::<f64>());
println!("{}", "abc".parse::<i32>().is_err());
println!("{:?}", "abc".as_bytes());
println!("{:?}", "abc".chars().collect::<Vec<char>>());
println!("{}", 42.to_string());
println!("{}", 3.5.to_string());
println!("{:?}", String::from("abc").into_bytes());
println!("{}", format!("{:?}", "a\nb"));
```

Output:

```
Ok(42)
Ok(3.5)
true
[97, 98, 99]
['a', 'b', 'c']
42
3.5
[97, 98, 99]
"a\nb"
```

`{:?}` on a string adds the quotes and escapes — useful for seeing whitespace.

`into_bytes` consumes the `String` and reuses the same allocation — no copy —
and `String::from_utf8` turns the bytes back. Lesson 6.7 covers parsing and byte
conversion in depth.

## Quick Reference

| Returns | Methods |
|---|---|
| `()` (mutates) | `push`, `push_str`, `insert`, `insert_str`, `truncate`, `clear`, `retain`, `replace_range`, `extend`, `reserve`, `shrink_to_fit`, `make_ascii_uppercase`, `make_ascii_lowercase` |
| `Option<_>` | `pop`, `get`, `strip_prefix`, `strip_suffix`, `chars().next/last/nth`, `find`, `split_once` |
| `Result<_, _>` | `parse`, `String::from_utf8` |
| `&str` (borrowed, free) | `trim*`, `as_str`, `&s[..]` |
| New `String` | `to_uppercase`, `to_lowercase`, `to_ascii_*case`, `replace`, `repeat`, `concat`, `join`, `format!`, `to_string` |
| `bool` | `is_empty`, `starts_with`, `ends_with`, `contains`, `eq_ignore_ascii_case`, `is_char_boundary` |
| `usize` / `char` | `len`, `capacity`, `remove` |
| Iterator | `chars`, `bytes`, `char_indices`, `lines`, `split*`, `matches`, `drain` |

| Panics on | Method |
|---|---|
| Index out of range or off char boundary | `&s[a..b]`, `insert`, `insert_str`, `remove`, `truncate`, `drain`, `replace_range` |
| Never (returns `Option`) | `get`, `pop`, `find`, `strip_prefix`, `strip_suffix` |

## Common Mistakes

**Using `len()` as a character count.** It is bytes. Use `chars().count()` when
you mean characters.

**`trim_end_matches(".txt")` to drop one extension.** It strips every
repetition. Use `strip_suffix`.

**`eq_ignore_ascii_case` on international text.** It only folds `A-Z`.
Lowercase both sides with `to_lowercase()` instead.

**Expecting `to_uppercase` to keep the length.** `ß` becomes `SS`. Never assume
a byte-for-byte mapping.

**`clear()` to release memory.** It resets the length only; capacity stays.

**Reaching for a `pad_start` method.** Padding lives in `format!`:
`format!("{:>8}", s)`.

## Exercises

1. Write `fn stats(s: &str)` printing byte length, char count, line count, and
   whether it is empty. Test it on `""`, `"a"`, `"héllo"`, and `"🦀🦀"`.
2. Write `fn clean(s: &str) -> String` that trims whitespace, lowercases, and
   removes every non-alphanumeric character. Use `retain`.
3. Write `fn drop_ext(name: &str) -> &str` that removes exactly one `.txt`
   suffix if present. Verify `"a.txt.txt"` becomes `"a.txt"`.
4. Given `let mut s = String::from("The quick brown fox");`, use `drain` to
   remove `"quick "` and `replace_range` to turn `"fox"` into `"dog"`. Print
   after each step.
5. Print a right-aligned table of five `(name, score)` pairs with names in a
   12-wide left column and scores in a 6-wide right column, separated by a
   `-`-filled line of the same total width.

---

[← Course Index](../README.md)
