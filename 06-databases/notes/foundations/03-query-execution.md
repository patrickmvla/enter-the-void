# Query Execution — From SQL to Result Set

## The Pipeline

When you send a SQL query to a database, it passes through a pipeline
of stages before any data is returned:

```
SQL Text
  │
  ▼
Parser         → Abstract Syntax Tree (AST)
  │
  ▼
Analyzer       → Resolved names, types, permissions
  │
  ▼
Rewriter       → View expansion, rule application
  │
  ▼
Planner/       → Execution plan (the physical operations)
Optimizer
  │
  ▼
Executor       → Actually reads data, applies operations
  │
  ▼
Result Set
```

Understanding each stage is essential for writing efficient queries,
reading EXPLAIN output, and diagnosing performance problems.

---

## Parsing

The parser converts SQL text into an Abstract Syntax Tree (AST):

```sql
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.active = true
GROUP BY u.name
HAVING COUNT(o.id) > 5
ORDER BY COUNT(o.id) DESC
LIMIT 10;
```

Becomes:

```
SelectStmt
├── targetList: [u.name, COUNT(o.id)]
├── fromClause:
│   └── JoinExpr
│       ├── larg: RangeVar(users, alias=u)
│       ├── rarg: RangeVar(orders, alias=o)
│       └── quals: u.id = o.user_id
├── whereClause: u.active = true
├── groupClause: [u.name]
├── havingClause: COUNT(o.id) > 5
├── sortClause: [COUNT(o.id) DESC]
└── limitCount: 10
```

The parser checks **syntax** only. It doesn't know if `users` is a real
table or if `name` is a real column — that's the analyzer's job.

---

## Analysis (Semantic Analysis)

The analyzer resolves names and checks types:

1. **Name resolution**: `users` → the `public.users` table.
   `u.name` → the `name` column of `users` (type: varchar).
   Checks table exists, column exists, alias is valid.

2. **Type checking**: `u.active = true` — is `active` a boolean?
   `COUNT(o.id)` — is `id` a valid argument to COUNT? Implicit casts
   are inserted if needed (comparing int to bigint).

3. **Permission checking**: Does the current user have SELECT on
   `users` and `orders`?

4. **Function resolution**: `COUNT` — which overloaded version?
   PostgreSQL has multiple count functions; the analyzer picks the
   one matching the argument type.

Errors at this stage: "table does not exist," "column not found,"
"permission denied," "operator does not exist."

---

## Planning and Optimization

The planner transforms the logical query (what data to get) into a
physical plan (how to get it). This is where the database's intelligence
lives.

### The Search Space

For a query joining 5 tables, there are:
- 5! = 120 possible join orderings
- Multiple join algorithms per pair (nested loop, hash, merge)
- Index scan vs sequential scan for each table
- Multiple indexes to choose from

The total number of possible plans is astronomical. The planner uses
heuristics and cost estimation to find a good plan without exhaustively
searching all possibilities.

### Cost Model

The planner assigns an estimated **cost** to each possible operation
based on:

**Table statistics** (stored in `pg_statistic`):
- Number of rows (`reltuples`)
- Number of pages (`relpages`)
- Column statistics: most common values, histogram of value
  distribution, fraction of NULLs, number of distinct values

**Selectivity estimation**: What fraction of rows match a predicate?

```sql
WHERE age > 30
```

The planner looks at the histogram for the `age` column:

```
Histogram bounds: [18, 22, 25, 28, 30, 33, 37, 42, 50, 65, 90]
                   10 equal-frequency buckets

Values > 30: approximately 6 out of 10 buckets → selectivity ≈ 0.6
Rows matching: 100,000 × 0.6 = 60,000 estimated rows
```

**Cost formula** (PostgreSQL):

```
Sequential scan cost:
  total_cost = seq_page_cost × pages + cpu_tuple_cost × rows
  seq_page_cost = 1.0 (baseline)
  cpu_tuple_cost = 0.01

Index scan cost:
  total_cost = random_page_cost × pages_fetched + cpu_index_tuple_cost × index_rows
  random_page_cost = 4.0 (random I/O is 4x more expensive than sequential)

The planner picks the plan with the lowest estimated total cost.
```

