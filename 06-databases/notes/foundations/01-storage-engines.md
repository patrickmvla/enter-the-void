# Storage Engines — How Data Lives on Disk

## The Fundamental Constraint

Disks are slow. An SSD random read takes ~100 microseconds. A memory
access takes ~100 nanoseconds — **1,000x faster**. An HDD seek takes
~10 milliseconds — **100,000x slower** than memory.

Everything in storage engine design is a response to this fact. The
entire architecture — pages, buffer pools, B-trees, LSM trees, WAL,
compaction — exists to minimize the number of times the database must
wait for the disk.

```
                  Latency          Relative
L1 cache:         ~1 ns            1x
L2 cache:         ~4 ns            4x
L3 cache:         ~10 ns           10x
Main memory:      ~100 ns          100x
SSD random read:  ~100 μs          100,000x
HDD random read:  ~10 ms           10,000,000x
Network (same DC): ~500 μs         500,000x
Network (cross-continent): ~150 ms  150,000,000x
```

**Sequential access changes everything.** SSDs read sequentially at
~3 GB/s, HDDs at ~200 MB/s. Random reads are slow because the device
must locate the data first (SSD: internal page lookup; HDD: physical
head movement). Sequential reads just stream through contiguous bytes.

This is why databases organize data in **pages** — fixed-size blocks
(typically 4 KB or 8 KB) that align with the disk's natural I/O unit.
Reading one byte costs the same as reading a full page, so you might
as well read the whole page and cache it.

---

## Pages — The Fundamental Unit

A database doesn't read individual rows. It reads **pages** — fixed-size
blocks of data (8 KB in PostgreSQL, 16 KB in InnoDB/MySQL). Every data
structure in the database — tables, indexes, metadata — is stored as
a collection of pages.

### Page Layout (Slotted Page)

The most common page format for heap tables (where rows are stored):

```
┌──────────────────────────────────────────────────┐
│ Page Header (24 bytes in PostgreSQL)              │
│   - Page LSN (Log Sequence Number)                │
│   - Checksum                                      │
│   - Free space pointers                           │
│   - Number of items                               │
├──────────────────────────────────────────────────┤
│ Item Pointers (Line Pointers)                     │
│   [offset, length] [offset, length] [offset, ...]│
│   ↓                ↓                              │
├──────────────────────────────────────────────────┤
│                                                   │
│              Free Space                           │
│                                                   │
├──────────────────────────────────────────────────┤
│ ┌─────────┐ ┌──────────┐ ┌──────────┐           │
│ │ Tuple 3 │ │ Tuple 2  │ │ Tuple 1  │           │
│ └─────────┘ └──────────┘ └──────────┘           │
└──────────────────────────────────────────────────┘
  ↑ grows downward                    grows upward ↑
  (tuples/rows)                    (item pointers)
```

**Why slotted pages?**

Item pointers grow forward (low to high addresses). Tuples grow backward
(high to low addresses). Free space is in the middle. This design allows:

- **Rows can be different sizes** within the same page
- **Rows can be moved within the page** without changing their external
  reference — the item pointer is updated, but anything referencing
  "page 42, slot 3" still works
- **Compaction** squeezes free space by moving tuples together, without
  changing any external references

**The item pointer is the stable identifier.** In PostgreSQL, a tuple's
physical address is a **TID (Tuple ID)**: `(page_number, item_number)`.
Indexes store TIDs pointing to heap pages. When a row is updated (which
in MVCC creates a new version), the item pointer can be redirected to
the new version's location without updating every index.

### Page LSN (Log Sequence Number)

Every page has an LSN — the position in the WAL (Write-Ahead Log) of
the last modification to this page. This is critical for crash recovery:

```
If page LSN < WAL position:
  This page is behind — WAL changes need to be replayed onto it
If page LSN >= WAL position:
  This page is up to date — no replay needed
```

After a crash, the recovery process scans the WAL from the last
checkpoint and replays all changes to pages whose LSN is behind. This
is how databases provide durability without flushing every write to
disk immediately.

