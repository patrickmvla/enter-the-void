# Deep Internals — The Machinery Under the Hood

## The Wire Protocol

When your application sends a SQL query to PostgreSQL, it doesn't send
raw text over TCP. The PostgreSQL wire protocol (also called the
frontend/backend protocol) defines a binary message format for
communication between client and server.

### Connection Startup

```
Client                                    Server (Postmaster)
  │                                         │
  │ StartupMessage                          │
  │   [4 bytes: length]                     │
  │   [4 bytes: protocol version 3.0]       │
  │   [key=value pairs: user, database]     │
  │   [null terminator]                     │
  │ ───────────────────────────────────────> │
  │                                         │  fork() → backend process
  │ AuthenticationMD5Password               │
  │   [1 byte: 'R'][4 bytes: len]           │
  │   [4 bytes: auth type = 5]              │
  │   [4 bytes: salt]                       │
  │ <─────────────────────────────────────── │
  │                                         │
  │ PasswordMessage                         │
  │   [1 byte: 'p'][4 bytes: len]           │
  │   [md5(md5(password+user)+salt)]        │
  │ ───────────────────────────────────────> │
  │                                         │
  │ AuthenticationOk                        │
  │ ParameterStatus (server_version, ...)   │
  │ BackendKeyData (PID, secret key)        │
  │ ReadyForQuery ('I' = idle)              │
  │ <─────────────────────────────────────── │
```

**Every message** (except StartupMessage and SSLRequest) has the format:

```
[1 byte: message type][4 bytes: length (including self)][payload]

Client → Server types:
  'Q' = Simple Query
  'P' = Parse (extended query)
  'B' = Bind
  'D' = Describe
  'E' = Execute
  'S' = Sync
  'X' = Terminate
  'C' = Close

Server → Client types:
  'R' = Authentication
  'K' = BackendKeyData
  'Z' = ReadyForQuery
  'T' = RowDescription
  'D' = DataRow
  'C' = CommandComplete
  'E' = ErrorResponse
  '1' = ParseComplete
  '2' = BindComplete
  'n' = NoData
```

### Simple Query Protocol

The simplest mode. Send the entire SQL string, get results back:

```
Client → Server:
  'Q' [length] "SELECT id, name FROM users WHERE id = 42\0"

Server → Client:
  'T' RowDescription
    [2 bytes: field count = 2]
    [field 1: name="id",   table_oid, col#, type_oid=23(int4), size=4, ...]
    [field 2: name="name", table_oid, col#, type_oid=25(text), size=-1, ...]
  'D' DataRow
    [2 bytes: column count = 2]
    [4 bytes: col 1 length = 2]["42"]     ← text format, not binary
    [4 bytes: col 2 length = 5]["alice"]
  'C' CommandComplete "SELECT 1"
  'Z' ReadyForQuery 'I'
```

**All values are sent as text** in the simple query protocol. The
integer 42 is sent as the ASCII string "42" (2 bytes), not the binary
representation 0x0000002A (4 bytes). The client library (libpq, node-pg,
psycopg) parses these text representations into native types.

### Extended Query Protocol

The extended protocol separates parsing from execution, enabling
**prepared statements**:

```
Step 1 — Parse: Create a prepared statement
  Client: 'P' [name="s1"][query="SELECT * FROM users WHERE id = $1"][param_types=[int4]]
  Server: '1' ParseComplete

Step 2 — Bind: Bind parameters to the prepared statement, creating a portal
  Client: 'B' [portal=""][statement="s1"][params=["42"]][result_format=text]
  Server: '2' BindComplete

Step 3 — Describe (optional): Get column metadata
  Client: 'D' [type='P'][name=""]
  Server: 'T' RowDescription [...]

Step 4 — Execute: Run the portal
  Client: 'E' [portal=""][max_rows=0]  (0 = all rows)
  Server: 'D' DataRow [...]
  Server: 'C' CommandComplete

Step 5 — Sync: Signal end of the message batch
  Client: 'S'
  Server: 'Z' ReadyForQuery 'I'
```

**Why extended matters:**

