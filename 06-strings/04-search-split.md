[← Course Index](../README.md)

# 6.4 — Searching, Splitting, and Replacing

## What You'll Learn

- `find`/`rfind` and the `Option<usize>` they return
- Every split variant and exactly how each treats empty fields
- `matches`, `match_indices`, `replace`, `replacen`
- The four pattern kinds, and when to reach for the `regex` crate

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `s.indexOf(x)` → `-1` | `s.find(x)` → `None` |
| JS `s.split(",")` → array | `s.split(',')` → lazy iterator |
| JS `s.replace(a, b)` (first only) | `s.replacen(a, b, 1)` |
| JS `s.replaceAll(a, b)` | `s.replace(a, b)` |
| Python `s.partition(",")` | `s.split_once(',')` → `Option` |
| Go `strings.Split` | `s.split(..).collect::<Vec<_>>()` |

Two differences dominate: there is no `-1` sentinel (you get an `Option`), and
splits are **lazy iterators** — you call `.collect()` only if you want a `Vec`.
`replace` is replace-all, which catches JS developers out.

## `find` and `rfind`

Both return the **byte** offset of a match, or `None`.

```rust
let s = "the quick brown fox the end";
println!("{:?}", s.find("the"));
println!("{:?}", s.rfind("the"));
println!("{:?}", s.find('q'));
println!("{:?}", s.find("zebra"));
println!("{:?}", s.find(""));
println!("{:?}", "".find("a"));
```

Output:

```
Some(0)
Some(20)
Some(4)
None
Some(0)
None
```

An empty pattern matches at 0. There is no `-1`, so you cannot accidentally use
a failure as an index.

```rust
let s = "the quick brown fox the end";
println!("{:?}", s.find(char::is_whitespace));
println!("{:?}", s.rfind(char::is_whitespace));
println!("{:?}", s.find(['x', 'o']));
println!("{:?}", "héllo".find('l'));
if let Some(i) = s.find("brown") {
    println!("{:?} / {:?}", &s[..i], &s[i..]);
}
println!("{:?}", s.find("the").map(|i| &s[i..i + 3]));
```

Output:

```
Some(3)
Some(23)
Some(12)
Some(3)
"the quick " / "brown fox the end"
Some("the")
```

In `"héllo"`, `'l'` is the third character but byte 3, because `é` takes two
bytes. A `find` index is always a char boundary, so slicing at it is safe.

## Patterns: the four kinds

Every search/split/trim/replace method takes a `Pattern`. There are four
implementations, and they are interchangeable.

```rust
let s = "a-b_c1d";
println!("{:?}", s.split('-').collect::<Vec<_>>());
println!("{:?}", s.split("_b").collect::<Vec<_>>());
println!("{:?}", s.split(['-', '_']).collect::<Vec<_>>());
println!("{:?}", s.split(char::is_numeric).collect::<Vec<_>>());
```

Output:

```
["a", "b_c1d"]
["a-b_c1d"]
["a", "b", "c1d"]
["a-b_c", "d"]
```

| Pattern | Written as | Matches |
|---|---|---|
| `char` | `'-'` | That one character |
| `&str` | `"ab"` | That exact substring |
| `[char; N]` / `&[char]` | `['-', '_']` | Any one of those chars |
| `FnMut(char) -> bool` | `char::is_numeric`, `\|c: char\| c == 'x'` | Any char satisfying it |

A `&str` pattern matches the **whole substring**, not any of its characters —
that's the difference between `split("_b")` and `split(['-', '_'])` above.

Note `&String` is also a pattern, and a closure must usually be annotated
`|c: char|` so inference can pick the right impl.

## The split family

| Method | Returns | Empty fields | Note |
|---|---|---|---|
| `split` | iterator of `&str` | kept | Baseline |
| `splitn(n, p)` | iterator | kept | At most `n` pieces; last holds the rest |
| `rsplit` | iterator | kept | Same pieces, reverse order |
| `rsplitn(n, p)` | iterator | kept | Splits from the right |
| `split_once(p)` | `Option<(&str, &str)>` | — | `None` if no match |
| `rsplit_once(p)` | `Option<(&str, &str)>` | — | Splits at the last match |
| `split_whitespace` | iterator | **dropped** | Any Unicode whitespace run |
| `split_ascii_whitespace` | iterator | dropped | ASCII only, faster |
| `split_terminator` | iterator | no trailing empty | For trailing-delimiter data |
| `split_inclusive` | iterator | — | Keeps the delimiter on each piece |
| `lines` | iterator | — | Splits on `\n`/`\r\n`, drops one trailing |

