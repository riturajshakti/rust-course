[← Course Index](../README.md)

# 17.1 — Serde: Serialization and Deserialization

## What You'll Learn

- `#[derive(Serialize, Deserialize)]` and the `serde_json` entry points
- Every field attribute that matters in a real API: `rename`, `rename_all`, `skip`, `default`, `flatten`, `alias`, `with`
- The four enum representations and the exact JSON each produces
- `serde_json::Value`, custom (de)serializers, and what a bad payload's error looks like

## Coming From Another Language

| Other | Rust |
|---|---|
| `JSON.stringify(obj)` | `serde_json::to_string(&v)?` |
| `JSON.parse(s)` as `any` | `serde_json::from_str::<T>(s)?` — typed, validated |
| Python `pydantic.BaseModel` | `struct` + `#[derive(Deserialize)]` |
| Go `json:"name,omitempty"` struct tags | `#[serde(rename = "name", skip_serializing_if = "...")]` |

The big difference: serde generates code at compile time, so there is no
reflection and no runtime schema. Parsing into a `struct` either produces that
exact type or an error with a line and column — there is no `any` escape hatch
unless you ask for `Value`.

`serde` is the trait layer; `serde_json` is one format. The same derives also
drive TOML, YAML, MessagePack, and bincode.

## Setup

```bash
cargo add serde --features derive
cargo add serde_json
```

```toml
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.154"
```

## The round trip

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    id: u64,
    name: String,
    active: bool,
}

fn main() {
    let u = User { id: 1, name: "raj".into(), active: true };
    println!("{}", serde_json::to_string(&u).unwrap());
    println!("{}", serde_json::to_string_pretty(&u).unwrap());

    let json = r#"{"id":2,"name":"sam","active":false}"#;
    let parsed: User = serde_json::from_str(json).unwrap();
    println!("{:?}", parsed);
}
```

Output:

```
{"id":1,"name":"raj","active":true}
{
  "id": 1,
  "name": "raj",
  "active": true
}
User { id: 2, name: "sam", active: false }
```

`Serialize` needs `&self`; `Deserialize` needs the target type known, either from
an annotation or a turbofish: `from_str::<User>(json)`.

## Bytes and `Value` instead of strings

```rust
let bytes = serde_json::to_vec(&o)?;
let from_bytes: Order = serde_json::from_slice(&bytes)?;

let v = serde_json::to_value(&o)?;          // struct -> Value
let back: Order = serde_json::from_value(v)?; // Value -> struct
```

Axum uses `from_slice` internally — the body arrives as bytes, never a `String`.

## Field attributes

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(rename_all = "camelCase")]
struct Account {
    user_id: u64,
    display_name: String,
    #[serde(rename = "email_address")]
    email: String,
    #[serde(skip)]
    internal_score: f64,
    #[serde(skip_serializing_if = "Option::is_none")]
    phone: Option<String>,
    #[serde(default)]
    tags: Vec<String>,
}

let a = Account {
    user_id: 7, display_name: "Raj".into(), email: "r@x.com".into(),
    internal_score: 9.5, phone: None, tags: vec![],
};
println!("{}", serde_json::to_string_pretty(&a).unwrap());
```

Output:

```
{
  "userId": 7,
  "displayName": "Raj",
  "email_address": "r@x.com",
  "tags": []
}
```

`internal_score` is gone entirely. `phone` is gone because it was `None`.
`rename` on a field beats `rename_all` on the struct.

Deserializing the same shape, with two fields absent:

```rust
let s = r#"{"userId":8,"displayName":"Sam","email_address":"s@x.com"}"#;
println!("{:?}", serde_json::from_str::<Account>(s).unwrap());
```

Output:

```
Account { user_id: 8, display_name: "Sam", email: "s@x.com", internal_score: 0.0, phone: None, tags: [] }
```

`#[serde(skip)]` fields are filled from `Default`, so a skipped field's type must
implement `Default`. `#[serde(default)]` does the same for a field that is simply
missing from the payload.

`rename_all` accepts `"camelCase"`, `"snake_case"`, `"PascalCase"`,
`"SCREAMING_SNAKE_CASE"`, `"kebab-case"`, `"lowercase"`, `"UPPERCASE"`.

## `flatten`

Inlines a nested struct's fields into the parent object, both directions.

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Meta { page: u32, total: u32 }

