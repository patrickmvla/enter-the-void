# HTTP — The Protocol of the Web

## What HTTP Is

HTTP (Hypertext Transfer Protocol) is a request-response protocol for
transferring data between clients and servers. It defines the syntax
and semantics of messages — how a client asks for a resource and how
a server responds.

HTTP sits at the top of the stack:

```
HTTP      → structures the data ("GET /index.html, here are my headers")
TLS       → encrypts the channel
TCP/QUIC  → delivers bytes reliably
IP        → routes packets across networks
```

Every web page, API call, image load, font download, and webhook
delivery uses HTTP. Understanding HTTP at the protocol level — not
just the API abstraction — is understanding how the web actually works.

---

## HTTP/1.1 — The Foundation

HTTP/1.1 (RFC 9110, 9112) is a text-based protocol. Messages are
human-readable ASCII. You can literally type an HTTP request into a
TCP connection and get a response.

### Request Format

```
GET /api/users?page=2 HTTP/1.1\r\n
Host: example.com\r\n
Accept: application/json\r\n
Authorization: Bearer eyJhbGc...\r\n
User-Agent: MyApp/1.0\r\n
Accept-Encoding: gzip, deflate, br\r\n
Connection: keep-alive\r\n
\r\n
```

Structure:
```
[Method] [Request-Target] [HTTP-Version]\r\n
[Header-Name]: [Header-Value]\r\n
[Header-Name]: [Header-Value]\r\n
...
\r\n
[Optional Body]
```

**`\r\n` (CRLF)** terminates every line. The empty line (`\r\n\r\n`)
separates headers from body. This is the parser's signal that headers
are done.

### Response Format

```
HTTP/1.1 200 OK\r\n
Content-Type: application/json\r\n
Content-Length: 47\r\n
Cache-Control: max-age=3600\r\n
ETag: "abc123"\r\n
\r\n
{"users": [{"id": 1, "name": "alice"}]}
```

Structure:
```
[HTTP-Version] [Status-Code] [Reason-Phrase]\r\n
[Header-Name]: [Header-Value]\r\n
...
\r\n
[Body]
```

### Methods — Semantics and Safety

| Method | Safe | Idempotent | Body | Purpose |
|--------|------|-----------|------|---------|
| GET | Yes | Yes | No | Retrieve a resource |
| HEAD | Yes | Yes | No | GET without the body (check existence, headers) |
| POST | No | No | Yes | Create a resource, submit data, trigger an action |
| PUT | No | Yes | Yes | Replace a resource entirely |
| PATCH | No | No | Yes | Partial modification |
| DELETE | No | Yes | No* | Remove a resource |
| OPTIONS | Yes | Yes | No | Discover allowed methods (CORS preflight) |

**Safe**: The request doesn't modify server state. Caches and prefetchers
can freely make safe requests. If your GET endpoint modifies data, you've
violated the contract and will have bugs (search engine crawlers triggering
deletions, browser prefetching creating side effects).

**Idempotent**: Making the same request N times has the same effect as
making it once. PUT and DELETE are idempotent — putting the same resource
twice results in the same state; deleting the same resource twice results
in the same state (already deleted). POST is not idempotent — submitting
a payment twice creates two payments.

**Why idempotency matters for infrastructure**: Load balancers, CDNs, and
retry logic depend on these semantics. If a request times out, is it safe
to retry? For GET/PUT/DELETE: yes (idempotent). For POST: dangerous
(might duplicate the action). This is why payment APIs use idempotency
keys:

```http
POST /charges HTTP/1.1
Idempotency-Key: unique-request-id-abc123
Content-Type: application/json

{"amount": 5000, "currency": "usd"}
```

The server stores the result keyed by the idempotency key. Replaying the
same request returns the stored result instead of creating a new charge.

### Status Codes — What the Numbers Mean

