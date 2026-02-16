# Message Queues & Event-Driven Architecture

## Why Queues

Synchronous request-response breaks down when:
- The downstream service is slow (caller blocks waiting)
- The downstream service is down (caller fails)
- A burst of traffic overwhelms the downstream service
- The caller doesn't need the result immediately

A message queue decouples producers from consumers:

```
Synchronous:
  Service A → Service B (A waits for B's response)
  If B is slow: A is slow. If B is down: A fails.

Asynchronous with queue:
  Service A → [Queue] → Service B
  A publishes and moves on. B consumes at its own pace.
  If B is slow: messages buffer in the queue.
  If B is down: messages wait in the queue until B recovers.
```

---

## Messaging Models

### Point-to-Point (Work Queue)

One producer, one consumer per message. Each message is processed
exactly once (by one consumer).

```
Producer → [Queue] → Consumer 1
                   → Consumer 2  (each message goes to ONE consumer)
                   → Consumer 3

Messages: M1, M2, M3, M4, M5, M6
Consumer 1 gets: M1, M4
Consumer 2 gets: M2, M5
Consumer 3 gets: M3, M6

Use case: task distribution (background jobs, email sending)
```

### Publish-Subscribe (Fan-Out)

One producer, multiple consumers. Each consumer gets every message.

```
Producer → [Topic] → Consumer Group A (all messages)
                   → Consumer Group B (all messages)
                   → Consumer Group C (all messages)

Use case: event broadcasting (order placed → notification service,
          analytics service, inventory service all get the event)
```

### Consumer Groups (Kafka Model)

Combines both: messages are published to a topic. Each consumer
**group** gets every message, but within a group, each message goes
to only one consumer.

```
Topic: "orders" (partitions: P0, P1, P2)

Consumer Group "notifications":
  Consumer N1 reads P0
  Consumer N2 reads P1, P2

Consumer Group "analytics":
  Consumer A1 reads P0, P1, P2

Every order event is processed by both groups.
Within each group, no duplication.
```

---

## Apache Kafka — Internals

### Log-Structured Storage

Kafka's core abstraction is a **commit log** — an append-only,
ordered, immutable sequence of records.

```
Topic: "orders"

Partition 0:  [0] [1] [2] [3] [4] [5] [6] [7] ...
Partition 1:  [0] [1] [2] [3] [4] [5] ...
Partition 2:  [0] [1] [2] [3] [4] [5] [6] [7] [8] ...
               ↑                              ↑
           oldest offset                  newest offset

Each partition is an independent, ordered log.
Records are assigned an offset (monotonically increasing integer).
Records are immutable — once written, never modified.
Records are retained for a configurable duration (default 7 days)
or size, then deleted.
```

### On-Disk Format

Each partition is a directory containing **segment files**:

```
/kafka-data/orders-0/          (partition 0 of topic "orders")
├── 00000000000000000000.log   (segment file: offsets 0-9999)
├── 00000000000000000000.index (sparse offset → file position index)
├── 00000000000000000000.timeindex (timestamp → offset index)
├── 00000000000000010000.log   (segment file: offsets 10000-19999)
├── 00000000000000010000.index
└── 00000000000000010000.timeindex
```

**Segment files** are the unit of retention. When a segment exceeds
`log.segment.bytes` (default 1 GB) or `log.segment.ms` (default 7
days), a new segment starts. Old segments are deleted whole — no
partial deletion within a segment.

**Record batch format** (on disk and on wire — same format):

```
RecordBatch:
  base_offset:           int64   (first offset in batch)
  batch_length:          int32   (total bytes)
  partition_leader_epoch: int32
  magic:                 int8    (format version = 2)
  crc:                   uint32  (CRC32-C of everything after crc)
  attributes:            int16   (compression, timestamp type, txn)
  last_offset_delta:     int32
  base_timestamp:        int64
  max_timestamp:         int64
  producer_id:           int64   (for exactly-once)
  producer_epoch:        int16
  base_sequence:         int32
  records_count:         int32
  [Record, Record, Record, ...]

Record:
  length:                varint
  attributes:            int8
  timestamp_delta:       varint  (delta from base_timestamp)
  offset_delta:          varint  (delta from base_offset)
  key_length:            varint
  key:                   bytes
  value_length:          varint
  value:                 bytes
  headers_count:         varint
  [Header, Header, ...]
```

Records are batched, compressed together (LZ4, Snappy, Zstd, or
gzip), and written as a single I/O operation. The **same byte format**
is used on disk, on the wire, and in the producer's send buffer —
Kafka uses **zero-copy transfer** (sendfile syscall) to stream segment
files directly from disk to the network socket without copying through
user space.

