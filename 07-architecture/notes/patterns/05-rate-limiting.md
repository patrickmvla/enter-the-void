# Rate Limiting & Resilience Patterns

## Why Rate Limit

Without rate limiting, a single client can overwhelm your service:
- A buggy client sends 10,000 requests/second in a tight loop
- A DDoS attack floods your API with traffic
- A legitimate traffic spike (viral content) exceeds capacity
- One noisy tenant in a multi-tenant system starves others

Rate limiting protects the system by capping how many requests a client
can make in a given time window.

---

## Rate Limiting Algorithms

### Fixed Window Counter

Divide time into fixed windows (e.g., 1-minute windows). Count
requests per window. Reject when the count exceeds the limit.

```
Limit: 100 requests per minute

Window: 12:00:00 - 12:00:59
  Request count: 0, 1, 2, ... 99 → allowed
  Request 101 → rejected (429 Too Many Requests)

Window: 12:01:00 - 12:01:59
  Counter resets to 0
```

**Implementation:**
```
Key: "ratelimit:{client_id}:{window}"
  window = floor(current_timestamp / window_size)

INCR key
IF value > limit: reject
EXPIRE key window_size  (auto-cleanup)

Example with Redis:
  key = "ratelimit:user42:202401151200"
  INCR → returns current count
  If first request: EXPIRE key 60
```

**The boundary problem:**

```
Limit: 100 requests per minute

12:00:30: 100 requests (allowed — fills the 12:00 window)
12:01:00: window resets
12:01:01: 100 requests (allowed — fills the 12:01 window)

Result: 200 requests in 31 seconds. The effective rate is 2x the limit
at the boundary between windows.
```

### Sliding Window Log

Track the exact timestamp of every request. Count requests within
the last N seconds.

```
Limit: 100 requests per 60 seconds

On each request:
  1. Remove all entries older than (now - 60s) from the log
  2. Count remaining entries
  3. If count >= 100: reject
  4. Else: add current timestamp to log, allow

Log: [12:00:05, 12:00:06, 12:00:07, ..., 12:00:45]
New request at 12:01:02:
  Remove entries before 12:00:02
  Count remaining: 78
  78 < 100 → allow, add 12:01:02
```

**Pros:** Perfectly accurate. No boundary problem.
**Cons:** Stores every timestamp. Memory: O(limit) per client.
For 100M clients × 1000 req/min limit = 100 billion timestamps.

### Sliding Window Counter

Approximation that eliminates the boundary problem without storing
every timestamp. Interpolate between the current and previous
fixed windows.

```
Limit: 100 requests per minute

Previous window (12:00): 84 requests
Current window (12:01): 36 requests
Current time: 12:01:15 (25% into the current window)

Weighted count = prev_count × (1 - elapsed_fraction) + current_count
               = 84 × 0.75 + 36
               = 63 + 36
               = 99

99 < 100 → allow

Next request:
  99 + 1 = 100 → at limit
  Next request rejected
```

**Pros:** Only stores 2 counters per client per window. Very efficient.
Eliminates the boundary problem (approximately — not mathematically
exact, but close enough in practice).

**This is the algorithm most production rate limiters use.**

### Token Bucket

A bucket holds tokens. Tokens are added at a fixed rate. Each request
consumes a token. If the bucket is empty, the request is rejected.

```
Capacity: 10 tokens (burst size)
Refill rate: 2 tokens/second

t=0:   bucket = 10
t=0:   5 requests arrive → bucket = 5 (burst allowed)
t=0.5: 1 token added → bucket = 6
t=1:   2 tokens added → bucket = 8
t=1:   3 requests arrive → bucket = 5
t=5:   bucket capped at 10 (full)
```

**Key property**: Allows bursts up to the bucket capacity, while
limiting the sustained rate to the refill rate. This is usually the
desired behavior — short bursts are fine, sustained overload is not.

**Implementation (atomic with Redis + Lua):**
```lua
-- Token bucket in Redis (Lua script for atomicity)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])     -- tokens per second
local now = tonumber(ARGV[3])      -- current timestamp
local requested = tonumber(ARGV[4]) -- tokens to consume (usually 1)

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens based on elapsed time
local elapsed = now - last_refill
local new_tokens = math.min(capacity, tokens + elapsed * rate)

if new_tokens >= requested then
    new_tokens = new_tokens - requested
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
    return 1  -- allowed
else
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
    return 0  -- rejected
end
```

