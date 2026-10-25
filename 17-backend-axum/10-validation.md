[← Course Index](../README.md)

# 17.10 — Validation and Request Guards

## What You'll Learn

- The `validator` crate: `derive(Validate)` and every attribute worth using
- A `Valid<T>` extractor that validates and returns 422 with field-level errors
- Validating query params, and the newtype/`TryFrom` alternative
- A field-error JSON shape a frontend can actually render

## Coming From Another Language

| Other | Rust |
|---|---|
| `zod` / `yup` schema | `#[derive(Validate)]` attributes |
| pydantic validators | `validator` for value rules, serde for shape |
| `class-validator` decorators | nearly identical attribute syntax |
| Go `validator.v10` struct tags | same idea, same attribute names |

Rust splits validation in two. **Shape** is serde's job: a missing field or a
string where a number belongs never reaches your code (lesson 17.4). **Values**
are `validator`'s job: a present, well-typed `age: u8` that happens to be `12`.

## Setup

```bash
cargo add validator --features derive
cargo add regex
```

```toml
validator = { version = "0.21.0", features = ["derive"] }
```

## The attributes

```rust
use validator::{Validate, ValidationError};

static ZIP_RE: std::sync::LazyLock<regex::Regex> =
    std::sync::LazyLock::new(|| regex::Regex::new(r"^\d{6}$").unwrap());

fn not_reserved(name: &str) -> Result<(), ValidationError> {
    if ["admin", "root"].contains(&name) {
        return Err(ValidationError::new("reserved").with_message("this name is reserved".into()));
    }
    Ok(())
}

#[derive(Debug, Deserialize, Validate)]
struct Address {
    #[validate(length(min = 2, max = 60))]
    city: String,
    #[validate(regex(path = "ZIP_RE"))]
    zip: String,
}

#[derive(Debug, Deserialize, Validate)]
struct SignUp {
    #[validate(length(min = 3, max = 20), custom(function = "not_reserved"))]
    username: String,
    #[validate(email)]
    email: String,
    #[validate(length(min = 8, message = "password must be at least 8 characters"))]
    password: String,
    #[validate(range(min = 18, max = 120))]
    age: u8,
    #[validate(url)]
    website: Option<String>,
    #[validate(nested)]
    address: Address,
    #[validate(length(min = 1, max = 5))]
    tags: Vec<String>,
}
```

| Attribute | Applies to | Notes |
|---|---|---|
| `length(min, max, equal)` | `String`, `Vec`, any `HasLen` | on a `Vec` it is element count |
| `range(min, max, exclusive_min, exclusive_max)` | numbers | |
| `email` | `String` | |
| `url` | `String` | |
| `regex(path = "STATIC")` | `String` | the static must be a `Regex` |
| `contains(pattern = "x")` | `String` | |
| `does_not_contain(pattern = "x")` | `String` | |
| `must_match(other = "field")` | `String` | password confirmation |
| `custom(function = "f")` | anything | `fn(&T) -> Result<(), ValidationError>` |
| `nested` | a struct or `Vec<struct>` field | recurses |
| `required` | `Option<T>` | errors when `None` |
| `skip_on_field_errors` | the struct | skip struct-level checks when fields failed |
| `message = "..."` | any rule | overrides the code in the output |

Two mechanical details. An `Option<T>` field with a rule is validated **only
when `Some`** — combine with `required` to insist on a value. And `nested` is
mandatory for recursion: without it a bad `Address` passes silently, which is the
most common mistake with this crate.

## Running validation

`validate()` returns `Result<(), ValidationErrors>`.

```rust
let good = SignUp {
    username: "raj".into(), email: "raj@x.com".into(), password: "longenough".into(),
    age: 31, website: Some("https://raj.dev".into()),
    address: Address { city: "Pune".into(), zip: "411001".into() },
    tags: vec!["rust".into()],
};
println!("{:?}", good.validate().is_ok());

let bad = SignUp {
    username: "ad".into(), email: "nope".into(), password: "short".into(),
    age: 12, website: Some("not a url".into()),
    address: Address { city: "P".into(), zip: "abc".into() },
    tags: vec![],
};
let e = bad.validate().unwrap_err();
println!("{} field(s) failed", e.field_errors().len());
```

Output:

```
true
6 field(s) failed
```

Every rule runs — the error carries all failures at once, which is what a form
needs.

`ValidationErrors` is a tree: `errors()` gives a map of field name to
`ValidationErrorsKind`, which is `Field(Vec<ValidationError>)`,
`Struct(Box<ValidationErrors>)` for a `nested` struct, or `List(BTreeMap<usize,
..>)` for a `Vec` of structs. Its `Display` is unreadable, so flatten it
yourself.