### Zero-Copy

The critical performance optimization. Without zero-copy:

```
Traditional path:
  Disk → kernel page cache → user space buffer → kernel socket buffer → NIC

4 copies, 2 context switches. For a broker serving 1 GB/s,
this would saturate CPU.

Zero-copy (sendfile):
  Disk → kernel page cache → NIC (DMA)

The data never enters user space. The kernel sends directly from
page cache to the network card. 2 fewer copies, 2 fewer context
switches. This is why Kafka can serve gigabytes/sec per broker
with minimal CPU usage.
```

### Producer Internals

```
Application → Producer API:

  producer.send(new ProducerRecord("orders", key, value))

Internal pipeline:
  1. Serializer: convert key and value to bytes
  2. Partitioner: determine which partition
     - If key is set: hash(key) % num_partitions
       (same key always goes to same partition → ordering guarantee)
     - If key is null: round-robin (or sticky partitioner in newer versions)
  3. Record accumulator: batch records per partition
     - Buffer in memory until batch.size (default 16 KB) or linger.ms (default 0)
     - linger.ms > 0 waits for more records before sending (higher throughput,
       higher latency)
  4. Sender thread: send batch to partition leader
  5. Broker acknowledges

Acknowledgment modes (acks):
  acks=0:  Fire and forget. Don't wait for broker. Fastest. Data loss possible.
  acks=1:  Wait for leader to write to its log. Fast. Data loss if leader
           crashes before replication.
  acks=all (acks=-1): Wait for ALL in-sync replicas (ISR) to write.
           Slowest. No data loss (unless all replicas fail simultaneously).
```

### Replication and ISR

Each partition has one leader and N-1 followers (replicas):

```
Topic: orders, partition 0, replication_factor=3

  Broker 1 (Leader):    [0][1][2][3][4][5][6]  offset 6
  Broker 2 (Follower):  [0][1][2][3][4][5]     offset 5 (1 behind)
  Broker 3 (Follower):  [0][1][2][3][4][5][6]  offset 6 (caught up)

ISR (In-Sync Replicas): {Broker 1, Broker 3}
  Broker 2 is not in ISR (too far behind)

Writes: only to leader. Leader waits for ISR replicas to replicate
  before acknowledging (when acks=all).

High watermark: offset 5 (the latest offset replicated to ALL ISR members)
  Consumers can only read up to the high watermark.
  This prevents consuming records that might be lost if the leader fails.
```

**ISR management:**
- Followers fetch from the leader continuously (like database replication)
- If a follower falls behind by more than `replica.lag.time.max.ms`
  (default 30s), it's removed from ISR
- When it catches up, it's added back to ISR
- `min.insync.replicas`: minimum ISR size for the leader to accept writes.
  With `min.insync.replicas=2` and `acks=all`, you're guaranteed data is
  on at least 2 brokers before acknowledging.

### Consumer Internals

```
Consumer Group: "order-processing"
  Consumer C1: assigned partitions [P0, P1]
  Consumer C2: assigned partitions [P2]

For each assigned partition:
  1. Fetch records starting from committed offset
     (fetch.min.bytes, fetch.max.wait.ms control batching)
  2. Process records
  3. Commit offset (mark records as processed)

Offset management:
  Stored in internal topic: __consumer_offsets
  Key: (group, topic, partition)
  Value: committed offset

  Auto-commit (enable.auto.commit=true):
    Commit every auto.commit.interval.ms (default 5s)
    Risk: if consumer crashes between commit and processing,
    some records may be lost (committed but not processed)

  Manual commit:
    consumer.commitSync();  // block until committed
    consumer.commitAsync(); // non-blocking

    Process-then-commit: at-least-once (may reprocess on crash)
    Commit-then-process: at-most-once (may lose on crash)
```

**Rebalancing**: When consumers join/leave a group, partitions are
reassigned:

```
Before: C1=[P0,P1,P2], C2=[P3,P4,P5]
C2 crashes.
Rebalance: C1=[P0,P1,P2,P3,P4,P5]  (C1 takes over C2's partitions)

C3 joins.
Rebalance: C1=[P0,P1], C2_new=[P2,P3], C3=[P4,P5]

During rebalance: all consumers pause (stop-the-world).
Cooperative rebalancing (Kafka 2.4+): only affected partitions pause.
```

---

## Delivery Guarantees

### At-Most-Once

Each message is delivered zero or one times. Messages can be lost
but never duplicated.

```
Producer: send message, don't wait for ack (acks=0)
Consumer: commit offset BEFORE processing

If producer or consumer crashes: message may be lost.
If no crash: exactly once.

Use when: losing some data is acceptable (metrics, logs, analytics
where approximate counts are fine).
```

