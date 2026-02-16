# JWTs — Stateless Authentication and Its Consequences

## The Motivation

Sessions require a server-side lookup on every request. At scale — millions
of requests per second across dozens of services — this becomes a bottleneck.
Every service needs access to the session store. The session store becomes a
single point of failure.

JWTs (JSON Web Tokens) flip the model: **put the session data in the token
itself, sign it so it can't be tampered with, and let the client carry it.**
No server-side lookup. Any service that has the signing key (or public key)
can verify the token independently.

This is the core tradeoff. You gain statelessness. You lose instant
revocation. Everything else about JWTs follows from this.

---

## Anatomy of a JWT

A JWT is three Base64url-encoded segments separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTcwODAxMjgwMCwiZXhwIjoxNzA4MDk5MjAwfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
│                                        │                                                                                    │
└── Header                               └── Payload                                                                          └── Signature
```

### Why Base64url and Not Base64

Standard Base64 uses `+`, `/`, and `=` as characters. These have meaning in
URLs (`+` is a space, `/` is a path separator, `=` is a query parameter
delimiter). Base64url replaces them:

```
Base64:    + / =
Base64url: - _ (padding omitted)
```

This lets JWTs be passed in URL query parameters, HTTP headers, and HTML
forms without encoding issues. It's defined in RFC 4648 Section 5.

### Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- **`alg`**: The signing algorithm. This tells the verifier how to check the
  signature. Common values: `HS256` (HMAC-SHA256), `RS256` (RSA-SHA256),
  `ES256` (ECDSA-P256-SHA256), `EdDSA` (Ed25519).

- **`typ`**: Token type. Almost always `"JWT"`. Exists for cases where JWTs
  are mixed with other token types in the same system.

- **`kid`** (optional): Key ID. When the server uses multiple signing keys
  (key rotation), this tells the verifier which key to use.

### Payload (Claims)

```json
{
  "sub": "42",
  "role": "admin",
  "iat": 1708012800,
  "exp": 1708099200
}
```

Claims are statements about the user and the token. Three categories:

**Registered claims** (RFC 7519, standardized names):

| Claim | Name | Meaning |
|-------|------|---------|
| `iss` | Issuer | Who created this token (e.g., `"auth.yourapp.com"`) |
| `sub` | Subject | Who this token is about (user ID) |
| `aud` | Audience | Who this token is intended for (e.g., `"api.yourapp.com"`) |
| `exp` | Expiration | Unix timestamp after which the token is invalid |
| `nbf` | Not Before | Unix timestamp before which the token is invalid |
| `iat` | Issued At | Unix timestamp when the token was created |
| `jti` | JWT ID | Unique identifier for this specific token (for revocation tracking) |

**Public claims**: Registered in the IANA JSON Web Token Claims registry or
defined as URIs to avoid collision. Example: `"email"`, `"name"` from OpenID
Connect.

**Private claims**: Application-specific. Anything you want. `"role"`,
`"tenant_id"`, `"permissions"`. No collision protection — just don't reuse
registered names.

**The payload is NOT encrypted.** It's Base64url-encoded, which is a
reversible encoding, not encryption. Anyone who has the token can decode the
payload and read every claim. Never put secrets in JWT payloads.

```bash
echo "eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiJ9" | base64 -d
# {"sub":"42","role":"admin"}
```

### Signature

The signature ensures the token hasn't been tampered with. It's computed
over the header and payload:

```
signature = HMAC-SHA256(
    base64url(header) + "." + base64url(payload),
    secret_key
)
```

The signed content includes the dot separator — it's a signature of the
exact string `"eyJhbGci...InR5cCI6IkpXVCJ9.eyJzdWIi..."`, not of the
decoded JSON objects. This prevents attacks where the same JSON could be
encoded differently.

---

## Signing Algorithms — The Crypto Under the Hood

### HMAC-SHA256 (HS256) — Symmetric

**One key for both signing and verification.**

```
HMAC(key, message):
    if len(key) > block_size:
        key = SHA256(key)
    key = pad(key, block_size)           // pad to 512 bits (SHA-256 block size)

    o_key_pad = key XOR 0x5c5c5c...     // outer padding
    i_key_pad = key XOR 0x3636363636... // inner padding

    return SHA256(o_key_pad || SHA256(i_key_pad || message))
