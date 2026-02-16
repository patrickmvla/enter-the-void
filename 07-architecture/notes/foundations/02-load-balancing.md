# Load Balancing — Distributing Traffic Across Servers

## Why Load Balance

A single server has finite capacity. When traffic exceeds that capacity,
you add more servers. A load balancer sits in front of them and
distributes incoming requests.

```
Without LB:                    With LB:

  Client → Server              Client → [Load Balancer] → Server 1
  (single point of failure,              ↘ Server 2
   capacity ceiling)                      ↘ Server 3
```

Load balancing provides:
- **Horizontal scaling**: Spread load across N servers
- **High availability**: If a server dies, traffic routes to healthy ones
- **Zero-downtime deploys**: Drain traffic from a server, deploy, add it back
- **Geographic distribution**: Route to the nearest datacenter

---

## L4 vs L7 Load Balancing

The two fundamentally different approaches, named after OSI layers.

### L4 (Transport Layer)

Operates on TCP/UDP connections. The load balancer sees source IP, dest
IP, source port, dest port, protocol. It does NOT parse HTTP, inspect
headers, or understand the application.

```
Client (1.2.3.4:5000)
  │
  │ TCP SYN to LB (10.0.0.1:80)
  ▼
L4 Load Balancer
  │
  │ Decision: based on IP/port tuple
  │ Method: NAT (rewrite dest IP to server IP)
  │   or DSR (Direct Server Return — server replies directly to client)
  ▼
Server 2 (10.0.1.2:80)
  │
  │ TCP SYN-ACK
  ▼
(rest of the TCP connection flows through the LB, or directly
 back to the client in DSR mode)
```

**How L4 routing works:**

```
NAT mode (most common):
  Inbound:  Client → LB → Server   (LB rewrites dest IP)
  Outbound: Server → LB → Client   (LB rewrites source IP)
  LB sees all traffic. Can be a bandwidth bottleneck.

DSR mode (Direct Server Return):
  Inbound:  Client → LB → Server   (LB rewrites dest MAC, not IP)
  Outbound: Server → Client         (direct, bypasses LB)
  LB only sees inbound traffic. Server must be configured with
  the LB's IP as a loopback address.
  Used for: high-bandwidth services (video streaming) where outbound
  traffic dwarfs inbound (response >> request).

IP tunneling (IPIP):
  LB encapsulates the original packet inside a new IP packet
  destined for the server. Server decapsulates and replies directly
  to the client. Similar benefits to DSR but works across subnets.
```

**L4 characteristics:**
- Extremely fast (~millions of connections/sec)
- Low overhead (no content parsing)
- Sticky by connection (same TCP connection always goes to same server)
- Cannot route based on URL, headers, cookies
- Cannot terminate TLS (no access to HTTP layer)
- Linux IPVS, AWS NLB, Maglev (Google), Katran (Facebook)

### L7 (Application Layer)

Operates on HTTP (or other application protocols). The load balancer
terminates the TCP/TLS connection, parses the HTTP request, makes
routing decisions based on content, then opens a new connection to
the backend.

```
Client
  │
  │ TLS + HTTP request
  ▼
L7 Load Balancer
  │ Terminates TLS
  │ Parses HTTP: GET /api/users?id=42 Host: api.example.com
  │ Inspects: URL path, Host header, cookies, JWT claims
  │ Decides: route to backend pool "api-service"
  │ Opens new HTTP connection to backend
  ▼
Server 3 (10.0.1.3:8080)
```

**What L7 can do that L4 can't:**

```
Content-based routing:
  /api/*     → api-service pool
  /static/*  → CDN or static file servers
  /ws/*      → WebSocket server pool

Host-based routing:
  api.example.com → api servers
  www.example.com → web servers
  admin.example.com → admin servers

Header-based routing:
  X-Version: v2 → canary deployment servers
  Accept-Language: ja → Japan-localized servers

Cookie-based session affinity:
  Cookie: session=abc123 → hash to a specific server

TLS termination:
  Decrypt HTTPS, route as HTTP internally
  One certificate managed at the LB, not on every backend

Request manipulation:
  Add X-Real-IP, X-Forwarded-For headers
  Rewrite URLs, inject headers, rate limit
  Compress responses, cache static content
```

