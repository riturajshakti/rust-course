[← 17.10 Validation and Request Guards](10-validation.md) · [Course Index](../README.md) · [17.12 HTTP Clients with reqwest →](12-reqwest.md)

# 17.11 — Authentication with JWT

## What You'll Learn

- Password hashing with argon2 — and why nothing faster is acceptable
- The full JWT flow with `jsonwebtoken`: `Claims`, `encode`, `decode`, validation
- An `AuthUser` extractor that pulls the Bearer token and rejects with 401
- Role-based authorization, token expiry, refresh tokens, and secret handling

## Coming From Another Language

| Other | Rust |
|---|---|
| `bcrypt.hash(pw, 10)` | `Argon2::default().hash_password(pw.as_bytes())` |
| `jwt.sign(payload, secret)` | `encode(&Header::default(), &claims, &enc_key)` |
| `jwt.verify(token, secret)` | `decode::<Claims>(token, &dec_key, &validation)` |
| `passport` / middleware + `req.user` | a `FromRequestParts` extractor in the signature |

The axum difference is the extractor. `async fn me(user: AuthUser)` is
**unauthenticated code cannot compile** — the handler either names the extractor
or has no user at all. Middleware plus `req.user` can be forgotten silently.

## Setup

```bash
cargo add argon2 --features getrandom,std
cargo add jsonwebtoken --features rust_crypto
cargo add serde --features derive
```

```toml
argon2 = { version = "0.6.0", features = ["getrandom", "std"] }
jsonwebtoken = { version = "11.1.0", features = ["rust_crypto"] }
```

`jsonwebtoken` 11 needs a crypto provider feature. Without `rust_crypto` (or
`aws_lc_rs`) the code compiles and then **panics at the first `encode`**:

```
Could not automatically determine the process-level CryptoProvider from jsonwebtoken crate features.
```

## Password hashing: argon2, nothing else

**Never store a password in plain text, and never hash one with SHA-256,
SHA-512, MD5, or any other general-purpose hash.** Those are designed to be
fast, which is exactly wrong: a GPU computes billions of SHA-256 hashes per
second, so a leaked table of SHA-256 password hashes is cracked in hours.

A password hash must be deliberately slow and memory-hard. Use **argon2id**
(the current recommendation), or bcrypt/scrypt in an existing codebase. Argon2's
defaults are tuned for this; do not lower them to speed up your login endpoint.

```rust
use argon2::{Argon2, password_hash::{PasswordHasher, PasswordVerifier, phc::PasswordHash}};

let argon = Argon2::default();
let hash: String = argon.hash_password(b"correct horse battery staple").unwrap().to_string();
println!("hash = {hash}");
println!("len = {}", hash.len());

let parsed = PasswordHash::new(&hash).unwrap();
println!("algorithm = {}", parsed.algorithm);
println!("verify correct: {:?}", argon.verify_password(b"correct horse battery staple", &parsed).is_ok());
println!("verify wrong:   {:?}", argon.verify_password(b"hunter2", &parsed).is_ok());
println!("error for wrong: {}", argon.verify_password(b"hunter2", &parsed).unwrap_err());

let hash2: String = argon.hash_password(b"correct horse battery staple").unwrap().to_string();
println!("same password, different salt -> same hash? {}", hash == hash2);
```

Output:

```
hash = $argon2id$v=19$m=19456,t=2,p=1$/XeYoPYSQcyDQ4vrJHOCnA$GXWvcWDGaZZZegrl2sbXJHgk4wMZxw1wD+kSjH1pmT4
len = 97
algorithm = argon2id
verify correct: true
verify wrong:   false
error for wrong: invalid password
same password, different salt -> same hash? false
```

Read the hash string: `$argon2id$` names the algorithm, `m=19456,t=2,p=1` is
memory/iterations/parallelism, then the **salt**, then the digest. All of it in
one 97-character column — you never store a salt separately, and raising the
parameters later still verifies old hashes because each one carries its own.

The last line is the salt doing its job: two hashes of the same password differ,
so a rainbow table is useless and a leak does not reveal which users share a
password.

`hash_password` in argon2 0.6 generates the salt itself (that is what the
`getrandom` feature is for). Older guides pass a `SaltString` explicitly; that
API is gone.

Hashing takes tens of milliseconds by design. That is long enough to matter on a
tokio worker thread, so in a high-traffic service move it off:

```rust
let hash = tokio::task::spawn_blocking(move || {
    Argon2::default().hash_password(password.as_bytes()).map(|h| h.to_string())
}).await??;
```