```
1xx: Informational (request received, continuing)
  100 Continue        — "Send the body, I accept your headers"
  101 Switching Protocols — "Upgrading to WebSocket" (or HTTP/2 over cleartext)
  103 Early Hints     — "Start preloading these resources while I prepare the response"

2xx: Success
  200 OK              — Standard success
  201 Created         — Resource created (POST). Location header has the URL
  204 No Content      — Success, but no body (common for DELETE)
  206 Partial Content — Range request fulfilled (resumable downloads, video seeking)

3xx: Redirection
  301 Moved Permanently — Permanent redirect. Browsers/caches remember it
  302 Found            — Temporary redirect. (Historically misused — clients
                          would change POST to GET. 303 and 307 fix this)
  303 See Other        — "Go GET this other URL" (POST→redirect→GET pattern)
  304 Not Modified     — Conditional request: resource hasn't changed
  307 Temporary Redirect — Like 302, but preserves the method (POST stays POST)
  308 Permanent Redirect — Like 301, but preserves the method

4xx: Client Error
  400 Bad Request      — Malformed request syntax
  401 Unauthorized     — Authentication required (misleading name — means unauthenticated)
  403 Forbidden        — Authenticated but not authorized
  404 Not Found        — Resource doesn't exist
  405 Method Not Allowed — e.g., POST on a read-only endpoint
  409 Conflict         — State conflict (optimistic concurrency, duplicate resource)
  413 Content Too Large — Request body exceeds server's limit
  415 Unsupported Media Type — Server doesn't accept this Content-Type
  422 Unprocessable Content — Syntactically valid but semantically wrong
  429 Too Many Requests — Rate limited. Retry-After header says when

5xx: Server Error
  500 Internal Server Error — Unhandled exception
  502 Bad Gateway      — Proxy/LB got invalid response from upstream
  503 Service Unavailable — Server overloaded or in maintenance
  504 Gateway Timeout  — Proxy/LB: upstream didn't respond in time
```

**301 vs 308**: 301 allows the client to change the method (POST → GET).
308 preserves it. If your API endpoint moves, use 308 to avoid breaking
POST requests. Browsers treat 301 as "change to GET" in practice.

**401 vs 403**: 401 means "I don't know who you are — authenticate."
403 means "I know who you are, and you can't do this." A 401 should
include a `WWW-Authenticate` header specifying the auth scheme.

### Content Negotiation

The client declares what it can handle; the server picks the best match:

```http
Accept: application/json, text/html;q=0.9, */*;q=0.1
Accept-Language: en-US, en;q=0.9, fr;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
```

**Quality values (q)**: 0.0 to 1.0, default 1.0. The server uses these
to choose the best representation. `application/json` at q=1.0 is
preferred over `text/html` at q=0.9.

The server responds with what it chose:

```http
Content-Type: application/json; charset=utf-8
Content-Language: en-US
Content-Encoding: gzip
Vary: Accept, Accept-Encoding, Accept-Language
```

**Vary header**: Tells caches that the response varies based on these
request headers. A cache must store separate entries for different
Accept-Encoding values (a gzipped response for `Accept-Encoding: gzip`
and an uncompressed one for no encoding). Without Vary, a cache might
serve a gzipped response to a client that can't decompress it.

### Chunked Transfer Encoding

When the server doesn't know the response size upfront (streaming data,
server-side rendering), it sends chunks:

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1a\r\n
This is the first chunk.\r\n
1c\r\n
And this is the second one.\r\n
0\r\n
\r\n
```

Each chunk: hex size + CRLF + data + CRLF. The final chunk has size 0.
Trailers (headers sent after the body) can follow the final chunk —
useful for checksums or metadata computed during streaming.

**Content-Length vs Transfer-Encoding**: Mutually exclusive. If the server
knows the size, use Content-Length (allows progress bars, download
estimation). If not, use chunked. A message with both is malformed —
historically used for request smuggling attacks.

### Connection Management

**Keep-Alive** (default in HTTP/1.1): The TCP connection persists after
the response, allowing multiple request-response pairs without new
handshakes. Before HTTP/1.1, every request opened a new TCP connection
(new 3-way handshake, new slow start).

```http
Connection: keep-alive       ← default in HTTP/1.1 (explicit in 1.0)
Connection: close            ← "close the connection after this response"
Keep-Alive: timeout=5, max=100  ← keep open for 5s idle, max 100 requests
```

**The Head-of-Line Blocking Problem in HTTP/1.1**:

A keep-alive connection processes requests **sequentially**. The client
sends request 1, waits for response 1, then sends request 2. If
response 1 takes 2 seconds, request 2 waits 2 seconds even if the
server could answer it instantly.

```
Connection 1: [──Req 1──][──wait──][──Req 2──][──wait──]
               ↑ blocked until response 1 completes

