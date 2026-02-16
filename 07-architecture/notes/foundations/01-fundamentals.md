# System Design Fundamentals — The Numbers and the Math

## Latency Numbers Every Engineer Must Know

These numbers are approximate but stable across hardware generations
(ratios stay roughly the same). Memorize the order of magnitude.

```
Operation                           Latency         Notes
─────────────────────────────────────────────────────────────────────
L1 cache reference                  ~1 ns           On-die. Per-core.
L2 cache reference                  ~4 ns           On-die. Per-core.
L3 cache reference                  ~10 ns          Shared across cores.
Branch mispredict                   ~3 ns           Pipeline flush.
Mutex lock/unlock                   ~17 ns          Uncontended.
Main memory reference               ~100 ns         DRAM access.
Compress 1KB with Snappy            ~3 μs           ~300 MB/s.
Read 1 MB sequentially from RAM     ~3 μs           ~10 GB/s bandwidth.
SSD random read (4 KB)              ~100 μs         NVMe: ~20 μs.
Read 1 MB sequentially from SSD     ~50 μs          NVMe: ~250 μs. ~3 GB/s.
HDD random read (4 KB)              ~10 ms          Physical head movement.
Read 1 MB sequentially from HDD     ~2 ms           ~200 MB/s.
Round trip same datacenter          ~500 μs         Within a rack: ~100 μs.
Round trip same region (AZ to AZ)   ~1-2 ms         Fiber between buildings.
TLS handshake                       ~5-10 ms        2 round trips (TLS 1.2).
Round trip US coast to coast        ~30-40 ms       ~4800 km of fiber.
Round trip US to Europe             ~70-90 ms       Transatlantic cable.
Round trip US to Asia               ~100-150 ms     Transpacific cable.
TCP handshake                       1 RTT           SYN, SYN-ACK, ACK.
```

### The Key Ratios

```
Memory vs SSD random:        100 ns vs 100 μs     → 1,000x
SSD random vs HDD random:    100 μs vs 10 ms      → 100x
Memory vs HDD random:        100 ns vs 10 ms      → 100,000x
Same-DC round trip vs memory: 500 μs vs 100 ns    → 5,000x
Cross-continent vs same-DC:   80 ms vs 500 μs     → 160x
```

**Design implications:**
- A single network round trip costs as much as 5,000 memory accesses.
  Batch operations. Avoid chatty protocols.
- SSD random read is 1,000x slower than memory. This is why buffer
  pools and caches exist.
- Cross-continent latency is 160x same-datacenter. This is why CDNs
  and regional deployments exist.
- Sequential I/O is dramatically faster than random on all media.
  This is why WAL (sequential) exists alongside random data page I/O.

---

## Back-of-Envelope Estimation

The ability to quickly estimate system capacity is essential for
architecture decisions. You don't need exact numbers — you need
to know whether you need 1 server or 100.

### The Framework

```
1. Clarify the requirements (reads/writes, data size, latency)
2. Estimate the scale (users, requests/sec, storage)
3. Calculate resource needs (CPU, memory, disk, bandwidth)
4. Identify bottlenecks (what runs out first?)
5. Design around the bottleneck
```

### Worked Example: Twitter-like Timeline Service

**Requirements:**
- 500 million users, 200 million daily active
- Average user reads timeline 10 times/day
- Average user posts 2 tweets/day
- Average tweet: 280 chars (~300 bytes) + metadata (~200 bytes) = ~500 bytes
- Timeline shows latest 200 tweets from followed accounts
- Average user follows 200 accounts

**Read load:**
```
Timeline reads:
  200M DAU × 10 reads/day = 2 billion reads/day
  2B / 86,400 sec = ~23,000 reads/sec average
  Peak (3x average): ~70,000 reads/sec
```

**Write load:**
```
Tweet posts:
  200M DAU × 2 tweets/day = 400 million tweets/day
  400M / 86,400 sec = ~4,600 writes/sec average
  Peak: ~14,000 writes/sec
```