```

HMAC is a nested hash: hash the message with the key (inner), then hash
that result with the key again (outer). The two different padding constants
(0x5c and 0x36) ensure the inner and outer operations are independent — you
can't use the inner hash to forge the outer hash.

**Why not just `SHA256(key + message)`?** Length extension attacks. SHA-256
(and all Merkle-Damgård hashes) have a property where knowing `SHA256(X)`
lets you compute `SHA256(X + padding + Y)` without knowing `X`. HMAC's
nested structure prevents this.

**Implication**: Both the token issuer and every service that verifies tokens
must have the same secret key. If any verifying service is compromised, the
attacker can forge tokens for all services. This is why HMAC is only
appropriate when the issuer and verifier are the same service, or when you
fully trust all verifiers.

### RSA-SHA256 (RS256) — Asymmetric

**Private key signs, public key verifies. Different keys.**

```
Signing:
    hash = SHA256(header + "." + payload)
    signature = RSA_SIGN(hash, private_key)
    // internally: signature = hash^d mod n
    // where d is the private exponent, n is the modulus

Verification:
    hash = SHA256(header + "." + payload)
    recovered = RSA_VERIFY(signature, public_key)
    // internally: recovered = signature^e mod n
    // where e is the public exponent (typically 65537)
    return hash == recovered
```

The math: RSA relies on the difficulty of factoring large numbers. The
modulus `n = p × q` (two large primes). Knowing `n` and `e` (public), you
can verify. Computing `d` (private) requires knowing `p` and `q`, which
requires factoring `n` — infeasible for 2048+ bit keys.

**Implication**: The auth server keeps the private key. Every other service
only needs the public key. A compromised API server can verify tokens but
can't forge new ones. This is the key advantage in microservice architectures.

**Cost**: RSA signing is ~100x slower than HMAC. RSA verification is ~10x
slower. For a high-throughput auth server issuing thousands of tokens per
second, this matters. For API servers verifying tokens, the ~10x overhead is
usually acceptable.

**Key size**: RSA-2048 produces a 256-byte signature. This makes the JWT
significantly larger than HMAC (32 bytes). RSA-4096 doubles it to 512 bytes.

### ECDSA-P256-SHA256 (ES256) — Asymmetric, Smaller

Elliptic Curve Digital Signature Algorithm. Same asymmetric property as RSA
but with smaller keys and signatures.

```
Signing:
    hash = SHA256(header + "." + payload)
    k = random_nonce()                  // CRITICAL: must be unique per signature
    (r, s) = ECDSA_SIGN(hash, private_key, k)
    signature = r || s                  // concatenation of two 32-byte integers

Verification:
    hash = SHA256(header + "." + payload)
    valid = ECDSA_VERIFY(hash, signature, public_key)
