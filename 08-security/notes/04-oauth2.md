# OAuth 2.0 — Delegated Authorization From First Principles

## The Problem OAuth Solves

Before OAuth, if a third-party app wanted to access your data on another
service, you gave it your **password**. Literally. Early Twitter clients,
Facebook apps, email integrations — you typed your Gmail password into a
third-party website and hoped they didn't store it, abuse it, or get hacked.

The problems:
1. **Overprivileged**: The third party has your full credentials. They can do
   anything you can do — read all your email, send as you, delete your
   account.
2. **No revocation granularity**: To cut off access, you change your password,
   which breaks every other service using it.
3. **Trust problem**: You're trusting every third party with the keys to your
   entire account.
4. **Password proliferation**: Your password exists on N different servers.
   Any one of them gets breached, your account is compromised.

OAuth exists to solve this: **let a user grant limited access to their
resources on one service to another service, without sharing their
credentials.**

---

## The Roles

OAuth defines four roles. Understanding these precisely prevents confusion
throughout every flow.

### Resource Owner
The user. The human who owns the data and can grant access to it.

### Resource Server
The API that holds the user's data. Example: GitHub's API
(`api.github.com`), Google's Gmail API. It validates access tokens and
returns data.

### Client
The third-party application requesting access. Example: a CI/CD tool that
wants to read your GitHub repos. The client is NOT the user — it's the
application acting on the user's behalf.

Two types:
- **Confidential client**: Can securely store a secret. Server-side web
  apps, backend services. The client has a `client_secret` that only it
  and the authorization server know.
- **Public client**: Cannot securely store a secret. SPAs (JavaScript in
  the browser), mobile apps, desktop apps. The source code is accessible
  to the user, so any embedded secret is extractable. These clients use
  PKCE instead of a client secret.

### Authorization Server
The service that authenticates the user and issues tokens. Example: GitHub's
OAuth server (`github.com/login/oauth`), Google's (`accounts.google.com`).
Often the same organization as the resource server, but architecturally
separate.

The authorization server has two endpoints:
- **Authorization endpoint** (`/authorize`): User-facing. Displays the
  consent screen. Returns an authorization code via redirect.
- **Token endpoint** (`/token`): Machine-facing. Exchanges the authorization
  code for tokens. Returns access token (and optionally refresh token).

---

## Why OAuth 1.0 Died

OAuth 1.0 required cryptographic request signing. Every API request needed:
- A nonce (unique per request)
- A timestamp
- A signature computed over the HTTP method, URL, all parameters, and a
  shared secret using HMAC-SHA1
- A specific parameter ordering and encoding scheme

The signature base string construction was notoriously error-prone:

```
POST&https%3A%2F%2Fapi.example.com%2Fstatus&oauth_consumer_key%3D...
%26oauth_nonce%3D...%26oauth_signature_method%3DHMAC-SHA1%26...
```

One wrong encoding, one parameter in the wrong order, one missing
percent-encode, and the signature fails. Debugging was a nightmare.
Libraries had subtle interoperability issues. The spec was ambiguous in
places.

OAuth 2.0 made a radical decision: **drop request signing entirely and
rely on TLS (HTTPS) for transport security.** This dramatically simplified
implementation but made TLS mandatory — if your connection isn't encrypted,
tokens on the wire are exposed.

OAuth 2.0 also split the monolithic spec into multiple RFCs, introduced
multiple grant types for different use cases, and added the concept of
scopes for granular permissions.

---

## The Authorization Code Flow

This is the primary flow. If you understand this one deeply, the others are
variations.

### Why It Exists

The user needs to authenticate with the authorization server (type their
password into Google, not into the third-party app). But the result (an
access token) needs to get to the client application. The user's browser
is the intermediary — it talks to both parties. But the browser is also
untrusted (JavaScript can read URLs, browser extensions can intercept
traffic).

The authorization code flow solves this with a two-step exchange:
1. The browser gets an **authorization code** (useless on its own)
2. The client's backend exchanges the code for tokens via a **server-to-
   server** call (the browser never sees the tokens)

### The Full Flow — Every HTTP Request

**Step 0: Client Registration (One-Time Setup)**

Before any OAuth flow, the developer registers their application with the
authorization server. They receive:
- `client_id`: Public identifier for the application
- `client_secret`: Secret key (confidential clients only)
- Registered `redirect_uri`(s): Where the authorization server is allowed
  to send the user back to

