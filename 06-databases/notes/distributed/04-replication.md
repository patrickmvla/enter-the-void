# Replication — Copies of Data Across Machines

## Why Replicate

A single database server is a single point of failure. If it dies,
the data is inaccessible (or lost, if the disk fails). Replication
copies data to multiple machines for:

- **High availability**: If the primary dies, a replica takes over
- **Read scaling**: Distribute read queries across replicas
- **Geographic distribution**: Place data close to users (lower latency)
- **Disaster recovery**: Survive data center failures

The fundamental challenge: keeping replicas consistent with each other
while handling network partitions, machine failures, and concurrent
writes.

---

## Single-Leader Replication

The most common architecture. One node (the **leader/primary**) accepts
all writes. It streams changes to **followers/replicas**, which apply
them in the same order.

```
Clients (reads + writes)
         │
         ▼
    ┌─────────┐
    │ Primary  │────────────────────┐
    │ (leader) │──────────┐        │
    └─────────┘          │        │
         │                │        │
    WAL stream       WAL stream  WAL stream
         │                │        │
         ▼                ▼        ▼
    ┌─────────┐    ┌─────────┐  ┌─────────┐
    │Replica 1│    │Replica 2│  │Replica 3│
    │(follower)│   │(follower)│ │(follower)│
    └─────────┘    └─────────┘  └─────────┘
         ↑              ↑            ↑
    Clients (reads only)
```

### Synchronous vs Asynchronous Replication

**Synchronous**: The primary waits for the replica to confirm the write
before acknowledging to the client.

```
Client → Primary: INSERT INTO users ...
Primary → Replica: "apply this WAL record"
Replica → Primary: "applied"
Primary → Client: "committed"

Guarantee: If the client sees "committed," the data is on at least 2 machines.
Cost: Every write adds a network round trip (~0.5-5ms within a DC).
Risk: If the replica is down, writes stall (or timeout).
```

**Asynchronous**: The primary acknowledges immediately, streams to
replicas in the background.

```
Client → Primary: INSERT INTO users ...
Primary → Client: "committed" (immediately)
Primary → Replica: "apply this WAL record" (background, eventual)

Guarantee: None for replicas. Data exists only on primary until replication catches up.
Cost: No added latency for writes.
Risk: If the primary dies before the replica receives the change, data is lost.
```

**Semi-synchronous** (practical compromise): One replica is synchronous,
the rest are asynchronous. Guarantees data is on at least 2 nodes
without the cost of waiting for all replicas. If the synchronous replica
falls behind, another replica is promoted to synchronous. PostgreSQL
calls this `synchronous_standby_names`.

### Replication Lag

With asynchronous replication, replicas are always slightly behind the
primary. This lag is typically milliseconds but can spike to seconds or
minutes under load.

**Read-after-write inconsistency:**
```
1. Client writes to primary: INSERT INTO posts (title) VALUES ('hello')
2. Client reads from replica: SELECT * FROM posts WHERE title = 'hello'
3. Replica hasn't received the INSERT yet → empty result
4. User: "Where's my post?!"
```

**Solutions:**
- **Read-your-writes consistency**: After a write, route subsequent
  reads from that user to the primary (for a short window, e.g., 10s)
- **Monotonic reads**: Ensure a user always reads from the same replica
  (consistent reads, even if lagging)
- **Causal consistency**: Track which writes a read depends on, and
  only serve reads from replicas that have received those writes

### Failover

When the primary dies, a replica must be promoted:

1. **Detect failure**: Heartbeat/health checks. Typically 3 missed
   heartbeats (30 seconds) before declaring the primary dead.
   (Too aggressive → false positives → unnecessary failover.
   Too conservative → extended downtime.)

2. **Choose a new primary**: The replica with the least replication lag
   (most up-to-date data). In Raft-based systems, an election determines
   the new leader.

3. **Reconfigure replicas**: Point remaining replicas to the new primary.
   Update the application's connection string (or use a connection proxy
   like PgBouncer/HAProxy that handles routing).

4. **Handle the old primary**: When the old primary comes back online,
   it must join as a replica (it may have writes that the new primary
   doesn't have — these are "divergent" and must be discarded or
   reconciled).

**Split-brain**: If the old primary isn't actually dead (just network-
partitioned), both nodes think they're the primary. Writes go to both.
Data diverges. This is catastrophic. Prevention:
- **Fencing**: Before the new primary starts, ensure the old one is
  stopped (STONITH — "Shoot The Other Node In The Head"). Power off
  the old node via IPMI, or revoke its access to shared storage.
- **Consensus protocols**: Raft/Paxos ensure only one leader exists at
  a time. A node can only be leader if it has a majority quorum.

### PostgreSQL Streaming Replication

PostgreSQL replicates by streaming WAL records from primary to replicas:

```
postgresql.conf (primary):
  wal_level = replica
  max_wal_senders = 10
  synchronous_standby_names = 'replica1'

On the replica:
  primary_conninfo = 'host=primary port=5432 user=replicator'

The replica:
  1. Connects to the primary
  2. Receives WAL records as they're generated
  3. Applies them (replays the WAL against its own data files)
  4. Becomes a near-exact copy of the primary
```