1. **Prepared statements avoid re-parsing and re-planning**. Parse once,
   execute many times with different parameters. The plan is cached
   (after 5 executions with custom plans, PostgreSQL switches to a
   generic plan if it's not much worse).

2. **SQL injection prevention**. Parameters are sent separately from the
   query — there's no way for a parameter value to escape into the SQL
   syntax. The server receives the query structure and the parameter
   values independently. This is why parameterized queries prevent SQL
   injection at the protocol level, not just by escaping.

3. **Binary format**. The Bind message can request binary result format
   instead of text. The integer 42 is sent as 4 bytes (0x0000002A)
   instead of 2 bytes of ASCII. For large result sets of numeric data,
   binary format saves bandwidth and parsing overhead.

### COPY Protocol

Bulk data loading, bypassing the normal query path:

```
Client: 'Q' "COPY users FROM STDIN WITH (FORMAT csv)"
Server: 'G' CopyInResponse [format=text, columns=3]
Client: 'd' CopyData "1,alice,alice@example.com\n"
Client: 'd' CopyData "2,bob,bob@example.com\n"
Client: 'd' CopyData "3,carol,carol@example.com\n"
Client: 'c' CopyDone
Server: 'C' CommandComplete "COPY 3"
Server: 'Z' ReadyForQuery 'I'
```

COPY bypasses the SQL parser and executor for each row. Data streams
directly into the table's heap pages. This is 10-100x faster than
individual INSERT statements for bulk loading. COPY also supports
binary format (more compact but less portable) and can stream to/from
files on the server.

---

## The Complete Write Path

What happens, byte by byte, when you execute:

```sql
INSERT INTO users (name, email) VALUES ('alice', 'alice@example.com');
```

### 1. Parse

The parser tokenizes the SQL and builds an AST:

```
InsertStmt
  relation: RangeVar("users")
  cols: [name, email]
  selectStmt: ValuesClause([Const("alice"), Const("alice@example.com")])
```

### 2. Analyze

The analyzer resolves names and types:
- `users` → `pg_class` entry (OID 16384, relkind='r')
- `name` → column 1, type text (OID 25)
- `email` → column 2, type text (OID 25)
- Permission check: current user has INSERT on `users`

### 3. Rewrite

The rewriter applies rules (e.g., rules from views, triggers). For a
simple INSERT on a base table, this is a no-op.

### 4. Plan

The planner creates an execution plan:

```
ModifyTable (operation=INSERT)
  → Result (values to insert)
```

For a simple INSERT, there's not much to plan. For `INSERT ... SELECT`,
the SELECT part goes through full planning/optimization.

### 5. Execute — The Critical Path

**5a. Begin implicit transaction** (if not already in one):
- Acquire a transaction ID (XID) from the XID counter
- Record in the proc array (shared memory) that this XID is active

**5b. Find a page with free space:**
- Consult the **Free Space Map (FSM)**: a binary tree tracking free
  space per page. FSM returns a page with enough room.
- If no page has space: extend the relation (add a new page to the file)

**5c. Pin and lock the target page:**
- Pin the page in the buffer pool (prevent eviction)
- Acquire an exclusive lock (buffer content lock) on the page

**5d. Build the tuple:**

```
HeapTupleHeader (23 bytes):
  t_xmin:    current XID (this transaction created the tuple)
  t_xmax:    0 (not deleted)
  t_cid:     command ID within the transaction (for subtransaction visibility)
  t_ctid:    (page_number, item_number) — self-referencing for new tuples
  t_infomask: flags (has nulls, has varlen attrs, etc.)
  t_hoff:    offset to user data
  t_bits:    null bitmap (if any columns are nullable)

User data:
  [4 bytes: varlena header for 'alice'][5 bytes: "alice"]
  [4 bytes: varlena header for 'alice@example.com'][17 bytes: "alice@example.com"]
```

**5e. Write WAL record FIRST:**
- Construct an xl_heap_insert WAL record:
  ```
  [WAL header: LSN, XID, resource manager=heap, info=XLOG_HEAP_INSERT]
  [target: relation OID, page, offset]
  [tuple data: the complete tuple bytes]
  ```
- Write to WAL buffers in shared memory
- WAL is NOT flushed to disk yet (that happens at COMMIT)

**5f. Insert into the heap page:**
- Add an item pointer to the page's line pointer array
- Copy the tuple data into the page's free space
- Update the page header (free space pointers, item count)
- Set the page's LSN to the WAL record's LSN
- Mark the page as dirty in the buffer pool

**5g. Update indexes:**
For each index on the table:
- Build the index tuple (key values + TID of the new heap tuple)
- Traverse the B+tree to find the correct leaf page
- Insert the index tuple (may trigger page splits)
- Each index insertion also writes its own WAL record

**5h. Check constraints:**
- NOT NULL: checked during tuple construction
- UNIQUE: checked during index insertion (the index rejects duplicates)
- CHECK constraints: evaluated against the tuple values
- Foreign keys: may require a lookup on the referenced table

**5i. Fire triggers** (if any AFTER INSERT triggers exist)

### 6. Commit

```
1. Write a COMMIT WAL record to the WAL buffer
2. Flush WAL buffers to disk (fsync)
   → This is the durability guarantee. Once fsync returns, the
     commit is durable even if the machine loses power.
3. Update CLOG: mark this XID as committed (set bits to 01)
4. Release all locks held by the transaction
5. Unpin any pinned buffer pages
6. Send CommandComplete to the client
```

**The dirty heap and index pages are NOT flushed.** They remain in the
buffer pool and will be written to disk later by the background writer
or checkpointer. The WAL is the durability guarantee — if the database
crashes, ARIES recovery replays the WAL to reconstruct the pages.

---

## The Complete Read Path

```sql
SELECT name, email FROM users WHERE id = 42;
```

Assuming a B+tree index on `id`:

### 1. Parse → Analyze → Rewrite → Plan

Plan chosen: Index Scan on users_pkey (id = 42)

### 2. Execute

**2a. Index traversal:**
- Read root page of the B+tree index (almost certainly in buffer pool)
- Compare 42 with separator keys, follow the correct child pointer
- Read internal page (likely in buffer pool)
- Follow pointer to leaf page
- Read leaf page, find key 42 → TID = (page 100, item 3)

**2b. Heap fetch:**
- Read heap page 100 from buffer pool (or disk if not cached)
- Follow item pointer 3 to the tuple
- **Visibility check:** Is this tuple visible to our snapshot?
  ```
  Tuple: xmin=50, xmax=0
  Our snapshot XID: 200
  Active transactions at snapshot time: [195, 198]

  Check:
    xmin=50 is committed? → check CLOG (or hint bits) → yes
    xmin=50 < our snapshot? → yes (50 < 200)
    xmax=0? → yes (not deleted)
    → Tuple IS visible ✓
  ```
- Extract `name` and `email` columns from the tuple data
- Apply any output expressions or formatting

**2c. Return to client:**
- RowDescription message (column names, types)
- DataRow message (column values as text)
- CommandComplete "SELECT 1"
- ReadyForQuery

### Index-Only Scan Optimization

If the index includes all needed columns (covering index), the heap
fetch can be skipped:

```sql
CREATE INDEX idx_users_id_name_email ON users(id) INCLUDE (name, email);
SELECT name, email FROM users WHERE id = 42;
```

But there's a catch: **MVCC visibility**. The index doesn't have
`xmin`/`xmax` — it doesn't know if the tuple is visible. PostgreSQL
uses the **visibility map** to check:

```
If the visibility map says page 100 is "all-visible":
  → Every tuple on that page is visible to all transactions
  → Skip the heap fetch, read directly from the index leaf
If not all-visible:
  → Must fetch the heap page to check tuple visibility
```

This is why VACUUM maintaining the visibility map matters for
index-only scan performance.

---

## The Executor Model — Volcano/Iterator

PostgreSQL's executor uses the **Volcano model** (also called the
iterator model or pull-based execution). Each node in the execution
plan is an iterator with three operations:

```
Init():  Prepare to produce tuples (allocate state, open files)
Next():  Return the next tuple (or NULL if done)
Close(): Clean up (free state, close files)
```

Nodes form a tree. The root calls Next() on its child, which calls
Next() on its children, recursively. Tuples flow **up** the tree,
one at a time.

```
Query: SELECT name FROM users WHERE age > 30 ORDER BY name LIMIT 10;

Execution tree:

    Limit (count=10)
      │
      │ Next() → pulls one tuple at a time
      ▼
    Sort (key=name)
      │
      │ Next() → must consume ALL input before returning first tuple
      ▼        (pipeline breaker)
    Filter (age > 30)
      │
      │ Next() → passes through tuples that match
      ▼
    SeqScan (users)
      │
      │ Next() → reads next tuple from heap pages
      ▼
    [Heap Pages]
```

### Pipeline Breakers

Most operators are **pipelining** — they process and return tuples one
at a time without buffering the entire input. Filter, Projection, and
Limit are pipelining.

**Pipeline breakers** (also called **blocking operators**) must consume
all input before producing any output:

| Operator | Why It Blocks |
|----------|---------------|
| Sort | Must see all tuples to sort them |
| HashAggregate | Must see all groups to compute aggregates |
| Hash Join (build phase) | Must build the entire hash table before probing |
| Materialize | Stores entire input for re-reading |

The Sort node above calls Next() on the Filter node repeatedly until it
returns NULL (no more tuples). Then it sorts all collected tuples and
starts returning them one at a time to the Limit node. The Limit node
stops pulling after 10 tuples.

**Top-N sort optimization**: When a Sort feeds a Limit, PostgreSQL uses
a **top-N heapsort** — it maintains a min-heap of size N=10, and for
each incoming tuple, inserts it and evicts the smallest if the heap
exceeds N. This uses O(N) memory instead of O(total rows). You see
this in EXPLAIN as `Sort Method: top-N heapsort`.

### Materialization

Some plans require reading the same data twice. The **Materialize**
node stores tuples in memory (or on disk if they exceed work_mem) so
they can be re-scanned without re-executing the child node.

```
Nested Loop Join:
  → Outer: SeqScan on orders (1000 rows)
  → Inner: Materialize
              → SeqScan on products (50 rows)

Without Materialize: SeqScan on products runs 1000 times
With Materialize: SeqScan on products runs once, tuples are cached
```

### Parallel Query Execution

The Volcano model is inherently single-threaded — one backend process
pulls tuples through the tree. PostgreSQL 9.6+ breaks this limitation
with **parallel query**: the leader process launches worker processes
that each execute a portion of the plan subtree, feeding results back
to the leader through shared memory.

**The Gather node** bridges serial and parallel execution:

```
Serial plan:                    Parallel plan:

    Aggregate                       Finalize Aggregate
      │                                  │
    SeqScan (users)                 Gather (workers=3)
      │                              / │ \
    [All pages]               Worker1  Worker2  Worker3
                                │        │        │
                           Partial     Partial   Partial
                           Aggregate   Aggregate Aggregate
                                │        │        │
                           Parallel    Parallel  Parallel
                           SeqScan     SeqScan   SeqScan
                           (pages     (pages    (pages
                            0-99)     100-199)   200-299)
```

Each worker is a full backend process (forked by the postmaster) that
shares the leader's transaction snapshot. Workers communicate with the
leader through **shared memory tuplequeues** — lock-free ring buffers
where workers deposit tuples and the leader (via the Gather node) pulls
them.

**How parallel seq scan divides work:**

