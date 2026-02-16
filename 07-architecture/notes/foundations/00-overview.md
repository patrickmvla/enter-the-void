# Architecture — Designing Systems That Scale

## The Core Problem

A single server handles ~10,000 requests per second. A single database
holds ~10 TB comfortably. These numbers are generous for most
applications, and a surprising number of systems never need to exceed
them.

But when they do — when you're serving millions of concurrent users,
processing billions of events per day, or storing petabytes — single
machines aren't enough. Architecture is the art of combining multiple
machines, services, and infrastructure components into a system that's
fast, reliable, and maintainable.

The difficulty isn't the individual components (load balancers, caches,
queues). It's the interactions between them — failure modes, consistency
trade-offs, latency propagation, and operational complexity. Every
architectural decision is a trade-off, and understanding what you're
trading away is more important than knowing the "right" answer.

---

## The Topics

```
Start here
│
├── foundations/
│   ├── 01-fundamentals.md
│   │   Latency numbers every engineer must know. Back-of-envelope
│   │   capacity estimation. SLAs, SLOs, SLIs. The math behind
│   │   availability. Capacity planning from first principles.
│   │
│   └── 02-load-balancing.md
│       L4 vs L7 load balancing. Algorithms (round-robin, least-conn,
│       consistent hashing). Health checking. Session affinity.
│       Connection draining. HAProxy/nginx internals.
│
├── patterns/
│   ├── 03-caching.md
│   │   Cache-aside, read-through, write-through, write-behind.
│   │   Eviction policies. Cache stampede and thundering herd.
│   │   Redis internals. CDN architecture. Memcached vs Redis.
│   │   Distributed caching and consistency.
│   │
│   ├── 04-message-queues.md
│   │   Kafka internals (log-structured storage, partitions, ISR,
│   │   consumer groups). AMQP and RabbitMQ. Delivery guarantees
│   │   (at-most-once, at-least-once, exactly-once). Event sourcing.
│   │   CQRS. Dead letter queues. Backpressure.
│   │
│   └── 05-rate-limiting.md
│       Token bucket, leaky bucket, sliding window algorithms.
│       Distributed rate limiting. Circuit breakers. Bulkhead
│       pattern. Retry strategies with exponential backoff and
│       jitter. Backpressure propagation.
│
└── design/
    └── 06-service-architecture.md
        Monolith vs microservices (honest trade-offs). Service
        decomposition. API gateways. Service discovery. Service
        mesh and sidecars. Saga pattern for distributed transactions.
        Strangler fig migration. Domain-driven design boundaries.
```

Each topic builds on the fundamentals. Load balancing distributes
traffic. Caching reduces load. Message queues decouple services.
Rate limiting protects them. Service architecture ties everything
together into a coherent system design.

Note: CAP theorem and consistency models are covered in
06-databases/notes/distributed/04-replication.md. API design
(REST, gRPC, GraphQL) is covered in
05-networking/notes/application-layer/03-apis.md. This section
focuses on infrastructure patterns, not protocol-level design.