```rust
let csv = "a,b,c,d";
println!("{:?}", csv.split(',').collect::<Vec<_>>());
println!("{:?}", csv.splitn(2, ',').collect::<Vec<_>>());
println!("{:?}", csv.splitn(3, ',').collect::<Vec<_>>());
println!("{:?}", csv.rsplit(',').collect::<Vec<_>>());
println!("{:?}", csv.rsplitn(2, ',').collect::<Vec<_>>());
println!("{:?}", csv.split(',').rev().collect::<Vec<_>>());
```

Output:

```
["a", "b", "c", "d"]
["a", "b,c,d"]
["a", "b", "c,d"]
["d", "c", "b", "a"]
["d", "a,b,c"]
["d", "c", "b", "a"]
```

`splitn(2, ..)` is the "key and everything else" tool. `rsplit` differs from
`split().rev()` only in that `rsplitn` can stop early from the right.

### `split_once` and `rsplit_once`

The cleanest way to parse `key=value` — one match, a tuple, no iterator.

```rust
let csv = "a,b,c,d";
println!("{:?}", csv.split_once(','));
println!("{:?}", csv.rsplit_once(','));
println!("{:?}", "noseparator".split_once(','));
println!("{:?}", "a=".split_once('='));
println!("{:?}", "=a".split_once('='));
```

Output:

```
Some(("a", "b,c,d"))
Some(("a,b,c", "d"))
None
Some(("a", ""))
Some(("", "a"))
```

`None` distinguishes "no delimiter" from "empty value" — something `splitn`
cannot express.

### Empty fields

```rust
println!("{:?}", "a,,b".split(',').collect::<Vec<_>>());
println!("{:?}", ",a,".split(',').collect::<Vec<_>>());
println!("{:?}", "".split(',').collect::<Vec<_>>());
println!("{:?}", "abc".split("").collect::<Vec<_>>());
```

Output:

```
["a", "", "b"]
["", "a", ""]
[""]
["", "a", "b", "c", ""]
```

`split` is faithful: an empty string yields **one** empty field, not zero. An
empty pattern matches between every char plus both ends.

### Whitespace splitting

```rust
println!("{:?}", "a b\tc\n d".split_whitespace().collect::<Vec<_>>());
println!("{:?}", "   ".split_whitespace().collect::<Vec<_>>());
println!("{:?}", "a b".split_ascii_whitespace().collect::<Vec<_>>());
println!("{:?}", "a b  c".split(' ').collect::<Vec<_>>());
```

Output:

```
["a", "b", "c", "d"]
[]
["a", "b"]
["a", "b", "", "c"]
```

`split_whitespace` collapses runs and drops empties; `split(' ')` does not. For
word splitting always use `split_whitespace`.

### Terminator and inclusive

```rust
println!("{:?}", "a.b.c.".split_terminator('.').collect::<Vec<_>>());
println!("{:?}", "a.b.c".split_terminator('.').collect::<Vec<_>>());
println!("{:?}", "a.b.c.".split('.').collect::<Vec<_>>());
println!("{:?}", "a.b.c.".split_inclusive('.').collect::<Vec<_>>());
println!("{:?}", "a.b.c".split_inclusive('.').collect::<Vec<_>>());
```

Output:

```
["a", "b", "c"]
["a", "b", "c"]
["a", "b", "c", ""]
["a.", "b.", "c."]
["a.", "b.", "c"]
```

`split_terminator` treats the delimiter as a line-ending rather than a
separator — exactly what you want for `"a;b;c;"`. `split_inclusive` keeps the
delimiter attached, so you can reassemble the input exactly.

### `lines`

```rust
println!("{:?}", "l1\nl2\n".lines().collect::<Vec<_>>());
```

Output:

```
["l1", "l2"]
```

It is `split_terminator('\n')` that also strips a `\r`.

## `matches`, `match_indices`, and the reverse forms

| Method | Returns | Note |
|---|---|---|
| `matches(p)` | iterator of `&str` | The matched text, left to right |
| `rmatches(p)` | iterator of `&str` | Right to left |
| `match_indices(p)` | iterator of `(usize, &str)` | Byte offset + text |
| `rmatch_indices(p)` | iterator of `(usize, &str)` | Right to left |
| `matches(p).count()` | `usize` | Occurrence count |

```rust
let s = "abcabcabc";
println!("{:?}", s.matches("abc").collect::<Vec<_>>());
println!("{}", s.matches("abc").count());
println!("{:?}", s.match_indices("abc").collect::<Vec<_>>());
println!("{:?}", s.rmatch_indices("bc").collect::<Vec<_>>());
```