```
Table: 300 pages

No static partitioning. Instead, workers use a shared atomic counter:

  ParallelBlockTableScan:
    phs_nblocks: 300        (total pages)
    phs_startblock: 0       (random start to avoid I/O contention)
    phs_nallocated: (atomic) (next page to claim)

  Worker execution loop:
    while true:
      page = atomic_fetch_add(&phs_nallocated, 1)
      if page >= 300: break
      scan page, emit matching tuples

Workers dynamically claim pages one at a time (or in small chunks).
Fast workers process more pages. Slow workers (preempted by OS, slow
disk) process fewer. This naturally load-balances without static
assignment.
```

**Parallel-aware operators** — nodes that know how to split work:

| Operator | Parallel Behavior |
|----------|-------------------|
| Parallel Seq Scan | Workers divide pages via atomic counter |
| Parallel Index Scan | Workers divide index leaf pages |
| Parallel Bitmap Heap Scan | Leader builds bitmap, workers divide heap pages |
| Parallel Hash Join | Workers cooperatively build a shared hash table, then each probes independently |
| Partial Aggregate | Each worker aggregates its own partition, leader combines partial results |
| Parallel Append | Workers claim partitions dynamically (for partitioned table scans) |

**Parallel Hash Join** (PostgreSQL 11+) is the most complex parallel
operator. Multiple workers build a **single shared hash table**
cooperatively:

```
Build phase (cooperative):
  Shared hash table in DSM (dynamic shared memory)

  1. Leader allocates hash buckets in shared memory
  2. All workers + leader scan the inner relation in parallel
  3. Each process inserts tuples into the shared hash table using
     atomic operations (no locking per insert — each bucket is
     a lock-free linked list with compare-and-swap)
  4. Barrier: all must finish building before probe begins

Probe phase (independent):
  1. Each worker scans a portion of the outer relation
  2. Each probes the shared hash table independently
  3. No synchronization needed (hash table is read-only during probe)

Multiple batches (when hash table exceeds work_mem):
  1. Both inner and outer are partitioned by hash into N batches
  2. Batch 0 is built and probed in memory
  3. Remaining batches are written to temp files
  4. Workers cooperatively process remaining batches one at a time
```

**Gather Merge** preserves sort order from parallel workers:

```
Query: SELECT * FROM events ORDER BY created_at LIMIT 100;

Plan:
    Limit (count=100)
      │
    Gather Merge (workers=3)     ← merge-sorts 3+1 sorted streams
      │
    [Worker 1]  [Worker 2]  [Worker 3]  [Leader]
      │            │            │           │
    Sort         Sort         Sort        Sort
      │            │            │           │
    Parallel     Parallel    Parallel    Parallel
    SeqScan      SeqScan     SeqScan     SeqScan

Gather Merge maintains a binary heap of the next tuple from each
worker. Each Next() call extracts the minimum, maintaining global
sort order. Total sort work is divided by worker count, and the
merge is O(log W) per tuple where W = number of workers.
```

**When PostgreSQL uses parallel query:**

```
Conditions (ALL must be met):
  max_parallel_workers_per_gather > 0  (default: 2)
  max_parallel_workers > 0             (default: 8)
  parallel_tuple_cost and parallel_setup_cost are reasonable
  Table size > min_parallel_table_scan_size (default: 8 MB)
  Not inside a serializable transaction (SSI conflicts with parallel)
  Not a cursor (cursors fetch incrementally, parallel is batch)
  No writes (parallel SELECT only, not INSERT/UPDATE/DELETE)
  No functions marked PARALLEL UNSAFE in the query

Worker count decision:
  Based on table size with log₃ scaling:
  8 MB - 24 MB:   1 worker
  24 MB - 72 MB:  2 workers
  72 MB - 216 MB: 3 workers
  ...up to max_parallel_workers_per_gather

  rel_size / (3 ^ nworkers) >= min_parallel_table_scan_size
```

**The cost model for parallel plans:**

```
Regular seq scan:
  cost = seq_page_cost × pages + cpu_tuple_cost × rows
       = 1.0 × 10000 + 0.01 × 1000000
       = 20,000

Parallel seq scan (3 workers + leader = 4 processes):
  cost = seq_page_cost × pages
       + (cpu_tuple_cost × rows) / 4      ← CPU work divided
       + parallel_tuple_cost × rows        ← tuple transfer overhead
       + parallel_setup_cost               ← worker launch overhead
       = 1.0 × 10000
       + (0.01 × 1000000) / 4
       + 0.1 × 1000000                    ← default 0.1 per tuple
       + 1000                              ← default 1000.0
       = 10000 + 2500 + 100000 + 1000
       = 113,500

Wait — that's higher than sequential! This is because the default
parallel_tuple_cost (0.1) is conservative. In practice, the CPU work
per tuple is often much higher (aggregations, complex expressions),
making the division by worker count a bigger win.

With a complex aggregate:
  Serial:   cpu_cost = 0.05 × 1M = 50,000
  Parallel: cpu_cost = 0.05 × 1M / 4 + 0.1 × 1M = 12,500 + 100,000
  Still not great with defaults. Many practitioners lower
  parallel_tuple_cost to 0.01 based on benchmarks.
```

**Shared memory infrastructure for parallel workers:**