## JWT: what it is

Three base64url segments joined by dots: `header.payload.signature`. The payload
is **signed, not encrypted** — anyone holding the token can read it.

```rust
#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,   // subject — the user id
    role: String,  // a custom claim
    exp: u64,      // expiry, seconds since epoch — REQUIRED
    iat: u64,      // issued at
}

fn now() -> u64 {
    std::time::SystemTime::now().duration_since(std::time::UNIX_EPOCH).unwrap().as_secs()
}

let secret = b"a-very-long-random-secret-from-env";
let enc = EncodingKey::from_secret(secret);
let dec = DecodingKey::from_secret(secret);

let iat = now();
let claims = Claims { sub: "42".into(), role: "admin".into(), iat, exp: iat + 900 };
let token = encode(&Header::default(), &claims, &enc).unwrap();
println!("token = {token}");
println!("segments = {}", token.split('.').count());
```

Output:

```
token = eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTc5Mjk0OTYzNywiaWF0IjoxNzkyOTQ4NzM3fQ.KzeUkpFJeorQZwN3u_xjkRpv2NRuAqEM6x_fgirKDRA
segments = 3
```

Decoding the first two segments with plain base64 — no key needed:

```
header = {"typ":"JWT","alg":"HS256"}
payload = {"sub":"42","role":"admin","exp":1792949637,"iat":1792948737}
```

That is the whole payload, in the clear. **Never put a secret in a JWT.** No
passwords, no API keys, no personal data you would not hand to the bearer.

Standard claims: `sub` (subject), `exp` (expiry), `iat` (issued at), `nbf` (not
before), `iss` (issuer), `aud` (audience), `jti` (token id). `exp` is the one
that must always be there.

Verifying:

```rust
let data = decode::<Claims>(&token, &dec, &Validation::new(Algorithm::HS256)).unwrap();
println!("decoded sub={} role={}", data.claims.sub, data.claims.role);
```

Output:

```
decoded sub=42 role=admin
```

HS256 is symmetric — the same secret signs and verifies, which suits one service
issuing tokens for itself. RS256/ES256 are asymmetric: a private key signs, a
public key verifies, so other services can validate without being able to mint
tokens. Use those once more than one service is involved.

## Every way decoding fails

```rust
let wrong = DecodingKey::from_secret(b"different-secret");
let e = decode::<Claims>(&token, &wrong, &Validation::new(Algorithm::HS256)).unwrap_err();
println!("wrong secret: {:?} / {}", e.kind(), e);

let expired = Claims { sub: "42".into(), role: "user".into(), iat: iat - 3600, exp: iat - 1800 };
let et = encode(&Header::default(), &expired, &enc).unwrap();
let e = decode::<Claims>(&et, &dec, &Validation::new(Algorithm::HS256)).unwrap_err();
println!("expired: {:?} / {}", e.kind(), e);
println!("is ExpiredSignature: {}", matches!(e.kind(), ErrorKind::ExpiredSignature));

let e = decode::<Claims>("not.a.token", &dec, &Validation::new(Algorithm::HS256)).unwrap_err();
println!("malformed: {:?} / {}", e.kind(), e);

let e = decode::<Claims>(&token, &dec, &Validation::new(Algorithm::HS512)).unwrap_err();
println!("wrong algorithm: {:?} / {}", e.kind(), e);
```

Output:

```
wrong secret: InvalidSignature / InvalidSignature
expired: ExpiredSignature / ExpiredSignature
is ExpiredSignature: true
malformed: Base64(InvalidLastSymbol(2, 116)) / Base64 error: Invalid last symbol 116, offset 2.
wrong algorithm: InvalidAlgorithm / InvalidAlgorithm
```

`Validation::new(alg)` checks `exp` and the algorithm by default. The algorithm
check is a real security control: without pinning it, an attacker can present a
token whose header says `alg: none` and skip the signature entirely.

More `Validation` knobs:

```rust
let mut v = Validation::new(Algorithm::HS256);
v.set_issuer(&["my-api"]);
v.set_audience(&["my-app"]);
v.set_required_spec_claims(&["exp", "iss"]);
v.leeway = 60;              // seconds of clock skew tolerance; 60 is the default
v.validate_exp = true;      // on by default
v.validate_nbf = false;
```

Note that `set_issuer` only checks `iss` **when present** — to make it mandatory
add it to `set_required_spec_claims`:

```
missing iss: MissingRequiredClaim("iss") / Missing required claim: iss
```