Time: =========================================>
```

**HTTP/1.1 pipelining** attempted to fix this: send multiple requests
without waiting for responses. But responses still had to arrive **in
order** (the server must respond to request 1 before request 2, even
if request 2 is ready first). A slow response 1 blocks all subsequent
responses. Pipelining was widely unsupported and effectively dead.

**The workaround**: Browsers open **6 parallel TCP connections** per
domain (a limit set by convention). Each connection handles one
request-response at a time, but 6 connections = 6 concurrent requests.
This is why HTTP/1.1 performance tuning involved domain sharding
(spreading resources across cdn1.example.com, cdn2.example.com, etc. to
get more connections) and concatenation (bundling CSS/JS into single
files to reduce the number of requests).

---

## HTTP/2 — Binary Framing and Multiplexing

HTTP/2 (RFC 9113, derived from Google's SPDY) solves HTTP/1.1's
head-of-line blocking at the application layer by multiplexing requests
over a single TCP connection.

### Binary Framing Layer

HTTP/2 replaces HTTP/1.1's text-based format with a binary framing
layer. Every message is split into **frames** — small binary units
that can be interleaved on a single connection.

```
HTTP/1.1 (text):
  GET /style.css HTTP/1.1\r\n
  Host: example.com\r\n\r\n

HTTP/2 (binary frames):
  [HEADERS frame, stream 1] [DATA frame, stream 1]
  [HEADERS frame, stream 3] [DATA frame, stream 3]
  [DATA frame, stream 1]    [DATA frame, stream 3]
```

### Frame Format (9-byte header)

```
+-----------------------------------------------+
|                 Length (24)                     |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+------+--------+
|R|                 Stream ID (31)               |
+-+----------------------------------------------+
|                Frame Payload (0...)             |
+------------------------------------------------+
```

- **Length**: Payload size (not including the 9-byte header). Max 16384
  by default, configurable up to 2^24 - 1 (~16 MB) via SETTINGS.
- **Type**: DATA (0), HEADERS (1), PRIORITY (2), RST_STREAM (3),
  SETTINGS (4), PUSH_PROMISE (5), PING (6), GOAWAY (7), WINDOW_UPDATE (8),
  CONTINUATION (9).
- **Stream ID**: Which stream this frame belongs to. Stream 0 is the
  connection itself (SETTINGS, PING, GOAWAY). Odd IDs are client-initiated,
  even IDs are server-initiated (push).

### Streams and Multiplexing

A **stream** is a bidirectional flow of frames within a connection.
Each HTTP request-response pair is a stream. Streams are independent —
frames from different streams can be interleaved:

```
Single TCP Connection:
  ┌──────────────────────────────────────────────────┐
  │ [H:1] [H:3] [D:1] [D:3] [D:1] [H:5] [D:3] [D:5]│
  └──────────────────────────────────────────────────┘
  H = HEADERS frame, D = DATA frame, number = stream ID

Stream 1: GET /index.html  → [H:1]......[D:1]...[D:1]
Stream 3: GET /style.css   → ...[H:3]......[D:3]........[D:3]
Stream 5: GET /app.js      → ..................[H:5]..........[D:5]
```

**No application-layer head-of-line blocking**: If stream 1's response
is slow, streams 3 and 5 can still send data. The server interleaves
frames as data becomes available.

**But TCP head-of-line blocking remains**: All streams share one TCP
connection. A lost TCP packet blocks ALL streams until retransmission
completes (as discussed in TCP/01-tcp-ip.md). This is why HTTP/3 moves
to QUIC — per-stream reliability eliminates even this blocking.

### HPACK Header Compression

HTTP headers are repetitive. Every request sends `Host`, `User-Agent`,
`Accept`, `Cookie` (often kilobytes). HTTP/1.1 sends these as
uncompressed text on every request.

HTTP/2 uses **HPACK** (RFC 7541), a header compression scheme designed
to be safe from compression-based attacks (CRIME).

**How HPACK works:**

1. **Static Table**: 61 pre-defined header name-value pairs for common
   headers (`":method": "GET"`, `":status": "200"`, etc.). Referenced by
   index instead of sending the full string.

2. **Dynamic Table**: Headers seen in previous requests are added to a
   connection-specific table. Subsequent requests reference them by index.
   The table has a size limit (configurable via SETTINGS).

3. **Huffman Encoding**: Header values can be Huffman-coded using a
   fixed code table optimized for HTTP header content. Typically saves
   ~30% on header size.

```
First request:
  :method: GET              → index 2 (static table)
  :path: /api/users         → literal, added to dynamic table as index 62
  authorization: Bearer xyz → literal, added to dynamic table as index 63
  cookie: session=abc123... → literal (1200 bytes → ~800 bytes Huffman)