```
Dynamic Shared Memory (DSM):
  Each parallel plan allocates a DSM segment containing:

  ┌─────────────────────────────────────┐
  │ ParallelContext                      │
  │   worker_count, transaction snapshot │
  │   combo_cid (command ID mapping)     │
  ├─────────────────────────────────────┤
  │ Tuple Queues (one per worker)       │
  │   shm_mq (shared memory message     │
  │   queue): lock-free ring buffer     │
  │   [header][tuple][tuple][...]       │
  ├─────────────────────────────────────┤
  │ Parallel State per node             │
  │   ParallelBlockTableScan: counter   │
  │   Parallel Hash: shared hash table  │
  │   Barrier nodes: synchronization    │
  ├─────────────────────────────────────┤
  │ Error Reporting                     │
  │   Workers write error info here     │
  │   Leader reads and re-throws        │
  └─────────────────────────────────────┘
```

Workers attach to the leader's transaction using **parallel worker
transaction** — they see the same snapshot (same xmin, same set of
in-progress transactions) but have their own command IDs. Each worker
runs within the leader's transaction boundaries but cannot COMMIT or
ABORT — only the leader controls the transaction.

**EXPLAIN ANALYZE with parallel:**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT status, COUNT(*) FROM events GROUP BY status;
```

```
Finalize HashAggregate (actual time=45.2..45.3 rows=5 loops=1)
  Group Key: status
  Batches: 1  Memory Usage: 24kB
  -> Gather (actual time=20.1..44.8 rows=15 loops=1)
        Workers Planned: 3
        Workers Launched: 3              ← all 3 actually started
        -> Partial HashAggregate (actual time=18.2..18.3 rows=5 loops=4)
              Group Key: status                              ↑
              Batches: 1  Memory Usage: 24kB          loops=4 means
              -> Parallel Seq Scan on events            3 workers + leader
                    (actual time=0.03..12.1 rows=250000 loops=4)
                    Buffers: shared hit=8334
Planning Time: 0.2 ms
Execution Time: 45.5 ms
```

Key observations:
- `loops=4`: 3 workers + 1 leader, each executed the node
- `rows=250000` is per-loop (each process scanned ~250K rows)
- Total rows = 250000 × 4 = 1,000,000
- `Workers Launched: 3` — all requested workers were available
  (if the system is under load, fewer may launch)
- `Buffers: shared hit=8334` — total across all workers
  (each scanned ~2083 pages of the ~8334 total)

### JIT Compilation

PostgreSQL 11+ includes a **JIT compiler** (using LLVM) that compiles
query expressions into native machine code at runtime, replacing the
interpreted expression evaluator.

**What the interpreter does** (without JIT):

```
For each tuple:
  1. Walk the expression tree node by node
  2. At each node: switch on node type, call the handler function
  3. Each handler: extract arguments, type-check, compute, return

Example: WHERE (a + b) > (c * 2)

Expression tree:
    OpExpr (>)
    ├── OpExpr (+)
    │   ├── Var (a) → fetch column a from tuple
    │   └── Var (b) → fetch column b from tuple
    └── OpExpr (*)
        ├── Var (c) → fetch column c from tuple
        └── Const (2)

Per-tuple overhead:
  - 5 function calls (one per node)
  - 5 switch/dispatch decisions
  - Null checks at every node
  - Datum boxing/unboxing
```

This overhead is tiny per tuple but multiplied by millions of tuples
in an analytical query, it dominates CPU time. The expression evaluator
spends more time in dispatch overhead than in actual computation.

**What JIT does:**

```
At plan time (after a cost threshold is exceeded):
  1. Translate the expression tree to LLVM IR (intermediate representation)
  2. LLVM optimizes the IR (inline functions, eliminate dead code,
     constant folding, vectorize where possible)
  3. Compile IR to native machine code for this CPU architecture

Generated code for WHERE (a + b) > (c * 2):
  ; Pseudo-assembly (simplified from actual LLVM IR):
  load a from tuple_slot + offset_a
  load b from tuple_slot + offset_b
  add  result1 = a + b
  load c from tuple_slot + offset_c
  shl  result2 = c << 1          ; * 2 optimized to shift
  cmp  result1 > result2
  ret  bool

  No function calls. No dispatch. No null checks (if columns are NOT NULL).
  Direct memory access at known offsets. One tight loop per tuple.
```

**Three levels of JIT optimization:**

| Level | What's JIT'd | Controlled By |
|-------|-------------|---------------|
| Expression evaluation | WHERE clauses, aggregation, projections | `jit_above_cost` (default 100,000) |
| Tuple deforming | Extracting column values from heap tuples | `jit_optimize_above_cost` (default 500,000) |
| Operator inlining | Inlining type-specific operator functions (int4_add, float8_mul) | `jit_inline_above_cost` (default 500,000) |

**Tuple deforming** deserves special attention. PostgreSQL heap tuples
store columns sequentially with variable-length headers. Extracting
column N requires walking past columns 0 through N-1 (checking each
for NULL and variable-length). The interpreted version calls
`slot_getattr()` which contains loops and branches:

```c
// Interpreted tuple deforming (simplified):
for (int i = 0; i <= attnum; i++) {
    if (hasnulls && att_isnull(i, bp))
        values[i] = (Datum) 0;
    else {
        values[i] = fetchatt(att[i], tp + off);
        off = att_addlength_pointer(off, att[i]->attlen, tp + off);
        off = att_align_nominal(off, att[i]->attalign);
    }
}
```

JIT-compiled tuple deforming generates straight-line code with known
column offsets (for fixed-width columns), eliminating the loop:

```
  ; JIT'd deforming for table with (int4, int4, text):
  load col0 from tuple + 0     ; int4, known offset, 4 bytes
  load col1 from tuple + 4     ; int4, known offset, 4 bytes
  load col2_len from tuple + 8 ; text varlena header
  load col2 from tuple + 12    ; text data
  ; No loop, no branches, no function calls