And `leeway` genuinely forgives recent expiry, which matters across machines
whose clocks differ:

```rust
let mut v = Validation::new(Algorithm::HS256);
v.leeway = 60;
let almost = Claims { sub: "42".into(), role: "user".into(), iat, exp: iat - 30 };
let at = encode(&Header::default(), &almost, &enc).unwrap();
println!("30s expired with 60s leeway: {:?}", decode::<Claims>(&at, &dec, &v).map(|d| d.claims.sub));
```

Output:

```
30s expired with 60s leeway: Ok("42")
```

## Secrets from the environment

```rust
let secret = std::env::var("JWT_SECRET")
    .expect("JWT_SECRET must be set");
```

Rules:

- **Never commit a secret.** A hardcoded fallback is the bug that leaks
  production — the example server below uses one only so it runs out of the box,
  and says so.
- At least 32 bytes of randomness: `openssl rand -base64 48`.
- Keep it out of logs — a `Debug` impl on a config struct will happily print it,
  so use a wrapper that redacts (`secrecy::Secret<String>`).
- Rotating it invalidates every live token. Support two decoding keys during a
  rotation window if that matters.

Lesson 18.1 covers configuration properly.

## Keys in state

Build the keys once at startup — `EncodingKey::from_secret` does work you do not
want per request.

```rust
#[derive(Clone)]
struct Keys { enc: Arc<EncodingKey>, dec: Arc<DecodingKey> }

impl Keys {
    fn new(secret: &[u8]) -> Self {
        Self {
            enc: Arc::new(EncodingKey::from_secret(secret)),
            dec: Arc::new(DecodingKey::from_secret(secret)),
        }
    }
}

#[derive(Clone)]
struct AppState { keys: Keys, users: Arc<RwLock<HashMap<String, StoredUser>>> }

impl FromRef<AppState> for Keys {
    fn from_ref(s: &AppState) -> Keys { s.keys.clone() }
}
```

The `FromRef` impl (lesson 17.5) is what makes the extractor below reusable: it
is generic over any state from which `Keys` can be extracted, so it drops into
another app unchanged.

## The auth error type

```rust
#[derive(Debug)]
enum AuthError { MissingHeader, BadScheme, Expired, Invalid, Forbidden(String) }

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, code, msg) = match &self {
            AuthError::MissingHeader => (StatusCode::UNAUTHORIZED, "missing_token",
                "Authorization header required".to_string()),
            AuthError::BadScheme => (StatusCode::UNAUTHORIZED, "bad_scheme",
                "expected `Authorization: Bearer <token>`".to_string()),
            AuthError::Expired => (StatusCode::UNAUTHORIZED, "token_expired",
                "token has expired".to_string()),
            AuthError::Invalid => (StatusCode::UNAUTHORIZED, "invalid_token",
                "token is not valid".to_string()),
            AuthError::Forbidden(r) => (StatusCode::FORBIDDEN, "forbidden",
                format!("requires role {r}")),
        };
        let mut res = (status, Json(json!({ "error": code, "message": msg }))).into_response();
        if status == StatusCode::UNAUTHORIZED {
            res.headers_mut().insert(header::WWW_AUTHENTICATE,
                header::HeaderValue::from_static("Bearer"));
        }
        res
    }
}
```

A distinct `token_expired` code is worth having: it is the signal for a client
to use its refresh token rather than send the user back to the login screen.
`WWW-Authenticate` on a 401 is what the HTTP spec asks for.

## The `AuthUser` extractor

```rust
struct AuthUser { id: u64, role: String }

impl<S> FromRequestParts<S> for AuthUser
where Keys: FromRef<S>, S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let raw = parts.headers.get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .ok_or(AuthError::MissingHeader)?;
        let token = raw.strip_prefix("Bearer ").ok_or(AuthError::BadScheme)?.trim();

        let keys = Keys::from_ref(state);
        let data = decode::<Claims>(token, &keys.dec, &Validation::new(Algorithm::HS256))
            .map_err(|e| match e.kind() {
                ErrorKind::ExpiredSignature => AuthError::Expired,
                _ => AuthError::Invalid,
            })?;

        Ok(AuthUser {
            id: data.claims.sub.parse().map_err(|_| AuthError::Invalid)?,
            role: data.claims.role,
        })
    }
}
```

Protecting a route is now one argument:

```rust
async fn me(user: AuthUser) -> impl IntoResponse {
    Json(json!({ "id": user.id, "role": user.role }))
}

async fn public() -> &'static str { "anyone can read this" }
```