Second request (same connection):
  :method: GET              → index 2
  :path: /api/users         → index 62 (dynamic table — 1 byte!)
  authorization: Bearer xyz → index 63 (1 byte!)
  cookie: session=abc123... → same cookie → Huffman only
```

**Why not gzip?** The CRIME attack (2012) showed that TLS-level
compression of headers leaks information through response size.
HPACK avoids this by never compressing across requests — each header
is either a table reference (fixed size) or a literal (independent
of other headers). There's no shared compression context that an
attacker can exploit.

### Server Push

HTTP/2 allows the server to push resources before the client requests
them:

```
Client: GET /index.html
Server: PUSH_PROMISE (stream 2: /style.css)
        PUSH_PROMISE (stream 4: /app.js)
        HEADERS + DATA (stream 1: /index.html)
        HEADERS + DATA (stream 2: /style.css)
        HEADERS + DATA (stream 4: /app.js)
```

The server knows that `/index.html` references `style.css` and `app.js`,
so it pushes them preemptively. The client receives them before it even
parses the HTML.

**In practice, server push failed.** Problems:
- The server guesses what the client needs and is often wrong (client
  already has the resource cached)
- Pushed resources compete with the response the client actually asked for
- No good mechanism for the client to cancel unwanted pushes efficiently
- Hard to implement correctly in application code
- CDNs and browsers had inconsistent support

Chrome removed server push support in 2022. The 103 Early Hints
response code (`Link: </style.css>; rel=preload`) is the replacement —
it tells the client to start fetching resources early without the
server actually pushing them.

### Flow Control

HTTP/2 has its own flow control layered on top of TCP's. Each stream
has a **flow control window** (default 65535 bytes), and there's also
a connection-level window. The receiver sends WINDOW_UPDATE frames to
grant more capacity.

**Why separate from TCP?** TCP flow control operates per-connection.
HTTP/2 needs per-stream control — a slow-to-consume stream shouldn't
block faster streams from receiving data. HTTP/2 flow control allows
the receiver to prioritize which streams get buffer space.

---

## HTTP/3 — QUIC and the End of TCP for HTTP

HTTP/3 (RFC 9114) replaces TCP with QUIC (RFC 9000) as the transport
layer. The HTTP semantics (methods, headers, status codes) are identical
to HTTP/2 — the difference is how they're carried on the wire.

### Why Replace TCP?

Two fundamental problems with TCP under HTTP/2:

1. **Head-of-line blocking** (covered in TCP/01-tcp-ip.md): A single
   lost packet blocks all streams. HTTP/2's multiplexing is undermined
   by TCP's in-order delivery guarantee.

2. **Handshake latency**: TCP handshake (1 RTT) + TLS handshake (1 RTT)
   = 2 RTT before data flows. On a 200ms RTT connection, that's 400ms
   of mandatory waiting.

QUIC solves both.

### QUIC Architecture

```
HTTP/2 stack:           HTTP/3 stack:
  HTTP/2                  HTTP/3
  TLS 1.3                 QUIC (includes TLS 1.3)
  TCP                     UDP
  IP                      IP
