# Transactions — Correctness Under Concurrency

## The Problem

A database serves many clients simultaneously. Each client reads and
writes data. Without coordination, concurrent access produces
nonsensical results:

```
Account balance: $1000

Thread A (withdraw $800):        Thread B (withdraw $500):
  1. Read balance: $1000           1. Read balance: $1000
  2. Check: 1000 >= 800 ✓         2. Check: 1000 >= 500 ✓
  3. Write balance: $200           3. Write balance: $500
                                    → Final balance: $500
                                    → $1300 withdrawn from $1000 account
```

A **transaction** is a group of operations that execute as a single
logical unit. Either all operations succeed and their effects are
permanent, or none of them take effect. The database guarantees this
even when many transactions run concurrently and even if the system
crashes mid-transaction.

---

## ACID Properties

### Atomicity

A transaction is **all or nothing**. If any operation within the
transaction fails (constraint violation, disk error, deadlock, explicit
rollback), all changes from the transaction are undone.

**How it's implemented**: The WAL records all changes. On COMMIT, the
changes become permanent. On ABORT (or crash before commit), the undo
log is used to reverse all changes (ARIES undo phase).

Atomicity is NOT about concurrency — it's about failure handling. Even
with a single user, atomicity ensures that a multi-step operation
(transfer money between accounts = debit + credit) either fully
completes or fully reverts.

### Consistency

The database moves from one valid state to another. All constraints
(primary keys, foreign keys, CHECK constraints, unique constraints,
triggers) are satisfied before and after the transaction.