**L7 characteristics:**
- Slower than L4 (~100K-500K requests/sec per core)
- Higher overhead (TLS termination, HTTP parsing)
- Can do A/B testing, canary deployments, circuit breaking
- nginx, HAProxy (L7 mode), Envoy, AWS ALB, Cloudflare

### Choosing L4 vs L7

| Scenario | Choice | Why |
|----------|--------|-----|
| Raw TCP throughput (database, game servers) | L4 | No need to parse application protocol |
| Simple HTTP round-robin | L4 | Lower latency, higher throughput |
| Path-based routing, microservices | L7 | Need URL/header inspection |
| TLS termination + HTTP/2 | L7 | Must terminate TLS to inspect HTTP |
| WebSockets with long-lived connections | L4 or L7 | L7 adds overhead for upgrade; L4 is simpler |
| gRPC | L7 | gRPC multiplexes streams over one connection; L4 can't distribute streams |
| Very high throughput (>1M req/sec) | L4 in front of L7 | L4 for raw distribution, L7 for intelligent routing |

**Two-tier architecture** (production standard):

```
Internet → L4 LB (NLB/IPVS) → L7 LB (nginx/Envoy) → backends

L4 distributes TCP connections across L7 instances.
L7 instances handle TLS termination and content routing.
This combines L4 throughput with L7 intelligence.
```

---

## Load Balancing Algorithms

### Static Algorithms

**Round Robin:**
```
Requests: R1, R2, R3, R4, R5, R6
Servers:  [A, B, C]

R1 → A
R2 → B
R3 → C
R4 → A
R5 → B
R6 → C

Simple. Even distribution IF all requests are equal.
Problem: if Server B is slower, it accumulates a queue while A and C
are idle.
```

**Weighted Round Robin:**
```
Servers: A (weight=5), B (weight=3), C (weight=2)
Distribution: A gets 50%, B gets 30%, C gets 20%

Use when: servers have different capacity (CPU, memory).
The LB sends more traffic to more capable servers.
```

**IP Hash:**
```
server = hash(client_ip) % num_servers

Same client IP always goes to the same server.
Provides session affinity without cookies.
Problem: adding/removing a server reshuffles most mappings.
```

### Dynamic Algorithms

**Least Connections:**
```
Servers:  A (active: 12), B (active: 5), C (active: 8)
Next request → B (fewest active connections)

Adapts to server speed: fast servers complete requests quickly,
so their connection count drops, and they get more requests.
Best general-purpose algorithm for heterogeneous workloads.
```

**Weighted Least Connections:**
```
Score = active_connections / weight
A: 12/5 = 2.4    B: 5/3 = 1.67    C: 8/2 = 4.0
Next request → B (lowest score)

Combines capacity awareness (weights) with real-time load (connections).
```

**Least Response Time:**
```
Track the average response time from each server.
Route to the server with the lowest response time.

More accurate than least connections (a server with few connections
might be slow because it's overloaded with heavy queries).
Requires the LB to measure response latency — adds overhead.
```

**Random with Two Choices (Power of Two Choices):**
```
1. Pick two servers at random
2. Send the request to the one with fewer connections

Mathematically: reduces max load from O(log n / log log n) to
O(log log n). Dramatic improvement over pure random, with almost
no overhead. Doesn't require global state — just two random picks.

Used by: Envoy (default), many modern LBs.
```

The power of two choices is a profound result from randomized
algorithms. Pure random has max load O(log n / log log n). With just
ONE additional check (compare two random choices), the max load
drops to O(log log n). For 1,000 servers: from ~7 to ~3. The
marginal value of a third choice is negligible.

### Consistent Hashing (for Stateful Services)

When servers hold state (cache, session data), removing or adding a
server shouldn't invalidate all state. Consistent hashing minimizes
disruption.