```

QUIC runs over UDP and implements:
- Reliable delivery (per-stream, not per-connection)
- Congestion control (similar to TCP's)
- TLS 1.3 (integrated into the handshake — not layered on top)
- Stream multiplexing (independent streams with independent ordering)
- Connection migration (connections survive IP changes)

### QUIC Handshake: 1-RTT (or 0-RTT)

QUIC integrates the cryptographic handshake into the transport handshake:

```
TCP + TLS 1.3:                    QUIC:
  SYN        ──>                   Initial (ClientHello) ──>
  SYN-ACK    <──                   Initial (ServerHello)  <──
  ACK + ClientHello ──>            Handshake + 1-RTT data  ──>
  ServerHello + ... <──
  Finished     ──>                 1 RTT total (vs 2 RTT)
  Finished     <──
  Application data ──>

  2 RTT total
```

For 0-RTT (returning clients with a PSK from a previous session):

```
QUIC 0-RTT:
  Initial (ClientHello + 0-RTT data) ──>    0 RTT!
  Server processes request immediately
```

Same replay caveats as TLS 1.3 0-RTT — only safe for idempotent requests.

### Per-Stream Reliability

The key innovation. Each QUIC stream has independent sequence numbers
and retransmission:

```
QUIC Connection:
  Stream 1: [Frame A1][Frame A2][Frame A3]     ← in order within stream 1
  Stream 3: [Frame B1][  LOST  ][Frame B3]     ← B2 lost, only stream 3 waits
  Stream 5: [Frame C1][Frame C2]               ← unaffected by stream 3's loss
```

Lost data on stream 3 is retransmitted without blocking streams 1 and 5.
This eliminates head-of-line blocking entirely. On lossy networks (mobile,
WiFi), HTTP/3 significantly outperforms HTTP/2.

### Connection Migration

TCP connections are identified by (src_ip, src_port, dst_ip, dst_port).
If your phone switches from WiFi to cellular, the IP changes, and all
TCP connections die. New handshakes, new slow start.

QUIC connections are identified by a **Connection ID** — a random token
in the QUIC header. When the IP changes, the connection continues with
the same Connection ID. The server recognizes the client by the ID, not
the IP address.

```
Phone on WiFi: 192.168.1.42 ──QUIC (CID: abc123)──> Server
Phone switches to LTE: 10.0.0.1 ──QUIC (CID: abc123)──> Server
→ Same connection. No handshake. No interruption.
```

This is critical for mobile — users constantly switch between networks.

### QPACK Header Compression

HTTP/3 can't use HPACK because HPACK's dynamic table assumes in-order
delivery (it references entries by their insertion order). QUIC streams
are independent and may arrive out of order.

**QPACK** (RFC 9204) solves this with a two-stream approach:
- A dedicated **encoder stream** carries dynamic table updates
- A dedicated **decoder stream** carries acknowledgments
- Request/response headers reference the static table (same as HPACK)
  or the dynamic table (with acknowledgment tracking to avoid
  referencing entries that haven't arrived yet)

### HTTP/3 Adoption

As of 2025, ~30% of web traffic uses HTTP/3. Supported by all major
browsers, Cloudflare, Google, Facebook, Akamai. Discovery works via:

```http
HTTP/2 Response:
  Alt-Svc: h3=":443"; ma=86400

→ "I support HTTP/3 on port 443. Remember this for 24 hours."
→ Browser tries QUIC on next request. Falls back to HTTP/2 if it fails.
```

The `Alt-Svc` header tells the client that HTTP/3 is available. The first
request always uses HTTP/2 (or 1.1), and subsequent requests upgrade to
HTTP/3. This means the very first visit to a site is always HTTP/2.

---

## HTTP Caching

Caching is HTTP's performance multiplier. A properly cached response
avoids the network entirely — zero latency, zero bandwidth.

### Cache-Control

The primary caching header. Directives for both requests and responses:

```http
# Response directives:
Cache-Control: public, max-age=3600
  → Any cache (browser, CDN, proxy) can store this for 1 hour

Cache-Control: private, max-age=600
  → Only the browser can cache (not CDN/proxies) — contains user-specific data

Cache-Control: no-cache
  → Cache can store it, but MUST revalidate with the server before using

Cache-Control: no-store
  → Don't cache at all. Not in memory, not on disk. For sensitive data.

Cache-Control: public, max-age=31536000, immutable
  → Cache forever (1 year). "immutable" means don't even revalidate.
    Used for fingerprinted assets: /app.a1b2c3d4.js

Cache-Control: s-maxage=3600
  → Like max-age but only for shared caches (CDNs). Overrides max-age for CDNs.