---

## B-Trees — The Index Structure

B-trees (and their variant B+trees) are the dominant index structure in
relational databases. Nearly every index you create in PostgreSQL, MySQL,
SQL Server, or Oracle is a B+tree.

### Why B-Trees?

The problem: given a key (e.g., `user_id = 42`), find the corresponding
row without scanning every page. A sequential scan of a 10 GB table
reads millions of pages. An index lookup reads 3-4 pages.

**Binary search trees** are O(log₂ n) but have high branching factor of
2. For 1 million keys, the tree is ~20 levels deep. Each level is a
random disk read. 20 random reads × 100 μs = 2 ms. Acceptable, but
we can do better.

**B-trees** increase the branching factor. Instead of 2 children per
node, a B-tree node fills an entire page (8 KB) with keys. With 8-byte
keys and 8-byte pointers, a single page holds ~500 key-pointer pairs.
The branching factor is ~500.

```
Branching factor: ~500
Keys: 1,000,000

Levels = log₅₀₀(1,000,000) ≈ 2.2

3 levels can index 500³ = 125,000,000 keys
4 levels can index 500⁴ = 62,500,000,000 keys
```

**Three page reads** to find any key among 125 million. And the root
page is always cached in memory, so in practice it's **2 page reads**
for 125 million keys. Even with 62 billion keys, it's 3-4 reads.

### B+Tree Structure

Databases use **B+trees** (a variant of B-trees) where:
- **Internal nodes** contain only keys and child pointers (no data)
- **Leaf nodes** contain keys and pointers to the actual rows (TIDs)
- **Leaf nodes are linked** in a doubly-linked list for range scans

```
                    [  30  |  70  ]              ← Root (internal)
                   /       |       \
         [10|20|30]    [40|50|60|70]    [80|90]  ← Internal nodes
         /  |  |  \    /  |  |  |  \    / |  \
        L1  L2 L3 L4  L5 L6 L7 L8 L9 L10 L11 L12  ← Leaf nodes
        ↔   ↔  ↔  ↔   ↔  ↔  ↔  ↔  ↔   ↔   ↔      (linked list)

Each leaf node:
  [key₁|TID₁] [key₂|TID₂] [key₃|TID₃] ... [→ next leaf]
```

**Lookup: WHERE id = 42**
1. Read root page: 42 is between 30 and 70 → follow middle pointer
2. Read internal page [40|50|60|70]: 42 is between 40 and 50 → follow pointer
3. Read leaf page: scan for key 42 → found → TID = (page 1234, slot 5)
4. Read heap page 1234, slot 5 → actual row data

**4 I/O operations** (root is cached → 3 in practice). Compared to a
sequential scan of potentially thousands of pages.

**Range scan: WHERE id BETWEEN 30 AND 60**
1. Find the leaf node containing key 30 (same traversal as above)
2. Follow the linked list of leaf nodes, reading all keys ≤ 60
3. For each matching key, fetch the row from the heap

The linked list makes range scans efficient — once you find the start,
you just follow pointers forward without going back to the root.

### B+Tree Operations

**Insertion:**
1. Find the leaf node where the key belongs
2. If the leaf has space, insert the key (maintaining sorted order)
3. If the leaf is full, **split** it:
   - Create a new leaf node
   - Move half the keys to the new node
   - Add a pointer to the new node in the parent
   - If the parent is also full, split the parent (cascading up)
   - If the root splits, create a new root (tree grows taller)

```
Before split (leaf full, inserting 25):
  [10|15|20|30]

After split:
  [10|15]  [20|25|30]
     ↑          ↑
  Parent gets new separator key: 20
```

**Deletion:**
1. Find the leaf node containing the key
2. Remove the key
3. If the leaf falls below minimum occupancy (~50%), either:
   - **Merge** with a sibling (if both are below threshold)
   - **Redistribute** keys from a sibling (borrow keys)