**Storage:**
```
Per day: 400M tweets × 500 bytes = 200 GB/day
Per year: 200 GB × 365 = ~73 TB/year
5-year retention: ~365 TB raw data
With indexes and replicas (3x): ~1 PB
```

**Timeline computation:**
```
Approach 1 — Fan-out on read (pull):
  When user opens timeline:
    1. Get list of 200 followed accounts
    2. Query latest tweets from each (or batch query)
    3. Merge-sort, return top 200
  Cost: ~200 point lookups per timeline read
  At 70K reads/sec: 14 million DB queries/sec. Too expensive.

Approach 2 — Fan-out on write (push):
  When user posts a tweet:
    1. Look up all followers
    2. Write the tweet to each follower's pre-computed timeline cache
  Average followers per user: ~200 (median is much lower)
  At 14K writes/sec: 14K × 200 = 2.8 million cache writes/sec
  Timeline read: single cache lookup. Fast.

  Problem: celebrities. A user with 50 million followers:
    1 tweet = 50 million cache writes. Takes minutes. Stale timelines.

Approach 3 — Hybrid (what Twitter actually does):
  Regular users (< 10K followers): fan-out on write
  Celebrities (> 10K followers): fan-out on read (merge at read time)
  Timeline = pre-computed cache + real-time merge of celebrity tweets
```

**Memory for timeline cache:**
```
Pre-computed timelines:
  200M active users × 200 tweets × 8 bytes (tweet ID) = 320 GB
  Fits in a distributed Redis cluster (~10 nodes with 32 GB each)
```

### Powers of Two (Quick Reference)

```
2^10 = 1,024                    ~1 thousand (1 KB)
2^20 = 1,048,576                ~1 million (1 MB)
2^30 = 1,073,741,824            ~1 billion (1 GB)
2^40 = 1,099,511,627,776        ~1 trillion (1 TB)
2^50                            ~1 quadrillion (1 PB)
```

### Throughput Rules of Thumb

```
Single server:
  Web server (nginx):            ~50,000 req/sec (static)
  Application server (Node.js):  ~10,000 req/sec (simple JSON API)
  Application server (Go):       ~30,000 req/sec (simple JSON API)
  PostgreSQL:                    ~10,000 simple queries/sec per core
  Redis:                         ~100,000 ops/sec (single thread)
  Kafka broker:                  ~200,000 messages/sec (per partition)

Network:
  1 Gbps link:                   ~125 MB/s, ~100K small packets/sec
  10 Gbps link:                  ~1.25 GB/s
  25 Gbps link (modern DC):      ~3.1 GB/s
```

---

## Availability — The Math of Uptime

### SLA, SLO, SLI

**SLI (Service Level Indicator)**: A measurable metric of service
behavior. Examples:
- Request latency (p50, p95, p99)
- Error rate (percentage of 5xx responses)
- Availability (percentage of successful requests)
- Throughput (requests per second)

**SLO (Service Level Objective)**: A target value for an SLI.
Internal commitment.
- "p99 latency < 200ms"
- "Error rate < 0.1%"
- "Availability > 99.9%"

**SLA (Service Level Agreement)**: A contract with consequences
(credits, refunds) if the SLO is not met. External commitment.
SLAs are always looser than SLOs — you want a buffer.

### Nines of Availability

```
Availability    Downtime/year    Downtime/month   Downtime/week
──────────────────────────────────────────────────────────────────
99%     (two 9s)  3.65 days        7.31 hours       1.68 hours
99.9%   (three 9s) 8.77 hours      43.8 minutes     10.1 minutes
99.95%            4.38 hours       21.9 minutes     5.04 minutes
99.99%  (four 9s)  52.6 minutes    4.38 minutes     1.01 minutes
99.999% (five 9s)  5.26 minutes    26.3 seconds     6.05 seconds
```

**What the nines mean in practice:**
- **99%**: Acceptable for internal tools. ~7 hours downtime per month.
- **99.9%**: Standard for most web applications. ~44 min downtime/month.
  Achievable with a single good deployment. This is what most services
  actually deliver.
- **99.99%**: Hard. Requires redundancy at every layer, automated
  failover, canary deployments. ~4.4 min downtime/month.