Cache-Control: stale-while-revalidate=60
  → Serve stale content for up to 60s while fetching a fresh version in the
    background. User gets instant response, cache updates asynchronously.

Cache-Control: stale-if-error=86400
  → If the origin is down, serve stale content for up to 24 hours.
    Better stale than broken.
```

### Conditional Requests (Revalidation)

When a cached response expires (max-age exceeded), the client doesn't
fetch the full response again. It asks the server: "Has this changed?"

**ETag-based:**
```http
# First response:
HTTP/1.1 200 OK
ETag: "abc123"
Cache-Control: max-age=60

# After 60 seconds, cache expired. Client revalidates:
GET /resource HTTP/1.1
If-None-Match: "abc123"

# Server checks: is the current ETag still "abc123"?
# If unchanged:
HTTP/1.1 304 Not Modified     ← no body, use cached version
# If changed:
HTTP/1.1 200 OK               ← full response with new body and ETag
```

**Last-Modified-based:**
```http
# First response:
HTTP/1.1 200 OK
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT

# Revalidation:
GET /resource HTTP/1.1
If-Modified-Since: Mon, 01 Jan 2024 00:00:00 GMT

# 304 if unchanged, 200 if changed
```

ETags are stronger (can detect changes that don't affect the modification
time, like a file being rewritten with identical content then changed
again). Both can be used together.

### Cache Invalidation

```
"There are only two hard things in Computer Science: cache
invalidation and naming things." — Phil Karlton
```

HTTP's cache invalidation is **time-based** (TTL) and **validation-based**
(conditional requests). There's no mechanism to proactively push
invalidation to clients or intermediate caches.

**The fingerprinting pattern**: For static assets, embed a content hash
in the URL:

```
/app.js              → /app.a1b2c3d4.js
/style.css           → /style.e5f6g7h8.css
```

Set `Cache-Control: public, max-age=31536000, immutable`. When the
content changes, the URL changes (new hash), so caches treat it as a
completely new resource. The old version can stay cached forever —
nothing will ever request it again.

The HTML page that references these assets is cached with a short TTL
(or `no-cache`) so it always fetches the latest asset URLs.

---

## HTTP Security Headers

Headers that harden the browser's security model:

### Strict-Transport-Security (HSTS)

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

"Only connect to this domain over HTTPS. If someone types http://,
the browser upgrades to https:// automatically without making the
insecure request." `includeSubDomains` applies to all subdomains.
`preload` requests inclusion in browser preload lists (hardcoded HSTS
— protects even the first visit).

**Without HSTS**: The first HTTP request to `http://example.com`
can be intercepted and the redirect to `https://` can be stripped
(SSL stripping attack). HSTS prevents this after the first visit
(or always, if preloaded).

### Content-Security-Policy (CSP)

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123';
  style-src 'self' 'unsafe-inline'; img-src 'self' data: https://cdn.example.com;
  connect-src 'self' https://api.example.com; frame-ancestors 'none';
  report-uri /csp-violations