4. If merging removes a key from the parent, cascading may occur

In practice, many databases use **lazy deletion** — mark the key as
deleted but don't immediately restructure the tree. The space is
reclaimed during subsequent insertions or background maintenance.

### Fill Factor

The **fill factor** controls how full pages are on initial load or
rebuild. Default is typically 90-100%.

```sql
CREATE INDEX idx_users_email ON users(email) WITH (fillfactor = 70);
```

A fill factor of 70% leaves 30% free space per leaf page. This space
absorbs future inserts without triggering splits. Useful for write-heavy
tables where the index key is not sequential (random UUIDs, emails).

For append-only workloads (auto-incrementing IDs), 100% fill factor is
optimal — new keys always go to the rightmost leaf, never causing splits
in existing pages.

### B+Tree Concurrency: Latching

Multiple threads accessing the same B+tree need synchronization.
Databases use **latches** (lightweight locks, not the transaction-level
locks discussed in the transactions topic):

**Latch crabbing (coupling):**
1. Acquire latch on root
2. Acquire latch on child
3. If the child is "safe" (won't split or merge on this operation),
   release the parent's latch
4. Repeat down to the leaf

"Safe" means: for insertions, the node is not full; for deletions,
the node is above minimum occupancy. This allows concurrent access —
different threads can traverse different parts of the tree simultaneously.

**Optimistic latching** (B-link trees): Assume no splits will happen.
Traverse down with read latches only. At the leaf, take a write latch.
If a split occurred during traversal (detected by checking the node's
right-link pointer), restart. Splits are rare, so this is faster than
pessimistic crabbing in most workloads.

---

## LSM Trees — The Write-Optimized Alternative

### The B-Tree Write Problem

B-tree writes are **random I/O**. Inserting a key into a B+tree
requires:
1. Read the leaf page from disk
2. Modify it in memory
3. Write it back to disk (eventually)

Each insert potentially touches a different page (random location on
disk). For write-heavy workloads (logging, time-series, event ingestion),
random I/O becomes the bottleneck.

### The LSM Tree Idea

LSM (Log-Structured Merge) trees convert random writes into **sequential
writes** by buffering writes in memory and flushing them to disk in
sorted batches.

```
Write path:
  1. Write to in-memory buffer (memtable — usually a red-black tree or skiplist)
  2. Also write to WAL (for durability)
  3. When memtable is full (~64 MB), flush to disk as a sorted, immutable SSTable
  4. Background compaction merges SSTables

Read path:
  1. Check memtable (newest data, in memory)
  2. Check each SSTable from newest to oldest (disk)
  3. First match wins (newest version)
```

### Architecture

```
         ┌────────────────┐
  Write ──>│   Memtable      │  In-memory sorted structure (skiplist)
         │  (active)       │  Writes go here first (fast)
         └───────┬────────┘
                 │ flush when full
         ┌───────▼────────┐
         │  Immutable      │  Being flushed to disk
         │  Memtable       │  New writes go to a fresh memtable
         └───────┬────────┘
                 │ write as SSTable
    ─────────────┼──────────── Disk ──────────────────
                 ▼
Level 0:  [SST₁] [SST₂] [SST₃] [SST₄]     ← recent flushes
          (may have overlapping key ranges)
                 │ compaction
                 ▼
Level 1:  [  SST_A  ] [  SST_B  ] [  SST_C  ]  ← merged, non-overlapping
                 │ compaction
                 ▼
Level 2:  [    SST_X    ] [    SST_Y    ] [    SST_Z    ]  ← larger, fewer
                 │
                 ▼
Level N:  [          SST_HUGE          ]  ← oldest, largest
```

### SSTables (Sorted String Tables)

An SSTable is an immutable, sorted file on disk:

```
┌──────────────────────────────────────────────────┐
│ Data Blocks                                       │
│   [key₁|value₁][key₂|value₂]...[keyₙ|valueₙ]   │
│   Block 1: keys aaa-czz                           │
│   Block 2: keys daa-fzz                           │
│   ...                                             │
├──────────────────────────────────────────────────┤
│ Index Block                                       │
│   [block1_first_key → offset]                    │
│   [block2_first_key → offset]                    │
│   ...                                             │
├──────────────────────────────────────────────────┤
│ Bloom Filter                                      │
│   Probabilistic: "Is key X definitely NOT here?"  │
├──────────────────────────────────────────────────┤
│ Footer / Metadata                                 │
│   Compression type, checksum, format version      │
└──────────────────────────────────────────────────┘
```

**Bloom filters** are critical for read performance. Before reading any
data blocks, the database checks the bloom filter:
- "Definitely not in this SSTable" → skip entirely (no I/O)
- "Possibly in this SSTable" → read the index block, then the data block

A bloom filter with 10 bits per key has a ~1% false positive rate. This
means 99% of unnecessary SSTable reads are avoided.

### Compaction

SSTables accumulate over time. Without compaction:
- Reads get slower (must check more SSTables)
- Disk space grows (obsolete versions aren't reclaimed)
- Bloom filters become less effective (more SSTables to query)

**Compaction** merges SSTables, discarding old versions and deleted keys:

```
Before compaction (Level 0, overlapping ranges):
  SST1: [a:1, b:2, d:4]         (newer)
  SST2: [a:0, c:3, d:3]         (older)

After compaction (merged):
  SST3: [a:1, b:2, c:3, d:4]   (a=1 wins over a=0, d=4 wins over d=3)
```

**Two compaction strategies:**

**Size-tiered compaction** (Cassandra default):
- Merge SSTables of similar sizes together
- Simpler, better write throughput
- Temporarily needs 2x disk space during compaction
- Can have many overlapping SSTables (worse read performance)

**Leveled compaction** (RocksDB, LevelDB):
- Each level (except L0) has non-overlapping key ranges
- Compaction picks an SSTable from level N and merges it into level N+1
- Each level is ~10x larger than the previous
- Better read performance (fewer SSTables to check per key)
- Higher write amplification (data is rewritten multiple times as it
  moves through levels)

### Write Amplification

The hidden cost of LSM trees. A single logical write may cause the data
to be written to disk multiple times:

```
1. Write to WAL                          (1x write)
2. Flush memtable to L0 SSTable          (1x write)
3. Compact L0 → L1                       (1x write)
4. Compact L1 → L2                       (1x write)
5. Compact L2 → L3                       (1x write)

Total: 5x write amplification for leveled compaction
Typical range: 10-30x for real workloads
```

This means writing 1 GB of data may cause 10-30 GB of actual disk
writes. This matters for SSD longevity (SSDs have limited write
endurance) and for I/O throughput.

**B-trees have write amplification too**: typically 2-3x (write the page
+ write the WAL entry). Lower than LSM, but the writes are random
(slower on HDDs).

### Read Amplification

The cost of reading in an LSM tree. A point lookup may need to check:
1. Memtable (memory access — fast)
2. L0 SSTables (potentially overlapping — may check all of them)
3. L1 SSTable (one — non-overlapping ranges)
4. L2 SSTable (one)
5. ...

**Bloom filters reduce this dramatically.** Without bloom filters, a
miss (key doesn't exist) would check every level. With bloom filters,
most levels are skipped.

---

## Write-Ahead Log (WAL)

### The Durability Problem

The buffer pool keeps modified pages in memory (they're "dirty"). If
the database crashes before dirty pages are flushed to disk, data is
lost. But flushing every write immediately would be catastrophically
slow (one disk I/O per write).

### How WAL Solves It

Before modifying any data page, write a description of the change to
the WAL (a sequential, append-only log file). The WAL is flushed to
disk (fsync) before the transaction is acknowledged as committed.

```
Transaction timeline:

  1. BEGIN
  2. UPDATE users SET name='bob' WHERE id=42
     → Write to WAL: "page 1234, slot 5: set name column to 'bob'"
     → Modify page 1234 in buffer pool (memory only — NOT flushed to disk)
  3. COMMIT
     → Write COMMIT record to WAL
     → fsync the WAL file to disk
     → Acknowledge to client: "committed"
  4. (Later, asynchronously)
     → Background writer flushes dirty page 1234 to data file
```

**If the database crashes between steps 3 and 4:** The data file
doesn't have the change, but the WAL does. On recovery:
1. Read the WAL from the last checkpoint
2. Replay all committed changes that aren't in the data files
3. Discard uncommitted changes

**The key insight**: WAL writes are **sequential** (append-only).
Sequential writes to disk are fast (no seeking). Data page writes are
**random** (each page is at a different location). By batching data
page writes and relying on the WAL for durability, the database gets
both performance AND durability.

### WAL Record Format

```
┌──────────┬──────────┬──────────┬──────────────────────┐
│   LSN    │  TxnID   │  Type    │     Data             │
│ (8 bytes)│ (4 bytes)│ (1 byte) │  (variable)          │
└──────────┴──────────┴──────────┴──────────────────────┘

Types: INSERT, UPDATE, DELETE, COMMIT, ABORT, CHECKPOINT
```

The **LSN (Log Sequence Number)** is a monotonically increasing
identifier for each WAL record. Every data page stores the LSN of the
last WAL record applied to it. During recovery, if the page's LSN is
less than a WAL record's LSN, that record needs to be replayed.

### ARIES Recovery Algorithm

ARIES (Algorithms for Recovery and Isolation Exploiting Semantics) is
the standard recovery algorithm used by PostgreSQL, MySQL/InnoDB,
SQL Server, and most modern databases.

Three phases:

**1. Analysis**: Scan the WAL from the last checkpoint. Determine:
   - Which transactions were active at crash time (might need undo)
   - Which pages might be dirty (might need redo)

**2. Redo**: Replay ALL WAL records from the last checkpoint forward,
   regardless of whether the transaction committed. This brings every
   page to its most recent state.

   ```
   For each WAL record:
     If page LSN < record LSN:
       Apply the change (the page is behind)
     Else:
       Skip (the page already has this change)
   ```

**3. Undo**: Roll back all transactions that were active at crash time
   (never committed). Walk backward through the WAL, undoing changes
   from uncommitted transactions.

   ```
   For each uncommitted transaction (from Analysis):
     Walk backward through its WAL records
     For each record: apply the inverse operation
     Write a compensation log record (CLR) for each undo
   ```

**CLRs (Compensation Log Records)**: Undo operations are themselves
logged. If the database crashes DURING recovery (crash during undo),
the CLRs ensure undo operations are not re-done. ARIES guarantees
recovery is idempotent — running it multiple times produces the same
result.

### Checkpoints

Without checkpoints, recovery would replay the ENTIRE WAL from the
beginning of time. Checkpoints limit how far back recovery needs to go.

**Fuzzy checkpoint** (what PostgreSQL and most databases use):
1. Write a CHECKPOINT record to the WAL
2. Record which pages are dirty and which transactions are active
3. The dirty pages are flushed to disk **in the background** (not
   immediately — that would stall everything)
4. Once all pages from before the checkpoint are flushed, the WAL
   before the checkpoint can be recycled

Recovery starts from the last checkpoint, not from the beginning.
Checkpoints happen periodically (every 5 minutes, or every N WAL
segments).

---

## Buffer Pool — The Page Cache

### Why Not Just Use the OS Page Cache?

The operating system already caches disk pages in memory. Why does the
database maintain its own cache?

**Control over eviction**: The OS uses LRU (Least Recently Used) or a
variant. A full table scan loads millions of pages, evicting frequently
accessed index pages. The database knows that a sequential scan's pages
won't be reused and can avoid caching them (or use a separate small
buffer for scans).

**Control over writes**: The database must write the WAL before dirty
pages. The OS might flush dirty pages in any order. If a data page is
flushed before its WAL record, and then a crash occurs, the data page
has changes that can't be recovered (the WAL record is gone). The
database controls the write order to prevent this.

**Pin/unpin semantics**: When a page is being accessed, it must not be
evicted. The database pins pages during use and unpins them when done.
The OS doesn't provide this granularity.

**Prefetching**: The database knows the query plan and can prefetch
pages it will need (e.g., for a sequential scan or index range scan).
The OS can only guess based on access patterns.

### Eviction Policies

**LRU (Least Recently Used)**: Evict the page that hasn't been accessed
for the longest time. Simple but suffers from the sequential scan
problem — a one-time scan of a large table evicts everything useful.

**Clock (approximation of LRU)**: Pages are arranged in a circular
buffer. Each page has a "reference bit" set to 1 on access. The clock
hand sweeps around: if the bit is 1, clear it and move on; if 0, evict.
Cheaper than maintaining a full LRU list. Used by PostgreSQL.

**LRU-K (LRU-2)**: Track the last K accesses to each page. Evict the
page whose K-th most recent access is oldest. LRU-2 (K=2) effectively
separates "accessed once" (scan pages) from "accessed multiple times"
(frequently used index pages). Used by SQL Server.

**ARC (Adaptive Replacement Cache)**: Maintains both a recency list and
a frequency list, dynamically adjusting the size of each based on
workload. Used by ZFS.

### Dirty Page Management

When a page is modified in the buffer pool, it's marked **dirty**. Dirty
pages must eventually be written to disk, but not immediately (that
would defeat the purpose of the buffer pool).

**Background writer**: A dedicated process periodically writes dirty
pages to disk, spreading the I/O over time. This prevents a spike of
I/O when the buffer pool is full and needs to evict a dirty page
(which requires writing it first).

**Double-write buffer** (InnoDB): A page write to disk isn't atomic —
if the database crashes mid-write, the page on disk is corrupt (a
"torn page"). InnoDB writes dirty pages to a sequential double-write
buffer first, then to their actual locations. On crash recovery, if
a page is corrupt, the clean copy from the double-write buffer is used.
PostgreSQL uses full-page writes in the WAL instead (writing the
entire page image to the WAL on first modification after a checkpoint).

---

## B-Tree vs LSM Tree — The Tradeoff

| Property | B-Tree | LSM Tree |
|----------|--------|----------|
| Read latency | Lower (one tree traversal) | Higher (check memtable + multiple SSTables) |
| Write latency | Higher (random I/O to leaf page) | Lower (sequential memtable + WAL) |
| Write amplification | Lower (2-3x) | Higher (10-30x) |
| Read amplification | Lower (1 tree traversal) | Higher (multiple levels, mitigated by bloom filters) |
| Space amplification | Lower (one copy of data + index) | Higher (multiple versions across SSTables until compaction) |
| Range scan | Fast (leaf linked list) | Fast (SSTables are sorted, merge during read) |
| Sequential writes | Yes (WAL), No (data pages) | Yes (all writes are sequential) |
| Concurrency | Latch crabbing/optimistic | Lock-free memtable + immutable SSTables |
| Space reclamation | In-place (page reuse) | Compaction (background) |

**B-trees are better when:**
- Read-heavy workloads (OLTP with many point queries)
- Low-latency reads are critical
- Strong transactional requirements (easier MVCC with in-place pages)
- Disk space is a concern

**LSM trees are better when:**
- Write-heavy workloads (logging, time-series, event streams)
- Write throughput matters more than read latency
- SSD (sequential writes play well with SSD write patterns)
- Compression is important (SSTables compress well — sorted data has
  higher compressibility)

**Who uses what:**
- B-tree: PostgreSQL, MySQL/InnoDB, SQL Server, Oracle
- LSM: RocksDB, LevelDB, Cassandra, ScyllaDB, CockroachDB (uses
  RocksDB/Pebble as storage engine), InfluxDB
- Hybrid: TiDB (B-tree for OLTP + columnar for analytics), many systems
  use B-tree indexes on top of LSM storage

---

## Column-Oriented Storage

Everything above assumes **row-oriented** storage — each page contains
complete rows (all columns of a row are stored together). This is
optimal for OLTP (transactional workloads) where you typically read
or write entire rows.

**Column-oriented** storage stores each column separately. All values
of a single column are stored contiguously:

```
Row-oriented (heap page):
  Page 1: [alice, 28, engineer] [bob, 35, manager] [carol, 42, director]
  → Great for: SELECT * FROM users WHERE id = 42 (one page read)

Column-oriented:
  Name column:  [alice] [bob] [carol] [dave] [eve] ...
  Age column:   [28] [35] [42] [29] [31] ...
  Role column:  [engineer] [manager] [director] [analyst] [designer] ...
  → Great for: SELECT AVG(age) FROM users (read only the age column)
```

### Why Column Storage for Analytics

**I/O efficiency**: `SELECT AVG(age) FROM users` on a row store reads
every column of every row (name, email, address, ...) just to get the
age. Column store reads only the age column — potentially 10-50x less
data from disk.

**Compression**: Values in a column are the same type and often similar.
Run-length encoding, dictionary encoding, delta encoding, and bitmap
encoding achieve 5-10x compression ratios. Compressed data means fewer
pages to read from disk.

```
Dictionary encoding:
  Role column (raw):    [engineer, manager, engineer, director, engineer, ...]
  Dictionary:           {0: engineer, 1: manager, 2: director}
  Encoded:              [0, 1, 0, 2, 0, ...]  (1 byte per value instead of ~8)

Run-length encoding:
  Status column (sorted): [active, active, active, active, inactive, inactive]
  Encoded:                 [(active, 4), (inactive, 2)]
```

**Vectorized execution**: Column stores process data in batches of
values from the same column (vectors). Modern CPUs can apply operations
to vectors using SIMD instructions, processing 4-8 values in a single
CPU instruction. Row stores process one row at a time (tuple-at-a-time).

### Who Uses Column Storage

- **Data warehouses**: Redshift, BigQuery, Snowflake, ClickHouse
- **Analytical engines**: Apache Parquet (file format), Apache Arrow
  (in-memory format)
- **Hybrid (HTAP)**: Some databases support both — PostgreSQL with
  columnar extensions (Citus columnar, pg_columnar), TiDB's TiFlash

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Disk latency drives everything | SSD random read: 100μs. Memory: 100ns. 1000x gap. Minimize disk I/O |
| Pages are the I/O unit | 4-16 KB fixed blocks. Reading 1 byte = reading a full page |
| Slotted pages allow variable-size rows | Item pointers provide stable addresses for row movement |
| B+trees have branching factor ~500 | 3-4 levels index billions of keys. Leaf linked list enables range scans |
| B+tree splits and merges maintain balance | Fill factor controls space for future inserts |
| LSM trees convert random writes to sequential | Memtable → SSTable flush → compaction. Write-optimized |
| Bloom filters prevent unnecessary SSTable reads | 10 bits/key = 1% false positive rate. Critical for LSM read performance |
| Write amplification: LSM 10-30x, B-tree 2-3x | LSM rewrites data through compaction levels. Matters for SSD endurance |
| WAL provides durability without synchronous page flushes | Sequential append. fsync on commit. Replay on crash recovery |
| ARIES: Analysis → Redo → Undo | The standard crash recovery algorithm. CLRs make recovery idempotent |
| Buffer pool gives the DB control OS page cache can't | Eviction policy, write ordering, pinning, prefetching |
| B-tree for read-heavy, LSM for write-heavy | PostgreSQL/MySQL = B-tree. RocksDB/Cassandra = LSM |
| Column stores for analytics, row stores for OLTP | Column: 10-50x less I/O for aggregation queries. Vectorized + compressible |