The `redirect_uri` registration is a critical security control — we'll see
why.

**Step 1: Client Redirects User to Authorization Server**

```http
GET /authorize?
    response_type=code&
    client_id=abc123&
    redirect_uri=https://myapp.com/callback&
    scope=read:repos read:user&
    state=xYz789RandomString
HTTP/1.1
Host: github.com
```

Parameters:
- `response_type=code`: "I want an authorization code" (not a token directly)
- `client_id`: Identifies which application is requesting access
- `redirect_uri`: Where to send the user after they consent
- `scope`: What permissions the application is requesting (more on this later)
- `state`: Random string for CSRF protection (critical — explained below)

The user's browser is redirected to this URL. They now see the authorization
server's consent screen: "MyApp wants to access your repositories and
profile. Allow?"

**Step 2: User Authenticates and Consents**

This happens entirely between the user and the authorization server. The
client application is not involved. The user types their password into
GitHub's login page, not the client's page. If they have 2FA, they complete
it here.

The authorization server shows a consent screen listing the requested
scopes. The user can approve or deny.

**Step 3: Authorization Server Redirects Back with Code**

```http
HTTP/1.1 302 Found
Location: https://myapp.com/callback?
    code=SplxlOBeZQQYbYS6WxSbIA&
    state=xYz789RandomString
```

The authorization server redirects the user's browser back to the
`redirect_uri` with two parameters:
- `code`: The authorization code. A short-lived (typically 30-60 seconds),
  single-use string.
- `state`: The same string the client sent in step 1 (CSRF verification).

**The code is in the URL query string.** This means it appears in:
- The browser's address bar
- The browser's history
- Server logs of the redirect target
- The Referer header of any subsequent requests from that page

This is why the code is short-lived and single-use. It's also why PKCE
exists (covered below) — to prevent an attacker who intercepts the code
from using it.

**Step 4: Client Verifies State Parameter**

Before doing anything with the code, the client checks that the `state`
parameter matches what it sent in step 1.

**Why**: Without `state`, an attacker could:
1. Start an OAuth flow with their own account
2. Get the authorization code for their account
3. Trick the victim's browser into hitting the client's callback with the
   attacker's code: `https://myapp.com/callback?code=ATTACKERS_CODE`
4. The client exchanges the code and links the attacker's account to the
   victim's session
5. The attacker now has access to whatever the victim does in the app

This is a **CSRF attack on the OAuth callback**. The `state` parameter is
the CSRF token. It must be:
- Generated per-authorization-request
- Stored in the user's session (or a secure cookie)
- Cryptographically random
- Verified before the code exchange

**Step 5: Client Exchanges Code for Tokens (Server-to-Server)**

```http
POST /token HTTP/1.1
Host: github.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
redirect_uri=https://myapp.com/callback
```

This is a **direct server-to-server call**. The user's browser is not
involved. The client authenticates itself with `client_id` and
`client_secret` (via Basic auth or POST body).

The `redirect_uri` must exactly match what was sent in step 1. The
authorization server verifies this to prevent code theft via manipulated
redirect URIs.