```rust
let app = Router::new()
    .route("/public", get(public))
    .route("/me", get(me))
    .with_state(state);
```

No `.layer`, no route grouping — the handler signature is the access control,
and it is visible at a glance.

## Role-based authorization

Wrap the extractor in another extractor.

```rust
struct AdminUser(AuthUser);

impl<S> FromRequestParts<S> for AdminUser
where Keys: FromRef<S>, S: Send + Sync,
{
    type Rejection = AuthError;
    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let u = AuthUser::from_request_parts(parts, state).await?;
        if u.role != "admin" {
            return Err(AuthError::Forbidden("admin".into()));
        }
        Ok(AdminUser(u))
    }
}

async fn admin_only(AdminUser(u): AdminUser) -> impl IntoResponse {
    Json(json!({ "secret": "the launch codes", "for_user": u.id }))
}
```

Composition all the way down: `AdminUser` reuses `AuthUser`'s header parsing and
token validation and adds one check. For finer-grained permissions, put a
`Vec<String>` of scopes in the claims and make the extractor generic over a
const scope name.

The caveat with claims-based roles: a role put in a token at login **stays
valid until the token expires**. Demote a user and they keep admin access for up
to 15 minutes. That is the trade-off stateless JWTs make, and the reason to keep
access-token lifetimes short.

## Register and login handlers

```rust
#[derive(Deserialize)]
struct Credentials { email: String, password: String }

async fn register(State(st): State<AppState>, Json(c): Json<Credentials>) -> impl IntoResponse {
    if c.password.len() < 8 {
        return (StatusCode::UNPROCESSABLE_ENTITY,
            Json(json!({"error":"password too short"}))).into_response();
    }
    let hash: String = Argon2::default().hash_password(c.password.as_bytes()).unwrap().to_string();
    let mut users = st.users.write().unwrap();
    if users.contains_key(&c.email) {
        return (StatusCode::CONFLICT, Json(json!({"error":"email taken"}))).into_response();
    }
    let id = users.len() as u64 + 1;
    users.insert(c.email.clone(), StoredUser {
        id, email: c.email.clone(), password_hash: hash, role: "user".into(),
    });
    (StatusCode::CREATED, Json(json!({"id": id, "email": c.email}))).into_response()
}

async fn login(State(st): State<AppState>, Json(c): Json<Credentials>) -> impl IntoResponse {
    let user = st.users.read().unwrap().get(&c.email).cloned();
    let Some(user) = user else {
        return (StatusCode::UNAUTHORIZED, Json(json!({"error":"invalid credentials"}))).into_response();
    };
    let parsed = PasswordHash::new(&user.password_hash).unwrap();
    if Argon2::default().verify_password(c.password.as_bytes(), &parsed).is_err() {
        return (StatusCode::UNAUTHORIZED, Json(json!({"error":"invalid credentials"}))).into_response();
    }

    let iat = now();
    let access = Claims { sub: user.id.to_string(), role: user.role.clone(), iat, exp: iat + 900 };
    let token = encode(&Header::default(), &access, &st.keys.enc).unwrap();
    let refresh = Claims { sub: user.id.to_string(), role: "refresh".into(), iat, exp: iat + 60 * 60 * 24 * 30 };
    let refresh = encode(&Header::default(), &refresh, &st.keys.enc).unwrap();

    (StatusCode::OK, Json(json!({
        "access_token": token, "refresh_token": refresh,
        "token_type": "Bearer", "expires_in": 900
    }))).into_response()
}
```

The `let Some(user) = user else` and the verify failure return the **same**
message. Distinguishing "no such email" from "wrong password" is a user
enumeration vulnerability — an attacker learns which addresses are registered.

Note the `.cloned()` before matching: it releases the read lock before the
argon2 verify, which takes tens of milliseconds.

## The flow end to end

```console
$ curl -s localhost:52512/public
anyone can read this

$ curl -si -X POST localhost:52512/register -H 'content-type: application/json' \
    -d '{"email":"raj@x.com","password":"longenough"}' | sed -n '1p;$p'
HTTP/1.1 201 Created
{"email":"raj@x.com","id":2}

$ curl -si -X POST localhost:52512/register -H 'content-type: application/json' \
    -d '{"email":"a@b.co","password":"x"}' | sed -n '1p;$p'
HTTP/1.1 422 Unprocessable Entity
{"error":"password too short"}

$ curl -si -X POST localhost:52512/register -H 'content-type: application/json' \
    -d '{"email":"raj@x.com","password":"longenough"}' | sed -n '1p;$p'
HTTP/1.1 409 Conflict
{"error":"email taken"}
```