### Leaky Bucket

Requests enter a queue (bucket). The queue drains at a fixed rate.
If the queue is full, new requests are rejected.

```
Queue capacity: 10
Drain rate: 2 requests/second

Requests arrive in bursts:
  t=0: 8 requests → queue = 8
  t=0.5: queue drains 1 → queue = 7
  t=1: queue drains 2 → queue = 5, 3 more arrive → queue = 8
  t=1: 5 more arrive → queue = 10 (full), 3 rejected

Output is always smooth: exactly 2 requests/second processed.
```

**Difference from token bucket:**
- Token bucket: allows bursts (sends all at once up to capacity)
- Leaky bucket: smooths output (processes at fixed rate regardless of burst)

Leaky bucket shapes traffic. Token bucket permits bursts.

### Algorithm Comparison

| Algorithm | Burst | Accuracy | Memory | Implementation |
|-----------|-------|----------|--------|----------------|
| Fixed Window | 2x at boundary | Approximate | O(1) per key | Simple counter |
| Sliding Window Log | None | Exact | O(limit) per key | Sorted set of timestamps |
| Sliding Window Counter | Minimal | Approximate | O(1) per key | Two counters |
| Token Bucket | Controlled burst | Exact | O(1) per key | Tokens + timestamp |
| Leaky Bucket | None (smoothed) | Exact | O(queue size) | FIFO queue |

---

## Distributed Rate Limiting

When your API runs on multiple servers, each server sees only a
fraction of the traffic. A per-server rate limit of 100/min means
a client can send 100 × N requests/min (where N = number of servers).

### Centralized (Redis)

All servers check/update a shared Redis counter:

```
Server 1 ─┐
Server 2 ─┤──→ Redis: INCR "ratelimit:user42:window"
Server 3 ─┘

Pros: Accurate global count
Cons: Redis is a single point of failure. Network latency on every request.
      Redis becomes a bottleneck at very high request rates.
```

### Local + Sync

Each server maintains a local counter and periodically syncs with
the central store:

```
Server 1: local_count = 30 (sync every 1s)
Server 2: local_count = 25
Server 3: local_count = 20

Every 1 second:
  Each server reports its count to Redis
  Redis: global_count = 75
  Each server gets the global count back
  If global_count > limit: reject locally

Trade-off: up to 1-second window where the limit can be exceeded
(each server doesn't know about the others' counts yet).
This is usually acceptable.
```

### Cell-Based Rate Limiting

Divide the global limit among servers, with periodic rebalancing:

```
Global limit: 300 req/min, 3 servers

Initial allocation:
  Server 1: 100 req/min
  Server 2: 100 req/min
  Server 3: 100 req/min

If Server 1 is getting more traffic:
  Rebalance: Server 1: 150, Server 2: 100, Server 3: 50

Each server rate-limits locally with its allocated share.
No per-request coordination needed. Rebalance periodically.
```

---

## HTTP Rate Limiting Headers

Standard headers to communicate rate limit status to clients:

```
HTTP/1.1 200 OK
RateLimit-Limit: 100          (max requests per window)
RateLimit-Remaining: 42       (requests left in current window)
RateLimit-Reset: 1705334400   (Unix timestamp when window resets)

HTTP/1.1 429 Too Many Requests
Retry-After: 30               (seconds to wait before retrying)
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1705334400
```

Clients should implement backoff based on Retry-After.

---

## Circuit Breaker

A pattern from electrical engineering. When a downstream service fails
repeatedly, stop sending requests to it (open the circuit) instead
of letting failures cascade.

### Three States