**Step 6: Authorization Server Returns Tokens**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
    "access_token": "gho_16C7e42F292c6912E7710c838347Ae178B4a",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "ghr_1B4a2e77170c838347Ae178B4a16C7e42F292c6912E7710c",
    "scope": "read:repos read:user"
}
```

`Cache-Control: no-store` is required by the spec — tokens must never be
cached by intermediaries.

**Step 7: Client Uses Access Token**

```http
GET /user/repos HTTP/1.1
Host: api.github.com
Authorization: Bearer gho_16C7e42F292c6912E7710c838347Ae178B4a
```

The resource server validates the token (either by introspecting with the
authorization server or by verifying it as a JWT) and returns the requested
data, limited to the granted scopes.

### The Complete Diagram

```
User          Browser          Client (myapp.com)     Auth Server (github.com)    Resource Server (api.github.com)
 │               │                    │                        │                           │
 │ clicks        │                    │                        │                           │
 │ "Login with   │                    │                        │                           │
 │  GitHub"      │                    │                        │                           │
 │──────────────>│                    │                        │                           │
 │               │ GET /auth/github   │                        │                           │
 │               │───────────────────>│                        │                           │
 │               │                    │                        │                           │
 │               │                    │ generate state,        │                           │
 │               │                    │ store in session       │                           │
 │               │                    │                        │                           │
 │               │ 302 → github.com/authorize?                 │                           │
 │               │     client_id&redirect_uri&scope&state      │                           │
 │               │<───────────────────│                        │                           │
 │               │                    │                        │                           │
 │               │ GET /authorize?... │                        │                           │
 │               │────────────────────────────────────────────>│                           │
 │               │                    │                        │                           │
 │               │ login page         │                        │                           │
 │               │<────────────────────────────────────────────│                           │
 │               │                    │                        │                           │
 │ enters        │                    │                        │                           │
 │ password      │                    │                        │                           │
 │──────────────>│                    │                        │                           │
 │               │ POST /login (to GitHub)                     │                           │
 │               │────────────────────────────────────────────>│                           │
 │               │                    │                        │                           │
 │               │ consent screen     │                        │                           │
 │               │<────────────────────────────────────────────│                           │
 │               │                    │                        │                           │
 │ clicks Allow  │                    │                        │                           │
 │──────────────>│                    │                        │                           │
 │               │ POST consent (to GitHub)                    │                           │
 │               │────────────────────────────────────────────>│                           │
 │               │                    │                        │                           │
 │               │ 302 → myapp.com/callback?code&state         │                           │
 │               │<────────────────────────────────────────────│                           │
 │               │                    │                        │                           │
 │               │ GET /callback?code&state                    │                           │
 │               │───────────────────>│                        │                           │
 │               │                    │                        │                           │
 │               │                    │ verify state matches   │                           │
 │               │                    │ session state          │                           │
 │               │                    │                        │                           │
 │               │                    │ POST /token            │                           │
 │               │                    │  code + client_secret  │                           │
 │               │                    │───────────────────────>│                           │
 │               │                    │                        │                           │
 │               │                    │ { access_token,        │                           │
 │               │                    │   refresh_token }      │                           │
 │               │                    │<───────────────────────│                           │
 │               │                    │                        │                           │
 │               │                    │ create session,        │                           │
 │               │                    │ store tokens           │                           │
 │               │                    │                        │                           │
 │               │ 302 → /dashboard   │                        │                           │
 │               │ Set-Cookie: sid=.. │                        │                           │
 │               │<───────────────────│                        │                           │
 │               │                    │                        │                           │
 │               │                    │ GET /user/repos        │                           │
 │               │                    │ Authorization: Bearer  │                           │
 │               │                    │─────────────────────────────────────────────────────>│
 │               │                    │                        │                           │
 │               │                    │ { repos: [...] }       │                           │
 │               │                    │<─────────────────────────────────────────────────────│
```

Notice: the user's password is **only typed into GitHub's domain**. The
client application never sees it. The access token travels only between
the client's server and GitHub's servers — the browser never sees it.

---

## PKCE — Proof Key for Code Exchange

### The Problem PKCE Solves

The authorization code travels through the browser (in the redirect URL).
On mobile apps, the redirect uses a custom URL scheme (e.g.,
`myapp://callback?code=...`). On many platforms, **any app can register
the same URL scheme** and intercept the redirect.

The attacker:
1. Registers `myapp://` scheme on the victim's device
2. Victim starts OAuth flow in the legitimate app
3. Authorization code redirects to `myapp://callback?code=XYZ`
4. The OS delivers this to the malicious app instead of (or in addition to)
   the legitimate app
5. Malicious app exchanges the code for tokens

For confidential clients, the code exchange requires the `client_secret`,
which the attacker doesn't have. But **public clients don't have a client
secret** — that's why they're public. So the attacker can exchange the
intercepted code.

PKCE prevents this by binding the code to the specific client instance that
started the flow.

### How PKCE Works

**Step 1: Client generates a random secret (code verifier)**

```
code_verifier = base64url(crypto.randomBytes(32))
// e.g., "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
// Must be 43-128 characters, URL-safe
```

**Step 2: Client computes a challenge from the verifier**

```
code_challenge = base64url(SHA256(code_verifier))
// e.g., "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"
```

**Step 3: Authorization request includes the challenge**

```http
GET /authorize?
    response_type=code&
    client_id=abc123&
    redirect_uri=https://myapp.com/callback&
    code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
    code_challenge_method=S256&
    ...
```

