# Real-Time Communication — WebSockets, SSE, and the Alternatives

## The Problem

HTTP is request-response. The client asks, the server answers. But many
applications need the server to push data to the client without being
asked — chat messages, live notifications, stock tickers, multiplayer
game state, collaborative editing, real-time dashboards.

HTTP wasn't designed for this. Every technique in this document is either
a workaround for HTTP's limitations or a protocol built specifically to
solve them.

---

## The Evolution: Polling → Long Polling → SSE → WebSockets

### Short Polling

The simplest approach. The client repeatedly asks "anything new?"

```
Client                              Server
  │  GET /messages?since=100         │
  │ ───────────────────────────────> │
  │  200 OK (empty — nothing new)   │
  │ <─────────────────────────────── │
  │                                  │
  │  (wait 2 seconds)                │
  │                                  │
  │  GET /messages?since=100         │
  │ ───────────────────────────────> │
  │  200 OK (empty)                 │
  │ <─────────────────────────────── │
  │                                  │
  │  (wait 2 seconds)                │
  │                                  │
  │  GET /messages?since=100         │
  │ ───────────────────────────────> │
  │  200 OK [{id: 101, text: "hi"}] │
  │ <─────────────────────────────── │
```

**Problems:**
- Most responses are empty — wasted bandwidth and server load
- Latency = polling interval. 2-second polls mean up to 2-second delay
- Reducing the interval increases server load quadratically with users
- 10,000 users polling every 1 second = 10,000 requests/second, mostly
  returning empty responses

**When it's acceptable**: Low-frequency updates (weather data, daily
reports), when simplicity outweighs efficiency, when the number of
clients is small.

### Long Polling

The client sends a request, and the server **holds it open** until
there's data to send:

```
Client                              Server
  │  GET /messages?since=100         │
  │ ───────────────────────────────> │
  │                                  │  (holds connection open)
  │                                  │  (waits for new data...)
  │                                  │  (30 seconds pass...)
  │                                  │  (new message arrives!)
  │  200 OK [{id: 101, text: "hi"}] │
  │ <─────────────────────────────── │
  │                                  │
  │  GET /messages?since=101         │  (immediately reconnects)
  │ ───────────────────────────────> │
  │                                  │  (holds again...)
```

**How it works at the server:**

```javascript
app.get('/messages', async (req, res) => {
    const since = req.query.since;

    // Check for existing messages
    let messages = await db.getMessagesSince(since);

    if (messages.length > 0) {
        return res.json(messages);  // Return immediately
    }

    // No messages — hold the connection
    const timeout = setTimeout(() => {
        res.json([]);  // Return empty after 30s to prevent connection timeout
    }, 30000);

    // Subscribe to new messages
    const unsubscribe = messageEmitter.on('new', (msg) => {
        clearTimeout(timeout);
        unsubscribe();
        res.json([msg]);
    });

    // Clean up on disconnect
    req.on('close', () => {
        clearTimeout(timeout);
        unsubscribe();
    });
});
```

**Improvements over short polling:**
- Near-zero latency when data is available (response sent immediately)
- No wasted empty responses (connection is held until data exists)

**Problems:**
- Each held connection consumes a server resource (file descriptor,
  memory for the request context). Thread-per-connection servers (like
  traditional Java/PHP) exhaust threads quickly. Event-driven servers
  (Node.js, Go) handle this better.
- Reconnection gap: Between receiving a response and establishing the
  next long poll, messages can be missed. Need sequence numbers to
  track position.
- HTTP overhead on every reconnection (headers, cookies, TLS if new
  connection).
- Load balancers and proxies may time out held connections (default
  timeouts of 30-60s).

**When to use**: Push notifications on legacy infrastructure, when
WebSockets aren't available (restrictive corporate proxies), simple
real-time features where the overhead is acceptable.

---

## Server-Sent Events (SSE)

SSE (EventSource API) is a standardized, HTTP-based protocol for
server-to-client streaming. The server sends a stream of events over
a single long-lived HTTP connection.

### The Protocol

SSE uses plain HTTP with `Content-Type: text/event-stream`. The server
sends events as formatted text chunks:

```http
GET /events HTTP/1.1
Accept: text/event-stream
Cache-Control: no-cache

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"user": "alice", "message": "hello"}\n\n

data: {"user": "bob", "message": "hi there"}\n\n

event: typing
data: {"user": "alice"}\n\n

id: 42
event: message
data: {"user": "alice", "message": "how are you?"}\n
data: continues on next line\n\n

retry: 5000\n\n
```

### Event Format

```
[field]: [value]\n

Fields:
  data:    Event payload (multiple data lines are concatenated with \n)
  event:   Event type (client can listen for specific types)
  id:      Event ID (for reconnection — client sends Last-Event-ID header)
  retry:   Reconnection delay in milliseconds
  : comment (lines starting with colon are ignored — used for keepalive)

Events are separated by a blank line (\n\n).
```