### At-Least-Once

Each message is delivered one or more times. No messages are lost
but duplicates are possible.

```
Producer: send message, wait for ack, retry on failure (acks=1 or all)
Consumer: process message, THEN commit offset

If consumer crashes after processing but before committing:
  On restart, reprocesses the same message → duplicate.

Use when: processing must be complete, duplicates are tolerable
(or handled by idempotent processing).
```

### Exactly-Once (Kafka's Approach)

Each message is processed exactly once. No loss, no duplicates.
The hardest guarantee to provide.

**Kafka's exactly-once = idempotent producer + transactions:**

**Idempotent producer:**
```
enable.idempotence=true

Each producer gets a unique producer_id.
Each record gets a sequence number (per partition).
Broker tracks (producer_id, partition, sequence_number).
If a broker receives a duplicate (same producer_id + sequence):
  It returns success without writing the duplicate.
  Network retries can't cause duplicates.

This gives exactly-once within a single partition from a single producer.
```

**Transactional producer** (exactly-once across partitions):
```
producer.initTransactions();
producer.beginTransaction();
  producer.send(topic1, record1);
  producer.send(topic2, record2);
  producer.sendOffsetsToTransaction(consumerOffsets, groupId);
producer.commitTransaction();
// or producer.abortTransaction();

All sends in the transaction either all commit or all abort.
Consumer offsets committed atomically with produced records.

Consumer must set: isolation.level=read_committed
  (only reads records from committed transactions)
```

**How it works internally:**
```
1. Producer sends records with transaction markers
2. Transaction coordinator (a Kafka broker) tracks the transaction state
   in an internal topic (__transaction_state)
3. On commitTransaction():
   a. Coordinator writes PREPARE to __transaction_state
   b. Coordinator writes COMMIT markers to all involved partitions
   c. Coordinator writes COMMITTED to __transaction_state
4. Consumers with read_committed filter out uncommitted/aborted records
```

**Important caveat**: Kafka's exactly-once guarantees are within the
Kafka ecosystem. If your consumer writes to an external system (HTTP
call, database insert), you need **idempotent consumers** — the
consumer must be able to handle duplicates gracefully (idempotency
keys, UPSERT, deduplication tables).

---

## Event-Driven Patterns

### Event Sourcing

Instead of storing current state, store the sequence of events that
led to the current state. The current state is derived by replaying
events.

```
Traditional (state-based):
  Account table: {id: 42, balance: 750}
  After deposit: UPDATE balance = 750 + 200 = 950

Event sourced:
  Events table:
    {id: 1, account: 42, type: "created",    amount: 0,    timestamp: ...}
    {id: 2, account: 42, type: "deposited",  amount: 1000, timestamp: ...}
    {id: 3, account: 42, type: "withdrawn",  amount: 250,  timestamp: ...}
    {id: 4, account: 42, type: "deposited",  amount: 200,  timestamp: ...}

  Current balance = replay: 0 + 1000 - 250 + 200 = 950
```

**Advantages:**
- **Complete audit trail**: Every change is recorded. You can answer
  "what happened at time T?" by replaying events up to T.
- **Temporal queries**: "What was the balance at 3pm yesterday?"
  Replay events up to that timestamp.
- **Event replay**: Fix a bug in event handling, replay all events
  through the fixed handler.
- **Decoupling**: Multiple services can react to the same events
  in different ways.

**Disadvantages:**
- **Complexity**: Reading current state requires replaying events
  (or maintaining a projection/snapshot).
- **Event schema evolution**: Changing event format is hard when
  you have billions of historical events.
- **Storage**: Potentially enormous event store.
- **Eventual consistency**: Projections may lag behind the event stream.

### CQRS (Command Query Responsibility Segregation)

Separate the read model from the write model. Writes go to one store
(optimized for writes), reads come from a different store (optimized
for the specific query patterns).

```
Command (write):
  Client → Command Handler → Event Store (append-only)
                                  │
                                  ▼
                           Event published
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼              ▼
              Read Model 1   Read Model 2   Read Model 3
              (user view)    (admin view)    (analytics)

Query (read):
  Client → Query Handler → Read Model (denormalized, optimized)
```

**Common pairing with event sourcing:**
- Write side: event store (append-only log, Kafka/EventStore)
- Read side: projected views in PostgreSQL, Elasticsearch, Redis
- Events flow from write to read via consumers that update projections

**Trade-off**: Read models are **eventually consistent** with the
write model. The delay depends on the event processing pipeline
latency (typically milliseconds to seconds).

### Saga Pattern