## A JSON shape a frontend can use

```rust
fn flatten(errs: &ValidationErrors) -> BTreeMap<String, Vec<String>> {
    let mut out = BTreeMap::new();
    collect(errs, "", &mut out);
    out
}

fn collect(errs: &ValidationErrors, prefix: &str, out: &mut BTreeMap<String, Vec<String>>) {
    for (field, kind) in errs.errors() {
        let path = if prefix.is_empty() { field.to_string() } else { format!("{prefix}.{field}") };
        match kind {
            ValidationErrorsKind::Field(v) => {
                out.insert(path, v.iter().map(|e| {
                    e.message.clone().map(|m| m.to_string()).unwrap_or_else(|| e.code.to_string())
                }).collect());
            }
            ValidationErrorsKind::Struct(inner) => collect(inner, &path, out),
            ValidationErrorsKind::List(map) => {
                for (i, inner) in map { collect(inner, &format!("{path}[{i}]"), out); }
            }
        }
    }
}
```

Flattened, the six failures above become:

```json
{"address.city":["length"],"address.zip":["regex"],"age":["range"],"email":["email"],"password":["password must be at least 8 characters"],"username":["length"]}
```

A dotted path per field and a list of problems — a frontend maps that straight
onto inputs. `e.code` is the rule name (`"length"`, `"range"`), stable enough to
translate client-side; `e.message` is the `message = "..."` override when given,
which is why `password` reads in English and the rest read as codes. Supply a
`message` on every rule whose output reaches a user.

Nested paths come out as `address.city` and list items as `tags[0].name`, which
is what the recursion is for.

## The `Valid<T>` extractor

Parse, validate, and reject in one extractor, so handlers only ever see valid
data.

```rust
struct Valid<T>(T);

impl<S, T> FromRequest<S> for Valid<T>
where
    T: Validate,
    Json<T>: FromRequest<S, Rejection = JsonRejection>,
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state).await.map_err(|rej| {
            (rej.status(), Json(ErrorBody {
                error: "invalid_json", message: Some(rej.body_text()), fields: None,
            })).into_response()
        })?;

        value.validate().map_err(|e| {
            (StatusCode::UNPROCESSABLE_ENTITY, Json(ErrorBody {
                error: "validation_error", message: None, fields: Some(flatten(&e)),
            })).into_response()
        })?;

        Ok(Valid(value))
    }
}

#[derive(Debug, Serialize)]
struct ErrorBody {
    error: &'static str,
    #[serde(skip_serializing_if = "Option::is_none")]
    message: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    fields: Option<BTreeMap<String, Vec<String>>>,
}
```

The handler then has no validation code at all:

```rust
async fn signup(Valid(body): Valid<SignUp>) -> impl IntoResponse {
    (StatusCode::CREATED, Json(json!({ "username": body.username, "email": body.email })))
}
```

A `Response` as the `Rejection` type is the trick that lets one extractor return
two different shapes. `Response` implements `IntoResponse` trivially.

The happy path:

```console
$ curl -si -X POST localhost:52430/signup -H 'content-type: application/json' -d '{
  "username":"raj","email":"raj@x.com","password":"longenough","age":31,
  "address":{"city":"Pune","zip":"411001"}}' | sed -n '1p;$p'
HTTP/1.1 201 Created
{"email":"raj@x.com","username":"raj"}
```

Every rule broken at once:

```console
$ curl -si -X POST localhost:52430/signup -H 'content-type: application/json' -d '{
  "username":"ad","email":"nope","password":"short","age":12,
  "address":{"city":"P","zip":"abc"}}' | sed -n '1p;$p'
HTTP/1.1 422 Unprocessable Entity
{"error":"validation_error","fields":{"address.city":["length"],"address.zip":["regex"],"age":["range"],"email":["email"],"password":["password must be at least 8 characters"],"username":["length"]}}
```

The custom rule, with its message:

```console
$ curl -s -X POST localhost:52430/signup -H 'content-type: application/json' -d '{
  "username":"admin","email":"a@b.co","password":"longenough","age":31,
  "address":{"city":"Pune","zip":"411001"}}'
{"error":"validation_error","fields":{"username":["this name is reserved"]}}
```

And the serde-layer failures, kept distinct from validation failures:

```console
$ curl -si -X POST localhost:52430/signup -H 'content-type: application/json' \
    -d '{"username":"raj"}' | sed -n '1p;$p'
HTTP/1.1 422 Unprocessable Entity
{"error":"invalid_json","message":"Failed to deserialize the JSON body into the target type: missing field `email` at line 1 column 18"}

$ curl -si -X POST localhost:52430/signup -H 'content-type: application/json' -d '{nope' | sed -n '1p;$p'
HTTP/1.1 400 Bad Request
{"error":"invalid_json","message":"Failed to parse the request body as JSON: key must be a string at line 1 column 2"}
```

Both carry `error: "invalid_json"`, so a client can tell "your payload is the
wrong shape" from "your values are wrong" and show the right thing. Note the
ordering: the body is deserialized **first**, so a missing field is a shape error
and never a validation error.

## Validating query params

`Query<T>` has no body, so a `FromRequestParts` version is cleanest. The quick
version calls `validate()` in the handler:

```rust
#[derive(Debug, Deserialize, Validate)]
struct Page {
    #[validate(range(min = 1, max = 1000))] #[serde(default = "one")] page: u32,
    #[validate(range(min = 1, max = 100))] #[serde(default = "twenty")] per_page: u32,
    #[validate(length(min = 2))] q: Option<String>,
}

async fn search(Query(p): Query<Page>) -> Result<Json<Value>, Response> {
    p.validate().map_err(|e| {
        (StatusCode::UNPROCESSABLE_ENTITY, Json(ErrorBody {
            error: "validation_error", message: None, fields: Some(flatten(&e)),
        })).into_response()
    })?;
    Ok(Json(json!({ "page": p.page, "per_page": p.per_page, "q": p.q })))
}
```

```console
$ curl -s 'localhost:52430/search?page=2&per_page=10&q=rust'
{"page":2,"per_page":10,"q":"rust"}

$ curl -si 'localhost:52430/search?page=0&per_page=500&q=a' | sed -n '1p;$p'
HTTP/1.1 422 Unprocessable Entity
{"error":"validation_error","fields":{"page":["range"],"per_page":["range"],"q":["length"]}}
```

Capping `per_page` is not cosmetic: without it a client sends `per_page=1000000`
and your `LIMIT` loads a million rows. Validate it at the edge.

The reusable version mirrors `Valid<T>` with `FromRequestParts`:

```rust
struct ValidQuery<T>(T);

impl<S, T> FromRequestParts<S> for ValidQuery<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let Query(v) = Query::<T>::from_request_parts(parts, state).await
            .map_err(|r| (r.status(), Json(ErrorBody {
                error: "invalid_query", message: Some(r.body_text()), fields: None,
            })).into_response())?;
        v.validate().map_err(|e| (StatusCode::UNPROCESSABLE_ENTITY, Json(ErrorBody {
            error: "validation_error", message: None, fields: Some(flatten(&e)),
        })).into_response())?;
        Ok(ValidQuery(v))
    }
}
```

Being a `FromRequestParts` extractor, it can sit anywhere in the argument list
and coexist with `Valid<T>` on the body.

## The newtype alternative

`validator` checks a struct you could still have built wrong. A newtype with
`TryFrom` makes the invalid state **unrepresentable** — Module 09's pattern,
applied to a web boundary.

```rust
#[derive(Debug, Serialize)]
struct Email(String);

impl TryFrom<String> for Email {
    type Error = String;
    fn try_from(s: String) -> Result<Self, String> {
        let s = s.trim().to_lowercase();
        if s.len() < 3 || !s.contains('@') || s.starts_with('@') || s.ends_with('@') {
            return Err(format!("{s:?} is not a valid email"));
        }
        Ok(Email(s))
    }
}
```

```console
$ curl -s -X POST localhost:52430/newtype -H 'content-type: application/json' -d '{"email":"  RAJ@X.COM  "}'
"raj@x.com"

$ curl -si -X POST localhost:52430/newtype -H 'content-type: application/json' -d '{"email":"nope"}' | sed -n '1p;$p'
HTTP/1.1 422 Unprocessable Entity
{"error":"\"nope\" is not a valid email"}
```

Note the trim and lowercase: a constructor is the natural place for
normalization, which is where the newtype beats an attribute. Once `Email`
exists, **every function taking an `Email` is guaranteed a valid one** — no
re-validation deeper in the call stack, and no "did we check this already?".

Wiring it into deserialization means the extractor enforces it:

```rust
impl<'de> Deserialize<'de> for Email {
    fn deserialize<D: Deserializer<'de>>(d: D) -> Result<Self, D::Error> {
        let s = String::deserialize(d)?;
        Email::try_from(s).map_err(serde::de::Error::custom)
    }
}
```

Now `Json<CreateUser>` with an `email: Email` field rejects a bad address as a
`JsonDataError`, before any handler code. The cost: the error is a serde message
on one field, not a multi-field report — so it suits ids and codes better than
user-facing forms.