The authorization server stores the challenge alongside the authorization
code.

**Step 4: Code exchange includes the original verifier**

```http
POST /token
grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk&
...
```

**Step 5: Authorization server verifies**

```
received_challenge = base64url(SHA256(code_verifier))
if received_challenge != stored_challenge:
    reject
```

### Why the Attacker Can't Win

The attacker intercepts the **authorization code** but not the **code
verifier** (which stays in the legitimate app's memory, never in a URL
or redirect). They can't compute the verifier from the challenge (SHA-256
is a one-way function — topic 01). They can't exchange the code without
the correct verifier.

### PKCE Is Now Required for All Clients

Originally designed for public clients (mobile/SPA), PKCE is now
recommended (and increasingly required) for **all** clients including
confidential ones. OAuth 2.1 (draft) makes PKCE mandatory. It's defense
in depth — even for confidential clients, it prevents authorization code
injection attacks.

---

## Scopes — Granular Permission Control

Scopes define **what** the client can do, not **who** the user is.

### How Scopes Work

The client requests specific scopes:
```
scope=read:repos read:user user:email
```

The authorization server shows these to the user on the consent screen:
"This application wants to: Read your repositories, Read your profile,
Access your email address."

The user can (in some implementations) approve only a subset. The granted
scopes are returned in the token response:
```json
{ "scope": "read:repos read:user" }
```

The resource server enforces scopes:
```
GET /user/repos  → requires "read:repos"  → allowed
POST /user/repos → requires "write:repos" → denied (not granted)
DELETE /user      → requires "admin:user"  → denied
```

### Scope Design Patterns

**GitHub style** — resource:action:
```
read:repos, write:repos, delete:repos
read:user, write:user
admin:org
```

**Google style** — URL-based:
```
https://www.googleapis.com/auth/gmail.readonly
https://www.googleapis.com/auth/gmail.send
https://www.googleapis.com/auth/calendar.events
```

**AWS Cognito style** — custom:
```
custom:read_data
custom:write_data
openid profile email
```

### The Principle of Least Privilege

Request the minimum scopes needed. A CI tool that only reads repos should
not request `write:repos`. Users are more likely to approve narrow
permissions, and if the token is compromised, the blast radius is smaller.

Some authorization servers let users **downscope** — approve only some of
the requested scopes. The client must handle this gracefully (check the
returned `scope` parameter, not assume it got everything it asked for).

---

## Other Grant Types

### Client Credentials

```http
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=client_credentials&
scope=read:analytics
```

**No user involved.** The client authenticates as itself (not on behalf of
a user) and gets a token representing its own identity. Used for
service-to-service communication, backend jobs, automated processes.

The "resource owner" and "client" are the same entity. There's no
authorization code, no redirect, no consent screen.

### Device Authorization (Device Code Flow)

For devices with no browser or limited input capability (smart TVs, CLI
tools, IoT devices).

```
1. Device requests a device code:
   POST /device/code
   → { "device_code": "GmRh...", "user_code": "WDJB-MJHT",
       "verification_uri": "https://github.com/login/device" }

2. Device displays: "Go to github.com/login/device and enter code WDJB-MJHT"

3. User opens browser on phone/laptop, navigates to the URL, enters code,
   authenticates, and consents.

4. Meanwhile, device polls the token endpoint:
   POST /token
   grant_type=urn:ietf:params:oauth:grant-type:device_code&
   device_code=GmRh...&
   client_id=abc123

   Response while waiting:
   { "error": "authorization_pending" }

   Response after user consents:
   { "access_token": "...", "refresh_token": "..." }
```

The device never handles user credentials. The user authenticates on a
trusted device (their phone) with a full browser. The device code binds
the two interactions.

**Polling interval**: The response includes an `interval` parameter (e.g.,
5 seconds). The device must not poll faster than this. The authorization
server may respond with `"error": "slow_down"` if the client polls too
aggressively.

### Resource Owner Password Credentials (ROPC)

```http
POST /token
grant_type=password&
username=user@example.com&
password=secretpassword&
client_id=abc123
```

The user's password is sent directly to the client, which sends it to the
authorization server. **This defeats the entire purpose of OAuth.** It
exists only for migration from legacy systems that already have the user's
password and need to transition to token-based auth.

