# Caching — Trading Space for Speed

## The Fundamental Trade-Off

A cache stores copies of data closer to where it's needed (in faster
storage, or geographically closer). Every cache introduces the same
fundamental problems:

1. **Consistency**: The cache copy can be stale (source changed)
2. **Invalidation**: How and when to update/remove stale entries
3. **Memory pressure**: The cache has finite size — what to evict?
4. **Cold start**: Empty cache = all requests hit the source

"There are only two hard things in Computer Science: cache invalidation
and naming things." — Phil Karlton

---

## Cache Placement in a System

```
User → [Browser Cache] → [CDN / Edge Cache] → [Load Balancer]
  → [Application Cache (Redis/Memcached)] → [Database Buffer Pool]
  → [OS Page Cache] → [Disk]

Each layer is a cache for the layer below it:
  Browser cache:     avoids network request entirely
  CDN:               avoids hitting your origin servers
  Application cache: avoids hitting the database
  Buffer pool:       avoids hitting disk
  OS page cache:     avoids hitting disk (redundant with buffer pool)
```

---

## Caching Strategies

### Cache-Aside (Lazy Loading)

The application manages the cache explicitly. The most common pattern.

```
Read:
  1. Check cache for key
  2. If hit: return cached value
  3. If miss: read from database
  4. Write result to cache
  5. Return result

Write:
  1. Write to database
  2. Invalidate (delete) the cache entry
  (Do NOT update the cache — see "why invalidate, not update" below)
```

```javascript
async function getUser(userId) {
    // 1. Check cache
    const cached = await redis.get(`user:${userId}`);
    if (cached) return JSON.parse(cached);

    // 2. Cache miss — read from DB
    const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

    // 3. Populate cache (with TTL)
    await redis.set(`user:${userId}`, JSON.stringify(user), 'EX', 3600);

    return user;
}

async function updateUser(userId, data) {
    // 1. Write to database
    await db.query('UPDATE users SET name = $1 WHERE id = $2', [data.name, userId]);

    // 2. Invalidate cache (don't update!)
    await redis.del(`user:${userId}`);
}
```

**Why invalidate, not update:**

```
Race condition with update:

  Thread A: UPDATE user SET name='alice' in DB
  Thread B: UPDATE user SET name='bob' in DB
  Thread B: SET cache user = 'bob'
  Thread A: SET cache user = 'alice'     ← Thread A's stale write wins
  Cache: 'alice', DB: 'bob' → INCONSISTENT

With invalidation:
  Thread A: UPDATE DB (name='alice')
  Thread B: UPDATE DB (name='bob')
  Thread A: DELETE cache
  Thread B: DELETE cache
  Next read: cache miss → reads 'bob' from DB → cache is correct
```

Invalidation is idempotent and order-independent. Updates are not.

**Pros:** Simple. Cache only contains data that's actually been
requested (no wasteful caching of unused data).

**Cons:** Cache miss penalty (first request is slow). If many requests
arrive simultaneously for a cold key: thundering herd (see below).

### Read-Through

The cache itself is responsible for loading data on a miss. The
application only talks to the cache.

```
Read:
  1. Application asks cache for key
  2. If hit: cache returns value
  3. If miss: cache reads from database, stores it, returns to application
  (Application never talks to the database directly for reads)

The cache library/layer handles the DB query:
  cache.get("user:42")
    → cache miss
    → cache calls loadFromDB("user:42")
    → cache stores result
    → returns to caller
```

**Difference from cache-aside**: In cache-aside, the application
handles misses. In read-through, the cache handles misses. This moves
the loading logic into the cache layer, which can provide built-in
protection against thundering herd (only one loader per key).

### Write-Through

Every write goes to the cache AND the database, synchronously:

```
Write:
  1. Application writes to cache
  2. Cache synchronously writes to database
  3. Both are updated before the write is acknowledged

Read:
  Always from cache (guaranteed to be up-to-date)
```

**Pros:** Cache is always consistent with the database. No stale reads.

**Cons:** Write latency increases (must wait for both cache and DB).
Every write populates the cache, even for data that may never be read
(wasteful for write-heavy workloads).

### Write-Behind (Write-Back)

Writes go to the cache immediately. The cache asynchronously flushes
to the database later.