```
         success
    ┌──────────────┐
    │              │
    ▼              │
 ┌──────┐   failure count    ┌──────┐   timeout    ┌───────────┐
 │CLOSED│ ──── > threshold ──>│ OPEN │ ──────────── > │HALF-OPEN │
 └──────┘                    └──────┘               └───────────┘
    ▲                          ▲                       │    │
    │                          │                       │    │
    │        success           │      failure          │    │
    └──────────────────────────┼───────────────────────┘    │
                               │                            │
                               └────────────────────────────┘

CLOSED:    Normal operation. Requests flow through.
           Track failure count. If failures exceed threshold: → OPEN

OPEN:      All requests immediately fail (fast failure, no waiting
           for timeout). No requests sent to downstream.
           After a timeout period: → HALF-OPEN

HALF-OPEN: Let ONE request through (probe).
           If it succeeds: → CLOSED (service recovered)
           If it fails: → OPEN (still broken)
```

**Implementation:**

```javascript
class CircuitBreaker {
    constructor(options) {
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 30000; // 30s
        this.halfOpenMaxAttempts = options.halfOpenMaxAttempts || 1;
    }

    async execute(fn) {
        if (this.state === 'OPEN') {
            if (Date.now() > this.nextAttempt) {
                this.state = 'HALF_OPEN';
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }

        try {
            const result = await fn();
            this.onSuccess();
            return result;
        } catch (err) {
            this.onFailure();
            throw err;
        }
    }

    onSuccess() {
        this.failureCount = 0;
        if (this.state === 'HALF_OPEN') {
            this.state = 'CLOSED';
        }
    }

    onFailure() {
        this.failureCount++;
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
            this.nextAttempt = Date.now() + this.resetTimeout;
        }
    }
}

// Usage:
const paymentCircuit = new CircuitBreaker({
    failureThreshold: 5,
    resetTimeout: 30000
});

try {
    const result = await paymentCircuit.execute(() =>
        fetch('https://payment-service/charge', { method: 'POST', body })
    );
} catch (err) {
    if (err.message.includes('Circuit breaker')) {
        // Fast failure — don't even try the downstream
        return fallbackResponse();
    }
    throw err;
}
```

**Why circuit breakers matter:**

```
Without circuit breaker:
  Payment service is down.
  Every checkout request waits 30 seconds for TCP timeout.
  Thread pool exhausted. Cascading failure to the checkout service.
  Checkout service becomes unresponsive. Cascades to the frontend.
  Entire system down because one dependency is slow.

With circuit breaker:
  Payment service is down. 5 requests fail.
  Circuit OPENS. All subsequent requests fail immediately (<1ms).
  Checkout service returns "payment temporarily unavailable" fast.
  System remains responsive. Only payment-dependent features degraded.
```

---

## Bulkhead Pattern

Isolate failures by partitioning resources. Named after ship
bulkheads — watertight compartments that prevent a hull breach
from sinking the entire ship.

```
Without bulkhead:
  Thread pool: 100 threads shared by all endpoints
  /api/search (slow DB) uses 95 threads
  /api/health (fast) can't get a thread → appears unhealthy → cascading failure

With bulkhead:
  Thread pool for /api/search:  50 threads (isolated)
  Thread pool for /api/orders:  30 threads (isolated)
  Thread pool for /api/health:  20 threads (isolated)
  If search exhausts its 50 threads, orders and health are unaffected.
```

**Implementation approaches:**
- **Thread pool isolation**: Separate thread pools per dependency
  (Hystrix-style, JVM)
- **Semaphore isolation**: Limit concurrent requests per dependency
  (lighter weight, any language)
- **Connection pool isolation**: Separate database connection pools
  per service/feature
- **Process/container isolation**: Separate containers per service
  (strongest isolation)

```javascript
// Semaphore bulkhead
class Bulkhead {
    constructor(maxConcurrent) {
        this.maxConcurrent = maxConcurrent;
        this.current = 0;
    }

    async execute(fn) {
        if (this.current >= this.maxConcurrent) {
            throw new Error('Bulkhead full — request rejected');
        }
        this.current++;
        try {
            return await fn();
        } finally {
            this.current--;
        }
    }
}

const searchBulkhead = new Bulkhead(50);
const ordersBulkhead = new Bulkhead(30);
```

---

## Retry Strategies

### Exponential Backoff with Jitter

When a request fails, retry with increasing delays:

```
Attempt 1: fail → wait 1s
Attempt 2: fail → wait 2s
Attempt 3: fail → wait 4s
Attempt 4: fail → wait 8s
Attempt 5: fail → give up

Exponential: delay = base × 2^attempt
With cap:    delay = min(base × 2^attempt, max_delay)
```