```

Whitelist of allowed content sources. Blocks inline scripts (XSS defense),
restricts where resources can be loaded from, prevents clickjacking
(`frame-ancestors`). The `nonce` attribute allows specific inline scripts
while blocking injected ones.

### Other Security Headers

```http
X-Content-Type-Options: nosniff
  → Prevent MIME-type sniffing (don't execute a CSS file as JavaScript)

X-Frame-Options: DENY
  → Prevent the page from being loaded in an iframe (clickjacking defense)
  → Superseded by CSP frame-ancestors but still widely used

Referrer-Policy: strict-origin-when-cross-origin
  → Control how much referrer information is sent with requests

Permissions-Policy: camera=(), microphone=(), geolocation=(self)
  → Control which browser features the page can use
```

---

## HTTP/1.1 vs HTTP/2 vs HTTP/3 — Summary

```
Feature              HTTP/1.1          HTTP/2              HTTP/3
─────────────────────────────────────────────────────────────────
Format               Text              Binary frames       Binary frames
Multiplexing         No (6 connections) Yes (1 connection)  Yes (1 connection)
HOL blocking         Application-layer  TCP-layer           None (per-stream)
Header compression   None              HPACK               QPACK
Server push          No                Yes (deprecated)    Yes (deprecated)
Transport            TCP               TCP                 QUIC (over UDP)
Handshake RTT        TCP(1) + TLS(1)   TCP(1) + TLS(1)    1 RTT (combined)
0-RTT                No                No                  Yes (with PSK)
Connection migration No                No                  Yes (Connection ID)
Encryption           Optional          Effectively required Mandatory
```

**Performance by scenario:**

| Scenario | Best Protocol | Why |
|----------|--------------|-----|
| High-bandwidth, low-loss | HTTP/2 ≈ HTTP/3 | HOL blocking is rare, both multiplex well |
| High-latency (satellite, intercontinental) | HTTP/3 | 1-RTT handshake saves 100-300ms |
| Lossy network (mobile, WiFi) | HTTP/3 | Per-stream reliability avoids HOL blocking |
| First visit to a site | HTTP/2 | HTTP/3 requires Alt-Svc discovery |
| Returning visit | HTTP/3 | 0-RTT resumption eliminates handshake |
| Small, simple API | HTTP/1.1 | Simple, debuggable, less overhead per request |

---

## HTTP Request Smuggling

A critical attack that exploits discrepancies between how front-end
(proxy/load balancer) and back-end servers parse HTTP messages.

### The Mechanism

HTTP/1.1 has two ways to determine message length:
1. `Content-Length: 13` — "The body is 13 bytes"
2. `Transfer-Encoding: chunked` — "The body is chunked"

If both headers are present, RFC 9112 says Transfer-Encoding wins.
But not all implementations agree.

### CL.TE Attack (Front-end uses Content-Length, back-end uses Transfer-Encoding)

```http
POST / HTTP/1.1
Host: example.com
Content-Length: 13
Transfer-Encoding: chunked

0\r\n
\r\n
SMUGGLED
```

**Front-end** sees Content-Length: 13 → reads 13 bytes (`0\r\n\r\nSMUGGLED`)
as one request. Forwards everything to the back-end.

**Back-end** sees Transfer-Encoding: chunked → reads `0\r\n\r\n` as the
end of the chunked body (zero-length chunk). `SMUGGLED` is left in the
buffer as the **start of the next request**.

The next legitimate user's request gets prepended with `SMUGGLED`. The
attacker has injected data into another user's request.

### Impact

- **Steal other users' requests** (attach your attack to their request)
- **Bypass access controls** (smuggle a request to an internal endpoint)
- **Cache poisoning** (smuggle a request that poisons the CDN cache)
- **Request hijacking** (redirect another user's request to your server)

### Defense

- Use HTTP/2 end-to-end (binary framing eliminates ambiguity)
- Normalize ambiguous requests at the front-end (reject if both
  Content-Length and Transfer-Encoding are present)
- Ensure front-end and back-end agree on parsing behavior
- Use the same HTTP implementation at both layers

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| HTTP/1.1 is text-based, sequential | One request at a time per connection. 6 connections = workaround |
| Method semantics (safe, idempotent) matter for infrastructure | Retry logic, caching, and prefetching depend on these contracts |
| HTTP/2 multiplexes streams over one TCP connection | Binary frames, HPACK compression, stream IDs. Still TCP HOL blocked |
| HTTP/3 replaces TCP with QUIC (over UDP) | Per-stream reliability, 1-RTT handshake, connection migration |
| HPACK: static table + dynamic table + Huffman | Safe from CRIME. References replace repeated headers |
| Server push is effectively dead | 103 Early Hints is the replacement |
| Cache-Control is the primary caching mechanism | max-age, no-cache, no-store, stale-while-revalidate, immutable |
| Content fingerprinting + immutable = optimal caching | /app.hash.js with max-age=1y. URL changes when content changes |
| ETags enable conditional requests (304 Not Modified) | Revalidation avoids re-downloading unchanged resources |
| HSTS prevents SSL stripping | First-visit vulnerability unless preloaded |
| CSP whitelists content sources | Primary defense against XSS. Use nonces for inline scripts |
| Request smuggling exploits CL/TE parsing mismatches | HTTP/2 end-to-end eliminates the attack vector |
| Alt-Svc header enables HTTP/3 discovery | First request is always HTTP/2. Subsequent requests upgrade |