Distributed transactions across services without 2PC. A saga is a
sequence of local transactions with **compensating actions** for
rollback.

```
Order saga:
  1. Create order (Order Service)
  2. Reserve inventory (Inventory Service)
  3. Charge payment (Payment Service)
  4. Confirm order (Order Service)

If step 3 fails (payment declined):
  Compensate step 2: Release inventory
  Compensate step 1: Cancel order
```

**Orchestration** (centralized coordinator):
```
Saga Orchestrator:
  1. Send "create order" to Order Service
     ← success
  2. Send "reserve inventory" to Inventory Service
     ← success
  3. Send "charge payment" to Payment Service
     ← FAILURE
  4. Send "release inventory" to Inventory Service (compensate)
  5. Send "cancel order" to Order Service (compensate)

The orchestrator knows the entire flow and drives it.
Single point of logic (and failure).
```

**Choreography** (decentralized, event-driven):
```
Order Service:
  Order created → publishes "OrderCreated" event

Inventory Service:
  Listens for "OrderCreated"
  Reserves inventory
  Publishes "InventoryReserved" event

Payment Service:
  Listens for "InventoryReserved"
  Charges payment
  Publishes "PaymentCharged" (or "PaymentFailed")

Inventory Service:
  Listens for "PaymentFailed"
  Releases inventory

No central coordinator. Each service reacts to events.
Harder to understand the full flow (logic is distributed).
```

---

## Dead Letter Queues

When a message can't be processed (bad format, business rule violation,
transient error after retries), it goes to a dead letter queue (DLQ)
instead of being silently dropped or blocking the main queue.

```
Main queue → Consumer
  ↓ (processing fails after N retries)
Dead Letter Queue → manual review / automated remediation

Configuration (Kafka):
  errors.deadletterqueue.topic.name = "orders.DLQ"
  errors.deadletterqueue.context.headers.enable = true
  (adds error context: original topic, partition, offset, exception)

Configuration (RabbitMQ):
  Queue declared with:
    x-dead-letter-exchange: "dlx"
    x-dead-letter-routing-key: "orders.dead"
  On reject/nack with requeue=false → message routed to DLX
```

**DLQ handling strategies:**
- Alert on DLQ depth (monitor queue size)
- Automated retry after delay (exponential backoff)
- Manual inspection dashboard
- Replay DLQ messages to main queue after fix deployed

---

## Backpressure

When a consumer can't keep up with a producer, the system must decide
what to do with the excess:

```
Strategy 1 — Drop: Discard messages (lossy)
  Use when: freshness matters more than completeness
  (real-time metrics, stock tickers)

Strategy 2 — Buffer: Queue messages (bounded)
  Use when: all messages must be processed eventually
  Risk: unbounded buffering = OOM. Always set queue size limits.

Strategy 3 — Slow down producer: Apply backpressure
  Use when: producer can afford to slow down
  How: TCP flow control, HTTP 429, blocking send()
```

Kafka handles backpressure naturally: the queue is on disk (bounded
by retention policy, not memory). Producers write at their pace.
Consumers read at their pace. If consumers fall behind, they just
read from older offsets (data is still on disk).

RabbitMQ handles backpressure with **flow control**: when memory or
disk reaches a threshold, the broker blocks publishers (stops reading
from their TCP connections). Publishers stall until the consumer
catches up.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Queues decouple producers from consumers | Buffer absorbs spikes. Consumer processes at its own pace |
| Kafka = distributed commit log | Append-only, immutable. Partitions for parallelism. Offsets for position |
| Zero-copy (sendfile): disk → NIC without user space | Why Kafka handles GB/s per broker with low CPU |
| Kafka record batch: same format on disk, wire, and producer buffer | Compressed together. Varint encoding. Efficient I/O |
| ISR: in-sync replicas. High watermark: safe to consume | acks=all + min.insync.replicas=2 prevents data loss |
| Consumer offsets in __consumer_offsets topic | Manual commit: process-then-commit = at-least-once |
| At-most-once: fire and forget. At-least-once: ack after write. Exactly-once: idempotent + txn | Exactly-once within Kafka. External systems need idempotent consumers |
| Event sourcing: store events, derive state by replay | Complete audit trail, temporal queries. High complexity, schema evolution is hard |
| CQRS: separate write model from read model | Write-optimized store + read-optimized projections. Eventually consistent |
| Saga: local transactions + compensating actions | Orchestration (centralized) vs choreography (decentralized events) |
| Dead letter queues: don't lose unprocessable messages | Monitor DLQ depth. Include error context. Replay after fix |
| Backpressure: drop, buffer, or slow down producer | Kafka: disk buffer (natural). RabbitMQ: flow control (blocks publishers) |
