# PostgreSQL Internals — The Concrete Implementation

## Why PostgreSQL

PostgreSQL is the most advanced open-source relational database. It
implements every concept covered in this section — B+trees, MVCC,
WAL/ARIES, cost-based optimization, streaming replication — in
production-grade code running the backends of Instagram, Discord,
Stripe, and thousands of others.

Understanding PostgreSQL's internals is understanding the theory made
real. This topic maps abstract concepts to specific PostgreSQL
mechanisms, configurations, and tools.

---

## Process Architecture

PostgreSQL uses a **multi-process model** (not multi-threaded). Each
client connection is handled by a separate OS process (a "backend").

```
                        ┌────────────────┐
                        │   Postmaster   │  Main process (PID 1)
                        │  (supervisor)  │  Accepts connections, forks backends
                        └───────┬────────┘
                     ┌──────────┼──────────────────────┐
                     │          │                       │
              ┌──────▼──────┐  │  ┌──────────────────┐ │
              │   Backend    │  │  │   Background      │ │
              │  (per client)│  │  │   Workers          │ │
              └─────────────┘  │  └──────────────────┘ │
              ┌─────────────┐  │  ┌──────────────────┐ │
              │   Backend    │  │  │   Autovacuum      │ │
              │  (per client)│  │  │   Launcher        │ │
              └─────────────┘  │  └──────────────────┘ │
              ┌─────────────┐  │  ┌──────────────────┐ │
              │   Backend    │  │  │   WAL Writer      │ │
              │  (per client)│  │  └──────────────────┘ │
              └─────────────┘  │  ┌──────────────────┐ │
                               │  │   Checkpointer    │ │
                               │  └──────────────────┘ │
                               │  ┌──────────────────┐ │
                               │  │   BG Writer       │ │
                               │  └──────────────────┘ │
                               │  ┌──────────────────┐ │
                               │  │  WAL Sender(s)    │ │
                               │  │  (replication)    │ │
                               │  └──────────────────┘ │
                               │  ┌──────────────────┐ │
                               │  │  Stats Collector  │ │
                               │  └──────────────────┘ │
                               └──────────────────────┘

                All processes share: Shared Memory (buffer pool, WAL buffers,
                                     lock tables, CLOG, etc.)
```

### Key Processes

**Postmaster**: The parent process. Listens for connections, forks a
new backend process for each client. If a backend crashes, the
postmaster kills all other backends and restarts (to ensure shared
memory consistency). This is why a single query crashing can briefly
disconnect all clients.

**Backend (per-connection)**: Handles one client's SQL queries. Parses,
plans, executes, returns results. Each backend has its own memory
context (work_mem, maintenance_work_mem) but shares the buffer pool.

**WAL Writer**: Flushes WAL buffers to disk. Backends write WAL records
to shared WAL buffers; the WAL writer periodically fsyncs them. On
COMMIT, the backend can force an immediate flush (synchronous_commit=on)
or accept the slight durability risk of waiting for the next periodic
flush (synchronous_commit=off — up to ~200ms of potential data loss).

**Background Writer**: Writes dirty pages from the buffer pool to data
files. Spreads I/O over time to avoid checkpoint spikes.

**Checkpointer**: Periodically writes all dirty pages and creates a
checkpoint record in the WAL. After a checkpoint, WAL before that
point can be recycled. Configurable via `checkpoint_timeout` (default
5 minutes) and `max_wal_size` (default 1 GB).

**Autovacuum Launcher**: Starts autovacuum workers for tables that
exceed dead tuple thresholds. Critical for preventing table bloat and
XID wraparound.

**WAL Sender**: One per streaming replication replica. Reads WAL records
and sends them to the replica's WAL receiver.

### Connection Handling

Each connection = one process. Creating a process takes ~5ms and ~5 MB
of memory. For an application with 500 connections opening and closing
rapidly, this overhead is significant.

**PgBouncer** (connection pooler) sits between the application and
PostgreSQL:

```
App (500 connections) → PgBouncer (20 actual connections) → PostgreSQL

Modes:
  Session:     One PG connection per client session (least useful)
  Transaction: PG connection shared between clients between transactions
  Statement:   PG connection shared between individual statements (most aggressive)
```