```
Hash ring (0 to 2^32):

Servers: hash(A)=1000, hash(B)=4000, hash(C)=7000

Request routing: hash(request_key) → walk clockwise → first server

hash("user:42") = 2500 → clockwise → Server B (at 4000)
hash("user:43") = 5500 → clockwise → Server C (at 7000)

Adding Server D at position 3000:
  Only requests between 1000 and 3000 move (from B to D).
  Everything else stays. Movement: ~1/N of total.

Virtual nodes: each physical server → 100-200 positions on the ring.
  Smooths out uneven distribution from hash collisions.
```

Used by: Redis Cluster, Memcached clients, Cassandra, many CDNs.

---

## Health Checking

The LB must detect unhealthy servers and stop routing to them.

### Active Health Checks

The LB periodically probes each server:

```
Configuration (nginx):
  upstream backend {
      server 10.0.1.1:8080 max_fails=3 fail_timeout=30s;
      server 10.0.1.2:8080 max_fails=3 fail_timeout=30s;
      server 10.0.1.3:8080 max_fails=3 fail_timeout=30s;
  }

  # nginx+ or HAProxy:
  health_check interval=5s fails=3 passes=2 uri=/health;

Probes:
  TCP check: can we open a connection? (L4)
  HTTP check: GET /health returns 200? (L7)
  Custom: does /health return {"status":"ok","db":"connected"}?

Timing:
  interval: how often to probe (5-10 seconds)
  fails: consecutive failures before marking unhealthy (2-3)
  passes: consecutive successes before marking healthy again (2)
  timeout: probe timeout (3-5 seconds)
```

**The health check endpoint** should verify actual readiness, not just
"the process is running":

```javascript
app.get('/health', async (req, res) => {
    try {
        await db.query('SELECT 1');           // database reachable
        await redis.ping();                    // cache reachable
        const memUsage = process.memoryUsage();
        if (memUsage.heapUsed > MAX_HEAP) {
            return res.status(503).json({ status: 'degraded', reason: 'memory' });
        }
        res.json({ status: 'ok' });
    } catch (err) {
        res.status(503).json({ status: 'unhealthy', error: err.message });
    }
});
```

### Passive Health Checks

Detect failures from actual traffic, not probes:

```
Track error rate per server over a sliding window.
If Server B returns 5xx for > 50% of requests in the last 30 seconds:
  Mark unhealthy. Stop routing traffic.
  Periodically retry (1 request every 10 seconds) to check recovery.
```

Passive checks are faster to detect (no probe interval delay) but can
cause user-visible errors before the server is removed.

**Best practice**: Use both active and passive health checks.

---

## Session Affinity (Sticky Sessions)

Some applications store session state in server memory (not recommended
but common). Requests from the same user must go to the same server.

### Cookie-Based Affinity

```
First request:
  Client → LB → Server B
  LB inserts: Set-Cookie: SERVERID=srv-b

Subsequent requests:
  Client sends Cookie: SERVERID=srv-b
  LB reads cookie → route to Server B
```

### IP-Based Affinity

```
hash(client_ip) % num_servers → consistent routing

Problem: NAT. Thousands of users behind a corporate NAT share one IP.
All traffic goes to one server. Extremely uneven distribution.
```

### Why Sticky Sessions Are Problematic

```
1. Uneven distribution: one server gets all the "heavy" users
2. Failover loses session: if Server B dies, user's session is gone
3. Can't scale down: removing a server loses all its sessions
4. Autoscaling is harder: new servers get no traffic until sessions expire

Solution: externalize session state.
  Store sessions in Redis/Memcached. Any server can handle any request.
  Sticky sessions become unnecessary.
```

---

## Connection Draining (Graceful Shutdown)

When removing a server (deploy, scale down, failure), in-flight
requests must complete.

```
Without draining:
  1. Remove Server B from LB pool
  2. Kill Server B
  3. All in-flight requests to B get connection reset → 502 errors

With draining:
  1. Mark Server B as "draining" (accept no NEW connections)
  2. Wait for existing connections to complete (with timeout)
  3. After timeout or all connections done: kill Server B
  4. No user-visible errors
```

```
HAProxy configuration:
  # Graceful shutdown with 30-second drain
  server srv-b 10.0.1.2:8080 check drain

Kubernetes:
  spec:
    terminationGracePeriodSeconds: 30
  # Pod receives SIGTERM → removed from Service endpoints
  # Application has 30s to complete in-flight requests
  # After 30s: SIGKILL
```