- **99.999%**: Extremely hard. Requires active-active multi-region,
  zero-downtime deploys, and the reality is that most "five nines"
  claims exclude planned maintenance.

### Serial vs Parallel Availability

**Serial** (components in sequence — all must work):

```
System = Service A → Service B → Service C

Availability = A(A) × A(B) × A(C)
             = 0.999 × 0.999 × 0.999
             = 0.997 (99.7%)

Each additional serial dependency REDUCES availability.
10 services at 99.9% each: 0.999^10 = 99.0%
```

This is why microservices are dangerous for availability — a request
touching 10 services has lower availability than a monolith, even if
each service is individually reliable.

**Parallel** (redundant components — any one is sufficient):

```
System = Service A || Service A (backup)

Availability = 1 - (1 - A(A))^N  where N = number of replicas
             = 1 - (1 - 0.999)^2
             = 1 - 0.001^2
             = 1 - 0.000001
             = 0.999999 (99.9999%)

Two replicas of a 99.9% service give 99.9999%.
```

**Real architecture** (serial path of parallel groups):

```
[LB₁ || LB₂] → [App₁ || App₂ || App₃] → [DB_primary || DB_replica]

LB availability:    1 - (0.001)^2 = 0.999999
App availability:   1 - (0.001)^3 = 0.999999999
DB availability:    1 - (0.001)^2 = 0.999999

Total = 0.999999 × 0.999999999 × 0.999999 ≈ 0.999998 (99.9998%)
```

### Error Budgets

If your SLO is 99.9% (43.8 minutes downtime/month), you have an
**error budget** of 43.8 minutes. Use it intentionally:

```
Monthly error budget: 43.8 minutes

Spend it on:
  Deployments (2 min each × 10 deploys):   20 min
  Incidents:                                 15 min
  Remaining:                                 8.8 min (buffer)

If you've burned your budget:
  Freeze deployments. Focus on reliability.
  No new features until the error budget recovers.
```