```
Write:
  1. Application writes to cache
  2. Cache acknowledges immediately (fast!)
  3. Cache asynchronously writes to database (batched, delayed)

Timeline:
  t=0:   Write to cache (acknowledged to client)
  t=0:   Cache: {user:42 → 'alice'}   DB: {user:42 → 'old_value'}
  t=5s:  Cache flushes batch to DB
  t=5s:  Cache: {user:42 → 'alice'}   DB: {user:42 → 'alice'}
```

**Pros:** Very fast writes (only cache latency). Database writes are
batched (fewer DB operations). Absorbs write spikes.

**Cons:** Data loss risk — if the cache crashes before flushing, writes
are lost. The cache must be durable (Redis with AOF persistence, or
replicated) for this to be safe. Complexity.

### Comparison

| Strategy | Read Perf | Write Perf | Consistency | Complexity |
|----------|-----------|------------|-------------|------------|
| Cache-Aside | Miss penalty | DB write speed | Eventual (TTL) | Low |
| Read-Through | Miss penalty (auto-loaded) | DB write speed | Eventual (TTL) | Medium |
| Write-Through | Always fast | Slower (cache + DB sync) | Strong | Medium |
| Write-Behind | Always fast | Fastest (cache only) | Weak (async) | High |

---

## Eviction Policies

When the cache is full, which entry to remove?

### LRU (Least Recently Used)

Evict the entry that hasn't been accessed for the longest time.

```
Access pattern: A, B, C, D, A, E (cache size: 4)

[A] → [A, B] → [A, B, C] → [A, B, C, D]
→ access A: [B, C, D, A]    (A moves to most recent)
→ insert E: [C, D, A, E]    (B evicted — least recently used)
```

**Implementation**: Doubly-linked list + hash map. O(1) for access,
insert, and eviction. The list maintains access order; the hash map
provides O(1) lookup.

```
Hash Map: key → pointer to list node
List:     [LRU] ←→ ... ←→ [MRU]

Get(key):
  node = hashmap[key]
  move node to tail (MRU end)
  return node.value

Put(key, value):
  if key in hashmap:
    update value, move to tail
  else:
    if full: remove head (LRU), delete from hashmap
    insert at tail, add to hashmap
```

**Redis approximate LRU**: Redis doesn't maintain a full LRU list
(too expensive for millions of keys). Instead, it samples N random
keys (default: 5, configurable via `maxmemory-samples`) and evicts
the least recently used among the sample. With 10 samples, this
approximation is nearly as good as true LRU.

### LFU (Least Frequently Used)

Evict the entry with the fewest accesses.

```
A (access count: 100), B (50), C (3), D (1)
Evict D (lowest frequency)
```

Better than LRU for: workloads where some items are consistently
popular (they won't be evicted even if not accessed in the last few
seconds).

Problem: new items have low frequency and get evicted immediately
(no time to build up count). Solution: LFU with aging — decay
frequency counts over time.

**Redis LFU**: Uses a logarithmic counter (8 bits) with a decay
factor. Counter increments probabilistically (harder to increment
at higher values). Decay happens based on elapsed time. Configurable
via `lfu-log-factor` and `lfu-decay-time`.

### Other Policies

**FIFO**: First in, first out. Simplest but ignores access patterns.

**Random**: Evict a random entry. Surprisingly effective — avoids
worst-case patterns that trip up LRU/LFU.

**TTL-based**: Every entry has an expiration time. Expired entries
are evicted on access (lazy) or by a background process (active).
Not a replacement for LRU/LFU — TTL handles staleness, LRU/LFU
handles memory pressure.

**ARC (Adaptive Replacement Cache)**: Maintains two LRU lists
(recently used once, recently used multiple times) and dynamically
adjusts the partition between them. Self-tuning — adapts to workload
without configuration. Used by ZFS. Patented by IBM.

---

## Cache Invalidation Patterns

### TTL (Time-To-Live)

Every entry expires after a fixed duration:

```
redis.set("user:42", data, "EX", 3600);  // expires in 1 hour

Pros: Simple. Bounded staleness (max 1 hour out of date).
Cons: Stale for up to TTL duration. Short TTL = more cache misses.
      Long TTL = more stale data.
```

**Choosing TTL**: Consider how often the data changes and how
tolerant the application is of stale data:

```
User profile:    TTL = 5-15 minutes (changes rarely)
Product price:   TTL = 1-5 minutes (changes sometimes, staleness has cost)
Session token:   TTL = session duration (must not outlive the session)
API rate limit:  TTL = window duration (1 second, 1 minute)
Static assets:   TTL = 1 year (versioned URL, never changes)
```