This is partially the **application's** responsibility. The database
enforces declared constraints, but business rules that aren't expressed
as constraints (e.g., "an account balance should match the sum of its
transactions") must be maintained by application logic within the
transaction.

### Isolation

Concurrent transactions don't interfere with each other. Each
transaction sees a consistent snapshot of the database as if it were
the only one running. In reality, the database interleaves operations
from many transactions — but the result is equivalent to some serial
execution order.

Isolation is the most complex ACID property. The level of isolation is
configurable (see Isolation Levels below) because full isolation is
expensive.

### Durability

Once a transaction is committed, its changes survive any subsequent
crash (power failure, kernel panic, disk failure). The data is on
permanent storage.

**How it's implemented**: WAL is fsynced to disk before COMMIT is
acknowledged. Even if the data pages are only in the buffer pool
(not yet written to the data files), the WAL has the information to
reconstruct them.

**Note**: Durability doesn't protect against disk failure. That's what
replication is for (covered in 04-replication.md). Durability means
"survives a crash of the running process or machine," not "survives
permanent hardware destruction."

---

## Concurrency Anomalies

Without isolation, concurrent transactions can observe several classes
of anomalies. Understanding these is essential for choosing the right
isolation level.

### Dirty Read

Transaction A reads data written by Transaction B before B commits.
If B rolls back, A has read data that never existed.

```
Transaction A:                    Transaction B:
                                  UPDATE accounts SET balance = 200
                                    WHERE id = 1;
SELECT balance FROM accounts
  WHERE id = 1;
  → Returns 200 (B's uncommitted change)
                                  ROLLBACK;
  → A has read a value that was never committed
```

### Non-Repeatable Read (Read Skew)

Transaction A reads a row, Transaction B modifies and commits it,
Transaction A reads the same row again and gets a different value.

```
Transaction A:                    Transaction B:
SELECT balance FROM accounts
  WHERE id = 1;
  → Returns 1000
                                  UPDATE accounts SET balance = 200
                                    WHERE id = 1;
                                  COMMIT;
SELECT balance FROM accounts
  WHERE id = 1;
  → Returns 200 (different from first read!)
```

### Phantom Read

Transaction A queries rows matching a predicate, Transaction B inserts
a new row matching that predicate, Transaction A re-queries and sees
a row that wasn't there before.

```
Transaction A:                    Transaction B:
SELECT COUNT(*) FROM orders
  WHERE status = 'pending';
  → Returns 5
                                  INSERT INTO orders (status)
                                    VALUES ('pending');
                                  COMMIT;
SELECT COUNT(*) FROM orders
  WHERE status = 'pending';
  → Returns 6 (phantom row appeared)
```

Non-repeatable reads are about existing rows changing. Phantoms are
about new rows appearing (or existing rows disappearing from a
predicate range).

### Lost Update

Two transactions read the same value, make a decision based on it,
and write back. One write overwrites the other.

```
Transaction A (counter++):       Transaction B (counter++):
  Read counter: 10                 Read counter: 10
  Compute: 10 + 1 = 11            Compute: 10 + 1 = 11
  Write counter: 11                Write counter: 11
  → Counter should be 12, but it's 11. One increment was lost.
```

### Write Skew

Two transactions read overlapping data, make disjoint writes based on
it, and the combined result violates a constraint that neither
transaction individually violated.

```
Constraint: At least one doctor must be on call

Doctor A on call: true           Doctor B on call: true

Transaction A:                    Transaction B:
  Check: 2 doctors on call ✓       Check: 2 doctors on call ✓
  Set doctor A off call             Set doctor B off call
  COMMIT                            COMMIT
  → 0 doctors on call. Constraint violated.
```

Neither transaction saw the other's change. Each individually left one
doctor on call. But together, both went off call. This is the hardest
anomaly to prevent — it requires serializable isolation.

---

## Isolation Levels

SQL defines four isolation levels, each preventing more anomalies at
higher cost:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom | Lost Update | Write Skew |
|----------------|-----------|-------------------|---------|------------|------------|
| Read Uncommitted | Possible | Possible | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible* | Prevented | Possible* |
| Serializable | Prevented | Prevented | Prevented | Prevented | Prevented |

*PostgreSQL's Repeatable Read prevents phantoms through MVCC snapshots
but allows write skew. MySQL/InnoDB's Repeatable Read uses gap locks
to prevent phantoms but may still allow write skew in some cases.

### Read Committed

**What you see**: Each statement within the transaction sees the most
recently committed data at the time the statement starts. Different
statements in the same transaction can see different snapshots.

**How it works**: Before each statement, the database takes a fresh
snapshot of committed data. Any uncommitted changes from other
transactions are invisible.

**This is the default in PostgreSQL.** Most applications run at this
level. It prevents dirty reads but allows non-repeatable reads and
phantoms.

```sql
-- Transaction A at Read Committed:
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- sees 1000
-- Transaction B commits an update: balance = 500
SELECT balance FROM accounts WHERE id = 1;  -- sees 500 (new snapshot!)
COMMIT;
```

### Repeatable Read (Snapshot Isolation)

**What you see**: The entire transaction sees a consistent snapshot
taken at the start of the transaction. All reads within the transaction
see the same data, regardless of concurrent commits.

**How it works**: The database takes a snapshot at transaction start (or
at the first statement). All reads return data from that snapshot. If
another transaction commits a change after your snapshot, you don't see
it.

```sql
-- Transaction A at Repeatable Read:
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- sees 1000
-- Transaction B commits an update: balance = 500
SELECT balance FROM accounts WHERE id = 1;  -- STILL sees 1000 (same snapshot)
COMMIT;
```

**Write conflicts**: If Transaction A tries to update a row that
Transaction B has modified and committed since A's snapshot, A gets
a serialization error and must retry:

```
ERROR: could not serialize access due to concurrent update
```

This prevents lost updates but not write skew.

### Serializable

**What you see**: The result is equivalent to some serial execution of
the transactions. All anomalies are prevented.

**How it works**: Two approaches:

**1. Serializable Snapshot Isolation (SSI)** — PostgreSQL's approach:
Run transactions at snapshot isolation, but track read and write sets.
At commit time, check if the transaction's reads would have been
different if any concurrently committed transaction's writes had been
visible. If so, abort and retry.

SSI is **optimistic** — transactions proceed without blocking, and
conflicts are detected at commit time. This means:
- No deadlocks from serializable isolation itself
- Some transactions will be aborted and must be retried
- Higher throughput under low contention (most transactions succeed)
- More aborts under high contention

**2. Two-Phase Locking (2PL)** — MySQL/InnoDB's approach (for true
serializability):
Transactions acquire locks on data they read and write. Locks are held
until the transaction commits (two phases: growing phase where locks
are acquired, shrinking phase where they're released at commit).

2PL is **pessimistic** — conflicts block rather than abort. This means:
- Deadlocks are possible (two transactions waiting for each other's locks)
- Lower throughput under contention (waiting for locks)
- No false aborts (if you get through, you're guaranteed to commit)

---

## MVCC — Multi-Version Concurrency Control

MVCC is the mechanism behind snapshot isolation. Instead of overwriting
data in place, the database keeps **multiple versions** of each row.
Readers see the version that was current at their snapshot time. Writers
create new versions. Readers never block writers, and writers never
block readers.

### PostgreSQL's MVCC

PostgreSQL stores multiple physical copies of each row in the heap.
Each row version (tuple) has hidden system columns:

```
Tuple header:
  xmin:  Transaction ID that created this version (INSERT or UPDATE)
  xmax:  Transaction ID that deleted/replaced this version
         (0 if the version is still live)
  ctid:  Current Tuple ID (physical location). Points to the next
         version if this one was updated.

Example:

UPDATE users SET name='bob' WHERE id=42;

Before (original tuple, page 100, slot 3):
  xmin=100, xmax=0, ctid=(100,3), data={id:42, name:'alice'}

After:
  Old tuple (page 100, slot 3):
    xmin=100, xmax=200, ctid=(100,4)  ← "killed" by txn 200
  New tuple (page 100, slot 4):
    xmin=200, xmax=0, ctid=(100,4), data={id:42, name:'bob'}
```

### Visibility Rules

A tuple is visible to Transaction T if:
1. `xmin` is committed AND `xmin` < T's snapshot
2. `xmax` is either 0 (not deleted) or the deleting transaction
   hasn't committed OR committed after T's snapshot

```
Snapshot at time 150:
  Tuple (xmin=100, xmax=0):     Visible ✓ (created before 150, not deleted)
  Tuple (xmin=100, xmax=200):   Visible ✓ (xmax=200 > 150, deletion not visible)
  Tuple (xmin=200, xmax=0):     Not visible ✗ (created after snapshot)
  Tuple (xmin=100, xmax=120):   Not visible ✗ (deleted before snapshot, if 120 committed)
```

**The commit log (CLOG/pg_xact)**: A bitmap that records whether each
transaction committed or aborted. Two bits per transaction:
- 00 = in progress
- 01 = committed
- 10 = aborted
- 11 = sub-committed (subtransaction)

Checking visibility requires consulting the CLOG. Recent entries are
cached in shared memory. Once a tuple's `xmin` is known to be committed
and visible to all active transactions, the tuple is marked with a
"hint bit" in its header — subsequent visibility checks skip the CLOG
lookup.

### The VACUUM Problem

PostgreSQL's MVCC creates dead tuples — old versions that are no longer
visible to any transaction. These dead tuples waste space and slow down
sequential scans (the database must skip over them).

**VACUUM** reclaims dead tuples:
1. Scan the table for dead tuples (visible to no active transaction)
2. Mark their space as reusable in the page's free space map
3. Update the visibility map (tracks which pages are all-visible —
   enables index-only scans)

**VACUUM FULL** goes further: rewrites the entire table to a new file,
eliminating all dead space. But it locks the table exclusively and
requires double the disk space temporarily. Use sparingly.

**Autovacuum**: A background process that automatically vacuums tables
based on configurable thresholds:
```
autovacuum_vacuum_threshold = 50        (min dead tuples before vacuum)
autovacuum_vacuum_scale_factor = 0.2    (vacuum when 20% of tuples are dead)
```

Autovacuum is critical. If it falls behind (large tables, high update
rates, long-running transactions that prevent dead tuple reclamation),
the table bloats. In extreme cases, table bloat can consume 10-100x
the actual data size.

**Transaction ID wraparound**: PostgreSQL uses 32-bit transaction IDs.
After ~4 billion transactions, IDs wrap around. Without VACUUM freezing
old tuples (replacing their `xmin` with a special "frozen" marker), the
database would think all old data was created by future transactions and
make it invisible. PostgreSQL forces an aggressive VACUUM to prevent
this — if the database reaches the wraparound danger zone, it refuses
to accept new transactions until VACUUM completes. This has caused
real production outages.

### InnoDB's MVCC (MySQL)

InnoDB takes a different approach:

```
Table row (clustered index):
  Points to → undo log containing previous versions

Current version:  {id:42, name:'bob',   trx_id=200, roll_ptr→undo1}
Undo log entry 1: {id:42, name:'alice', trx_id=100, roll_ptr→undo2}
Undo log entry 2: {id:42, name:'adam',  trx_id=50,  roll_ptr→null}
```

InnoDB stores only the **latest version** in the table (clustered
B+tree). Previous versions live in the **undo log** (a separate
structure). To read an old version, InnoDB follows the `roll_ptr` chain
back through the undo log until it finds a version visible to the
transaction's snapshot.

**Advantage over PostgreSQL**: Only one copy in the main table (less
bloat). Updates modify the row in place and push the old version to the
undo log.

**Disadvantage**: Reading old snapshots requires traversing the undo
chain (multiple I/O operations for long chains). Long-running
transactions can cause massive undo log growth.

---

## Locking

While MVCC handles most concurrency, some operations need explicit
locks.

### Lock Granularity

| Level | What's Locked | Concurrency | Overhead |
|-------|--------------|------------|----------|
| Row | Single row | Highest | Highest (many locks) |
| Page | One page (many rows) | Medium | Medium |
| Table | Entire table | Lowest | Lowest |

Most databases use **row-level locks** for DML (INSERT, UPDATE, DELETE)
and **table-level locks** for DDL (ALTER TABLE, DROP TABLE).

### Lock Modes

**Shared (S)**: Read lock. Multiple transactions can hold shared locks
on the same row simultaneously. Blocks exclusive locks.

**Exclusive (X)**: Write lock. Only one transaction can hold an exclusive
lock. Blocks both shared and exclusive locks.

**Compatibility matrix:**

```
        Requested
        S     X
Held S  ✓     ✗
     X  ✗     ✗
```

PostgreSQL additionally has:
- **FOR UPDATE**: Exclusive lock on selected rows (prevents other
  transactions from modifying or locking them)
- **FOR SHARE**: Shared lock on selected rows
- **FOR NO KEY UPDATE**: Weaker exclusive lock (doesn't block foreign
  key checks)
- **FOR KEY SHARE**: Weakest lock (allows FOR NO KEY UPDATE)

```sql
-- Lock the row to prevent concurrent modification
SELECT * FROM accounts WHERE id = 42 FOR UPDATE;
-- Now safe to read-modify-write
UPDATE accounts SET balance = balance - 100 WHERE id = 42;
COMMIT;  -- lock released
```

### Gap Locks and Next-Key Locks (InnoDB)

InnoDB's approach to preventing phantoms at Repeatable Read. A gap lock
locks the **range between** existing keys, preventing inserts in that
range:

```
Existing index keys: 10, 20, 30

SELECT * FROM t WHERE id BETWEEN 15 AND 25 FOR UPDATE;

Locks:
  Gap lock: (10, 20)     ← prevents INSERT of id=15
  Record lock: 20        ← prevents UPDATE/DELETE of id=20
  Gap lock: (20, 30)     ← prevents INSERT of id=25
```

A **next-key lock** = record lock + gap lock before it. This prevents
both modifications to existing rows and insertions of new rows in the
range.

### Deadlocks

Two transactions each hold a lock that the other needs:

```
Transaction A:                    Transaction B:
  LOCK row 1 (acquired)            LOCK row 2 (acquired)
  LOCK row 2 (waiting for B...)    LOCK row 1 (waiting for A...)
  → DEADLOCK
```

**Detection**: The database maintains a **wait-for graph** — a directed
graph where Transaction A → Transaction B means A is waiting for B's
lock. A cycle in this graph is a deadlock. One transaction is chosen as
the **victim** and rolled back (usually the one with the least work
done).

PostgreSQL detects deadlocks by checking the wait-for graph periodically
(every `deadlock_timeout`, default 1 second). InnoDB detects them
immediately on lock wait.

**Prevention**: Acquire locks in a consistent order. If all transactions
lock rows in ascending ID order, cycles can't form:

```sql
-- Instead of random lock order:
SELECT * FROM accounts WHERE id IN (42, 17) FOR UPDATE;
-- Use consistent order:
SELECT * FROM accounts WHERE id IN (17, 42) ORDER BY id FOR UPDATE;
```

---

## Isolation in Practice

### The Default Is Usually Fine

**Read Committed** (PostgreSQL default) handles most workloads:
- No dirty reads
- Short-lived queries see recent committed data
- Simple to reason about
- No serialization errors to handle

### When You Need Stronger Isolation

**Repeatable Read** for:
- Reports that must see a consistent point-in-time snapshot
- Read-modify-write operations (`SELECT ... FOR UPDATE` also works
  at Read Committed for single-row cases)
- Any operation where re-reading must return the same result

**Serializable** for:
- Write skew prevention (the doctor on-call example)
- When application logic can't easily be restructured to avoid anomalies
- Financial calculations requiring strict correctness
- Must handle serialization failures with retry logic:

```javascript
async function runSerializable(fn) {
    for (let attempt = 0; attempt < 3; attempt++) {
        try {
            await db.transaction(async (tx) => {
                await tx.query('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
                return fn(tx);
            });
            return;  // success
        } catch (err) {
            if (err.code === '40001') {  // serialization_failure
                continue;  // retry
            }
            throw err;  // real error
        }
    }
    throw new Error('Transaction failed after 3 retries');
}
```

### Advisory Locks

Application-level locks managed by the database:

```sql
-- Acquire a lock on a logical resource (not a row)
SELECT pg_advisory_lock(12345);
-- ... do work ...
SELECT pg_advisory_unlock(12345);

-- Try without blocking:
SELECT pg_try_advisory_lock(12345);  -- returns true/false
```

Useful for:
- Cron job coordination (only one instance runs)
- Rate limiting at the application level
- Mutual exclusion for non-database resources
- Distributed locks (all instances use the same database)

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| ACID: Atomicity, Consistency, Isolation, Durability | Atomicity = all-or-nothing. Isolation = concurrent txns don't interfere |
| Dirty read, non-repeatable read, phantom, lost update, write skew | Each anomaly breaks different assumptions. Write skew is the hardest |
| Read Committed: each statement sees latest committed data | PostgreSQL default. Prevents dirty reads. Allows non-repeatable reads |
| Repeatable Read: entire txn sees snapshot from start | Prevents lost updates. PostgreSQL uses MVCC snapshots |
| Serializable: equivalent to serial execution | SSI (optimistic, PostgreSQL) vs 2PL (pessimistic, InnoDB). Must retry on failure |
| MVCC keeps multiple row versions | Readers never block writers. PostgreSQL: heap versions. InnoDB: undo log chain |
| PostgreSQL VACUUM reclaims dead tuples | Autovacuum is critical. Bloat and XID wraparound are real production risks |
| Row-level locks: S (shared) and X (exclusive) | FOR UPDATE locks rows. FOR SHARE allows concurrent reads |
| Gap locks prevent phantoms in InnoDB | Lock the range between keys, not just existing rows |
| Deadlocks: cycle in wait-for graph | Database detects and kills one transaction. Prevent with consistent lock ordering |
| Advisory locks for application-level coordination | pg_advisory_lock for cron jobs, distributed mutex, rate limiting |