This framework (from Google's SRE book) makes reliability a
quantitative decision, not a cultural one. Teams that ship too
fast and cause outages literally run out of budget and must stop.

---

## Capacity Planning

### Identifying Bottlenecks

Every system has a bottleneck — the resource that runs out first:

```
Resource        What Saturates It                How to Measure
──────────────────────────────────────────────────────────────────
CPU             Computation-heavy tasks          top, vmstat
                (crypto, compression, JSON        CPU utilization %
                parsing, regex)

Memory          Working set doesn't fit          free -h, /proc/meminfo
                (buffer pool, caches, session    Swap usage
                state, connection overhead)

Disk I/O        Random reads (database pages)    iostat, iotop
                or write throughput (WAL,         IOPS, throughput, queue depth
                compaction)

Network         High bandwidth (video, large     iftop, /proc/net/dev
                payloads) or high packet rate    Bandwidth, packets/sec
                (many small requests)

Connections     File descriptors, TCP states,    ss, /proc/sys/fs/file-nr
                database connections             Connection count, TIME_WAIT
```

### Horizontal vs Vertical Scaling

**Vertical scaling** (scale up): Bigger machine.
```
Pros:
  - Simple. No distributed systems complexity.
  - Single machine = no network partitions, no distributed transactions.
  - Modern machines are powerful: 128 cores, 2 TB RAM, NVMe RAID.

Cons:
  - Hardware limits. You can't buy a machine with 1 PB of RAM.
  - Single point of failure (unless replicated).
  - Non-linear cost. 2x CPU ≠ 2x price (often 3-5x).
  - Downtime for upgrades (unless live migration is possible).
```

**Horizontal scaling** (scale out): More machines.
```
Pros:
  - Theoretically unlimited capacity.
  - Redundancy built in (machines fail independently).
  - Cost-efficient at scale (commodity hardware).

Cons:
  - Distributed systems complexity (consistency, partitioning, coordination).
  - Network overhead between nodes.
  - Not all workloads parallelize (Amdahl's Law).
  - Operational complexity (deployment, monitoring, debugging).
```

**The right answer**: Scale vertically until you can't, then
horizontally. A single large PostgreSQL instance handles most
applications. Don't distribute prematurely.

### Amdahl's Law

The speedup from parallelization is limited by the serial portion
of the workload:

```
Speedup = 1 / (S + P/N)

Where:
  S = fraction of work that's serial (can't be parallelized)
  P = fraction of work that's parallel (= 1 - S)
  N = number of parallel workers

Example: 95% of work is parallelizable, 5% is serial
  Speedup with 10 workers:  1 / (0.05 + 0.95/10) = 1 / 0.145 = 6.9x
  Speedup with 100 workers: 1 / (0.05 + 0.95/100) = 1 / 0.0595 = 16.8x
  Speedup with ∞ workers:   1 / 0.05 = 20x (theoretical maximum)

Even with infinite parallelism, 5% serial work limits you to 20x speedup.
```

This applies to distributed systems: if every request requires a
single serial database write, adding more application servers won't
help once the database is the bottleneck.

### Little's Law

The fundamental relationship between throughput, latency, and
concurrency:

```
L = λ × W

Where:
  L = average number of concurrent requests in the system
  λ = average arrival rate (requests/second)
  W = average time each request spends in the system (seconds)

Example:
  λ = 1,000 requests/sec
  W = 200 ms = 0.2 sec
  L = 1000 × 0.2 = 200 concurrent requests

  You need at least 200 concurrent connections/threads to handle
  this load without queuing.

Capacity planning:
  If each server handles 50 concurrent requests:
  Servers needed = 200 / 50 = 4 servers (minimum, add headroom)
```

### Universal Scalability Law (USL)

Extends Amdahl's Law with **coherence overhead** — the cost of
keeping parallel workers synchronized:

```
Throughput(N) = N / (1 + σ(N-1) + κN(N-1))

Where:
  N = number of nodes/workers
  σ = contention parameter (serial fraction, like Amdahl's S)
  κ = coherence parameter (coordination overhead)

Contention (σ): waiting for shared resources (locks, serial writes)
  Throughput plateaus as N increases.

Coherence (κ): keeping state synchronized (cache invalidation,
  distributed consensus, lock coordination)
  Throughput DECREASES past a peak — adding nodes makes things WORSE.
```

```
Throughput vs Nodes:

  │            Peak (diminishing returns)
  │           ╱╲
  │         ╱    ╲  ← coherence overhead dominates
  │       ╱        ╲   (more nodes = more coordination = slower)
  │     ╱
  │   ╱  ← near-linear scaling
  │ ╱
  │╱
  └──────────────────── Nodes (N)

  σ only (Amdahl):  throughput plateaus
  σ + κ (USL):      throughput peaks then DROPS
```

This is why adding more nodes to a system with heavy coordination
(distributed transactions, distributed locks, cache invalidation)
can make it slower. The coherence penalty grows quadratically with N.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Memory is 1,000x faster than SSD, SSD is 100x faster than HDD | Design implications: buffer pools, caches, sequential I/O over random |
| Cross-continent RTT is 160x same-DC | CDNs, regional deployments, minimize cross-region calls |
| Back-of-envelope: estimate before you design | Requests/sec, storage, bandwidth. Know if you need 1 server or 100 |
| Fan-out on write vs read: hybrid is the answer | Push for regular users, pull for celebrities. The Twitter pattern |
| 99.9% = 43.8 min downtime/month | Each nine is 10x harder. Five nines requires multi-region active-active |
| Serial availability: multiply. Parallel: 1-(1-A)^N | 10 services at 99.9% each = 99.0% end-to-end. Microservices reduce availability |
| Error budgets make reliability quantitative | SLO defines the budget. Burn it → freeze deploys |
| Amdahl's Law: serial fraction limits speedup | 5% serial = max 20x speedup regardless of parallelism |
| Little's Law: L = λ × W | Concurrency = throughput × latency. Use for capacity planning |
| USL: coherence overhead can make more nodes WORSE | Coordination cost grows quadratically. Don't add nodes to a coordination-bound system |
| Scale vertically first, horizontally when forced | A big PostgreSQL instance handles most applications. Don't distribute prematurely |