```

**When JIT hurts:**

```
Compilation cost: 5-50ms per query (LLVM compilation is not free)

OLTP queries (simple point lookups returning few rows):
  Without JIT: 0.1ms execution
  With JIT:    5ms compilation + 0.05ms execution = 5.05ms ← 50x slower

OLAP queries (scanning millions of rows):
  Without JIT: 2000ms execution
  With JIT:    30ms compilation + 800ms execution = 830ms ← 2.4x faster

This is why jit_above_cost defaults to 100,000 — only queries with
estimated cost exceeding this threshold trigger JIT. The compilation
cost must be amortized over enough tuples to be worthwhile.
```

**EXPLAIN shows JIT activity:**

```sql
SET jit = on;
SET jit_above_cost = 0;  -- force JIT for demonstration
EXPLAIN (ANALYZE) SELECT SUM(amount) FROM transactions WHERE status = 'completed';
```

```
Aggregate (actual time=312.5..312.5 rows=1 loops=1)
  -> Seq Scan on transactions (actual time=0.05..245.2 rows=4500000 loops=1)
        Filter: (status = 'completed')
        Rows Removed by Filter: 500000
JIT:
  Functions: 4
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 1.2 ms, Inlining 3.5 ms, Optimization 8.2 ms,
          Emission 12.1 ms, Total 25.0 ms
```

Generation → Inlining → Optimization → Emission (machine code
generation). Total 25ms of JIT overhead, worthwhile against 312ms
of execution.

---

## Lock Manager Internals

PostgreSQL has three layers of locking, each at a different level:

### Spinlocks (Lowest Level)

CPU-level atomic test-and-set operations. Protect very short critical
sections (a few instructions). A thread that fails to acquire a
spinlock spins in a tight loop (busy-wait) until it succeeds.

```c
// Simplified spinlock (actual uses platform-specific atomics)
while (atomic_test_and_set(&lock) == ALREADY_SET) {
    // spin — burn CPU cycles waiting
    // after too many spins, yield to OS scheduler
}
// critical section (a few instructions)
atomic_clear(&lock);
```

Spinlocks are never held across I/O or system calls. They protect
shared data structures for nanoseconds. If you hold a spinlock and
do anything slow, every waiting process burns CPU.

### Lightweight Locks (LWLocks)

Protect shared memory data structures: buffer pool headers, WAL
buffers, lock table, CLOG, proc array. Two modes:

```
LW_SHARED:    Multiple readers concurrently (read the buffer pool hash table)
LW_EXCLUSIVE: Single writer (modify the buffer pool hash table)
```

LWLocks use a combination of spinlock + OS semaphore:
1. Try to acquire via atomic operation (fast path)
2. If contended, add self to a wait queue and sleep on a semaphore
3. When released, the lock holder wakes up the next waiter

LWLocks are internal to PostgreSQL — not visible to users. They don't
have deadlock detection (lock ordering conventions prevent deadlocks).
You can see LWLock waits in `pg_stat_activity.wait_event`:

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock';
```

Common LWLock waits:
- `BufferContent`: Waiting for exclusive access to a buffer page
- `WALInsertLock`: Waiting to write a WAL record
- `BufferMapping`: Waiting to look up a page in the buffer hash table
- `lock_manager`: Waiting to access the heavyweight lock table

### Heavyweight Locks (Regular Locks)

The locks visible to users: row locks, table locks, advisory locks.
Managed by the lock manager in shared memory.

**Lock table structure:**

```
Shared Memory:
┌────────────────────────────────────────────────┐
│ Lock Hash Table                                 │
│   Hash(lock_tag) → LOCK structure               │
│                                                 │
│   LOCK {                                        │
│     tag: {relation_oid, page, tuple, xid}       │
│     granted_mask: which lock modes are held      │
│     wait_mask: which lock modes are being waited │
│     procLocks: list of PROCLOCK entries           │
│   }                                              │
│                                                 │
│   PROCLOCK {                                     │
│     tag: {LOCK*, PGPROC*}                        │
│     holdMask: which modes this process holds     │
│     releaseMask: modes to release at txn end     │
│   }                                              │
│                                                 │
│   PGPROC {                                       │
│     pid, xid, waitLock, waitStatus               │
│     links for the deadlock detector's wait-for   │
│     graph traversal                              │
│   }                                              │
└────────────────────────────────────────────────┘
```

**Lock acquisition flow:**
1. Hash the lock tag → find the LOCK entry
2. Check granted_mask: is the requested mode compatible?
3. If compatible: grant immediately, update masks
4. If not compatible: add to wait queue, sleep on semaphore
5. When the holding transaction commits: release locks, wake waiters

**Deadlock detection:**
When a process has waited longer than `deadlock_timeout` (default 1s):
1. Build the wait-for graph from the lock table
2. DFS for cycles
3. If a cycle exists: choose victim (typically the newest transaction
   in the cycle), abort it