**When the planner gets it wrong**: If the statistics are stale (table
was recently bulk-loaded but `ANALYZE` hasn't run), the planner
estimates the wrong number of rows. A selectivity estimate of 10 rows
(use index) when the real answer is 100,000 rows (use sequential scan)
produces a plan that's 100x slower than optimal.

```sql
-- Force statistics refresh:
ANALYZE users;

-- See what the planner estimates vs reality:
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 30;
-- Shows: "rows=60000" (estimated) vs "actual rows=58742"
```

### Join Algorithms

The planner chooses between three fundamental join algorithms:

**Nested Loop Join:**
```
For each row in outer table:
  For each row in inner table:
    If join condition matches:
      Output the combined row

Cost: O(N × M) without index
      O(N × log M) with index on inner table
```

Best when: one table is small (drives the outer loop) and there's an
index on the inner table's join column. The planner converts this to
an **Index Nested Loop** — for each outer row, do an index lookup on
the inner table.

**Hash Join:**
```
Phase 1 (Build): Scan the smaller table, build a hash table on the join key
Phase 2 (Probe): Scan the larger table, probe the hash table for each row

Cost: O(N + M) — linear in the size of both tables
Memory: Proportional to the smaller table
```

Best when: no useful indexes, both tables are large, the smaller table
fits in memory. If the hash table doesn't fit in memory, it spills to
disk (partitioned into batches).

```
Build phase:
  Hash table: {hash(user_id) → [row1, row2, ...]}

Probe phase:
  For each order row:
    hash(order.user_id) → lookup in hash table → emit matches
```

**Merge Join (Sort-Merge Join):**
```
Phase 1: Sort both tables on the join key (or use pre-sorted index)
Phase 2: Merge the sorted streams (like merge step of merge sort)

Cost: O(N log N + M log M) for sorting + O(N + M) for merging
      O(N + M) if both inputs are already sorted (from an index)
```

Best when: both inputs are already sorted (via index scan) or the
result needs to be sorted anyway (ORDER BY on the join key). Merge
join is the only algorithm that produces sorted output naturally.

**Planner selection:**

| Scenario | Algorithm |
|----------|-----------|
| Small outer + indexed inner | Nested Loop (Index) |
| No indexes, both large | Hash Join |
| Both sorted (from indexes) | Merge Join |
| One very small, one very large | Nested Loop |
| Join + ORDER BY on join key | Merge Join |

### Scan Methods

**Sequential Scan**: Read every page of the table in order. O(pages).
Used when: no useful index, or the query reads a large fraction of
the table (an index scan would be slower due to random I/O).

**Index Scan**: Traverse the B+tree index, then fetch each matching
row from the heap. Each heap fetch is a random I/O. Used when: small
fraction of rows match.

**Index-Only Scan**: The index contains all the columns needed by the
query. No heap fetch needed. Requires the visibility map to confirm
all tuples on the page are visible (no need to check the heap for
MVCC visibility). This is why VACUUM maintains the visibility map.

```sql
CREATE INDEX idx_users_name ON users(name);

-- Index-only scan possible (only needs name):
SELECT name FROM users WHERE name = 'alice';

-- Must fetch from heap (needs email, not in index):
SELECT name, email FROM users WHERE name = 'alice';

-- Covering index enables index-only scan:
CREATE INDEX idx_users_name_email ON users(name) INCLUDE (email);
SELECT name, email FROM users WHERE name = 'alice';  -- index-only scan ✓
```

**Bitmap Index Scan**: A hybrid approach for medium selectivity:

```
1. Scan the index, collect matching TIDs into a bitmap (1 bit per page)
2. Sort the bitmap by page number
3. Fetch pages in sequential order (converting random I/O to sequential)

Multiple indexes can be combined with bitmap AND/OR:
  Bitmap(age > 30) AND Bitmap(city = 'NYC') → rows matching both
```

Used when: too many rows for index scan (random I/O cost), too few
for sequential scan, or when combining multiple indexes.

---

## EXPLAIN — Reading Execution Plans

EXPLAIN is the primary tool for understanding and optimizing query
performance.

### Basic EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE age > 30;
```

```
Seq Scan on users  (cost=0.00..1693.00 rows=60000 width=68)
  Filter: (age > 30)
```

**Reading the output:**
- `Seq Scan`: Sequential scan (reading every page)
- `cost=0.00..1693.00`: Startup cost..total cost (in arbitrary units)
  - Startup cost: cost before the first row is returned
  - Total cost: cost to return all rows
- `rows=60000`: Estimated number of rows returned
- `width=68`: Estimated average row width in bytes
- `Filter`: Predicate applied to each row (after reading from disk)

### EXPLAIN ANALYZE

Actually runs the query and shows actual vs estimated numbers:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.active = true
GROUP BY u.name
ORDER BY COUNT(o.id) DESC
LIMIT 10;
```

```
Limit  (cost=4520.10..4520.13 rows=10 width=44) (actual time=45.2..45.3 rows=10 loops=1)
  -> Sort  (cost=4520.10..4523.40 rows=1320 width=44) (actual time=45.2..45.2 rows=10 loops=1)
        Sort Key: (count(o.id)) DESC
        Sort Method: top-N heapsort  Memory: 26kB
        -> HashAggregate  (cost=4480.00..4493.20 rows=1320 width=44) (actual time=44.1..44.5 rows=1320 loops=1)
              Group Key: u.name
              Batches: 1  Memory Usage: 209kB
              -> Hash Join  (cost=1250.00..4230.00 rows=50000 width=20) (actual time=12.3..38.5 rows=48756 loops=1)
                    Hash Cond: (o.user_id = u.id)
                    -> Seq Scan on orders o  (cost=0.00..1540.00 rows=100000 width=12) (actual time=0.01..5.2 rows=100000 loops=1)
                          Buffers: shared hit=540
                    -> Hash  (cost=1200.00..1200.00 rows=4000 width=16) (actual time=8.1..8.1 rows=3876 loops=1)
                          Buckets: 4096  Batches: 1  Memory Usage: 184kB
                          -> Seq Scan on users u  (cost=0.00..1200.00 rows=4000 width=16) (actual time=0.02..6.8 rows=3876 loops=1)
                                Filter: active
                                Rows Removed by Filter: 6124
                                Buffers: shared hit=400
Planning Time: 0.85 ms
Execution Time: 45.5 ms
```

**What to look for:**

1. **Estimated vs actual rows**: `rows=4000` estimated, `rows=3876`
   actual. Close enough — statistics are accurate. If these diverge
   wildly (estimated 100, actual 50,000), the planner chose poorly.

2. **Seq Scan where Index Scan expected**: The planner chose a sequential
   scan on `orders`. With 100,000 rows, this might be correct (reading
   the whole table is faster than 50,000 random index lookups). But if
   this were a point query returning 10 rows, a Seq Scan would be wrong.

3. **Buffers**: `shared hit=540` means 540 pages were read from the
   buffer pool (cache). `shared read=X` means X pages were read from
   disk. High read count = cold cache or working set exceeds memory.

4. **Sort Method**: `top-N heapsort` is efficient for LIMIT (only
   tracks the top N). `external merge` means the sort spilled to disk
   (not enough `work_mem`).

5. **Hash Batches**: `Batches: 1` means the hash table fit in memory.
   `Batches: N` means it spilled to disk (increase `work_mem`).

### Common EXPLAIN Patterns and Fixes

**Problem: Sequential scan on a large table with low selectivity**
```
Seq Scan on users  (cost=0.00..25000.00 rows=100 width=68)
  Filter: (email = 'alice@example.com')
  Rows Removed by Filter: 999900
```
Fix: Create an index on `email`.

**Problem: Index scan with too many rows**
```
Index Scan using idx_users_age on users  (cost=0.42..8500.00 rows=500000 width=68)
  Index Cond: (age > 20)
```
If this fetches 50% of the table, Seq Scan is cheaper. The planner
should choose Seq Scan here — if it doesn't, statistics may be wrong.
Run `ANALYZE users`.

**Problem: Nested Loop where Hash Join is better**
```
Nested Loop  (cost=0.42..500000.00 rows=100000 width=80)
  -> Seq Scan on orders  (actual rows=100000)
  -> Index Scan using users_pkey on users  (actual loops=100000)
```
100,000 index lookups. A Hash Join would build a hash table of users
(one pass) and probe it for each order (one pass). Set `enable_nestloop = off`
temporarily to see if the planner produces a better plan, then investigate
why it chose poorly (likely bad row estimates).

**Problem: Sort spilling to disk**
```
Sort  (cost=50000.00..52500.00 rows=1000000)
  Sort Method: external merge  Disk: 75MB
```
Fix: Increase `work_mem` (per-operation memory limit for sorts,
hashes, etc.).

```sql
SET work_mem = '256MB';  -- session-level, for the next query
```

---

## Common Optimization Patterns

### Index Design

**Composite indexes** — column order matters:

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Uses the index (leftmost prefix):
SELECT * FROM orders WHERE user_id = 42;
SELECT * FROM orders WHERE user_id = 42 AND created_at > '2024-01-01';

-- Cannot use the index efficiently:
SELECT * FROM orders WHERE created_at > '2024-01-01';
-- (the leading column user_id is not constrained)
```

The **leftmost prefix rule**: A composite index (a, b, c) can be used
for queries on (a), (a, b), or (a, b, c), but NOT (b), (c), or (b, c).
The index is a B+tree sorted by a, then by b within each a, then by c.
Without knowing a, the b values are scattered.

**Covering indexes** (INCLUDE):

```sql
-- The query needs name and email, filtered by age:
SELECT name, email FROM users WHERE age > 30;

-- Standard index — must fetch from heap for name and email:
CREATE INDEX idx_age ON users(age);

-- Covering index — all needed columns in the index:
CREATE INDEX idx_age_covering ON users(age) INCLUDE (name, email);
-- → Index-only scan (no heap access)
```

**Partial indexes:**

```sql
-- Only index rows where the predicate is true:
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Smaller index, faster lookups, only for active users
-- Queries must include the predicate:
SELECT * FROM users WHERE email = 'x@y.com' AND active = true;  -- uses index ✓
SELECT * FROM users WHERE email = 'x@y.com';                     -- can't use ✗
```

### Query Anti-Patterns

**Functions on indexed columns kill index usage:**

```sql
-- BAD: function on the column prevents index use
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- → Sequential scan (index on email can't help because LOWER() transforms the value)

-- FIX: Expression index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- OR: Store emails in lowercase and compare directly
```

**Implicit type casts prevent index use:**

```sql
-- Column user_id is integer, but parameter is text:
SELECT * FROM orders WHERE user_id = '42';
-- PostgreSQL may cast the column: CAST(user_id AS text) = '42'
-- → Function on column → no index use

-- FIX: Use the correct type:
SELECT * FROM orders WHERE user_id = 42;
```

**SELECT * fetches unnecessary data:**

```sql
-- BAD: fetches all 20 columns when you need 2
SELECT * FROM users WHERE id = 42;

-- GOOD: only the columns you need (enables index-only scan with covering index)
SELECT name, email FROM users WHERE id = 42;
```

**N+1 queries (application-level):**

```javascript
// BAD: 1 query + N queries
const users = await db.query('SELECT * FROM users LIMIT 100');
for (const user of users) {
    const orders = await db.query('SELECT * FROM orders WHERE user_id = $1', [user.id]);
}
// 101 queries

// GOOD: 2 queries
const users = await db.query('SELECT * FROM users LIMIT 100');
const userIds = users.map(u => u.id);
const orders = await db.query('SELECT * FROM orders WHERE user_id = ANY($1)', [userIds]);
// 2 queries, same data
```

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Pipeline: Parse → Analyze → Rewrite → Plan → Execute | Each stage transforms the query. Optimization happens at the planning stage |
| Cost-based optimization uses table statistics | Row counts, histograms, distinct values. Stale stats → bad plans. Run ANALYZE |
| Selectivity estimation drives plan choice | Wrong estimate → wrong algorithm. seq scan vs index scan depends on row count |
| Three join algorithms: nested loop, hash, merge | NL for small+indexed, Hash for large+no index, Merge for sorted inputs |
| Index scan, seq scan, bitmap scan, index-only scan | Bitmap bridges index and seq scan. Index-only avoids heap access |
| EXPLAIN ANALYZE shows estimated vs actual rows | Divergence = stale statistics or bad estimates. Check Buffers for I/O |
| Composite index uses leftmost prefix rule | Index(a,b,c) works for (a), (a,b), (a,b,c) but NOT (b) or (c) |
| Covering indexes (INCLUDE) enable index-only scans | Put all queried columns in the index. Avoids heap fetches |
| Functions on columns prevent index use | WHERE LOWER(email) = x can't use index(email). Use expression indexes |
| work_mem controls per-operation memory for sorts/hashes | Too low → disk spill. Set appropriately per query or session |
