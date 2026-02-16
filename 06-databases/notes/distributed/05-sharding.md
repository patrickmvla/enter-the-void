# Sharding — Splitting Data Across Machines

## Why Shard

Replication gives you copies of the same data on multiple machines.
It scales reads (distribute queries across replicas) but not writes
(all writes go to one leader) or storage (each replica holds the full
dataset).

When the data doesn't fit on one machine, or a single node can't handle
the write throughput, you split the data across machines. Each machine
holds a **shard** (partition) — a subset of the total data.

```
Single node:
  [All data: 10 TB, 50K writes/sec]  ← disk full, CPU maxed

Sharded (4 shards):
  Shard 1: [A-F data: 2.5 TB, 12.5K writes/sec]
  Shard 2: [G-L data: 2.5 TB, 12.5K writes/sec]
  Shard 3: [M-R data: 2.5 TB, 12.5K writes/sec]
  Shard 4: [S-Z data: 2.5 TB, 12.5K writes/sec]
```

Each shard is an independent database (its own storage, its own
indexes, its own write throughput). Sharding provides horizontal
scaling — add more shards for more capacity.

---

## Partitioning Strategies

The core question: given a row, which shard does it go to?

### Range Partitioning

Assign contiguous ranges of the partition key to each shard:

```
Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000
```

**Advantages:**
- Range queries are efficient (all data for a range is on one shard or
  contiguous shards)
- Easy to understand and implement

**Disadvantages:**
- **Hot spots**: If user_id is auto-incrementing, all new writes go to
  the last shard. One shard is overwhelmed while others are idle.
- Uneven distribution: If users 1-1M are very active and users 2M-3M
  are dormant, shard 1 is overloaded.

**When to use**: Time-series data (partition by time range — recent data
is hot, old data is cold, which aligns with access patterns). Data with
natural ranges that correspond to access patterns.

### Hash Partitioning

Apply a hash function to the partition key and assign hash ranges to
shards:

```
shard_number = hash(partition_key) % num_shards

hash(user_id=42) % 4 = 2 → Shard 2
hash(user_id=43) % 4 = 0 → Shard 0
hash(user_id=44) % 4 = 3 → Shard 3
```

**Advantages:**
- Even distribution regardless of key patterns (hash destroys ordering)
- No hot spots from sequential keys

**Disadvantages:**
- Range queries become scatter-gather (data for user_id 1-1000 is spread
  across all shards)
- Adding or removing shards requires rehashing (see Consistent Hashing)

**When to use**: Most OLTP workloads where point lookups dominate and
range scans are rare.

### Consistent Hashing

The `hash(key) % num_shards` approach has a problem: when you add or
remove a shard, almost every key maps to a different shard. Resharding
requires moving ~(N-1)/N of the data (adding one shard to 4 moves ~75%
of keys).

Consistent hashing minimizes data movement:

```
Hash ring (0 to 2³²):

            Shard A (pos: 1000)
           /
    ──────●──────────────────●─── Shard B (pos: 5000)
   │                              │
   │          Hash Ring            │
   │                              │
    ──────●──────────────────●───
           \
            Shard C (pos: 8000)   Shard D (pos: 3000)

Key assignment: hash the key, walk clockwise to the first shard.
  hash("user:42") = 2500 → clockwise → Shard D (pos: 3000)
  hash("user:43") = 6000 → clockwise → Shard C (pos: 8000)

Adding Shard E at position 4000:
  Only keys between 3000 and 4000 move (from Shard B to Shard E)
  All other keys stay put. Movement: ~1/N of the data.
```

**Virtual nodes**: Each physical shard is placed at multiple positions
on the ring (e.g., 150 virtual nodes per shard). This improves balance
— without virtual nodes, uneven spacing between shard positions creates
uneven load. With many virtual nodes, the distribution approaches
uniform.

Used by: DynamoDB, Cassandra, Riak, many caching systems (memcached,
Redis Cluster).

### Compound Partition Keys

Combine hash partitioning for distribution with range ordering within
each partition:

```sql
-- Cassandra-style compound key:
PRIMARY KEY ((user_id), created_at)
               ↑              ↑
          partition key    clustering key
          (hash → shard)   (sorted within partition)
```

All rows for `user_id=42` are on the same shard (hash of user_id
determines the shard). Within that shard, rows are sorted by
`created_at`. This enables efficient range queries within a partition:

```sql
-- Efficient: single shard, sorted range scan
SELECT * FROM posts WHERE user_id = 42 AND created_at > '2024-01-01';

-- Expensive: scatter-gather across all shards
SELECT * FROM posts WHERE created_at > '2024-01-01';
```

The partition key determines **which shard** (distribution). The
clustering key determines **order within the shard** (range queries).
This pattern is fundamental to Cassandra's data model and applicable
to any sharded system.

---

## Routing — Finding the Right Shard

Three approaches to routing queries to the correct shard:

### Client-Side Routing

The client knows the partitioning scheme and connects directly to the
correct shard:

```
Client: hash("user:42") % 4 = 2 → connect to Shard 2

Client library maintains:
  - Partition map (which key ranges → which shards)
  - Shard addresses
  - Handles rebalancing updates
```

Used by: Redis Cluster (client-side), Cassandra (client driver).
Pros: No extra hop. Cons: Client complexity, every client must be
updated when topology changes.

### Proxy-Based Routing

A proxy layer sits between clients and shards:

```
Client → Proxy → determines shard → routes to Shard N

Client is shard-unaware. Proxy handles:
  - Key-to-shard mapping
  - Connection pooling
  - Load balancing
  - Topology changes
```

Used by: Vitess (MySQL), Citus (PostgreSQL), Twemproxy (Redis).
Pros: Clients are simple, centralized routing logic. Cons: Extra
network hop, proxy is a potential bottleneck.

### Coordinator Node

Any node can accept the query and route it:

```
Client → any node → that node determines the correct shard → forwards
                                                              or proxies
```

Used by: Cassandra (coordinator node), CockroachDB. The contacted node
acts as a proxy for that specific request.

---

## Cross-Shard Operations

The hard problem. Once data is split across shards, operations that
span shards become complex and expensive.

### Cross-Shard Queries (Scatter-Gather)

```sql
SELECT COUNT(*) FROM orders WHERE status = 'pending';
```

If orders are sharded by `order_id`, every shard has some pending orders.
The query must:
1. Send the query to ALL shards
2. Each shard returns its local count
3. Aggregate the results

Latency = the slowest shard. One slow shard blocks the entire query.

### Cross-Shard Joins

```sql
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.city = 'NYC';
```

If `users` is sharded by `user_id` and `orders` is sharded by
`order_id`, the join requires data from different shards. Options:

- **Broadcast join**: Send the filter condition to all user shards,
  collect matching user IDs, then query all order shards for those
  user IDs. Expensive.
- **Co-located sharding**: Shard both tables by `user_id`. Then
  `users` and `orders` for the same user are on the same shard, and
  the join is local. This is the preferred approach — choose partition
  keys so related data is co-located.

### Distributed Transactions (2PC)

When a write spans multiple shards, all shards must agree to commit or
abort. **Two-Phase Commit (2PC)** coordinates this:

```
Phase 1 (Prepare):
  Coordinator → Shard A: "Prepare to commit txn 123"
  Coordinator → Shard B: "Prepare to commit txn 123"
  Shard A → Coordinator: "Prepared" (data locked, WAL written)
  Shard B → Coordinator: "Prepared"

Phase 2 (Commit):
  Coordinator → Shard A: "Commit txn 123"
  Coordinator → Shard B: "Commit txn 123"
  Shard A → Coordinator: "Committed"
  Shard B → Coordinator: "Committed"
```

**The blocking problem**: If the coordinator crashes after Phase 1 but
before Phase 2, the shards are stuck — they've prepared (data is locked)
but don't know whether to commit or abort. They must wait for the
coordinator to recover. This is why 2PC is called a **blocking protocol**.

**Three-phase commit (3PC)** adds a "pre-commit" phase to reduce
blocking, but it's more complex and still not partition-tolerant.

**In practice**: Most applications avoid distributed transactions.
Instead:
- Design the partition key so transactions stay within one shard
- Use **sagas** (a sequence of local transactions with compensating
  actions) for cross-shard workflows
- Accept eventual consistency for some operations

---

## Resharding

When you need to add or remove shards (data growth, load changes).

### The Naive Approach

```
Old: 4 shards, hash(key) % 4
New: 5 shards, hash(key) % 5

Problem: ~80% of keys change shards. Massive data movement.
```

### Online Resharding

Move data while the system continues serving requests:

```
1. Create new shard
2. Start copying data from existing shards to the new shard
   (background, while serving reads/writes from old shards)
3. Apply writes that happened during the copy (catch-up phase)
4. Atomically switch routing to include the new shard
5. Clean up old copies
```