Transaction mode is the standard. A PgBouncer instance handling 500
application connections might only maintain 20 actual PostgreSQL
connections, dramatically reducing process overhead.

**Limitation**: PgBouncer in transaction mode doesn't support
prepared statements (they're per-session state), SET commands, LISTEN/
NOTIFY, or advisory locks — anything that depends on session state.

---

## Shared Memory Layout

All backend processes share a region of memory allocated at startup:

```
Shared Memory:
┌───────────────────────────────────────────────┐
│ Buffer Pool (shared_buffers)                   │
│   Default: 128 MB. Production: 25% of RAM     │
│   Array of 8 KB pages (same as on-disk pages)  │
│   Hash table: (relation, page#) → buffer slot  │
│   Clock sweep for eviction                     │
├───────────────────────────────────────────────┤
│ WAL Buffers (wal_buffers)                      │
│   Default: 1/32 of shared_buffers (min 64 KB)  │
│   Ring buffer for WAL records before flush     │
├───────────────────────────────────────────────┤
│ CLOG (pg_xact)                                │
│   Transaction commit status: 2 bits per txn    │
│   00=in progress, 01=committed, 10=aborted     │
├───────────────────────────────────────────────┤
│ Lock Table                                     │
│   Hash table of all held locks                 │
│   Row locks, table locks, advisory locks       │
├───────────────────────────────────────────────┤
│ Proc Array                                     │
│   List of active backend processes             │
│   Used for snapshot computation (which txns    │
│   are visible?)                                │
└───────────────────────────────────────────────┘
```

### shared_buffers

The most important memory setting. This IS the buffer pool — the cache
of data pages in memory.

**Sizing**: Start with 25% of total RAM. PostgreSQL also relies on the
OS page cache (effective_cache_size tells the planner how much total
cache to assume, typically 75% of RAM). Setting shared_buffers to 50%+
of RAM usually hurts — double caching (buffer pool + OS cache) wastes
memory, and the OS cache is better at I/O scheduling.

```
Server: 64 GB RAM
  shared_buffers = 16 GB       (25%)
  effective_cache_size = 48 GB (75%, tells planner to assume this much cache)
```

### work_mem

Per-operation memory for sorts, hash joins, and hash aggregates.
**Not per-connection** — per-operation. A complex query with 3 hash
joins and 2 sorts uses up to 5 × work_mem.

```
Default: 4 MB
100 connections × average 3 operations = 300 × 4 MB = 1.2 GB

Set higher for analytical queries:
SET work_mem = '256MB';  -- for this session only

Too low: sorts spill to disk (external merge), hash joins spill to batches
Too high: OOM risk under concurrent load
```

### maintenance_work_mem

Memory for maintenance operations: VACUUM, CREATE INDEX, ALTER TABLE.
These are infrequent but benefit enormously from more memory:

```
Default: 64 MB
Production: 1-2 GB (allows VACUUM to process more dead tuples in one pass)
```

---

## The System Catalog

PostgreSQL stores its own metadata (table definitions, column types,
indexes, functions, users, permissions) in regular tables — the
**system catalog**. You can query the database to learn about the
database.

```sql
-- All tables (schema + name):
SELECT schemaname, tablename FROM pg_tables WHERE schemaname = 'public';

-- Column info:
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'users';

-- Index info:
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';

-- Table statistics (what the planner sees):
SELECT relname, reltuples, relpages, relallvisible
FROM pg_class WHERE relname = 'users';

-- Column statistics (histograms, MCVs):
SELECT attname, n_distinct, most_common_vals, histogram_bounds
FROM pg_stats WHERE tablename = 'users';

-- Running queries:
SELECT pid, state, query, query_start, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle';

-- Table bloat estimate:
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) as dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## VACUUM Deep Dive

### What VACUUM Does

1. **Scans** the table for dead tuples (xmax is set, and the deleting
   transaction is committed and older than any active snapshot)
2. **Removes** dead tuples from heap pages, adds space to the free
   space map (FSM)
3. **Updates** the visibility map (VM) — marks pages as "all-visible"
   (enabling index-only scans)
4. **Freezes** old tuples (replaces xmin with FrozenTransactionId)
   to prevent XID wraparound
5. **Updates** pg_class statistics (reltuples, relpages)
6. **Truncates** trailing empty pages (returns space to the OS, but
   only from the end of the file)

### VACUUM vs VACUUM FULL

| | VACUUM | VACUUM FULL |
|---|--------|-------------|
| Locks | ShareUpdateExclusiveLock (non-blocking) | AccessExclusiveLock (blocks ALL access) |
| Space | Marks dead space as reusable (within the file) | Rewrites entire table (reclaims space to OS) |
| Speed | Fast (single pass) | Slow (full rewrite) |
| Concurrent access | Yes | No |
| When to use | Regularly (autovacuum) | Only when table is severely bloated |

### XID Wraparound Protection

PostgreSQL uses 32-bit transaction IDs (XIDs). At the default rate of
~1000 transactions/second, wraparound occurs in ~49 days of continuous
heavy load.

```
XID space (circular):
  ←───── past ────── current ────── future ─────→
  [2B old txns]  [current XID]  [2B future XIDs]

If a tuple's xmin is 2 billion XIDs in the past, the circular comparison
breaks — the tuple appears to be from the future and becomes invisible.
```

**VACUUM FREEZE** marks tuples with xmin older than
`vacuum_freeze_min_age` (default 50 million XIDs) as "frozen" —
replacing their xmin with a special FrozenTransactionId (2) that is
always considered in the past.

**Autovacuum freeze**: When a table approaches the wraparound danger
zone (`autovacuum_freeze_max_age`, default 200 million XIDs),
autovacuum triggers an aggressive freeze vacuum regardless of the
dead tuple count.

**The emergency**: If a database reaches `2 billion - autovacuum_freeze_max_age`
XIDs without completing VACUUM FREEZE, PostgreSQL sets the database
to read-only mode and refuses new transactions:

```
WARNING: database "mydb" must be vacuumed within X transactions
ERROR: database is not accepting commands to avoid wraparound data loss
```

This has caused real production outages. Prevention: monitor
`age(datfrozenxid)` in `pg_database` and ensure autovacuum keeps up.

---

## TOAST — Handling Large Values

PostgreSQL pages are 8 KB. A single row must fit on one page (with
its item pointer and header). But a text column can hold megabytes.

**TOAST** (The Oversized-Attribute Storage Technique) handles this:

```
Row on main page:
  [id=42, name='alice', bio=TOAST_pointer]
                              ↓
TOAST table (separate heap):
  [chunk 0: first 2KB of bio text]
  [chunk 1: next 2KB]
  [chunk 2: remaining 1.5KB]
```

**TOAST strategies** (per-column):

| Strategy | Behavior |
|----------|----------|
| PLAIN | No TOAST (value must fit on a page). Used for fixed-size types |
| EXTENDED | Compress first, then store externally if still too large (default for text, bytea) |
| EXTERNAL | Store externally without compression (for pre-compressed data, or when partial reads are needed) |
| MAIN | Compress but keep on main page if possible. External only as last resort |

```sql
-- Check TOAST strategy:
SELECT attname, attstorage FROM pg_attribute
WHERE attrelid = 'users'::regclass;
-- p=PLAIN, x=EXTENDED, e=EXTERNAL, m=MAIN

-- Change strategy:
ALTER TABLE users ALTER COLUMN bio SET STORAGE EXTERNAL;
```

**Performance implication**: TOASTed values require extra I/O (reading
from the TOAST table). Queries that don't need the TOASTed column
(SELECT id, name FROM users) don't read it — only accessed when the
column is actually referenced.

---

## HOT Updates (Heap-Only Tuples)

In standard MVCC, every UPDATE creates a new tuple version. If the
row has 5 indexes, each index must be updated to point to the new
tuple location. For tables with many indexes and frequent updates,
this is expensive.

**HOT (Heap-Only Tuple)** optimization: If the updated columns are
not part of any index, the new tuple can be placed on the **same page**
as the old tuple. No index updates needed.

```
Before HOT update of non-indexed column:
  Index → Page 10, Slot 3 → (xmin=100, xmax=0, data={name:'alice', age:28})

After HOT update (only age changed, not indexed):
  Index → Page 10, Slot 3 → (xmin=100, xmax=200, ctid=(10,4))  ← old version
                              Page 10, Slot 4 → (xmin=200, xmax=0, data={name:'alice', age:29})
                                                 ↑ heap-only tuple (no index entry)

Index still points to Slot 3. When accessed via index:
  1. Read Slot 3, see it's dead and has ctid=(10,4)
  2. Follow the HOT chain to Slot 4
  3. Return the live tuple
```

**Conditions for HOT:**
- The updated columns must NOT be in any index
- The new tuple must fit on the same page as the old one
- The page must have enough free space (see fill factor)

**Fill factor** matters here: `CREATE TABLE users (...) WITH (fillfactor=80)`
leaves 20% free space per page for HOT updates. Without free space,
new tuples go to a different page and HOT can't be used.

---

## Index Types Beyond B-Tree

PostgreSQL's B+tree is the default and handles most workloads. But
PostgreSQL supports specialized index types for access patterns where
B+trees are inefficient or useless.

### GIN (Generalized Inverted Index)

An inverted index: maps each value (or element) to the set of rows
that contain it. The inverse of a normal index (which maps rows to
values).

```
Normal B-tree index:
  row 1 → tags: ['python', 'web']
  row 2 → tags: ['python', 'data']
  row 3 → tags: ['web', 'react']

GIN inverted index:
  'data'   → {row 2}
  'python' → {row 1, row 2}
  'react'  → {row 3}
  'web'    → {row 1, row 3}
```

**Use cases:**
- **JSONB queries**: `WHERE data @> '{"role": "admin"}'` — find documents
  containing a key-value pair. GIN indexes every key and value in the
  JSONB document.
- **Full-text search**: `WHERE to_tsvector(body) @@ to_tsquery('database & performance')`
  — GIN indexes each lexeme (word stem) to the rows containing it.
- **Array containment**: `WHERE tags @> ARRAY['python']` — find rows
  where the array contains a value.

```sql
-- JSONB index:
CREATE INDEX idx_data_gin ON documents USING GIN (data);

-- Full-text search index:
CREATE INDEX idx_body_fts ON articles USING GIN (to_tsvector('english', body));

-- Array index:
CREATE INDEX idx_tags_gin ON posts USING GIN (tags);
```

**Internals**: GIN stores a B-tree of keys (the indexed values/elements),
where each leaf entry points to a **posting list** (sorted list of TIDs)
or a **posting tree** (B-tree of TIDs, for keys with many matches).
Insertion is expensive — adding a row may require updating many keys
(every word in a document). GIN uses a **pending list** to batch
inserts, merging them into the main index periodically or when the
pending list exceeds `gin_pending_list_limit`.

### GiST (Generalized Search Tree)

A balanced tree that supports arbitrary indexing schemes through a
pluggable interface. Each internal node contains a **bounding
predicate** that covers all children.

**Use cases:**
- **Spatial data (PostGIS)**: `WHERE ST_Contains(area, point)` — find
  which polygon contains a point. GiST uses R-tree-like bounding boxes.
- **Range types**: `WHERE daterange @> '2024-06-15'` — find ranges
  containing a value.
- **Nearest-neighbor search**: `ORDER BY location <-> ST_MakePoint(lon, lat) LIMIT 10`
  — find the 10 closest points. GiST supports KNN (k-nearest-neighbor)
  search natively.

```sql
-- PostGIS spatial index:
CREATE INDEX idx_location ON buildings USING GiST (geom);

-- Range index:
CREATE INDEX idx_booking_dates ON bookings USING GiST (during);

-- Exclusion constraint (no overlapping bookings for the same room):
ALTER TABLE bookings ADD CONSTRAINT no_overlap
    EXCLUDE USING GiST (room_id WITH =, during WITH &&);
```

**Exclusion constraints** are a powerful GiST feature: enforce that no
two rows satisfy a set of operators simultaneously. "No two bookings
for the same room can have overlapping date ranges" — expressed
declaratively, enforced by the index.

### BRIN (Block Range INdex)

Stores summary statistics (min/max values) for ranges of physical
pages. Extremely compact — a BRIN index on a 100 GB table might be
only a few MB.

```
Pages 0-127:     min_timestamp = 2024-01-01, max_timestamp = 2024-01-15
Pages 128-255:   min_timestamp = 2024-01-15, max_timestamp = 2024-02-01
Pages 256-383:   min_timestamp = 2024-02-01, max_timestamp = 2024-02-15
...

Query: WHERE timestamp > '2024-02-10'
BRIN check: Skip pages 0-255 (max < 2024-02-10). Scan pages 256+.
```

**Requires physical correlation**: BRIN only works when the indexed
column's values correlate with the physical row order. Time-series data
inserted chronologically is perfect — timestamps increase with page
number. Random data has no correlation, and BRIN can't eliminate any
pages.

```sql
-- Perfect for append-only time-series:
CREATE INDEX idx_events_time ON events USING BRIN (created_at)
    WITH (pages_per_range = 128);

-- Check correlation (1.0 = perfect, 0.0 = no correlation):
SELECT attname, correlation FROM pg_stats WHERE tablename = 'events';
```

**Tradeoff**: BRIN uses almost no disk space and no maintenance
overhead, but it's lossy — it can only eliminate page ranges, not
pinpoint exact rows. After BRIN eliminates ranges, a sequential scan
reads the remaining pages. For time-series with good correlation,
this is extremely efficient.

### Hash Index

A hash table on disk. O(1) lookups for equality comparisons.

```sql
CREATE INDEX idx_session_token ON sessions USING HASH (token);

-- Only supports equality:
SELECT * FROM sessions WHERE token = 'abc123';  -- uses hash index ✓
SELECT * FROM sessions WHERE token > 'abc';     -- cannot use hash ✗
```

In PostgreSQL 10+, hash indexes are WAL-logged and crash-safe (they
weren't before — a crash required rebuilding). Still rarely used in
practice because B-tree handles equality just as well and also supports
range queries, sorting, and index-only scans. Hash indexes are
marginally faster for pure equality lookups on very large tables.

### Choosing the Right Index Type

| Index | Best For | Size | Insert Speed |
|-------|----------|------|-------------|
| B-tree | Equality, range, sorting, most queries | Medium | Fast |
| GIN | Multi-valued columns (JSONB, arrays, full-text) | Large | Slow (batched) |
| GiST | Spatial, ranges, nearest-neighbor, exclusion | Medium | Medium |
| BRIN | Time-series, append-only with physical correlation | Tiny | Minimal overhead |
| Hash | Pure equality on very large tables | Medium | Fast |

---

## Table Partitioning

Splitting a single logical table into multiple physical tables within
one database instance. Not distributed sharding — all partitions are
on the same machine. PostgreSQL 10+ supports declarative partitioning.

### Why Partition

- **Query performance**: Queries that filter on the partition key only
  scan relevant partitions (partition pruning). A query for January
  data only reads the January partition, not the entire year.
- **Maintenance**: VACUUM, ANALYZE, and index rebuilds operate per
  partition. Dropping old data = `DROP TABLE` on the old partition
  (instant, vs DELETE which creates dead tuples).
- **Bulk loading**: Load data into a new partition without affecting
  queries on existing partitions.

### Partition Strategies

**Range partitioning** (most common — time-series, logs):

```sql
CREATE TABLE events (
    id          bigserial,
    created_at  timestamptz NOT NULL,
    data        jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE events_2024_03 PARTITION OF events
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Query only scans events_2024_01:
SELECT * FROM events WHERE created_at >= '2024-01-15' AND created_at < '2024-02-01';

-- Drop old data instantly:
DROP TABLE events_2024_01;
```

**List partitioning** (categorical — region, status, tenant):

```sql
CREATE TABLE orders (
    id      bigserial,
    region  text NOT NULL,
    amount  numeric
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('us');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('eu', 'uk');
CREATE TABLE orders_asia PARTITION OF orders FOR VALUES IN ('jp', 'kr', 'sg');
```

**Hash partitioning** (even distribution when no natural range/list):

```sql
CREATE TABLE sessions (
    id      uuid NOT NULL,
    data    jsonb
) PARTITION BY HASH (id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Partition Pruning

The planner eliminates irrelevant partitions at plan time:

```sql
EXPLAIN SELECT * FROM events WHERE created_at = '2024-02-15';
```

```
Append
  -> Seq Scan on events_2024_02
        Filter: (created_at = '2024-02-15')
```

Only `events_2024_02` is scanned. The other partitions are pruned
entirely. This works for static values. For parameterized queries
(`WHERE created_at = $1`), **runtime pruning** (PostgreSQL 11+)
evaluates the parameter at execution time and prunes then.

### Partition Limitations

- Every unique constraint (including primary keys) must include the
  partition key. You can't have a globally unique `id` column without
  also including the partition key in the constraint.
- Foreign keys referencing partitioned tables are supported only in
  PostgreSQL 12+.
- Too many partitions (thousands) can slow down planning — the planner
  must consider each partition.
- Indexes are per-partition — each partition has its own B-tree. This
  is usually a benefit (smaller indexes, faster maintenance) but means
  global index lookups scan multiple partition indexes.

---

## Performance Monitoring

### Key Views

```sql
-- Slow queries (requires pg_stat_statements extension):
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Cache hit ratio (should be > 99%):
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS ratio
FROM pg_statio_user_tables;

-- Index usage (unused indexes waste space and slow writes):
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Table access patterns:
SELECT relname, seq_scan, seq_tup_read, idx_scan, idx_tup_fetch,
       n_tup_ins, n_tup_upd, n_tup_del, n_dead_tup
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Lock waits:
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype AND kl.relation = bl.relation AND kl.pid != bl.pid
JOIN pg_stat_activity blocking ON kl.pid = blocking.pid
WHERE NOT bl.granted;
```

### Critical Metrics to Monitor

| Metric | Source | Alert If |
|--------|--------|----------|
| Replication lag | pg_stat_replication.replay_lag | > 30 seconds |
| Dead tuples | pg_stat_user_tables.n_dead_tup | > 10% of live tuples |
| XID age | pg_database.datfrozenxid | > 500M XIDs |
| Cache hit ratio | pg_statio_user_tables | < 99% |
| Connection count | pg_stat_activity | > 80% of max_connections |
| Long-running queries | pg_stat_activity.query_start | > 5 minutes |
| Lock waits | pg_locks.granted = false | Any significant duration |
| WAL generation rate | pg_stat_wal | Spikes indicate problems |
| Checkpoint duration | pg_stat_bgwriter | > checkpoint_timeout |

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Multi-process model: one process per connection | Backend crash → postmaster kills all backends. Use PgBouncer for pooling |
| shared_buffers = 25% of RAM, effective_cache_size = 75% | Buffer pool IS the page cache. OS also caches, so don't over-allocate |
| work_mem is per-operation, not per-connection | 3 hash joins × work_mem. Too low → disk spill. Too high → OOM risk |
| System catalog: pg_class, pg_stats, pg_stat_activity | Query the database about itself. Stats drive planner decisions |
| VACUUM reclaims dead tuples, updates visibility map, freezes XIDs | Autovacuum is critical. Monitor n_dead_tup and datfrozenxid |
| XID wraparound forces read-only mode if VACUUM can't keep up | Monitor age(datfrozenxid). Real production outage vector |
| TOAST handles oversized values in a separate table | Chunked storage. EXTENDED = compress + external. Only read when column accessed |
| HOT updates avoid index updates for non-indexed column changes | Requires same-page placement. Fill factor reserves space for HOT |
| GIN: inverted index for JSONB, arrays, full-text search | Maps each value to rows containing it. Pending list batches inserts |
| GiST: spatial, ranges, nearest-neighbor, exclusion constraints | R-tree-like bounding predicates. PostGIS depends on GiST |
| BRIN: tiny index for time-series with physical correlation | Min/max per page range. Correlation must be near 1.0 to be effective |
| Table partitioning: range, list, hash within one instance | Partition pruning eliminates irrelevant partitions. DROP partition for instant data removal |
| pg_stat_statements shows slow query patterns | total_exec_time, mean_exec_time, calls. The first tool for optimization |
| Cache hit ratio should be > 99% | < 99% = shared_buffers too small or working set exceeds memory |
