# Databases — How Data Is Stored, Queried, and Scaled

## The Core Problem

An application needs to store data so that it survives process crashes,
machine reboots, and disk failures. It needs to retrieve that data
efficiently — by key, by range, by complex predicates. It needs to
modify data safely when multiple clients are reading and writing
simultaneously. And when the data outgrows a single machine, it needs
to distribute across many.

A database is the system that solves all of this. Understanding databases
at the internals level — how bytes are laid out on disk, how indexes
accelerate lookups, how transactions maintain consistency, how queries
are planned and optimized, how data is replicated and partitioned —
is understanding the foundation everything persistent is built on.

---

## The Mental Model: Layers

```
┌─────────────────────────────┐
│       SQL / Query Layer      │  Parse SQL, plan execution, optimize
│                             │  "What's the fastest way to answer this?"
├─────────────────────────────┤
│     Transaction Layer        │  ACID guarantees, isolation levels
│                             │  "How do concurrent writes stay consistent?"
├─────────────────────────────┤
│       Access Methods         │  B-trees, hash indexes, LSM trees
│                             │  "How do I find data without scanning everything?"
├─────────────────────────────┤
│     Buffer Pool / Cache      │  Page cache, dirty page management
│                             │  "How do I minimize disk I/O?"
├─────────────────────────────┤
│       Storage Engine         │  Pages on disk, WAL, data files
│                             │  "How do bytes get to and from the disk?"
├─────────────────────────────┤
│     Replication / Sharding   │  Distributed consensus, partitioning
│                             │  "How does this scale beyond one machine?"
└─────────────────────────────┘
```

---

## The Topics — Bottom Up

```
Start here
│
├── foundations/
│   ├── 01-storage-engines.md
│   │   How data lives on disk. Pages, B-trees vs LSM trees,
│   │   write-ahead logging, buffer pools, compaction. The physical
│   │   layer that everything else builds on.
│   │
│   ├── 02-transactions.md
│   │   ACID guarantees. Isolation levels (read committed through
│   │   serializable). MVCC, 2PL, WAL. How concurrent access stays
│   │   correct. Deadlocks and how to handle them.
│   │
│   ├── 03-query-execution.md
│   │   How SQL becomes a result set. Parsing, binding, planning,
│   │   optimization. Cost-based vs rule-based. EXPLAIN ANALYZE.
│   │   Join algorithms (nested loop, hash, merge). Index selection.
│   │
│   └── 04-sql.md
│       SQL as a language. Execution order, relational algebra,
│       all join types (LATERAL, anti, semi), window functions,
│       recursive CTEs, NULL semantics, data types, DDL internals,
│       zero-downtime migration patterns.
│
├── distributed/
│   ├── 04-replication.md
│   │   Copies of data across machines. Single-leader, multi-leader,
│   │   leaderless. Synchronous vs asynchronous. Consensus (Raft).
│   │   Conflict resolution. Read replicas.
│   │
│   └── 05-sharding.md
│       Splitting data across machines. Hash vs range partitioning.
│       Consistent hashing. Resharding. Cross-shard queries and
│       distributed transactions.
│
└── internals/
    ├── 06-postgresql.md
    │   PostgreSQL architecture: processes, shared memory, catalog,
    │   MVCC implementation, VACUUM, TOAST, connection handling.
    │   The concrete implementation of everything above.
    │
    └── 07-deep-internals.md
        Wire protocol byte format, complete write/read paths,
        Volcano executor model, parallel query, JIT compilation,
        lock manager (spinlocks → LWLocks → heavyweight), memory
        contexts, buffer pool internals, file layout on disk.
```

Each topic builds on the previous. Storage engines explain how data
reaches disk. Transactions explain how concurrent access is safe.
Query execution explains how the database decides what to read.
Replication and sharding explain how to scale beyond one machine.
PostgreSQL ties it all together in a real system you'll actually use.