```

The math: ECDSA relies on the difficulty of the Elliptic Curve Discrete
Logarithm Problem. Given a point `P` on a curve and `Q = k × P` (scalar
multiplication of a point), finding `k` is infeasible.

**The nonce `k` is critical.** If the same `k` is ever used for two
different messages, the private key can be recovered algebraically. This
is what broke the PlayStation 3 code signing — Sony used a fixed `k` for
every signature, allowing the private key to be extracted from any two
signed updates.

Modern implementations use RFC 6979 (deterministic nonce derived from the
private key and message) to eliminate this risk.

**Key/signature size comparison:**

| Algorithm | Key size | Signature size |
|-----------|----------|----------------|
| HS256 | 256 bits (shared) | 32 bytes |
| RS256 | 2048 bits (private) | 256 bytes |
| ES256 | 256 bits (private) | 64 bytes |
| EdDSA (Ed25519) | 256 bits (private) | 64 bytes |

ES256 gives you asymmetric security with signature sizes close to HMAC.
This matters because the JWT is sent on **every request** — a 256-byte RSA
signature adds ~350 bytes to every HTTP request after Base64url encoding.

### EdDSA (Ed25519) — Modern Asymmetric

Edwards-curve Digital Signature Algorithm using Curve25519. Fastest
asymmetric option with strong security properties.

Advantages over ECDSA:
- **Deterministic**: No random nonce needed — eliminates the class of bugs
  that broke PS3
- **Constant-time by design**: The algorithm is designed to resist timing
  side-channels without special implementation effort
- **Faster**: ~5x faster than ECDSA for signing, ~2x for verification
- **Simpler implementation**: Fewer parameters, less room for error

Growing adoption. Supported in JOSE (JSON Object Signing and Encryption)
since 2017. Use this if your ecosystem supports it.

---

## The Revocation Problem

This is the fundamental limitation of JWTs and the reason they don't simply
replace sessions.

### Why Revocation Is Hard

A session is a pointer to server-side state. To revoke it, delete the state.
Next request fails. Instant.

A JWT is self-contained. The server doesn't track it. To "revoke" it, you'd
need to... what? The token is valid because its signature is valid and its
`exp` hasn't passed. There's no server-side state to delete.

**Scenarios where you need revocation:**
1. User logs out — their token should stop working
2. User changes password — all existing tokens should be invalid
3. Admin bans a user — immediate effect needed
4. Token is stolen — must be invalidated before it expires
5. User's permissions change — old token has stale claims

### Attempted Solutions (All Re-Introduce State)

**Token blacklist / deny list:**

```
// On logout or revocation:
redis.set(`blacklist:${token.jti}`, true, 'EX', token.exp - now());

// On every request:
if (redis.exists(`blacklist:${token.jti}`)) {
    return 401;
}
```

This works but defeats the purpose. You're doing a server-side lookup on
every request — exactly what JWTs were supposed to eliminate. You now have
a stateful "stateless" system.

The blacklist is at least smaller than a session store (you only store
revoked tokens, not all tokens), and entries auto-expire when the token
would have expired anyway. But under adversarial conditions (mass revocation
during a security incident), the blacklist grows.

**Short-lived tokens + refresh tokens:**

Instead of one long-lived JWT, use two:
- **Access token**: Short-lived (5-15 minutes). Used on every request. Never
  checked against server state.
- **Refresh token**: Long-lived (days/weeks). Used only to get new access
  tokens. Checked against server state (it's basically a session).

When a user is banned or logs out, revoke the refresh token. The access
token continues working for up to 15 minutes, but then the user can't get
a new one.

**This is the industry standard pattern.** It's a compromise: you accept
up to 15 minutes of stale authorization in exchange for eliminating per-
request state lookups. Whether that window is acceptable depends on your
threat model.

**Token versioning:**

Store a `tokenVersion` counter per user in the database. Include it in the
JWT. On each request, compare the token's version to the stored version.
On revocation, increment the user's version.

```javascript
// In the JWT:
{ "sub": "42", "tokenVersion": 3 }

// On verification:
const user = await db.users.findById(token.sub);
if (token.tokenVersion !== user.tokenVersion) return 401;
```

This requires a database lookup per request — but it's a simple integer
comparison on a primary key lookup, which is fast. And it gives you instant
revocation for all tokens at once (just increment the version).

Tradeoff: you've lost pure statelessness but kept most of the benefits
(any service with DB access can verify, no session store, no session data
to synchronize).

---

## Where to Store JWTs on the Client

This is one of the most debated topics in web security. Each option has
real tradeoffs.

### `localStorage`

```javascript
localStorage.setItem('token', jwt);

// On each request:
fetch('/api/data', {
    headers: { 'Authorization': `Bearer ${token}` }
});
```

**Vulnerability: XSS**. Any JavaScript running on your page can read
`localStorage`. A single XSS vulnerability means the attacker exfiltrates
the token and uses it from their own machine, from anywhere, for the full
token lifetime. The token is completely stolen — not just used in-context
like an XSS with HttpOnly cookies.

**Advantage**: No CSRF vulnerability. The token isn't sent automatically —
your code explicitly attaches it. Cross-origin requests from `evil.com`
don't include it.

### `HttpOnly Cookie`

```
Set-Cookie: token=eyJ...; HttpOnly; Secure; SameSite=Lax
```

**Protection**: JavaScript can't read it. XSS can't exfiltrate the token.
The browser manages it automatically.

**Vulnerability: CSRF** (mitigated by SameSite=Lax). Also, you're now
using cookies — which means you're essentially doing cookie-based auth with
a JWT as the cookie value. At this point, you have most of the downsides of
JWTs (size, can't revoke) without the main upside (statelessness, since
you're coupled to cookie behavior anyway).

**Size concern**: Cookies have a ~4KB limit per cookie. A JWT with many
claims + an RS256 signature can approach this. `localStorage` has a ~5MB
limit.

### In-Memory (JavaScript Variable)

```javascript
let token = null;