### Event-Based Invalidation

Delete or update the cache when the source data changes:

```
Database trigger/CDC → message queue → cache invalidation

1. Application writes to PostgreSQL
2. PostgreSQL WAL (via Debezium/CDC) emits change event
3. Change event consumed by invalidation service
4. Invalidation service deletes the cache key

Or simpler: application code invalidates after write
  await db.query('UPDATE users SET ...');
  await redis.del('user:42');
```

More precise than TTL but requires reliable delivery of invalidation
events. If an invalidation event is lost, the cache stays stale
indefinitely (unless TTL provides a safety net).

**Best practice**: Use event-based invalidation + TTL as a safety net.

### Versioned Keys

Include a version in the cache key. When data changes, increment the
version:

```
Key: "user:42:v3" → data
After update: increment version to 4
New reads: cache miss on "user:42:v4" → load from DB

Old keys ("user:42:v3") expire via TTL.
No explicit invalidation needed — the old key is simply abandoned.
```

---

## Cache Problems

### Thundering Herd (Cache Stampede)

A popular cache key expires. Thousands of simultaneous requests see
a cache miss and all hit the database simultaneously:

```
t=0:    Cache key "popular:item" expires
t=0.001: 1000 concurrent requests check cache → all miss
t=0.002: 1000 requests simultaneously query the database
         → Database overloaded or crashes
```

**Solutions:**

**Lock/Semaphore (single-flight):**
```
1. First request to miss acquires a lock (redis SETNX)
2. First request loads from DB and populates cache
3. Other requests wait for the lock (or return stale data)
4. Lock released — all subsequent requests hit cache

await redis.set("lock:popular:item", "1", "NX", "EX", 5);
// If acquired: load from DB, set cache, release lock
// If not acquired: wait and retry (or return stale)
```

**Stale-While-Revalidate:**
```
Store data with both a TTL and a "soft" expiry:
  { value: data, expires: now + 1h, stale_until: now + 1h10m }

On read:
  If not expired: return immediately
  If expired but within stale_until:
    Return stale data to the client
    Trigger background refresh (only one — use lock)
  If past stale_until: cache miss (treat as above)
```

**Preemptive refresh:**
```
Background job refreshes popular keys BEFORE they expire.
If TTL = 1 hour, refresh at 50 minutes.
The key never expires → no stampede.
Requires knowing which keys are "popular" (track access counts).
```

### Cache Penetration

Requests for keys that don't exist in the cache OR the database.
Every request goes to the database (cache can never satisfy it):

```
Attacker: GET /user/99999999 (doesn't exist)
  Cache: miss → DB: miss → return 404

Millions of these requests bypass the cache entirely and hit the DB.
```

**Solutions:**

**Cache negative results:**
```
If DB returns not found: cache the null result with a short TTL
redis.set("user:99999999", "NULL", "EX", 60);

Next request: cache hit → return 404 without touching DB
```

**Bloom filter:**
```
Maintain a bloom filter of all existing keys.
Before checking cache/DB: check bloom filter.
  "Definitely not in DB" → return 404 immediately (no cache/DB hit)
  "Possibly in DB" → proceed with cache/DB lookup
```

### Cache Avalanche

Many cache keys expire simultaneously (e.g., all populated at the
same time with the same TTL). Sudden spike of cache misses:

```
t=0:    Bulk load 100K items with TTL=3600s
t=3600: All 100K items expire simultaneously
        → 100K cache misses → 100K DB queries → DB overwhelmed
```

**Solutions:**

**Jittered TTL:**
```
// Instead of fixed TTL:
redis.set(key, value, 'EX', 3600);

// Add random jitter:
const ttl = 3600 + Math.floor(Math.random() * 600); // 3600-4200s
redis.set(key, value, 'EX', ttl);

// Expirations spread over 10 minutes instead of all at once
```

**Staggered warm-up:**
```
After a cold start (deploy, cache flush), don't let all traffic
hit the DB. Rate-limit cache misses:

const semaphore = new Semaphore(20); // max 20 concurrent DB loads
async function getFromCacheOrDB(key) {
    const cached = await redis.get(key);
    if (cached) return cached;
    await semaphore.acquire();
    try {
        // Double-check after acquiring (another thread may have loaded it)
        const cached2 = await redis.get(key);
        if (cached2) return cached2;
        const data = await db.query(...);
        await redis.set(key, data, 'EX', ttl);
        return data;
    } finally {
        semaphore.release();
    }
}
```