Output:

```
["abc", "abc", "abc"]
3
[(0, "abc"), (3, "abc"), (6, "abc")]
[(7, "bc"), (4, "bc"), (1, "bc")]
```

`matches(p).count()` is how you count occurrences — there is no `count_of`.

```rust
println!("{:?}", "a1b2".matches(char::is_numeric).collect::<Vec<_>>());
println!("{:?}", "aaa".matches("aa").collect::<Vec<_>>());
println!("{:?}", "xyz".matches("q").collect::<Vec<_>>());
```

Output:

```
["1", "2"]
["aa"]
[]
```

Matches never overlap: `"aaa"` contains one `"aa"`, not two. The scan resumes
after each match.

## `replace` and `replacen`

Both return a **new** `String`; the original is untouched.

```rust
let s = "abcabcabc";
println!("{}", s.replace("abc", "X"));
println!("{}", s.replacen("abc", "X", 2));
println!("{}", s.replacen("abc", "X", 0));
println!("{}", s.replacen("abc", "X", 99));
```

Output:

```
XXX
XXabc
abcabcabc
XXX
```

`replace` is **replace-all**. `replacen(.., 1)` is the JS `replace` equivalent.
A count beyond the number of matches is harmless.

```rust
println!("{}", "a b c".replace(' ', ""));
println!("{}", "a1b2".replace(char::is_numeric, "#"));
println!("{}", "a-b_c".replace(['-', '_'], "."));
println!("{}", "hello".replace("", "-"));
println!("{}", "aaa".replace("aa", "b"));

let owned = String::from("x-y");
println!("{}", owned.replace('-', "+"));
println!("{}", owned);
```

Output:

```
abc
a#b#
a.b.c
-h-e-l-l-o-
ba
x+y
x-y
```

Replacing with `""` deletes, an empty pattern inserts everywhere, and matches
are non-overlapping again (`"aaa"` → `"ba"`). Nothing mutates in place — use
`replace_range` (lesson 6.3) for that.

## Splitting CSV-ish input

```rust
let input = "name,age,city\nraj,30,Pune\nsam,25,Delhi";
let mut lines = input.lines();
let headers: Vec<&str> = lines.next().unwrap().split(',').collect();
println!("{:?}", headers);

for line in lines {
    let fields: Vec<&str> = line.split(',').collect();
    let pairs: Vec<String> = headers
        .iter()
        .zip(&fields)
        .map(|(h, f)| format!("{h}={f}"))
        .collect();
    println!("{}", pairs.join(" "));
}
```

Output:

```
["name", "age", "city"]
name=raj age=30 city=Pune
name=sam age=25 city=Delhi
```

Every `&str` here borrows from `input` — no allocation per field. This is fine
for well-formed data; real CSV with quoted commas needs the `csv` crate.

A config parser with `split_once`:

```rust
let cfg = "host = localhost\nport=8080\n# comment\nbad line";
for line in cfg.lines() {
    if line.starts_with('#') {
        continue;
    }
    match line.split_once('=') {
        Some((k, v)) => println!("{:?} -> {:?}", k.trim(), v.trim()),
        None => println!("skipping {:?}", line),
    }
}
```

Output:

```
"host" -> "localhost"
"port" -> "8080"
skipping "bad line"
```

Path splitting:

```rust
let path = "/usr/local/bin/rustc";
println!("{:?}", path.rsplit_once('/'));
println!("{:?}", path.split('/').filter(|s| !s.is_empty()).collect::<Vec<_>>());
println!("{:?}", path.rsplit('/').next());
```

Output:

```
Some(("/usr/local/bin", "rustc"))
["usr", "local", "bin", "rustc"]
Some("rustc")
```

Split-then-parse:

```rust
let nums = "1, 2,3 , 4";
let parsed: Vec<i32> = nums.split(',').map(|t| t.trim().parse().unwrap()).collect();
println!("{:?}", parsed);
println!("{}", parsed.iter().sum::<i32>());
```

Output:

```
[1, 2, 3, 4]
10
```

Always `trim()` between split and parse — `" 2".parse::<i32>()` is an error.
Lesson 6.7 covers parsing failures.

## Regex, when the std methods run out

Regex is **not** in the standard library. Add it:

```
cargo add regex
```

```toml
[dependencies]
regex = "1.13.1"
```