async function login(email, password) {
    const res = await fetch('/auth/login', { method: 'POST', ... });
    token = res.json().token;
}

// Lost on page refresh, tab close, navigation
```

**Most secure against persistent theft**: The token only exists in the
JavaScript runtime. It's not in any storage an attacker can access
post-XSS (if the XSS is transient). It's not in cookies.

**Problem**: Lost on every page refresh. Users must re-authenticate
constantly. Only viable in Single Page Applications (SPAs) that rarely
do full-page navigations.

**Pattern**: Store the access token in memory, store the refresh token in
an HttpOnly cookie. On page load, silently use the refresh token to get a
new access token. This gives you:
- Access token: immune to CSRF (not in a cookie), short-lived if XSS
  exfiltrates it
- Refresh token: immune to XSS exfiltration (HttpOnly), protected from
  CSRF by SameSite

This is the most security-conscious approach for SPAs.

### The Honest Answer

There is no perfectly secure client-side token storage on the web. Every
option trades one vulnerability class for another. The real defense is
**preventing XSS in the first place** — Content Security Policy, input
sanitization, output encoding, avoiding `innerHTML` and `eval()`. If you
have XSS, you've already lost regardless of where the token is stored.

---

## Known Attacks on JWTs

### The `alg: none` Attack

The JWT spec defines `"alg": "none"` as a valid algorithm — "no digital
signature or MAC performed." This was intended for cases where the JWT is
transmitted over a channel that already provides integrity (like inside an
encrypted payload).

**The attack (CVE-2015-9235):**

1. Take a valid JWT
2. Change the header to `{"alg": "none"}`
3. Modify the payload however you want (change role to admin)
4. Remove the signature (just leave the trailing dot)
5. Send: `eyJ...none...}.eyJ...admin...}.`

Early JWT libraries would read the `alg` header and process accordingly.
`alg: none` → skip signature verification → accept any payload.

**The fix**: Never trust the `alg` header from the token. The server should
have an explicit allowlist of algorithms and key configurations. Modern
libraries require you to specify the expected algorithm:

```javascript
// GOOD — server specifies algorithm
jwt.verify(token, publicKey, { algorithms: ['ES256'] });

// BAD — trusts the token's alg header
jwt.verify(token, key); // library reads alg from token header
```

### Algorithm Confusion (RSA/HMAC)

**The attack**: The server is configured to verify with RSA (RS256). The
attacker:

1. Obtains the server's **public** key (it's public — this is expected)
2. Creates a JWT with `{"alg": "HS256"}` (HMAC instead of RSA)
3. Signs it using `HMAC-SHA256(payload, public_key_as_hmac_secret)`
4. Sends the token to the server

If the library trusts the `alg` header:
- It sees `HS256` and does HMAC verification
- It uses the "key" it has configured — which is the RSA public key
- The verification succeeds because the attacker used that same public key
  as the HMAC secret

**The fix**: Same as above. Never trust the `alg` header. Explicitly
configure which algorithm and key type to expect. Modern libraries enforce
this.

### JWK/JKU Injection

The JWT header can contain:
- `jwk`: An embedded JSON Web Key (the public key to verify with)
- `jku`: A URL pointing to a JSON Web Key Set

**The attack**: The attacker sets `jwk` to their own public key and signs
with their own private key. If the server reads the key from the JWT header,
it verifies against the attacker's key — always valid.

Similarly, `jku` can point to `https://evil.com/keys.json` where the
attacker hosts their public key.

**The fix**: Never load verification keys from the token itself. Keys should
come from server configuration, a trusted JWKS endpoint that you control, or
a `kid` header mapped to a pre-registered key.

### `kid` Injection