### Hot Key Problem

A single cache key receives disproportionate traffic (celebrity tweet,
viral product). Even Redis (100K ops/sec single thread) can be
overwhelmed by one hot key:

```
"trending:product:123" → 500K reads/sec → single Redis shard overloaded
```

**Solutions:**

**Local cache**: Cache hot keys in application memory (in-process).
No network round trip. Short TTL (5-10s) to limit staleness.

```javascript
const localCache = new Map();
async function getHotKey(key) {
    if (localCache.has(key)) {
        const entry = localCache.get(key);
        if (Date.now() < entry.expires) return entry.value;
    }
    const value = await redis.get(key);
    localCache.set(key, { value, expires: Date.now() + 5000 });
    return value;
}
```

**Key replication**: Store the same value under multiple keys
distributed across shards:

```
Instead of: "product:123" (one shard)
Use:        "product:123:0", "product:123:1", ..., "product:123:7"
Read:       pick random suffix → distributed across 8 shards
Write:      update all 8 keys
```

---

## Redis Internals

### Architecture

Redis is single-threaded for command execution (no locks needed for
data structures). Network I/O uses epoll with optional I/O threads
(Redis 6+).

```
Client connections (thousands)
  │
  ▼
Event loop (single thread, epoll):
  1. Read commands from all ready connections
  2. Execute commands sequentially (single-threaded)
  3. Write responses to connections

Single-threaded execution means:
  - Every command is atomic (no partial execution)
  - No locks on data structures
  - O(n) commands (KEYS *, SMEMBERS on large set) block EVERYTHING
  - Throughput: ~100K-200K ops/sec (pipelining can reach 1M+)
```

**I/O threads** (Redis 6+): Offload socket read/write to multiple
threads. Command execution remains single-threaded. Improves throughput
for large payloads (the bottleneck shifts from network I/O to execution).

### Data Structures

Redis values are not just strings — they use specialized internal
data structures based on size:

```
Type         Small encoding            Large encoding
──────────────────────────────────────────────────────────────
String       int (if numeric)          SDS (Simple Dynamic String)
             embstr (≤44 bytes)        raw SDS (>44 bytes)

List         listpack (≤128 items      quicklist (linked list
             AND each ≤64 bytes)       of listpack nodes)

Hash         listpack (≤128 fields     hashtable
             AND each ≤64 bytes)

Set          listpack (≤128 items      hashtable
             AND all integers)         (or intset for small int sets)

Sorted Set   listpack (≤128 items)     skiplist + hashtable
                                       (skiplist for range queries,
                                        hashtable for O(1) lookup)
```

**SDS (Simple Dynamic String)**: Redis strings are not C strings.
SDS stores the length and available space in a header, making strlen
O(1) and append O(1) amortized. Binary-safe (can contain \0).

**Skiplist** (sorted sets): A probabilistic data structure with O(log n)
search, insert, and delete. Each node has random "levels" — higher
levels are express lanes that skip over lower-level nodes:

```
Level 3:  Head ──────────────────────────────── 50 ─── NIL
Level 2:  Head ──────── 20 ──────────────────── 50 ─── NIL
Level 1:  Head ── 10 ── 20 ── 30 ── 40 ── 50 ── 60 ── NIL

Search for 40:
  Start at Level 3: Head → 50 (overshoot) → drop to Level 2
  Level 2: Head → 20 → 50 (overshoot) → drop to Level 1
  Level 1: 20 → 30 → 40 (found!)

Average O(log n) — similar to balanced BST but simpler to implement
and naturally supports range queries (walk forward from found node).
```

### Persistence

**RDB (snapshotting)**: Fork the process, child writes entire dataset
to a point-in-time snapshot file. Copy-on-write semantics mean the
fork is fast even for large datasets.

```
Pros: Compact file. Fast restart from RDB.
Cons: Data loss between snapshots. Fork can cause memory spike
      (CoW pages modified during snapshot = doubled memory worst case).
```

**AOF (Append-Only File)**: Log every write command to a file.
Replay on restart.

```
Configurable fsync:
  always:     fsync after every command (slow, durable)
  everysec:   fsync once per second (default — max 1s of data loss)
  no:         let OS decide when to fsync (fastest, up to 30s loss)

AOF rewriting: background process compacts the AOF (e.g., 1000
INCRs on a counter → single SET with final value). Similar to
LSM compaction.
```