4. The aborted transaction releases its locks, unblocking others

### Fast-Path Locks

For the common case of locking a single row or a small number of
resources, PostgreSQL avoids the shared lock table entirely. Each
backend has a small array (16 slots) in its PGPROC structure for
**fast-path locks**:

```
PGPROC.fpLockBits: [slot 0: relation 16384, AccessShareLock]
                   [slot 1: relation 16385, RowExclusiveLock]
                   [slot 2: empty]
                   ...
                   [slot 15: empty]
```

Fast-path locks don't require the lock manager's LWLock — they're local
to the backend process. This avoids contention on the shared lock table
for the most common operations (simple SELECTs and UPDATEs that only
touch a few tables). If a backend needs more than 16 fast-path locks,
it overflows to the shared lock table.

---

## Memory Management — Memory Contexts

PostgreSQL doesn't use malloc/free directly. All memory allocation goes
through **memory contexts** — hierarchical pools that provide bulk
deallocation, leak prevention, and debugging.

```
TopMemoryContext (lives for the entire backend session)
├── MessageContext (reset per client message)
├── CacheMemoryContext (catalog caches, long-lived)
├── TopTransactionContext (reset on transaction end)
│   ├── CurTransactionContext
│   │   ├── ExprContext (per-tuple evaluation, reset per tuple)
│   │   ├── AggContext (accumulator state for aggregates)
│   │   └── ...
│   └── ...
└── ErrorContext (reserved for out-of-memory error reporting)
```

### How Memory Contexts Work

Each context manages a set of memory **blocks** (chunks allocated from
the OS via malloc). Individual allocations come from these blocks:

```
MemoryContext:
  blocks: [Block 1 (8 KB)] → [Block 2 (16 KB)] → [Block 3 (32 KB)]

  palloc(100 bytes):
    1. Find space in current block
    2. If no space: allocate a new block (doubling size)
    3. Return pointer within the block
    4. Track: this allocation belongs to this context

  pfree(ptr):
    1. Mark the space as free within the block (or don't — see below)

  MemoryContextReset(context):
    1. Free ALL blocks at once
    2. Every allocation in this context is gone
    → No need to individually pfree() each allocation
```

**Why this design:**

1. **Bulk deallocation**: At the end of a transaction, all memory in
   TopTransactionContext is freed at once — no need to track individual
   allocations. At the end of processing each tuple in a scan, the
   ExprContext is reset. This is far cheaper than individual frees.

2. **Leak prevention**: If code forgets to free memory, it's freed when
   the context is destroyed. A function that allocates in
   TopTransactionContext can't leak past the transaction boundary.

3. **Debugging**: `MemoryContextStats(TopMemoryContext)` dumps the
   entire context tree with allocation counts and sizes. Essential for
   finding memory bloat.

### palloc vs malloc

```c
// PostgreSQL style:
char *str = palloc(100);        // allocates in CurrentMemoryContext
char *str = palloc0(100);       // allocates and zeros
MemoryContext old = MemoryContextSwitchTo(CacheMemoryContext);
char *cached = palloc(100);     // allocates in CacheMemoryContext
MemoryContextSwitchTo(old);     // switch back

// At transaction end: everything in TopTransactionContext is freed
// No leaks possible (unless you allocated in a longer-lived context)
```

The per-tuple ExprContext is the most critical for performance. A
query scanning 10 million rows evaluates expressions (WHERE predicates,
column projections) for each row. The ExprContext is reset per tuple —
any temporary memory allocated during expression evaluation is freed
without individual pfree calls. This is one reason PostgreSQL can
process millions of rows without running out of memory.

---

## Buffer Pool Internals

### Buffer Tags and the Hash Table

Every page in the buffer pool is identified by a **buffer tag**:

```
BufferTag {
    RelFileLocator: {spcOid, dbOid, relNumber}  — which tablespace/db/relation
    ForkNumber:     MAIN_FORKNUM (data), FSM_FORKNUM, VM_FORKNUM
    BlockNumber:    page number within the relation
}
```

The buffer pool maintains a hash table: `BufferTag → buffer_id`. When
a backend needs a page:

```
1. Compute the BufferTag for the desired page
2. Look up in hash table (requires BufferMapping LWLock in shared mode)
3. If found (cache hit):
   a. Pin the buffer (increment pin count — prevents eviction)
   b. Acquire content lock (shared for reads, exclusive for writes)
   c. Read/modify the page
   d. Release content lock, unpin
4. If not found (cache miss):
   a. Find a victim buffer (clock sweep — see below)
   b. If victim is dirty: write it to disk first (or let bgwriter handle it)
   c. Read the new page from disk into the victim's slot
   d. Insert into hash table
   e. Pin and lock as above
```

### Clock Sweep Algorithm

PostgreSQL uses a clock sweep (approximation of LRU) for buffer
eviction:

```
Buffer pool slots: [0] [1] [2] [3] [4] [5] [6] [7] ...
                                ↑
                           clock hand

Each slot has:
  - usage_count: incremented on access (max 5)
  - pin_count: number of backends currently using this buffer
  - dirty flag: has this page been modified?

Clock sweep to find a victim:
  while true:
    slot = clock_hand
    advance clock_hand
    if slot.pin_count > 0:
      skip (in use, can't evict)
    elif slot.usage_count > 0:
      decrement usage_count (give it another chance)
    else:
      return slot (victim found: unpinned and usage_count = 0)
```