The `kid` (Key ID) header identifies which key to use. If the server uses
`kid` in a database lookup or file path:

```javascript
// VULNERABLE
const key = fs.readFileSync(`/keys/${header.kid}`);
// attacker sets kid to "../../etc/passwd" or "../../dev/null"

// VULNERABLE
const key = db.query(`SELECT key FROM keys WHERE id = '${header.kid}'`);
// SQL injection through kid header
```

**The fix**: Validate and sanitize `kid`. Use it only as a lookup into a
pre-defined key map, never as a file path or raw SQL input.

---

## JWT Size and Performance

A JWT is sent on **every request** in the `Authorization` header. Size
matters.

### Typical Sizes

```
Minimal (HS256):
Header: {"alg":"HS256","typ":"JWT"}     → 36 bytes → 48 base64
Payload: {"sub":"42","exp":1708099200}  → 31 bytes → 44 base64
Signature: 32 bytes                     → 43 base64
Total: ~140 bytes

Typical (RS256, more claims):
Header + kid: ~80 bytes                 → 108 base64
Payload with role, email, etc: ~200 bytes → 268 base64
Signature: 256 bytes                    → 342 base64
Total: ~730 bytes

vs. Session cookie:
sid=a3f8c9d1e4b7... → ~70 bytes
```

A JWT with RSA is ~10x larger than a session cookie. Over millions of
requests, this means more bandwidth, larger HTTP headers, and potentially
exceeding cookie size limits (4KB) if stored in cookies.

### Impact

- **Bandwidth**: 730 bytes × 1 million requests = 730MB. Not catastrophic
  but not free.
- **Header parsing**: HTTP/1.1 sends headers uncompressed on every request.
  HTTP/2 compresses headers (HPACK) which helps, but the first request in a
  connection still sends the full token.
- **Lambda/serverless cold starts**: Larger headers mean more data to parse
  before the function can start processing.

**Practical advice**: Keep JWT payloads small. Don't stuff user profiles,
permissions arrays, or metadata into them. Use the JWT for identity
(`sub`, `role`) and fetch additional data from a service when needed.

---

## JWS vs JWE — Signing vs Encryption

JWT is actually an umbrella term. The token structure with three dot-
separated segments is a JWS (JSON Web Signature). There's also JWE (JSON
Web Encryption).

### JWS (What Everyone Calls "JWT")

```
header.payload.signature
```

The payload is **readable by anyone**. The signature guarantees it hasn't
been **modified**. This provides **integrity** and **authenticity**, not
**confidentiality**.

Use case: The client and intermediaries can read claims (for routing,
caching, logging) but can't modify them.

### JWE (Encrypted JWT)

```
header.encrypted_key.iv.ciphertext.auth_tag
```

Five segments instead of three. The payload is **encrypted**. Only the
holder of the decryption key can read it.