### Client API (Browser)

```javascript
const source = new EventSource('/events');

// Default "message" event
source.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log(data);
};

// Named events
source.addEventListener('typing', (event) => {
    const data = JSON.parse(event.data);
    showTypingIndicator(data.user);
});

// Connection lifecycle
source.onopen = () => console.log('Connected');
source.onerror = (e) => console.log('Error, reconnecting...');
// EventSource automatically reconnects on failure
```

### Automatic Reconnection

The killer feature of SSE. If the connection drops:

1. The client waits `retry` milliseconds (default: browser-specific,
   usually 3 seconds)
2. Reconnects to the same URL
3. Sends `Last-Event-ID: 42` header (the last received event ID)
4. The server resumes from event 43

```http
GET /events HTTP/1.1
Last-Event-ID: 42

HTTP/1.1 200 OK
Content-Type: text/event-stream

id: 43
data: {"message": "you missed this one"}\n\n

id: 44
data: {"message": "and this one"}\n\n
```

This is built into the browser's EventSource API — no application code
needed for reconnection. The server just needs to support the
`Last-Event-ID` header.

### SSE vs WebSockets

| Feature | SSE | WebSockets |
|---------|-----|------------|
| Direction | Server → Client only | Bidirectional |
| Protocol | HTTP (text/event-stream) | ws:// / wss:// (custom) |
| Auto-reconnect | Built-in (EventSource API) | Manual (application code) |
| Binary data | No (text only, base64 for binary) | Yes (native binary frames) |
| HTTP/2 multiplexing | Yes (one stream per SSE) | No (separate TCP connection) |
| Proxy/firewall compatibility | Excellent (it's just HTTP) | Can be blocked |
| Browser API | EventSource (simple) | WebSocket (more complex) |
| Max connections per domain | 6 on HTTP/1.1 (shared with other requests) | No HTTP limit (separate protocol) |
| Compression | HTTP gzip/br works | Per-message deflate extension |

**When SSE wins**: Server-to-client notifications, live feeds, dashboards,
any case where the client doesn't need to send frequent data back. SSE's
simplicity, auto-reconnection, and HTTP compatibility make it the better
choice for most real-time features.

**When WebSockets win**: Bidirectional communication (chat, gaming,
collaborative editing), binary protocols, high-frequency client→server
messages.

---

## WebSockets

WebSockets (RFC 6455) provide full-duplex, bidirectional communication
over a single TCP connection. After an HTTP upgrade handshake, the
protocol switches from HTTP to a lightweight framing protocol.

### The Opening Handshake

WebSockets start as HTTP and upgrade:

```http
# Client request:
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Extensions: permessage-deflate
Origin: https://example.com

# Server response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
Sec-WebSocket-Extensions: permessage-deflate
```

**Sec-WebSocket-Key / Sec-WebSocket-Accept**: Anti-cache mechanism, NOT
security. The server concatenates the client's key with a fixed GUID
(`258EAFA5-E914-47DA-95CA-5AB0DC85B11B`), SHA-1 hashes it, and
base64-encodes the result:

```
Accept = base64(SHA-1(Key + "258EAFA5-E914-47DA-95CA-5AB0DC85B11B"))
       = base64(SHA-1("dGhlIHNhbXBsZSBub25jZQ==" + "258EAFA5-..."))
       = "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

This proves the server understands WebSocket (not a regular HTTP server
accidentally accepting the upgrade). It does NOT authenticate anything.
Authentication happens at the application level (tokens in query params,
cookies, or in the first WebSocket message).

**Sec-WebSocket-Protocol**: Subprotocol negotiation. The client offers
protocols it supports (`chat, superchat`), the server picks one (`chat`).
This is application-level — WebSocket doesn't define what the subprotocols
mean.

### The Frame Format

After the handshake, communication switches to WebSocket frames:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+------+------------------------+
|                    Masking-key (0 or 4 bytes)                  |
+---------------------------------------------------------------+
|                    Payload Data (variable)                     |
+---------------------------------------------------------------+
```

**Key fields:**

- **FIN** (1 bit): Is this the final fragment of a message? Messages can
  be split across multiple frames for streaming.

- **Opcode** (4 bits):
  - 0x0 = Continuation frame
  - 0x1 = Text frame (UTF-8)
  - 0x2 = Binary frame
  - 0x8 = Close
  - 0x9 = Ping
  - 0xA = Pong

- **MASK** (1 bit): Client-to-server frames MUST be masked. Server-to-client
  frames MUST NOT be masked.

- **Payload length**: 7 bits (0-125: actual length), 126 (next 2 bytes are
  the length), 127 (next 8 bytes are the length). Max frame size: 2^63 bytes.

- **Masking key** (4 bytes, client→server only): Each byte of the payload
  is XORed with the masking key: `payload[i] ^= mask[i % 4]`.

### Why Masking?

WebSocket masking exists to prevent **cache poisoning attacks** against
intermediary infrastructure (proxies that don't understand WebSocket).

The attack scenario without masking:
1. Attacker's JavaScript opens a WebSocket connection through a
   non-WebSocket-aware proxy
2. The proxy sees the upgrade but doesn't understand it — treats
   subsequent frames as regular HTTP
3. Attacker crafts a WebSocket frame whose payload looks like an HTTP
   response: `HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<script>evil</script>`
4. The proxy interprets this as a cacheable HTTP response and stores it
5. Other users requesting the same URL get the poisoned cache entry

Masking with a random key makes it impossible for the attacker to
predict what the frame will look like on the wire, preventing this attack.
The masking is NOT for confidentiality (the key is sent in plaintext in
the frame header) — it's purely anti-cache-poisoning.

### Control Frames

**Ping/Pong** (keepalive):
```
Server sends: Ping (opcode 0x9, optional payload)
Client responds: Pong (opcode 0xA, echo the payload)
```
Used to detect dead connections. If a Pong doesn't arrive within a
timeout, the connection is presumed dead. Application-level keepalive
on top of TCP's (because TCP keepalive has long default intervals).

**Close**:
```
Initiator sends: Close frame (opcode 0x8, status code + reason)
Responder sends: Close frame (echoing the status code)
Both sides close the TCP connection.
```

Close status codes:
- 1000 = Normal closure
- 1001 = Going away (server shutdown, page navigation)
- 1002 = Protocol error
- 1003 = Unsupported data
- 1006 = Abnormal closure (no close frame received — connection dropped)
- 1008 = Policy violation
- 1011 = Unexpected condition (server error)
- 4000-4999 = Application-defined codes

### Authentication

WebSockets don't have a built-in authentication mechanism. Common
patterns:

**1. Cookie-based** (simplest for browser clients):
The WebSocket handshake is an HTTP request — cookies are sent
automatically. The server validates the session cookie during the
upgrade handshake and rejects unauthorized connections with 401/403.

```javascript
// Server (Node.js)
wss.on('connection', (ws, req) => {
    const session = parseSessionCookie(req.headers.cookie);
    if (!session.isValid()) {
        ws.close(1008, 'Unauthorized');
        return;
    }
    ws.userId = session.userId;
});
```

**2. Token in query parameter** (common for non-browser clients):
```javascript
const ws = new WebSocket('wss://example.com/ws?token=eyJhbGc...');
```
The server validates the token during the handshake. **Caution**: The
token appears in server access logs and potentially in proxy logs.
Use short-lived tokens.

**3. Token in first message** (most flexible):
```javascript
const ws = new WebSocket('wss://example.com/ws');
ws.onopen = () => {
    ws.send(JSON.stringify({ type: 'auth', token: 'eyJhbGc...' }));
};
```
The server holds the connection in an "unauthenticated" state and
rejects any non-auth messages until a valid token is received. Most
complex but avoids token in URL.

**Note**: The `Authorization` header cannot be set on WebSocket
connections from browsers (the WebSocket API doesn't support custom
headers). This is a known limitation — hence the workarounds above.

### Scaling WebSockets

WebSockets are stateful. Each connection is a persistent TCP connection
pinned to a specific server. This creates challenges that stateless
HTTP doesn't have.

**The problem with load balancers:**
```
              ┌──> Server A (has Alice's WebSocket)
Client ──LB──┤
              └──> Server B (has Bob's WebSocket)

Alice wants to message Bob.
Alice's message arrives at Server A.
Bob's WebSocket is on Server B.
How does Server A send to Bob?
```

**Solutions:**

**1. Pub/Sub backbone** (most common):

```
Server A ──┐                ┌── Server A
            ├── Redis Pub/Sub ──┤
Server B ──┘                └── Server B

1. Alice sends message to Server A
2. Server A publishes to Redis channel "chat:room1"
3. Redis delivers to all subscribers (Server A and B)
4. Server B receives, finds Bob's WebSocket, sends to Bob
```

Redis, NATS, RabbitMQ, or Kafka as the message bus. Every server
subscribes to relevant channels and forwards messages to local
connections.

**2. Sticky sessions:**
The load balancer routes all of a client's connections to the same
server (based on cookie or IP). Avoids cross-server communication
but creates uneven load distribution and makes server failure affect
all its connected clients.

**3. Dedicated real-time infrastructure:**
Services like Ably, Pusher, or self-hosted solutions (Centrifugo,
Socket.IO with Redis adapter) handle the fan-out. Application servers
publish events to the infrastructure, which handles delivery to
connected clients.

### Connection Limits

A single server's WebSocket capacity is limited by:

- **File descriptors**: One per connection. Default limit ~1024, set to
  65535+ for production. Linux: `ulimit -n` / `/etc/security/limits.conf`.
- **Memory**: Each connection holds buffers, state, and application data.
  ~10-50 KB per idle connection. 100,000 connections ≈ 1-5 GB.
- **CPU**: Message serialization, deserialization, and routing. More
  relevant for high-throughput than high-connection-count scenarios.
- **Bandwidth**: Each connection might receive N messages/second. 100,000
  connections × 10 messages/second × 200 bytes = 200 MB/s outbound.

A well-optimized server (Go, Rust, or C with epoll/kqueue) can handle
1M+ concurrent WebSocket connections on a single machine. Node.js
with uWebSockets.js handles ~100K-500K depending on message patterns.

---

## WebSocket over HTTP/2 (RFC 8441)

HTTP/2's multiplexing creates an opportunity: WebSocket connections
can share the same HTTP/2 connection instead of requiring a separate
TCP connection each.

```
HTTP/2 Connection:
  Stream 1: Regular HTTP request/response
  Stream 3: WebSocket connection (CONNECT method with :protocol pseudo-header)
  Stream 5: Another regular request
  Stream 7: Another WebSocket connection
```

The client sends an extended CONNECT request:
```
:method: CONNECT
:protocol: websocket
:scheme: https
:path: /chat
:authority: example.com
sec-websocket-version: 13
```

Benefits:
- Shares the TCP connection (no additional handshake)
- Benefits from HTTP/2's header compression
- Multiple WebSocket connections multiplex on one TCP connection
- Better for HTTP/2-aware proxies and load balancers

Adoption is still growing — not all servers and proxies support RFC 8441.

---

## Choosing the Right Approach

```
Do you need server → client push?
├── No → Regular HTTP (request-response)
└── Yes
    ├── How often do updates occur?
    │   ├── Rarely (minutes/hours) → Short polling or webhooks
    │   └── Frequently (seconds or faster)
    │       ├── Does the client need to send data back frequently?
    │       │   ├── No → SSE (simpler, auto-reconnect, HTTP-native)
    │       │   └── Yes → WebSockets
    │       ├── Is the data binary?
    │       │   ├── No → SSE or WebSockets
    │       │   └── Yes → WebSockets
    │       └── Are there proxy/firewall constraints?
    │           ├── Yes → SSE (it's just HTTP) or long polling
    │           └── No → WebSockets or SSE based on above
    └── Scale considerations
        ├── < 10K concurrent → Any approach works
        ├── 10K-100K concurrent → SSE or WebSockets with pub/sub
        └── 100K+ concurrent → Dedicated real-time infrastructure
```

### Common Patterns

| Use Case | Best Approach | Why |
|----------|--------------|-----|
| Live notifications | SSE | Server→client only, auto-reconnect |
| Chat application | WebSockets | Bidirectional, low-latency |
| Live dashboard | SSE | Server pushes metrics, client just displays |
| Collaborative editing | WebSockets | Bidirectional operational transforms |
| Multiplayer game | WebSockets (or UDP/WebRTC) | Low-latency bidirectional, binary data |
| Live sports scores | SSE | One-way push, infrequent updates |
| Stock ticker | WebSockets or SSE | High-frequency server push |
| File upload progress | SSE | Server pushes progress to client |
| IoT device communication | WebSockets + MQTT | Lightweight bidirectional |

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Short polling wastes resources, long polling holds connections | Both are HTTP workarounds for push. Long polling has reconnection gaps |
| SSE is HTTP-native server→client streaming | text/event-stream, auto-reconnect via Last-Event-ID, works with HTTP/2 |
| WebSockets upgrade from HTTP to a binary framing protocol | Full-duplex after 101 Switching Protocols. Not HTTP anymore |
| WebSocket masking prevents cache poisoning | XOR with random key. Not for confidentiality — for proxy safety |
| Sec-WebSocket-Key/Accept is anti-cache, not auth | SHA-1 + magic GUID. Proves server understands WebSocket protocol |
| WebSocket auth: cookie, query param, or first message | No custom headers from browser WebSocket API |
| Scaling WebSockets requires a pub/sub backbone | Redis/NATS between servers to fan out messages across connections |
| SSE beats WebSockets for most server→client push | Simpler, auto-reconnect, HTTP-native, HTTP/2 compatible |
| WebSockets beat SSE for bidirectional, binary, high-frequency | Chat, gaming, collaboration — where the client sends as much as it receives |
| Connection limits: FDs, memory, bandwidth | 100K connections ≈ 1-5 GB RAM. Tune ulimits for production |