Both login failures look identical from outside:

```console
$ curl -si -X POST localhost:52512/login -H 'content-type: application/json' \
    -d '{"email":"raj@x.com","password":"wrongpass"}' | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":"invalid credentials"}

$ curl -si -X POST localhost:52512/login -H 'content-type: application/json' \
    -d '{"email":"nope@x.com","password":"whatever"}' | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":"invalid credentials"}
```

A successful login (tokens truncated):

```console
$ curl -s -X POST localhost:52512/login -H 'content-type: application/json' \
    -d '{"email":"raj@x.com","password":"longenough"}'
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ...",
  "expires_in": 900,
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ...",
  "token_type": "Bearer"
}
```

Then every way `/me` can be rejected, and the one way it succeeds:

```console
$ curl -si localhost:52512/me | sed -n '1p;/www-auth/p;$p'
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"error":"missing_token","message":"Authorization header required"}

$ curl -si -H "Authorization: Token $TOK" localhost:52512/me | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":"bad_scheme","message":"expected `Authorization: Bearer <token>`"}

$ curl -si -H "Authorization: Bearer not.a.jwt" localhost:52512/me | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":"invalid_token","message":"token is not valid"}

$ curl -si -H "Authorization: Bearer $TOK" localhost:52512/me | sed -n '1p;$p'
HTTP/1.1 200 OK
{"id":2,"role":"user"}
```

An expired token, and one with its last character changed:

```console
$ curl -si -H "Authorization: Bearer $ETOK" localhost:52512/me | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":"token_expired","message":"token has expired"}

$ curl -si -H "Authorization: Bearer ${TOK%?}X" localhost:52512/me | sed -n '1p;$p'
HTTP/1.1 401 Unauthorized
{"error":"invalid_token","message":"token is not valid"}
```

One flipped character invalidates the signature — that is the whole point.

Authorization:

```console
$ curl -si -H "Authorization: Bearer $TOK" localhost:52512/admin | sed -n '1p;$p'
HTTP/1.1 403 Forbidden
{"error":"forbidden","message":"requires role admin"}

$ curl -si -H "Authorization: Bearer $ATOK" localhost:52512/admin | sed -n '1p;$p'
HTTP/1.1 200 OK
{"for_user":100,"secret":"the launch codes"}
```

401 for the missing/bad token, 403 for the valid token without the role —
exactly the distinction lesson 17.6 insisted on.

## Expiry and refresh tokens

Two tokens, different lifetimes:

| | Access token | Refresh token |
|---|---|---|
| Lifetime | 5-15 minutes | days to weeks |
| Sent | on every request | only to `/refresh` |
| Stored | memory | httpOnly cookie or secure storage |
| Revocable | no (too short to matter) | yes, if you persist it |

A `/refresh` endpoint takes the refresh token, verifies it, and issues a new
access token:

```rust
async fn refresh(State(st): State<AppState>, Json(body): Json<RefreshRequest>) -> impl IntoResponse {
    let data = match decode::<Claims>(&body.refresh_token, &st.keys.dec,
                                      &Validation::new(Algorithm::HS256)) {
        Ok(d) if d.claims.role == "refresh" => d,
        _ => return (StatusCode::UNAUTHORIZED, Json(json!({"error":"invalid refresh token"}))).into_response(),
    };
    // Look the user up again here — role and status may have changed.
    let iat = now();
    let access = Claims { sub: data.claims.sub, role: lookup_role(), iat, exp: iat + 900 };
    (StatusCode::OK, Json(json!({ "access_token": encode(&Header::default(), &access, &st.keys.enc).unwrap() }))).into_response()
}
```

Three things make this actually secure, none of which the stateless version
above does:

- **Mark the token type in the claims** (the `role == "refresh"` check), so an
  access token cannot be used to refresh or vice versa.
- **Re-read the user on refresh.** This is the moment a revoked account or
  changed role takes effect.
- **Store refresh tokens server-side** (a `jti` claim and a table) if you need
  real logout. Rotate on each use and treat reuse of an old one as theft —
  invalidate the whole family.

Because a JWT is stateless, there is no server-side logout for access tokens.
"Logout" means the client discards them; a short expiry is what limits the
damage. If you need instant revocation, keep a deny-list of `jti`s in Redis
checked by the extractor — at which point you have accepted statefulness and
server-side sessions become the simpler option.

## Header or cookie?