The application must handle SIGTERM gracefully:

```javascript
process.on('SIGTERM', () => {
    console.log('SIGTERM received. Draining connections...');
    server.close(() => {
        // All connections completed
        db.end();
        process.exit(0);
    });
    // Force exit after timeout
    setTimeout(() => process.exit(1), 25000);
});
```

---

## Advanced Patterns

### Global Server Load Balancing (GSLB)

Route users to the nearest datacenter using DNS:

```
User in Tokyo → DNS query: api.example.com
  DNS returns: 103.x.x.x (Tokyo DC)

User in New York → DNS query: api.example.com
  DNS returns: 34.x.x.x (US-East DC)

Implementation:
  GeoDNS: route based on client IP geolocation
  Anycast: multiple DCs advertise the same IP via BGP,
           network routing naturally goes to closest
  Latency-based: measure actual latency, route to fastest
```

**Anycast** is the most elegant approach:

```
Same IP (1.1.1.1) announced from:
  Tokyo DC:    via AS path through Asian transit providers
  London DC:   via AS path through European transit providers
  New York DC: via AS path through American transit providers

BGP routing naturally sends packets to the topologically nearest DC.
No DNS-level routing needed. Failover is automatic — if a DC goes
down, its BGP announcement is withdrawn, and traffic reroutes.

Used by: Cloudflare (all traffic), Google (DNS and frontend),
         all major CDNs.

Limitation: Anycast is IP-level. A TCP connection to an anycast IP
can break if routing changes mid-connection (packets go to a different
DC). This is why anycast is common for UDP (DNS) and short-lived
TCP connections, but long-lived connections use unicast behind anycast.
```

### Request Hedging

Send the same request to multiple servers and use the first response:

```
1. Send request to Server A
2. If no response within p95 latency (say 50ms), send same request to Server B
3. Use whichever responds first, cancel the other

This cuts tail latency dramatically:
  Without hedging: p99 latency = 500ms
  With hedging: p99 latency ≈ p50 latency = 20ms

Cost: ~5% more load (only hedge after p95, so only 5% of requests are duplicated).
Requires idempotent requests (safe to execute twice).
```

Used by Google (described in "The Tail at Scale" paper by Jeff Dean).

### Load Shedding

When overloaded, reject requests early rather than processing them
slowly (which cascades failure):

```
Server at capacity:
  Option 1: Accept all, respond slowly to everyone (cascading failure)
  Option 2: Reject excess requests with 503 (load shedding)

Implementation:
  Track: current request count, queue depth, or CPU usage
  Threshold: if concurrent_requests > 1000, reject with 503
  Priority: shed low-priority requests first (analytics before checkout)

HTTP: 503 Service Unavailable + Retry-After header
  The LB sees 503 and can route to another server.
```

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| L4: TCP/IP level, fast, no content inspection | NAT or DSR mode. Millions of conn/sec. IPVS, NLB |
| L7: HTTP level, TLS termination, content routing | Path/host/header routing. nginx, Envoy, ALB |
| Two-tier: L4 in front of L7 is production standard | L4 for distribution, L7 for intelligence |
| Least connections is the best general-purpose algorithm | Adapts to server speed. Weighted for heterogeneous capacity |
| Power of two choices: pick 2 random, choose the less loaded | O(log log n) max load vs O(log n / log log n) for pure random |
| Consistent hashing for stateful routing | Virtual nodes for balance. 1/N data movement on resize |
| Health checks: active (probe) + passive (error rate) | Endpoint should verify DB, cache, not just process liveness |
| Sticky sessions are an anti-pattern | Externalize state to Redis. Any server handles any request |
| Connection draining prevents 502s during deploys | Mark draining → wait for in-flight → kill. SIGTERM handler in app |
| Anycast: same IP from multiple DCs, BGP routes to nearest | Elegant for DNS/CDN. TCP connections can break on route changes |
| Request hedging cuts tail latency (send to 2, use first response) | 5% more load eliminates p99 spikes. Requires idempotent requests |
| Load shedding: reject early rather than process slowly | 503 + Retry-After. Prevents cascading failure under overload |