**Without jitter**: If 1000 clients fail simultaneously and all retry
at the exact same intervals, they create synchronized retry storms:

```
t=0:    1000 clients fail
t=1s:   1000 clients retry simultaneously → server overwhelmed again
t=3s:   1000 clients retry simultaneously → overwhelmed again
...forever (thundering herd of retries)
```

**With jitter**: Add randomness to break synchronization:

```
Full jitter:
  delay = random(0, base × 2^attempt)

Equal jitter:
  half = base × 2^attempt / 2
  delay = half + random(0, half)

Decorrelated jitter (AWS recommendation):
  delay = random(base, prev_delay × 3)
```

```javascript
async function retryWithBackoff(fn, maxRetries = 5) {
    let delay = 1000; // 1 second base
    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            return await fn();
        } catch (err) {
            if (attempt === maxRetries - 1) throw err;
            if (!isRetryable(err)) throw err;

            // Decorrelated jitter
            delay = Math.min(
                30000, // 30s max
                Math.random() * delay * 3
            );
            await sleep(delay);
        }
    }
}

function isRetryable(err) {
    // Retry: network errors, 429, 500, 502, 503, 504
    // Don't retry: 400, 401, 403, 404 (client errors won't fix themselves)
    return err.status >= 500 || err.status === 429 || err.code === 'ECONNRESET';
}
```

### Retry Budget

Limit total retries across all requests to prevent retry amplification:

```
Without budget:
  1000 requests fail → 1000 retries → 1000 more fail → 1000 more retries
  → 3000 total requests to a failing service (amplification factor: 3x)

With budget (max 10% retries):
  1000 requests fail → 100 retries allowed → stop
  Total: 1100 requests. Controlled amplification.

Track: total_requests and retry_requests over sliding window
  if retry_requests / total_requests > 0.1: stop retrying
```

---

## Timeout Strategy

Every remote call needs a timeout. Without timeouts, slow dependencies
cause threads to hang indefinitely, exhausting resources.

```
Connection timeout: how long to wait for TCP handshake (1-5s)
  If the server is down, fail fast instead of waiting 2 minutes for TCP timeout.

Request timeout: how long to wait for the response (1-30s, depends on operation)
  If the server is overloaded, don't wait forever.

Idle timeout: how long to keep an idle connection open (60-300s)
  Close stale connections to free resources.

End-to-end timeout: total allowed time for a user request (5-30s)
  Even if individual calls are within timeout, the cumulative
  time must not exceed the user's patience.
```

**Timeout propagation** (deadline propagation):

```
User → API Gateway (30s deadline)
  → Service A (25s remaining)
    → Service B (20s remaining)
      → Database (15s remaining)

Each hop subtracts elapsed time from the deadline.
If Service A took 5s, Service B gets 25-5 = 20s.
If the deadline is reached at any point, cancel immediately.

Implementation: pass the deadline as a header or context value.
gRPC does this natively with deadlines.
```

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Sliding window counter is the production standard | Two counters, weighted interpolation. Approximate but memory-efficient |
| Token bucket allows controlled bursts | Capacity = burst size. Refill rate = sustained rate. Most flexible |
| Distributed rate limiting: Redis centralized or local + periodic sync | Per-request Redis adds latency. Periodic sync trades accuracy for speed |
| 429 Too Many Requests + Retry-After header | Standard HTTP rate limit response. Clients should respect Retry-After |
| Circuit breaker: CLOSED → OPEN → HALF-OPEN | Prevents cascading failure. Fast-fail instead of waiting for timeout |
| Bulkhead: isolate resources per dependency | Thread/semaphore/connection pool isolation. Failure in one doesn't starve others |
| Exponential backoff with jitter prevents retry storms | Full jitter or decorrelated jitter. Cap the max delay. Budget total retries |
| Retry only retryable errors (5xx, 429, network) | Never retry 4xx client errors. Use retry budgets (10% cap) to prevent amplification |
| Every remote call needs a timeout | Connection timeout (short), request timeout (medium), end-to-end deadline propagation |
| Timeout propagation: subtract elapsed time at each hop | gRPC does this natively. Pass deadline in headers/context |