**OAuth 2.1 removes this grant type entirely.** Do not use it in new
applications.

### Implicit Flow (Deprecated)

```http
GET /authorize?response_type=token&client_id=abc123&redirect_uri=...
→ 302 Location: https://myapp.com/callback#access_token=...
```

The authorization server returns the access token directly in the redirect
URL fragment (`#access_token=...`). No authorization code, no code exchange.
Designed for browser-only SPAs that couldn't make server-to-server calls.

**Why it's deprecated:**
1. The token is in the URL fragment — visible in browser history, Referer
   headers, and potentially to JavaScript on the redirect page
2. No mechanism to verify the client (no client secret, no PKCE in the
   original spec)
3. Tokens can be injected — an attacker substitutes their own token in the
   redirect
4. No refresh tokens — the spec forbids them in implicit flow, so the user
   must re-authorize when the token expires

**Replaced by**: Authorization code flow with PKCE. Even for SPAs, the
authorization code exchange provides a layer of protection (the code is
useless without the PKCE verifier).

---

## Token Endpoint Authentication

How the client proves its identity when exchanging the authorization code.

### client_secret_post

```http
POST /token
grant_type=authorization_code&
code=...&
client_id=abc123&
client_secret=s3cret
```

Secret in the POST body. Simple but the secret travels in the request body,
which may be logged by proxies or application logging.

### client_secret_basic

```http
POST /token HTTP/1.1
Authorization: Basic YWJjMTIzOnMzY3JldA==

grant_type=authorization_code&
code=...
```

`client_id:client_secret` Base64-encoded in the Authorization header.
Marginally better — less likely to be logged (many systems redact auth
headers but log POST bodies).

### private_key_jwt

```http
POST /token
grant_type=authorization_code&
code=...&
client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer&
client_assertion=eyJ...
```

The client creates a JWT signed with its private key. The authorization
server verifies with the client's registered public key. **The secret never
leaves the client.** Most secure option. Required by some financial-grade
APIs (FAPI).

### None (Public Clients)

```http
POST /token
grant_type=authorization_code&
code=...&
client_id=abc123&
code_verifier=dBjftJeZ4CVP...
```

No client secret. The `code_verifier` (PKCE) is the only proof that this
is the same client that started the flow.

---

## Redirect URI Security

The redirect URI is one of the most critical security points in OAuth.

### Open Redirect Attack

If the authorization server doesn't strictly validate the redirect URI,
an attacker can:

```
https://auth.example.com/authorize?
    client_id=legitimate_app&
    redirect_uri=https://evil.com/steal&
    response_type=code
```

The user sees the legitimate app's consent screen, approves, and gets
redirected to `evil.com` with the authorization code.

### Validation Rules

**Exact match** (recommended): The redirect URI in the authorization
request must exactly match one of the pre-registered URIs. No wildcards,
no partial matching.

```
Registered:  https://myapp.com/callback
Request:     https://myapp.com/callback          ✓ match
Request:     https://myapp.com/callback?extra=1  ✗ reject
Request:     https://myapp.com/callback/..        ✗ reject
Request:     https://evil.com/callback            ✗ reject
```

**Why not allow subdirectories or query parameters?** Path traversal and
open redirects within the legitimate domain could be chained. If the
legitimate app has an open redirect at `myapp.com/callback?next=`, the
authorization code could be forwarded to an attacker-controlled URL.

**Localhost exception**: For native/desktop apps, redirect URIs like
`http://localhost:8080/callback` or `http://127.0.0.1:PORT/callback` are
allowed over HTTP (the traffic never leaves the machine). The port may
vary, so some authorization servers allow port wildcarding for localhost
only.

---

## OpenID Connect (OIDC) — Identity Layer on Top of OAuth

OAuth 2.0 is an **authorization** framework — it answers "what can this
app do?" It doesn't answer "who is the user?" The access token grants
access to resources but doesn't inherently carry user identity.

OpenID Connect adds **authentication** on top of OAuth 2.0.

### What OIDC Adds

1. **ID Token**: A JWT containing identity claims about the user
2. **UserInfo endpoint**: A standard API to fetch user profile data
3. **Standard scopes**: `openid`, `profile`, `email`, `address`, `phone`
4. **Discovery**: A well-known endpoint for server configuration
5. **Dynamic registration**: Clients can register programmatically