### Choosing between them

| | `validator` | newtype + `TryFrom` |
|---|---|---|
| Error shape | all fields at once, structured | one field, serde-style |
| Guarantees | the struct was checked once | the type is always valid |
| Normalization | awkward | natural in the constructor |
| Boilerplate | one attribute | a type + 2 impls |
| Best for | request DTOs, forms | domain types reused across the codebase |

Use both: `validator` on the DTO at the HTTP edge for a good error response, and
newtypes for domain values (`Email`, `Slug`, `NonEmptyName`, `PositiveQty`) that
travel deep into the code. The DTO converts into domain types once validation
passes.

## Sanitization

Validation says yes or no; sanitization changes the value. `validator` does not
sanitize — do it explicitly, before or after.

```rust
#[derive(Deserialize)]
struct CreatePost { title: String, body: String }

impl CreatePost {
    fn sanitized(mut self) -> Self {
        self.title = self.title.trim().to_string();
        self.body = self.body.trim().to_string();
        self
    }
}
```

Three rules worth stating:

- **Trim before validating length**, or `"   "` passes `length(min = 1)`.
- **Never sanitize HTML to make it safe** — escape it at render time, or use
  `ammonia` if you must store rich text.
- `#[serde(deserialize_with = ...)]` (lesson 17.1) can normalize during parsing,
  which is tidier than a separate pass.

SQL injection needs no sanitization at all — `.bind()` handles it structurally
(lesson 17.8).

## Validation is not authorization

A `Valid<UpdatePost>` says the payload is well-formed, not that this user may
edit that post. Keep the two separate: the extractor validates, the handler (or
the auth extractor of lesson 17.11) authorizes. Conflating them is how
`user_id` ends up trusted from a request body.

## Quick Reference

| Need | Form |
|---|---|
| Derive | `#[derive(Validate)]` |
| Run | `value.validate()?` → `Result<(), ValidationErrors>` |
| String length | `#[validate(length(min = 3, max = 20))]` |
| Number bounds | `#[validate(range(min = 18, max = 120))]` |
| Email / URL | `#[validate(email)]` / `#[validate(url)]` |
| Pattern | `#[validate(regex(path = "STATIC_RE"))]` |
| Confirm field | `#[validate(must_match(other = "password"))]` |
| Custom rule | `#[validate(custom(function = "f"))]` |
| Recurse | `#[validate(nested)]` |
| Require an `Option` | `#[validate(required)]` |
| Custom text | `message = "..."` on any rule |
| Error map | `errs.errors()` → `Field` / `Struct` / `List` |
| Extractor (body) | `impl FromRequest for Valid<T>` delegating to `Json<T>` |
| Extractor (query) | `impl FromRequestParts for ValidQuery<T>` |
| Two rejection shapes | `type Rejection = Response` |
| Always-valid value | newtype + `TryFrom` (+ a `Deserialize` impl) |

## Common Mistakes

**Forgetting `#[validate(nested)]`.** The inner struct is never checked and bad
data sails through. The single most common bug with this crate.

**Returning the first error only.** `validate()` already collects them all;
don't throw that away.

**Using the default `Display` of `ValidationErrors`.** It is not a client-facing
message. Flatten it into a field map.

**Validating before trimming.** `"   "` has length 3.

**No cap on `per_page`.** Your `LIMIT` becomes whatever the client typed.

**Validating `Option<T>` and expecting `None` to fail.** Rules are skipped for
`None`; add `required`.

**Treating validation as authorization.** A valid `user_id` in a body is still
attacker-controlled.

## Exercises

1. Add `#[derive(Validate)]` to a `CreateProduct { name, price_cents, sku }` with
   `length`, `range`, and a `regex` on the SKU. Print `validate()` for a good and
   a bad instance.
2. Add `password` and `password_confirm` with `must_match`, and confirm the error
   appears on the right field.
3. Build the `Valid<T>` extractor and use it on a POST handler. Verify three
   responses with curl: 201, a 422 validation body, and a 400 syntax body.
4. Write a `custom` validator rejecting a username that is not lowercase
   alphanumeric-with-hyphens, with a `message`, and confirm the message appears
   in the JSON.
5. Define a `Slug` newtype with `TryFrom<String>` that lowercases and replaces
   spaces with hyphens, plus a `Deserialize` impl. Use it as a field in a
   `Json<T>` payload and record the error for an invalid value.
6. Write `ValidQuery<T>` as a `FromRequestParts` extractor and use it alongside
   `Valid<T>` in one handler that takes both a validated query and a validated
   body.

---

[← Course Index](../README.md)