**Logical replication** (PostgreSQL 10+): Instead of streaming raw WAL
(physical bytes), stream logical change records (INSERT, UPDATE, DELETE
with the actual data). Enables:
- Replicating between different PostgreSQL versions
- Replicating a subset of tables
- Replicating to different schema structures
- Cross-database replication (PostgreSQL → other systems via CDC)

---

## Multi-Leader Replication

Multiple nodes accept writes. Each node replicates its writes to the
others. Used for multi-datacenter deployments where each datacenter
has its own primary.

```
Datacenter A          Datacenter B          Datacenter C
┌──────────┐          ┌──────────┐          ┌──────────┐
│ Primary A │ ←─────→ │ Primary B │ ←─────→ │ Primary C │
│ (accepts  │         │ (accepts  │         │ (accepts  │
│  writes)  │         │  writes)  │         │  writes)  │
└──────────┘          └──────────┘          └──────────┘
```

### The Conflict Problem

If two leaders modify the same row concurrently, which write wins?

```
Leader A: UPDATE users SET name='alice' WHERE id=42; (timestamp: 100)
Leader B: UPDATE users SET name='bob'   WHERE id=42; (timestamp: 101)
```

**Conflict resolution strategies:**

**Last Write Wins (LWW)**: The write with the highest timestamp wins.
Simple but lossy — one write is silently discarded. Clock skew between
datacenters makes "last" ambiguous. Used by Cassandra and DynamoDB
(with vector clocks for more nuanced ordering).

**Merge**: Application-specific logic combines the writes. For example,
a shopping cart: add both items rather than choosing one. Requires
domain knowledge.

**Custom resolution**: Store both versions, present the conflict to
the application (or user) for manual resolution. CouchDB does this.

**CRDTs (Conflict-Free Replicated Data Types)**: Data structures
mathematically designed to merge without conflicts. A G-Counter
(grow-only counter) where each node maintains its own count and the
total is the sum. An OR-Set (observed-remove set) where additions and
removals are tracked by unique tags. CRDTs guarantee eventual
convergence regardless of the order operations are applied.

### When Multi-Leader Makes Sense

- **Multi-datacenter**: Each datacenter writes locally with low latency;
  replication happens asynchronously across DCs
- **Offline-capable applications**: Each device is a "leader" that
  syncs when it reconnects (CouchDB, PouchDB)
- **Collaborative editing**: Each user's edits are local writes that
  sync (though most collaborative editors use OT or CRDTs, not
  database-level multi-leader)

---

## Leaderless Replication

No designated leader. Any node can accept reads and writes. Quorum
mechanisms ensure consistency.

### Dynamo-Style (Cassandra, Riak, DynamoDB)

```
Write: Client sends to ALL replicas (or a coordinator sends on its behalf)
  W replicas must acknowledge the write for it to be considered successful

Read: Client reads from ALL replicas (or a coordinator reads)
  R replicas must respond for the read to return

Quorum condition: W + R > N  (where N = total replicas)
  → At least one node in the read set has the latest write
```

**Example with N=3, W=2, R=2:**

```
Nodes: [A, B, C]

Write value=42:
  A: ack ✓    B: ack ✓    C: (slow/down, not ack'd)
  W=2 met → write succeeds

Read:
  A: returns 42 ✓   B: returns 42 ✓   C: (returns stale value or timeout)
  R=2 met → return 42 (latest value is in the read set)
```

**Tunable consistency:**

| W | R | Behavior |
|---|---|----------|
| N | 1 | Write to all, read from one. Fast reads, slow writes |
| 1 | N | Write to one, read from all. Fast writes, slow reads |
| ⌈N/2⌉+1 | ⌈N/2⌉+1 | Balanced. Tolerates ⌊N/2⌋ failures |
| 1 | 1 | Fastest but NO consistency guarantee (W+R ≤ N) |

### Anti-Entropy and Read Repair

Even with quorums, replicas can diverge (a write reaches 2 of 3, then
the stale replica needs to catch up):

**Read repair**: During a quorum read, the coordinator notices that
node C has a stale value. It sends the latest value to C. The read
itself triggers the repair. Only fixes rows that are actually read —
cold data stays stale.

**Anti-entropy process**: A background process compares data across
replicas (using Merkle trees — hash trees that efficiently identify
differing ranges) and fixes inconsistencies. Slower but comprehensive.

### Sloppy Quorums and Hinted Handoff

If the required W nodes are unavailable (network partition), do you
reject the write (strict quorum) or write to any W reachable nodes
(sloppy quorum)?