```rust
use regex::Regex;

let re = Regex::new(r"(\w+)@(\w+)\.com").unwrap();
let text = "mail raj@example.com or sam@test.com now";

println!("{}", re.is_match(text));
println!("{:?}", re.find(text).map(|m| m.as_str()));
println!("{:?}", re.find_iter(text).map(|m| m.as_str()).collect::<Vec<_>>());

for c in re.captures_iter(text) {
    println!("user={} host={}", &c[1], &c[2]);
}

println!("{}", re.replace_all(text, "<hidden>"));
println!("{}", re.replace_all(text, "$1 at $2"));
```

Output:

```
true
Some("raj@example.com")
["raj@example.com", "sam@test.com"]
user=raj host=example
user=sam host=test
mail <hidden> or <hidden> now
mail raj at example or sam at test now
```

Named groups, splitting, and validation:

```rust
use regex::Regex;

let num = Regex::new(r"\d+").unwrap();
println!("{:?}", num.find_iter("a1b22c333").map(|m| m.as_str()).collect::<Vec<_>>());
println!("{:?}", num.split("a1b22c").collect::<Vec<_>>());

let date = Regex::new(r"^(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})$").unwrap();
if let Some(c) = date.captures("2026-10-25") {
    println!("{} / {} / {}", &c["y"], &c["m"], &c["d"]);
}
println!("{:?}", date.captures("nope").is_none());
println!("{:?}", Regex::new(r"[").is_err());
```

Output:

```
["1", "22", "333"]
["a", "b", "c"]
2026 / 10 / 25
true
true
```

`Regex::new` returns a `Result` because the pattern is checked at runtime. Use a
raw string `r"..."` so backslashes reach the regex engine untouched.

Compiling a `Regex` is expensive — never do it inside a loop. Build it once with
`std::sync::LazyLock` or pass it in. And prefer the std methods when they
suffice: `split_once('=')` beats a regex for `key=value` on every axis.

## Quick Reference

| Task | Call |
|---|---|
| First index of | `s.find(p) -> Option<usize>` |
| Last index of | `s.rfind(p) -> Option<usize>` |
| Contains | `s.contains(p) -> bool` |
| Count occurrences | `s.matches(p).count()` |
| All match positions | `s.match_indices(p)` |
| Split on delimiter | `s.split(p)` |
| Split, limit pieces | `s.splitn(n, p)` |
| Split at first/last | `s.split_once(p)` / `s.rsplit_once(p)` |
| Split on words | `s.split_whitespace()` |
| Split, no trailing empty | `s.split_terminator(p)` |
| Split, keep delimiter | `s.split_inclusive(p)` |
| Split lines | `s.lines()` |
| Replace all | `s.replace(p, to)` |
| Replace first n | `s.replacen(p, to, n)` |
| Split into a `Vec` | `s.split(p).collect::<Vec<_>>()` |
| Regex | `regex::Regex::new(r"..")` |

| Pattern kind | Example |
|---|---|
| Single char | `s.split(',')` |
| Substring | `s.split("::")` |
| Any of several chars | `s.split([',', ';'])` |
| Predicate | `s.split(char::is_numeric)` |

## Common Mistakes

**Expecting `replace` to replace once.** It replaces all. Use
`replacen(p, to, 1)`.

**Treating a `find` index as a character position.** It is a byte offset. Fine
for slicing, wrong for `chars().nth()`.

**`split(' ')` for words.** Consecutive spaces produce empty fields. Use
`split_whitespace`.

**Forgetting `.collect()`.** Splits are lazy iterators; printing one shows the
iterator's `Debug`, not the pieces.

**Parsing split fields without trimming.** `" 2".parse::<i32>()` fails. Chain
`.trim()` first.

**Compiling a `Regex` inside a loop.** It is the expensive part. Build it once
outside.

## Exercises

1. Write `fn word_count(s: &str) -> usize` with `split_whitespace`. Confirm it
   returns 0 for `"   "` and 3 for `"a  b\tc"`.
2. Write `fn parse_kv(line: &str) -> Option<(&str, &str)>` using `split_once`
   that trims both sides and returns `None` for a line with no `=`.
3. Count how many times `"ab"` occurs in `"ababab"` and in `"aabab"`. Explain
   the non-overlapping result in one line.
4. Given `"2026-10-25T14:30:00"`, use `split_once('T')` then `split(':')` to
   print the hour, minute, and second as `i32`s.
5. Add `regex` to a scratch project and write a function that extracts every
   `#tag` from a sentence, returning `Vec<&str>` without the `#`.

---

[← Course Index](../README.md)