Frequently accessed pages have high usage_count — the clock hand
decrements but doesn't evict them. Rarely accessed pages (one-time
scan) quickly reach 0 and are evicted. This approximates LRU without
maintaining an explicit LRU list.

**The sequential scan problem**: A large sequential scan visits many
pages, each used once. Without protection, these pages would evict
frequently-accessed index pages. PostgreSQL mitigates this with a
**buffer ring**: sequential scans use a small ring of dedicated buffer
slots (~256 KB). Pages enter and leave the ring without touching the
main buffer pool.

---

## File Layout on Disk

PostgreSQL stores data in the `PGDATA` directory:

```
$PGDATA/
├── base/                    ← database directories
│   ├── 1/                   ← template1
│   ├── 12345/               ← your database (OID)
│   │   ├── 16384            ← table data file (relation OID)
│   │   ├── 16384_fsm        ← Free Space Map
│   │   ├── 16384_vm         ← Visibility Map
│   │   ├── 16385            ← index data file
│   │   └── ...
│   └── ...
├── global/                  ← cluster-wide tables (pg_database, pg_authid)
├── pg_wal/                  ← WAL segment files
│   ├── 000000010000000000000001   (16 MB each)
│   ├── 000000010000000000000002
│   └── ...
├── pg_xact/                 ← CLOG (transaction commit status)
├── pg_stat/                 ← statistics files
├── pg_tblspc/               ← tablespace symlinks
├── postgresql.conf          ← main configuration
├── pg_hba.conf              ← host-based authentication
└── postmaster.pid           ← PID file (prevents double-start)
```

### Relation Forks

Each table/index has multiple **forks** (separate files):

| Fork | Suffix | Purpose |
|------|--------|---------|
| Main | (none) | Actual data (heap tuples or index entries) |
| FSM | _fsm | Free Space Map — tracks free space per page |
| VM | _vm | Visibility Map — tracks all-visible pages |
| Init | _init | Unlogged table initialization fork |

### WAL Segment Files

WAL is stored as a sequence of 16 MB segment files (configurable via
`wal_segment_size`). File names encode the timeline and position:

```
000000010000000000000001
│        │              │
│        │              └── segment number (1)
│        └───────────────── high 32 bits of LSN (0)
└────────────────────────── timeline (1)
```

WAL segments are recycled after checkpoint — renamed and reused rather
than deleted and recreated. `pg_wal` should be on fast storage (SSD)
because WAL writes are on the critical commit path.

### Segment Size Limits

A single data file can't exceed 1 GB (configurable at compile time via
`--with-segsize`). Larger relations are split into 1 GB segments:

```
16384        ← first 1 GB
16384.1      ← second 1 GB
16384.2      ← third 1 GB
```

This limit exists for portability (some older filesystems didn't support
files larger than 2 GB). The relation internally is a logical sequence
of 8 KB pages numbered 0, 1, 2, ... regardless of which segment file
they're in.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Wire protocol: type byte + length + payload | Simple query sends text SQL. Extended query separates parse/bind/execute |
| Extended query protocol prevents SQL injection at protocol level | Parameters are sent separately from query structure. No escaping needed |
| COPY bypasses the executor for 10-100x bulk load performance | Streams data directly into heap pages. Always prefer COPY for bulk loads |
| Write path: WAL first, then heap page, then indexes | WAL is the durability guarantee. Dirty pages flushed later by background writer |
| Tuple header: 23 bytes of xmin, xmax, ctid, infomask | Every row carries MVCC metadata. xmin=creator, xmax=deleter |
| Visibility check: xmin committed + in snapshot, xmax not | Requires CLOG lookup (cached as hint bits after first check) |
| Volcano model: Init/Next/Close iterator tree | Tuples pulled up one at a time. Sort/Hash are pipeline breakers |
| Parallel query: Gather node + worker processes via shared memory | Workers divide pages via atomic counter. Parallel hash join builds shared hash table cooperatively |
| Parallel cost model: CPU work ÷ workers, plus tuple transfer overhead | parallel_tuple_cost (0.1) is conservative. Only wins for CPU-heavy queries on large tables |
| JIT compilation: LLVM compiles expressions to native code at runtime | Eliminates interpreter dispatch overhead. 2-3x speedup for analytical queries, hurts OLTP |
| JIT tuple deforming: straight-line code replaces column extraction loop | Fixed-width columns become direct memory loads at known offsets |
| Three lock layers: spinlocks, LWLocks, heavyweight locks | Spinlocks: nanoseconds. LWLocks: internal shared memory. Heavyweight: user-visible |
| Fast-path locks: 16 slots per backend, skip shared lock table | Avoids contention for simple queries touching few tables |
| Memory contexts: hierarchical pools with bulk deallocation | ExprContext reset per tuple. TopTransactionContext freed at txn end |
| Buffer pool: hash table lookup + clock sweep eviction | Buffer ring protects against sequential scan cache pollution |
| File layout: base/OID/relOID, pg_wal/ for WAL segments | 1 GB segments, FSM/VM forks per relation, 16 MB WAL segments |