### The ID Token

```json
{
    "iss": "https://accounts.google.com",
    "sub": "110169484474386276334",
    "aud": "abc123.apps.googleusercontent.com",
    "exp": 1708099200,
    "iat": 1708012800,
    "nonce": "n-0S6_WzA2Mj",
    "email": "user@gmail.com",
    "email_verified": true,
    "name": "Jane Doe",
    "picture": "https://lh3.googleusercontent.com/a/..."
}
```

The ID token is **always a JWT**, signed by the authorization server. The
client verifies the signature and reads the claims to identify the user.

**Critical difference**: The access token is opaque to the client (it
doesn't need to understand it — just pass it to the resource server). The
ID token is **meant to be read by the client** — that's its entire purpose.

### ID Token vs Access Token

| | ID Token | Access Token |
|---|----------|-------------|
| Purpose | Identify the user to the **client** | Access resources at the **resource server** |
| Format | Always JWT | JWT or opaque string |
| Audience | The client application (`aud` = client_id) | The resource server |
| Read by | The client | The resource server |
| Sent to | Decoded locally by the client | Sent in Authorization header to APIs |

A common mistake: using the access token to identify the user, or sending
the ID token to an API. Each has a specific purpose and audience.

### The `nonce` Parameter

Like `state` prevents CSRF on the authorization callback, `nonce` prevents
replay attacks on the ID token.

```
1. Client generates random nonce, stores it in session
2. Authorization request includes nonce parameter
3. Authorization server includes nonce in the ID token
4. Client verifies ID token's nonce matches the stored value
```

Without `nonce`, an attacker could capture an ID token from one flow and
inject it into a different flow (replay attack).

### Discovery Document

```
GET /.well-known/openid-configuration HTTP/1.1
Host: accounts.google.com
```

Returns a JSON document with all the server's endpoints and capabilities:

```json
{
    "issuer": "https://accounts.google.com",
    "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
    "token_endpoint": "https://oauth2.googleapis.com/token",
    "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
    "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
    "scopes_supported": ["openid", "email", "profile"],
    "response_types_supported": ["code", "token", "id_token"],
    "subject_types_supported": ["public"],
    "id_token_signing_alg_values_supported": ["RS256"],
    "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic", "private_key_jwt"]
}
```

The `jwks_uri` points to the **JSON Web Key Set** — the public keys used
to sign ID tokens. The client fetches these to verify token signatures
without any pre-shared secret.

This is how "Login with Google" works without any bilateral key exchange:
your app discovers Google's JWKS URI from the discovery document, fetches
the public keys, and verifies ID token signatures locally.

---

## Token Introspection and Revocation

### Token Introspection (RFC 7662)

When access tokens are opaque strings (not JWTs), the resource server needs
to ask the authorization server if a token is valid:

```http
POST /introspect HTTP/1.1
Host: auth.example.com
Authorization: Basic base64(resource_server_credentials)
Content-Type: application/x-www-form-urlencoded

token=gho_16C7e42F292c6912E7710c838347Ae178B4a
```

```json
{
    "active": true,
    "sub": "42",
    "client_id": "abc123",
    "scope": "read:repos read:user",
    "exp": 1708099200
}
```

This is the session-store lookup equivalent for OAuth. Every API request
triggers an introspection call. The authorization server is now a bottleneck
— same scalability concern as sessions.

**JWT access tokens eliminate this** — the resource server verifies the
signature locally. Tradeoff: no instant revocation (same as topic 03).

### Token Revocation (RFC 7009)

```http
POST /revoke HTTP/1.1
Host: auth.example.com
Authorization: Basic base64(client_credentials)
Content-Type: application/x-www-form-urlencoded

token=ghr_1B4a2e77170c838347Ae178B4a...&
token_type_hint=refresh_token
```

The authorization server invalidates the token. For opaque tokens, this
removes them from the store. For JWTs, this adds them to a deny list
(with the same problems discussed in topic 03).

`token_type_hint` helps the server look in the right store first
(`access_token` or `refresh_token`) but isn't binding — the server should
check both.

---

## Common Vulnerabilities

### Authorization Code Injection

An attacker obtains a valid authorization code (through interception,
logging, or Referer leakage) and injects it into their own session:

```
1. Attacker intercepts code meant for victim
2. Attacker navigates to: myapp.com/callback?code=STOLEN_CODE&state=ATTACKERS_STATE
3. If the app doesn't bind the code to the specific flow, the attacker
   gets tokens for the victim's account
```

**Defense**: PKCE. The code is bound to the `code_verifier` that only the
legitimate client instance has. Also, `state` prevents injection into a
different session.

### Scope Escalation

Some authorization servers have had bugs where:
- Requesting `scope=admin` when only `scope=read` was approved
- Modifying the scope during the token refresh
- The resource server not actually checking scopes (just checking if the
  token is valid)

**Defense**: Authorization servers must validate scopes at issuance. Resource
servers must validate scopes on every request. Refresh tokens must not grant
broader scopes than the original authorization.

### Covert Redirect

A variant of the open redirect attack where the attacker uses a legitimate
redirect URI that has its own open redirect:

```
redirect_uri=https://legitimate-app.com/page?next=https://evil.com
```

If `legitimate-app.com/page` redirects to whatever `next` specifies, the
authorization code is forwarded to `evil.com`.

**Defense**: Strict redirect URI matching (no query parameters allowed
beyond what's registered). The application at the redirect URI must not
have open redirects.

### Token Leakage via Referer

After the redirect to `myapp.com/callback?code=XYZ`, if that page loads
any external resources (images, scripts, analytics), the full URL including
the code appears in the `Referer` header to those external servers.

**Defense**: The callback endpoint should immediately exchange the code and
redirect to a clean URL (without query parameters). Use
`Referrer-Policy: no-referrer` on the callback page. The code's short
lifetime (30-60 seconds) limits the window.

---

## How "Login with Google/GitHub" Actually Works

When you implement "Login with GitHub" on your site, here's exactly what
happens, tying all the concepts together:

```
1. You register an OAuth App on GitHub
   → client_id: "Iv1.abc123"
   → client_secret: "secret_xyz"
   → redirect_uri: "https://yourapp.com/auth/github/callback"

2. User clicks "Login with GitHub"
   → Browser navigates to:
     https://github.com/login/oauth/authorize?
       client_id=Iv1.abc123&
       redirect_uri=https://yourapp.com/auth/github/callback&
       scope=read:user user:email&
       state=RANDOM_STATE

3. User logs into GitHub (if not already)
   → GitHub shows: "YourApp wants to access your profile and email"
   → User clicks Authorize

4. GitHub redirects:
   → https://yourapp.com/auth/github/callback?
       code=TEMP_CODE&
       state=RANDOM_STATE

5. Your server:
   → Verifies state matches
   → POST https://github.com/login/oauth/access_token
     { client_id, client_secret, code }
   → Gets: { access_token: "gho_..." }

6. Your server:
   → GET https://api.github.com/user
     Authorization: Bearer gho_...
   → Gets: { id: 12345, login: "user", email: "user@example.com" }

7. Your server:
   → Finds or creates a user in YOUR database with github_id: 12345
   → Creates a session (topic 02) for that user
   → Sets session cookie
   → Redirects to dashboard

The user is now logged into YOUR app. GitHub's access token is stored
in your database in case you need to make more GitHub API calls. The
user's session with YOUR app is a standard session cookie — not the
GitHub token.
```

The user never typed their password into your site. Your site never saw
their GitHub password. You can revoke the OAuth grant at any time from
GitHub's settings without affecting the user's GitHub password. GitHub
can revoke your app's access without affecting other apps. This is the
separation that OAuth provides.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| OAuth is authorization, not authentication | It answers "what can this app do" not "who is the user" |
| Authorization code flow is the primary flow | Two-step: browser gets code, server exchanges code for tokens |
| The code exchange is server-to-server | The browser never sees the access token |
| `state` prevents CSRF on the callback | Random, per-request, verified before code exchange |
| PKCE binds the code to the client instance | SHA-256 of a random verifier — prevents code interception |
| Redirect URI must be exactly validated | Open redirects on the callback path chain into code theft |
| Scopes are not all-or-nothing | Request minimum, handle partial grants, enforce on every API call |
| OIDC adds identity on top of OAuth | ID token = who the user is (for the client). Access token = what the app can do (for the API) |
| Implicit flow is dead | Authorization code + PKCE replaces it for all client types |
| OAuth 2.1 makes PKCE mandatory | Even for confidential clients — defense in depth |