| | `Authorization: Bearer` header | httpOnly cookie |
|---|---|---|
| CSRF | immune — the browser doesn't attach it | vulnerable; needs `SameSite` + a CSRF token |
| XSS | token in JS reach if stored in `localStorage` | `httpOnly` keeps JS out |
| Mobile / server-to-server | natural | awkward |
| Setup | none | `Secure`, `HttpOnly`, `SameSite=Lax` |

Headers for APIs and mobile clients; `httpOnly` cookies for a browser app on
your own domain, with `SameSite=Lax` or `Strict`. Never put a JWT in
`localStorage` for a site that could have an XSS bug — any injected script can
read it. For cookies, `axum_extra::extract::cookie::CookieJar` is the extractor.

## Sessions are the alternative

Server-side sessions — a random opaque id in a cookie, state in Redis or
Postgres — are revocable instantly, carry no claims staleness, and are simpler to
reason about; the cost is a store lookup per request and sticky state. For a
single app with a browser frontend they are often the better choice; `tower-sessions`
integrates with axum.

## Quick Reference

| Need | Form |
|---|---|
| Hash a password | `Argon2::default().hash_password(pw.as_bytes())?.to_string()` |
| Verify | `argon.verify_password(pw, &PasswordHash::new(&stored)?).is_ok()` |
| Off the async thread | `tokio::task::spawn_blocking(move \|\| ...)` |
| Keys | `EncodingKey::from_secret(s)` / `DecodingKey::from_secret(s)` |
| Sign | `encode(&Header::default(), &claims, &enc)?` |
| Verify | `decode::<Claims>(tok, &dec, &Validation::new(Algorithm::HS256))?` |
| Required claims | `sub`, `exp`, `iat` |
| Pin algorithm | `Validation::new(Algorithm::HS256)` |
| Require `iss` | `v.set_issuer(&[..]); v.set_required_spec_claims(&["exp","iss"])` |
| Clock skew | `v.leeway = 60` |
| Expired? | `matches!(e.kind(), ErrorKind::ExpiredSignature)` |
| Protect a route | add `user: AuthUser` to the handler |
| Require a role | a second extractor wrapping the first |
| Reusable extractor | `impl<S> FromRequestParts<S> for T where Keys: FromRef<S>` |
| 401 header | `WWW-Authenticate: Bearer` |
| Secret | `std::env::var("JWT_SECRET")`, 32+ random bytes |

## Common Mistakes

**SHA-256 (or MD5, or plain text) for passwords.** Fast hashes are crackable at
GPU speed. argon2id, bcrypt, or scrypt only.

**Putting anything secret in a JWT.** The payload is base64, not encrypted —
anyone can read it.

**No `exp` claim.** A token without an expiry is valid forever.

**Not pinning the algorithm.** Opens the `alg: none` attack.

**Different messages for "unknown email" and "wrong password".** That is user
enumeration.

**A hardcoded or committed secret.** Read it from the environment and keep it
out of `Debug` output.

**Expecting a role change to take effect immediately.** Claims are frozen until
the token expires. Keep access tokens short and re-read the user on refresh.

**Returning 403 for a missing token.** 401 means "authenticate"; 403 means
"authenticated, not allowed".

**A JWT in `localStorage`.** Any XSS reads it. Use a header with in-memory
storage, or an `httpOnly` cookie.

**Forgetting the `rust_crypto` feature.** It compiles and panics on the first
`encode`.

## Exercises

1. Hash `"password123"` with argon2, print the PHC string, then verify it against
   the right and the wrong password. Hash it twice and confirm the hashes differ.
2. Mint a token with a 2-second expiry, decode it successfully, `sleep(3)`, and
   decode again. Match `ErrorKind::ExpiredSignature` in the error.
3. Build the `AuthUser` extractor and a `/me` route. Verify four responses:
   no header, wrong scheme, garbage token, and a valid token.
4. Add a `permissions: Vec<String>` claim and a `RequirePermission<const P: &str>`
   extractor that 403s when the permission is absent. Use it on two routes.
5. Add `/refresh` that accepts a refresh token, rejects an access token presented
   to it, and issues a new access token. Prove both branches with curl.
6. Make the `AuthUser` extractor reject a token whose `iss` is not `"my-api"`,
   using `set_issuer` plus `set_required_spec_claims`, and record the error.

---

[← 17.10 Validation and Request Guards](10-validation.md) · [Course Index](../README.md) · [17.12 HTTP Clients with reqwest →](12-reqwest.md)