This is a **double-write** or **ghost table** approach. The system
temporarily writes to both old and new locations, then cuts over.
Tools like Vitess, gh-ost, and pt-online-schema-change automate this
for MySQL. CockroachDB and TiDB handle resharding automatically as
ranges split and move.

### Auto-Sharding

Some databases handle partitioning automatically:

- **CockroachDB/TiDB**: Data is stored in ranges. When a range exceeds
  a size threshold, it splits automatically. Ranges are moved between
  nodes for balance.
- **DynamoDB**: Partitions automatically based on throughput and storage.
  You specify the partition key; AWS manages the physical distribution.
- **MongoDB**: Configurable sharding with automatic chunk migration
  between shards.

---

## Shard Key Selection

The most important design decision. A bad shard key creates hot spots,
makes queries expensive, and is difficult to change.

### Good Shard Keys

| Pattern | Shard Key | Why |
|---------|-----------|-----|
| Multi-tenant SaaS | tenant_id | All of a tenant's data co-located. Queries are shard-local |
| Social media | user_id | User's posts, followers, activity on one shard |
| E-commerce | order_id (hash) | Evenly distributed. Order details co-located |
| Time-series | device_id + time bucket | Each device's data co-located. Time range for aging |
| Chat | conversation_id | All messages in a conversation on one shard |

### Bad Shard Keys

| Shard Key | Problem |
|-----------|---------|
| Auto-increment ID | Hot spot: all inserts go to one shard |
| Timestamp | Hot spot: all writes go to the current time shard |
| Country code | Uneven: US shard gets 50% of traffic |
| Boolean (active/inactive) | Only 2 partitions. Not enough distribution |

### The Impossible Query

Once you choose a shard key, queries without it become expensive:

```
Shard key: user_id

-- Efficient (shard-local):
SELECT * FROM orders WHERE user_id = 42;

-- Expensive (scatter-gather across ALL shards):
SELECT * FROM orders WHERE order_date > '2024-01-01';
SELECT * FROM orders WHERE product_id = 100;
```

If the application needs to query by both `user_id` AND `product_id`
efficiently, options:
- **Secondary index sharding**: A global index on `product_id` (itself
  sharded), mapping product_id → shard + order_id. Two lookups.
- **Denormalization**: Store the data twice, sharded differently for
  each access pattern.
- **Don't shard**: If possible, scale vertically or use read replicas
  instead. Sharding has massive complexity costs.

---

## When Not to Shard

Sharding is a last resort, not a first choice.

**Sharding adds complexity:**
- Cross-shard queries and joins are expensive or impossible
- Distributed transactions are slow and complex
- Application code must be shard-aware (or use a proxy)
- Resharding is operationally risky
- Backups, migrations, and schema changes become per-shard
- Debugging is harder (which shard has the bug?)

**Before sharding, try:**
1. **Vertical scaling**: Bigger machine. 256 GB RAM, 64 cores, NVMe
   SSDs. Handles most workloads.
2. **Read replicas**: Scale reads without splitting writes.
3. **Caching**: Redis/Memcached for hot data.
4. **Query optimization**: Indexes, query rewriting, EXPLAIN ANALYZE.
5. **Connection pooling**: PgBouncer handles thousands of connections
   efficiently.
6. **Archiving**: Move old data to cheaper storage.
7. **Table partitioning**: PostgreSQL declarative partitioning (by
   range, list, hash) splits a table within one database instance.
   Not distributed but gives some benefits of sharding without the
   complexity.

If you've exhausted all of these and still need more write throughput
or storage, then shard.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Sharding splits data across machines for write scaling and storage | Replication scales reads. Sharding scales writes and storage |
| Range partitioning: hot spots with sequential keys | Good for time-series. Bad for auto-increment IDs |
| Hash partitioning: even distribution, no range queries | Destroys ordering. Good for point lookups |
| Consistent hashing minimizes data movement on resize | Virtual nodes improve balance. Used by DynamoDB, Cassandra |
| Compound keys: hash for distribution, sort for range queries | Partition key (which shard) + clustering key (order within shard) |
| Co-locate related data on the same shard | Shard both tables by the same key to keep joins local |
| Cross-shard queries = scatter-gather | Latency = slowest shard. Avoid by design |
| 2PC coordinates distributed transactions | Blocking protocol: coordinator crash = stuck transactions |
| Shard key is the most important design decision | Bad key → hot spots + expensive queries. Hard to change later |
| Sharding is a last resort | Try vertical scaling, replicas, caching, query optimization first |