Use case: The token contains sensitive data that intermediaries shouldn't
see (PII, internal identifiers you don't want exposed).

### Nested JWT (JWS inside JWE)

Sign the token first (JWS), then encrypt it (JWE). The recipient decrypts,
then verifies the signature. This provides both confidentiality and integrity
with separate keys for each operation.

In practice, JWE is rarely used for authentication tokens. If you need
confidentiality, you're usually better off keeping data server-side
(sessions) rather than encrypting it in a token. JWE is more common in
inter-service communication where both parties are servers.

---

## The Access + Refresh Token Pattern (Deep Dive)

This is how production systems actually use JWTs:

### Token Roles

```
Access Token:
- Short-lived: 5-15 minutes
- Contains: sub, role, permissions, exp
- Sent on: every API request (Authorization: Bearer <token>)
- Stored: in-memory (SPA) or HttpOnly cookie
- Verified: signature + exp check only (no server state)

Refresh Token:
- Long-lived: 7-30 days
- Contains: sub, jti, exp (minimal claims)
- Sent to: only the /auth/refresh endpoint
- Stored: HttpOnly cookie (Secure, SameSite=Strict, Path=/auth)
- Verified: signature + exp + checked against server-side store
```

### The Flow

```
1. Login:
   POST /auth/login { email, password }
   → Verify password
   → Generate access token (JWT, 15min)
   → Generate refresh token (opaque or JWT, 30 days)
   → Store refresh token server-side (Redis or DB)
   → Return access token in body, refresh token in HttpOnly cookie

2. API requests:
   GET /api/data
   Authorization: Bearer <access_token>
   → Verify JWT signature + exp (no server lookup)
   → Process request

3. Access token expires:
   GET /api/data → 401 (token expired)

   POST /auth/refresh
   Cookie: refresh_token=<token>
   → Verify refresh token signature
   → Check refresh token against server-side store
   → If valid: generate new access token, return it
   → If revoked: return 401

4. Logout:
   POST /auth/logout
   → Delete refresh token from server-side store
   → Clear refresh token cookie
   → Access token still works for up to 15 minutes (acceptable tradeoff)

5. Security event (password change, suspicious activity):
   → Delete ALL refresh tokens for user
   → Within 15 minutes, all access tokens expire
   → User must re-authenticate on all devices
```

### Refresh Token Rotation

An additional defense against refresh token theft:

```
1. Client sends refresh token R1 to get new access token
2. Server validates R1
3. Server issues new access token AND new refresh token R2
4. Server invalidates R1 (one-time use)
5. Client now holds R2

If attacker stole R1:
   - If attacker uses R1 first: they get tokens, legitimate user's next
     refresh fails (R1 already used). User must re-login.
   - If legitimate user uses R1 first: R1 is consumed. Attacker tries R1,
     fails. Server detects reuse of consumed token → revokes entire
     refresh token family → all sessions for that user are terminated.
```

The "refresh token family" concept: track which refresh tokens descended
from the same login. If any consumed token in the family is reused, revoke
the entire family. This limits the damage window of a stolen refresh token.

---

## JWT vs Sessions — The Real Decision

| Factor | Sessions | JWTs (access + refresh) |
|--------|----------|------------------------|
| Server state | Required (every request) | Only for refresh tokens |
| Revocation | Instant | Delayed (up to token lifetime) |
| Scalability | Bounded by session store | Stateless verification scales infinitely |
| Cross-service auth | Each service needs session store access | Each service needs only the public key |
| Token size | ~70 bytes (cookie) | ~140-730 bytes |
| Complexity | Simple | Access/refresh/rotation logic |
| Attack surface | Session fixation, hijacking | alg:none, key confusion, storage debates |
| Best for | Monoliths, traditional web apps | Microservices, APIs, SPAs |

### When to Use Sessions

- Traditional server-rendered web applications
- When you need instant revocation (banking, healthcare)
- When you have a single server or already have Redis
- When simplicity matters more than horizontal scaling

### When to Use JWTs

- Microservice architectures where many services need to verify identity
- Third-party API access (the service can't access your session store)
- Mobile apps (cookies are awkward in native apps)
- Serverless / edge computing (no persistent connection to a session store)
- When you accept the delayed revocation tradeoff

### The Honest Take

Most applications don't need JWTs. A session cookie backed by Redis handles
millions of users. JWTs add complexity (two token types, rotation, storage
debates, algorithm attacks) for a scaling benefit most apps never reach.

But if you're building a platform with multiple services, third-party
integrations, or serverless functions — JWTs are the right tool. Just don't
use them because they're trendy.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| JWT = header + payload + signature, dot-separated, Base64url-encoded | It's signed, not encrypted — anyone can read the payload |
| Symmetric (HS256) vs Asymmetric (RS256, ES256, EdDSA) | Symmetric: same key signs and verifies. Asymmetric: private signs, public verifies |
| Revocation is the fundamental problem | Self-contained tokens can't be revoked without re-introducing server state |
| Access + refresh token pattern | Short-lived access (stateless), long-lived refresh (stateful). Industry standard |
| Never trust the `alg` header | Server must enforce expected algorithm. Prevents alg:none and key confusion |
| ECDSA nonce reuse = private key leak | PS3 was broken this way. Use Ed25519 or deterministic nonces |
| Token storage has no perfect answer | In-memory access + HttpOnly refresh is the best SPA pattern |
| Keep payloads small | The token is sent on every request. Identity only, not user profiles |
| JWTs don't replace sessions for most apps | Use JWTs when you need cross-service auth or stateless verification at scale |