**Best practice**: Use AOF (everysec) for durability + RDB for fast
backup/restart. Redis 7+ uses multi-part AOF for more efficient
rewriting.

### Redis Cluster

Distributes data across multiple nodes using hash slots:

```
16,384 hash slots distributed across N nodes:

Node A: slots 0-5460
Node B: slots 5461-10922
Node C: slots 10923-16383

Key assignment: CRC16(key) % 16384 → slot number → node

"user:42": CRC16("user:42") % 16384 = 9231 → Node B
```

**Hash tags** allow related keys to land on the same node:

```
Keys "user:{42}:profile" and "user:{42}:orders"
Both hash on "{42}" → same slot → same node
Enables multi-key operations (MGET, transactions) on related data.
```

---

## CDN — Edge Caching

A Content Delivery Network caches content at edge servers
geographically close to users.

### How CDNs Work

```
User in Tokyo → CDN edge in Tokyo → origin server in US-East

First request (cache miss):
  User → Tokyo edge (miss) → US-East origin (200 OK, Cache-Control: max-age=3600)
  Tokyo edge caches response
  User gets response (high latency, ~150ms RTT to origin)

Subsequent requests (cache hit):
  User → Tokyo edge (hit) → immediate response
  User gets response (low latency, ~5ms to edge)
```

### Cache-Control Headers

The origin controls CDN caching behavior via HTTP headers:

```
Cache-Control: public, max-age=31536000
  Public: CDN can cache (not just browser)
  max-age: cache for 1 year (immutable content, versioned URL)

Cache-Control: private, no-store
  Private: only browser can cache, not CDN
  no-store: don't cache at all (sensitive data)

Cache-Control: public, max-age=0, must-revalidate
  CDN caches but must check with origin every time (ETag/304)

Cache-Control: public, s-maxage=60, max-age=3600
  s-maxage: CDN caches for 60s (overrides max-age for shared caches)
  max-age: browser caches for 1 hour
```

### Cache Key

The CDN uses a **cache key** to identify cached responses. Default:
URL (method + host + path + query string). Can be customized:

```
Default key: GET https://api.example.com/users?page=2

Vary header: origin adds "Vary: Accept-Encoding"
  Key becomes: GET https://api.example.com/users?page=2 + Accept-Encoding: gzip
  Same URL but different encoding → different cache entries

Custom keys (Cloudflare, Fastly):
  Include: specific cookies, headers, device type
  Exclude: tracking query parameters (?utm_source=...)
```

### Purge and Invalidation

When content changes, you need to invalidate the CDN cache:

```
Purge by URL:
  POST /purge https://cdn.example.com/api/purge
  Body: { "url": "https://example.com/product/123" }

Purge by tag (surrogate keys):
  Origin response: Surrogate-Key: product-123 category-electronics
  Purge: invalidate all responses tagged "product-123"
  Much more efficient than purging individual URLs.

Purge everything:
  Nuclear option. Entire CDN cache cleared. Cold start.
  Only for emergencies.

Best practice: version URLs for static assets.
  /static/app.abc123.js (content hash in filename)
  Cache forever. New deploy = new URL. No purging needed.
```

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Cache-aside is the default pattern | App manages cache. Invalidate on write (don't update — race condition) |
| Write-behind absorbs write spikes but risks data loss | Cache writes async to DB. Must be durable (AOF, replication) |
| LRU: doubly-linked list + hash map, O(1) all operations | Redis uses approximate LRU (sample N keys). LFU for frequency-based workloads |
| TTL provides bounded staleness, event invalidation provides precision | Use both: event invalidation + TTL as safety net |
| Thundering herd: lock/semaphore (single-flight) or stale-while-revalidate | Only one request loads on miss. Others wait or get stale data |
| Cache penetration: cache negative results + bloom filter | Prevents DB hit for keys that will never exist |
| Cache avalanche: jittered TTLs prevent simultaneous expiration | Random TTL spread eliminates coordinated misses |
| Hot key: local in-process cache (5-10s TTL) or key replication across shards | Local cache eliminates network round trip for hottest keys |
| Redis is single-threaded for execution | Atomic commands, no locks. O(n) commands block everything |
| Redis sorted sets use skiplist internally | O(log n) with natural range query support |
| Redis Cluster: 16,384 hash slots, CRC16(key) % 16384 | Hash tags {tag} colocate related keys |
| CDN: Cache-Control headers control caching. s-maxage overrides for CDN | Versioned URLs (content hash) eliminate purging |