**Sloppy quorum**: Write to any W nodes, even if they're not the
"home" nodes for that key. When the home nodes come back, the
temporary nodes "hand off" the data. This improves write availability
but weakens consistency — a sloppy quorum read might miss the write
(it's on non-home nodes).

---

## Consensus — Agreeing in a Distributed System

When nodes need to agree on something (who is the leader, whether a
transaction committed), they need a **consensus protocol**.

### The Problem

In a network, messages can be delayed, lost, or reordered. Nodes can
crash and restart. How do N nodes agree on a value when any of them
might fail at any time?

**FLP Impossibility Result** (1985): In an asynchronous system (no
bound on message delivery time), no consensus protocol can guarantee
both safety (agreement) and liveness (progress) if even one node can
crash. In practice, protocols trade theoretical guarantees for
practical timeout-based detection.

### Raft (Understandable Consensus)

Raft (Ongaro & Ousterhout, 2014) is the most widely used consensus
protocol in modern systems (etcd, CockroachDB, TiDB, Consul).

**Three roles:**
- **Leader**: Handles all client requests, replicates log entries
- **Follower**: Passive, responds to leader's requests
- **Candidate**: A follower trying to become leader (during election)

**Leader election:**

```
1. Followers expect heartbeats from the leader
2. If no heartbeat within election timeout (randomized, 150-300ms):
   - Follower becomes Candidate
   - Increments its term number
   - Votes for itself
   - Sends RequestVote RPCs to all other nodes
3. A node grants its vote if:
   - It hasn't voted in this term yet
   - The candidate's log is at least as up-to-date as its own
4. Candidate receiving majority of votes becomes Leader
5. Leader sends heartbeats to prevent new elections
```

The randomized election timeout prevents split votes (two candidates
simultaneously). If a split vote occurs, a new election starts with
new random timeouts.

**Log replication:**

```
1. Client sends command to Leader
2. Leader appends to its log (uncommitted)
3. Leader sends AppendEntries RPC to all Followers
4. Follower appends to its log, responds success
5. When majority have appended: entry is committed
6. Leader applies to state machine, responds to client
7. Followers learn of commit via next heartbeat, apply to their state machines
```

**Safety guarantee**: Once a log entry is committed (majority ack'd),
it will be present in all future leaders' logs. A candidate can't win
an election unless its log is at least as up-to-date as a majority of
nodes — this ensures committed entries are never lost.

**Raft vs Paxos**: Raft was designed to be understandable (the paper
explicitly states this as a goal). Paxos (Lamport, 1989) is the
original consensus protocol and is provably correct, but notoriously
difficult to implement correctly. Most modern systems choose Raft.

---

## CAP Theorem and Consistency Models

### CAP Theorem (Brewer, 2000)

A distributed system can provide at most two of three guarantees:

- **Consistency**: Every read receives the most recent write
- **Availability**: Every request receives a response (not an error)
- **Partition tolerance**: The system continues operating despite
  network partitions

**In reality**: Network partitions happen (they're not optional in a
distributed system). So the real choice is between **CP** (consistent
but may be unavailable during partition) and **AP** (available but may
return stale data during partition).

```
Network partition occurs:

CP system (e.g., PostgreSQL with sync replication):
  Node A: "I can't reach Node B. I'll reject writes until the partition heals."
  → Consistent but unavailable for writes

AP system (e.g., Cassandra with W=1, R=1):
  Node A: "I can't reach Node B. I'll accept writes and reconcile later."
  → Available but may return stale data
```

**CAP is a spectrum, not a binary choice.** Most systems offer tunable
trade-offs (Cassandra's tunable consistency, DynamoDB's strong vs
eventual consistency options).

### Consistency Models

From strongest to weakest:

**Linearizability** (strongest): Every operation appears to take effect
at a single point in time between its start and end. Equivalent to a
single-threaded execution. Required for: distributed locks, leader
election, unique constraints.

**Sequential consistency**: Operations from each client appear in the
order that client issued them, but operations from different clients
may be reordered. Weaker than linearizability (no real-time ordering
guarantee between clients).

**Causal consistency**: If operation A causally precedes B (A happened
before B and B could have observed A), then all nodes see A before B.
Concurrent operations (no causal relationship) can be seen in any order.

**Eventual consistency** (weakest): If no new writes occur, all replicas
will eventually converge to the same value. No guarantees about how
long "eventually" takes. Reads may return any previous version.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Single-leader: all writes to one node, replicated to followers | Simplest model. PostgreSQL streaming replication. Semi-sync for durability |
| Sync replication: no data loss, added latency | Write waits for replica ack. Replica down = writes stall |
| Async replication: no latency, possible data loss | Replica lag causes read-after-write inconsistency |
| Failover: detect, elect, reconfigure, fence old primary | Split-brain is catastrophic. STONITH prevents it |
| Multi-leader: multiple write nodes | Conflicts are inevitable. LWW, merge, CRDTs, or manual resolution |
| Leaderless: quorum reads/writes (W + R > N) | Read repair + anti-entropy for convergence. Sloppy quorums trade consistency for availability |
| Raft: leader election + log replication with majority quorum | Randomized election timeout prevents split votes. Committed = majority ack'd |
| CAP: choose CP or AP during partitions | Partitions are inevitable. PostgreSQL = CP. Cassandra = tunable |
| Linearizability: strongest consistency | Ops appear atomic and ordered in real time. Required for distributed locks |
| Eventual consistency: weakest guarantee | Replicas converge eventually. Fine for read-heavy, conflict-rare workloads |