#[derive(Serialize, Deserialize, Debug)]
struct Page { items: Vec<String>, #[serde(flatten)] meta: Meta }

let p = Page { items: vec!["a".into()], meta: Meta { page: 1, total: 42 } };
println!("{}", serde_json::to_string(&p).unwrap());

let back: Page = serde_json::from_str(r#"{"items":["a","b"],"page":2,"total":42}"#).unwrap();
println!("{:?}", back);
```

Output:

```
{"items":["a"],"page":1,"total":42}
Page { items: ["a", "b"], meta: Meta { page: 2, total: 42 } }
```

Flatten into a `HashMap<String, Value>` to keep unknown fields instead of
discarding them:

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Loose { id: u32, #[serde(flatten)] extra: HashMap<String, Value> }

let l: Loose = serde_json::from_str(r#"{"id":3,"color":"red","size":42}"#).unwrap();
println!("{:?}", l);
println!("{}", serde_json::to_string(&l).unwrap());
```

Output:

```
Loose { id: 3, extra: {"size": Number(42), "color": String("red")} }
{"id":3,"size":42,"color":"red"}
```

## `alias` — accepting several input names

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Config { #[serde(alias = "hostname", alias = "host_name")] host: String }

for s in [r#"{"host":"a"}"#, r#"{"hostname":"b"}"#, r#"{"host_name":"c"}"#] {
    println!("{:?}", serde_json::from_str::<Config>(s).unwrap());
}
```

Output:

```
Config { host: "a" }
Config { host: "b" }
Config { host: "c" }
```

`alias` is deserialize-only. Serializing always writes `host`.

## Enums: externally tagged (the default)

```rust
#[derive(Serialize, Deserialize, Debug)]
enum Ext { Ping, Text(String), Move { x: i32, y: i32 } }

for v in [Ext::Ping, Ext::Text("hi".into()), Ext::Move { x: 1, y: 2 }] {
    println!("{}", serde_json::to_string(&v).unwrap());
}
```

Output:

```
"Ping"
{"Text":"hi"}
{"Move":{"x":1,"y":2}}
```

A unit variant becomes a bare string; everything else becomes a single-key
object. This is the most faithful representation but the ugliest to consume from
TypeScript.

## Internally tagged

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type")]
enum Int { Ping, Move { x: i32, y: i32 } }
```

Output:

```
{"type":"Ping"}
{"type":"Move","x":1,"y":2}
```

This is the TypeScript discriminated-union shape, and the one to reach for in a
public API. Restriction: no newtype variants holding non-struct data — a
`Text(String)` variant will not compile with `tag`.

## Adjacently tagged

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type", content = "data")]
enum Adj { Ping, Text(String), Move { x: i32, y: i32 } }
```

Output:

```
{"type":"Ping"}
{"type":"Text","data":"hi"}
{"type":"Move","data":{"x":1,"y":2}}
```

Works for every variant shape, at the cost of a wrapper key.

## Untagged

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(untagged)]
enum Untagged { Num(i64), Text(String), List(Vec<i64>) }

for v in [Untagged::Num(5), Untagged::Text("hi".into()), Untagged::List(vec![1,2])] {
    println!("{}", serde_json::to_string(&v).unwrap());
}
for s in ["7", r#""seven""#, "[7,8]"] {
    println!("{} -> {:?}", s, serde_json::from_str::<Untagged>(s).unwrap());
}
```

Output:

```
5
"hi"
[1,2]
7 -> Num(7)
"seven" -> Text("seven")
[7,8] -> List([7, 8])
```

No tag at all — serde tries each variant in declaration order and takes the
first that fits. Useful for messy third-party APIs (`"id": 5` or `"id": "5"`),
but its error messages are poor and ambiguous variants silently win in order.

`rename_all` works on enums too, applying to variant names:

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
enum Status { Active, PendingReview }

println!("{}", serde_json::to_string(&vec![Status::Active, Status::PendingReview]).unwrap());
println!("{:?}", serde_json::from_str::<Status>(r#""PENDING_REVIEW""#).unwrap());
```

Output:

```
["ACTIVE","PENDING_REVIEW"]
PendingReview
```

## `Option` and `null`

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Opt { a: Option<i32>, b: Option<i32> }

println!("{}", serde_json::to_string(&Opt { a: Some(1), b: None }).unwrap());
println!("{:?}", serde_json::from_str::<Opt>(r#"{"a":1,"b":null}"#).unwrap());
println!("{:?}", serde_json::from_str::<Opt>(r#"{"a":1}"#).err());
```

Output:

```
{"a":1,"b":null}
Opt { a: Some(1), b: None }
None
```

Three facts worth internalizing: `None` serializes to `null` unless you add
`skip_serializing_if`; `null` deserializes to `None`; and an **absent**
`Option` field is also `None` with no error. A non-`Option` field that is absent
*is* an error. That asymmetry is how you express "required" vs "optional" in a
request DTO — no attribute needed.

## Nested structs and collections

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Address { city: String, zip: String }

#[derive(Serialize, Deserialize, Debug)]
struct Order { id: u32, ship_to: Address, lines: Vec<Line>, note: Option<String> }

#[derive(Serialize, Deserialize, Debug)]
struct Line { sku: String, qty: u32 }

let o = Order {
    id: 1,
    ship_to: Address { city: "Pune".into(), zip: "411001".into() },
    lines: vec![Line { sku: "A1".into(), qty: 2 }, Line { sku: "B2".into(), qty: 1 }],
    note: None,
};
println!("{}", serde_json::to_string_pretty(&o).unwrap());
```

Output:

```
{
  "id": 1,
  "ship_to": {
    "city": "Pune",
    "zip": "411001"
  },
  "lines": [
    {
      "sku": "A1",
      "qty": 2
    },
    {
      "sku": "B2",
      "qty": 1
    }
  ],
  "note": null
}
```

`HashMap` becomes a JSON object; tuples and arrays become JSON arrays.

```rust
let mut m = HashMap::new();
m.insert("a", vec![1, 2]);
println!("{}", serde_json::to_string(&m).unwrap());
println!("{:?}", serde_json::from_str::<HashMap<String, Vec<i32>>>(r#"{"x":[9]}"#).unwrap());
println!("{}", serde_json::to_string(&(1, "two", true)).unwrap());
println!("{:?}", serde_json::from_str::<(i32, String)>(r#"[5,"z"]"#).unwrap());
```

Output:

```
{"a":[1,2]}
{"x": [9]}
[1,"two",true]
(5, "z")
```

A `HashMap` key must serialize to a string — `HashMap<u32, _>` works (keys are
stringified), `HashMap<SomeStruct, _>` does not.

## `json!` and `Value`

`Value` is the dynamically typed escape hatch: an enum of
`Null | Bool | Number | String | Array | Object`.

```rust
use serde_json::{json, Value};

let v = json!({ "name": "raj", "scores": [1, 2, 3], "nested": { "ok": true } });
println!("{}", v);
println!("{}", v["name"]);
println!("{:?}", v["name"].as_str());
println!("{:?}", v["scores"][1].as_u64());
println!("{:?}", v["missing"]);
println!("{:?}", v.get("missing"));
println!("{}", v["nested"]["ok"]);
```

Output:

```
{"name":"raj","nested":{"ok":true},"scores":[1,2,3]}
"raj"
Some("raj")
Some(2)
Null
None
true
```

Two things to note. Keys come out sorted — `serde_json`'s default `Map` is a
`BTreeMap`; enable the `preserve_order` feature for insertion order. And `v["x"]`
on a missing key returns `Value::Null` rather than panicking, which is why
`.get()` exists when you need to tell "absent" from "present and null" apart.

Inspecting unknown shapes:

```rust
let v: Value = serde_json::from_str(r#"[1, "two", null, true, 3.5, {}, []]"#).unwrap();
for item in v.as_array().unwrap() {
    println!("{:<6} num={} str={} null={}",
        item.to_string(), item.is_number(), item.is_string(), item.is_null());
}
```

Output:

```
1      num=true str=false null=false
"two"  num=false str=true null=false
null   num=false str=false null=true
true   num=false str=false null=false
3.5    num=true str=false null=false
{}     num=false str=false null=false
[]     num=false str=false null=false
```

The `as_*` family returns `Option`: `as_str`, `as_i64`, `as_u64`, `as_f64`,
`as_bool`, `as_array`, `as_object`, `as_null`. Use `Value` for pass-through
payloads and webhooks; use a struct everywhere you actually care about the shape.

## Dates with chrono

```bash
cargo add chrono --features serde
cargo add uuid --features v4,serde
```

```rust
use chrono::{DateTime, NaiveDate, Utc};
use uuid::Uuid;

#[derive(Serialize, Deserialize, Debug)]
struct Event { id: Uuid, at: DateTime<Utc>, day: NaiveDate }

let e = Event {
    id: Uuid::parse_str("8f9a2c1e-0000-4000-8000-000000000001").unwrap(),
    at: "2026-10-25T09:30:00Z".parse().unwrap(),
    day: NaiveDate::from_ymd_opt(2026, 10, 25).unwrap(),
};
println!("{}", serde_json::to_string_pretty(&e).unwrap());
```

Output:

```
{
  "id": "8f9a2c1e-0000-4000-8000-000000000001",
  "at": "2026-10-25T09:30:00Z",
  "day": "2026-10-25"
}
```

`DateTime<Utc>` is RFC 3339, `NaiveDate` is `YYYY-MM-DD`. Both round-trip:

```rust
let s = r#"{"id":"8f9a2c1e-0000-4000-8000-000000000001","at":"2026-10-25T09:30:00Z","day":"2026-10-25"}"#;
println!("{:?}", serde_json::from_str::<Event>(s).unwrap());
println!("{}", serde_json::from_str::<Event>(
    r#"{"id":"nope","at":"2026-10-25T09:30:00Z","day":"2026-10-25"}"#).unwrap_err());
```

Output:

```
Event { id: 8f9a2c1e-0000-4000-8000-000000000001, at: 2026-10-25T09:30:00Z, day: 2026-10-25 }
UUID parsing failed: invalid character: found `n` at 0 at line 1 column 12
```

Without the `serde` feature on chrono you get `the trait bound DateTime<Utc>:
Serialize is not satisfied` — a very common first-day error.

## Custom serializers

Two free functions, one for each direction.

```rust
use serde::{Deserializer, Serializer};

fn cents_to_string<S: Serializer>(v: &i64, s: S) -> Result<S::Ok, S::Error> {
    s.serialize_str(&format!("{}.{:02}", v / 100, v % 100))
}

fn string_to_cents<'de, D: Deserializer<'de>>(d: D) -> Result<i64, D::Error> {
    let s = String::deserialize(d)?;
    let (a, b) = s.split_once('.').ok_or_else(|| serde::de::Error::custom("expected N.NN"))?;
    Ok(a.parse::<i64>().map_err(serde::de::Error::custom)? * 100
        + b.parse::<i64>().map_err(serde::de::Error::custom)?)
}
```

`serde::de::Error::custom` is how you raise a message from inside a
deserializer. Attach them per direction with `serialize_with` /
`deserialize_with`, or bundle both into a module and use `with`:

```rust
mod epoch {
    use chrono::{DateTime, Utc};
    use serde::{Deserialize, Deserializer, Serializer};
    pub fn serialize<S: Serializer>(v: &DateTime<Utc>, s: S) -> Result<S::Ok, S::Error> {
        s.serialize_i64(v.timestamp())
    }
    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<DateTime<Utc>, D::Error> {
        let secs = i64::deserialize(d)?;
        DateTime::from_timestamp(secs, 0).ok_or_else(|| serde::de::Error::custom("bad timestamp"))
    }
}

#[derive(Serialize, Deserialize, Debug)]
struct Invoice {
    #[serde(serialize_with = "cents_to_string", deserialize_with = "string_to_cents")]
    amount: i64,
    #[serde(with = "epoch")]
    issued: DateTime<Utc>,
}

let i = Invoice { amount: 123456, issued: "2026-10-25T09:30:00Z".parse().unwrap() };
println!("{}", serde_json::to_string(&i).unwrap());
println!("{:?}", serde_json::from_str::<Invoice>(r#"{"amount":"99.50","issued":1761384600}"#).unwrap());
println!("{}", serde_json::from_str::<Invoice>(r#"{"amount":"99","issued":0}"#).unwrap_err());
```

Output:

```
{"amount":"1234.56","issued":1792920600}
Invoice { amount: 9950, issued: 2025-10-25T09:30:00Z }
expected N.NN at line 1 column 14
```

`with = "epoch"` expands to `serialize_with = "epoch::serialize"` plus
`deserialize_with = "epoch::deserialize"`.

## Rejecting unknown fields

By default serde ignores extra keys. `deny_unknown_fields` makes them an error —
worth it on config files and admin endpoints, where a typo should fail loudly
rather than be silently dropped.

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(deny_unknown_fields)]
struct Strict { id: u32 }

println!("{}", serde_json::from_str::<Strict>(r#"{"id":1,"x":2}"#).unwrap_err());
```

Output:

```
unknown field `x`, expected `id` at line 1 column 11
```

It is incompatible with `flatten` on the same struct — the flattened half has no
way to know which keys belong to it.

## Error messages from bad JSON

These are the strings your API will end up showing clients, so look at them
closely.

```rust
println!("{}", serde_json::from_str::<Opt>(r#"{"a":"nope"}"#).unwrap_err());
println!("{}", serde_json::from_str::<Opt>(r#"{"a":1,"#).unwrap_err());
println!("{}", serde_json::from_str::<Ext>(r#""Nope""#).unwrap_err());
let e = serde_json::from_str::<Opt>("{\n  \"a\": true\n}").unwrap_err();
println!("line={} column={} classify={:?}", e.line(), e.column(), e.classify());
```

Output:

```
invalid type: string "nope", expected i32 at line 1 column 11
EOF while parsing a value at line 1 column 7
unknown variant `Nope`, expected one of `Ping`, `Text`, `Move` at line 1 column 6
line=2 column=11 classify=Data
```

`classify()` returns `Io`, `Syntax`, `Data`, or `Eof`. `Data` means the JSON was
valid but did not match the type — that is a client 422, whereas `Syntax`/`Eof`
is a client 400. Lesson 17.4 wires exactly this distinction into axum's
`JsonRejection`.

## Quick Reference

| Need | Form |
|---|---|
| To JSON string | `serde_json::to_string(&v)` / `to_string_pretty` |
| To bytes / `Value` | `to_vec(&v)` / `to_value(&v)` |
| From string / bytes / `Value` | `from_str::<T>(s)` / `from_slice` / `from_value` |
| Rename one field | `#[serde(rename = "x")]` |
| Rename all | `#[serde(rename_all = "camelCase")]` |
| Never serialize or read | `#[serde(skip)]` |
| Omit when `None` | `#[serde(skip_serializing_if = "Option::is_none")]` |
| Fill when missing | `#[serde(default)]` or `#[serde(default = "path")]` |
| Inline nested fields | `#[serde(flatten)]` |
| Accept alternate input key | `#[serde(alias = "x")]` |
| Custom codec | `serialize_with` / `deserialize_with` / `with` |
| Reject extra keys | `#[serde(deny_unknown_fields)]` |
| Tagged union | `#[serde(tag = "type")]` |
| Tag + payload key | `#[serde(tag = "type", content = "data")]` |
| Shape-sniffing union | `#[serde(untagged)]` |
| Dynamic JSON | `serde_json::Value`, `json!{...}`, `.as_str()`, `.get()` |

## Common Mistakes

**Forgetting `features = ["derive"]` on serde.** You get `cannot find derive
macro Serialize`. Add it to `serde`, not `serde_json`.

**Forgetting `features = ["serde"]` on chrono or uuid.** The derive compiles but
the field does not: `the trait bound DateTime<Utc>: Serialize is not satisfied`.

**Using `#[serde(default)]` to mean optional.** An `Option<T>` field is already
optional on input. `default` is for non-`Option` fields that need a fallback.

**Expecting `None` to disappear from output.** It serializes as `null`. Add
`skip_serializing_if = "Option::is_none"` if the key should be absent.

**Combining `flatten` with `deny_unknown_fields`.** They are mutually exclusive;
the compiler error is unhelpful, so remember the rule.

**Using externally tagged enums in a public API.** `{"Move":{"x":1}}` is painful
for clients. `#[serde(tag = "type")]` gives the discriminated-union shape every
frontend expects.

## Exercises

1. Define `Product { id: u32, name: String, price_cents: i64, tags: Vec<String> }`
   with `rename_all = "camelCase"`, serialize one, and parse the result back.
2. Add an `Option<String>` description that is omitted from output when absent,
   and a `#[serde(default)]` `in_stock: bool`. Prove both with a payload that
   leaves them out.
3. Model `Payment` as an internally tagged enum with variants `Card { last4:
   String }`, `Bank { iban: String }`, and `Cash`. Print the JSON for each, then
   deserialize each back.
4. Write `serialize_with`/`deserialize_with` for a `price_cents` field so it
   appears in JSON as a `"12.34"` string. Make the deserializer reject `"12"`
   with a custom message and print the error.
5. Parse `{"id":1,"meta":{"a":[1,2,{"b":null}]}}` into a `serde_json::Value` and
   print the value at `meta.a[2].b`, distinguishing "key absent" from "value is
   null" using `.get()`.

---

[← Course Index](../README.md)
