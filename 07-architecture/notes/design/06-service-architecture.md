# Service Architecture — How to Structure a System

## The Spectrum

Service architecture isn't a binary choice between monolith and
microservices. It's a spectrum, and where you sit on it should be
driven by your team size, operational maturity, and actual scaling
needs — not by hype.

```
Monolith ←──────────────────────────────────────→ Microservices

  Single deployable           Modular monolith           Microservices
  (one codebase,              (one deployable,           (many deployables,
   one process,                clear module              many processes,
   one database)               boundaries,              many databases)
                               one database)
```

---

## Monolith

All functionality in a single deployable unit. One process, one
database, one codebase.

```
┌─────────────────────────────────────────────────────┐
│                    Application                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐│
│  │   Auth   │ │  Orders  │ │  Users   │ │Payments││
│  └──────────┘ └──────────┘ └──────────┘ └────────┘│
│  ┌──────────────────────────────────────────────────┤
│  │              Shared Libraries                    │
│  └──────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────┤
│  │                 ORM / Data Layer                  │
│  └──────────────────────────────────────────────────┤
└──────────────────────────────────────────────────────┘
                         │
                    ┌────▼────┐
                    │   DB    │
                    └─────────┘
```

### Why Monoliths Are Good

**Simplicity**: One codebase to understand, one deployment pipeline,
one set of logs, one database to query. A new developer can run the
entire system on their laptop.

**Refactoring**: Changing a function signature that's called from 50
places is a single commit. In microservices, it's a coordinated
change across 5 repos with versioned APIs.

**Data consistency**: Transactions are ACID. Transfer money between
accounts = one transaction, one database. No distributed transactions,
no eventual consistency, no saga pattern.

**Performance**: Function call latency is nanoseconds. Service call
latency is milliseconds. A monolith composing data from 5 modules
is 1,000x faster than a microservice doing 5 network calls.

**Debugging**: Stack trace shows the entire call path. In
microservices, you need distributed tracing (Jaeger, Zipkin) to
follow a request across services.

### When Monoliths Break Down

**Team scaling**: When 50+ developers are committing to the same
codebase, merge conflicts are constant, deploys are risky (one bad
commit blocks everyone), and build times grow.

**Independent scaling**: If the image processing module needs 10x
the CPU of the user profile module, you're scaling the entire
monolith 10x.

**Technology lock-in**: The entire application uses one language,
one framework, one database. Adding a specialized tool (graph
database, ML runtime) requires bending it into the existing stack.

**Deploy risk**: Deploying a fix to the payment module also deploys
untested changes to the user module. Blast radius is the entire
application.

---

## Modular Monolith

The practical middle ground. One deployable, but with strict module
boundaries that enforce separation.

```
┌─────────────────────────────────────────────────────┐
│                    Application                       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐│
│  │  Auth Module  │ │ Order Module │ │ User Module  ││
│  │  ┌─────────┐ │ │  ┌─────────┐ │ │  ┌─────────┐││
│  │  │Public API│ │ │  │Public API│ │ │  │Public API│││
│  │  └─────────┘ │ │  └─────────┘ │ │  └─────────┘││
│  │  ┌─────────┐ │ │  ┌─────────┐ │ │  ┌─────────┐││
│  │  │Internal │ │ │  │Internal │ │ │  │Internal │││
│  │  │ logic   │ │ │  │ logic   │ │ │  │ logic   │││
│  │  └─────────┘ │ │  └─────────┘ │ │  └─────────┘││
│  │  ┌─────────┐ │ │  ┌─────────┐ │ │  ┌─────────┐││
│  │  │  Data   │ │ │  │  Data   │ │ │  │  Data   │││
│  │  │ (schema)│ │ │  │ (schema)│ │ │  │ (schema)│││
│  │  └─────────┘ │ │  └─────────┘ │ │  └─────────┘││
│  └──────────────┘ └──────────────┘ └──────────────┘│
│         ↕ only through public API ↕                 │
└─────────────────────────────────────────────────────┘
                         │
                    ┌────▼────┐
                    │   DB    │  (shared DB, separate schemas)
                    └─────────┘
```

**Rules:**
1. Modules communicate ONLY through defined public interfaces (not
   direct database queries across module boundaries)
2. Each module owns its data (separate schema or table prefix)
3. No circular dependencies between modules
4. Enforced by linter/build rules (not just convention)

**Advantages**: Gets 80% of the modularity benefits of microservices
with 20% of the operational complexity. When (if) you need to extract
a module into a service, the boundary is already clean.

Shopify runs a modular monolith at massive scale. So does Basecamp.

---

## Microservices

Each service is independently deployable, owns its data, and
communicates over the network.

```
┌───────────┐    ┌───────────┐    ┌───────────┐
│ Auth Svc  │    │ Order Svc │    │ User Svc  │
│  ┌─────┐  │    │  ┌─────┐  │    │  ┌─────┐  │
│  │ API │  │◄──►│  │ API │  │◄──►│  │ API │  │
│  └─────┘  │    │  └─────┘  │    │  └─────┘  │
│  ┌─────┐  │    │  ┌─────┐  │    │  ┌─────┐  │
│  │ DB  │  │    │  │ DB  │  │    │  │ DB  │  │
│  └─────┘  │    │  └─────┘  │    │  └─────┘  │
└───────────┘    └───────────┘    └───────────┘
```

### The Real Costs

Microservices trade code complexity for **operational complexity**:

```
What you get:
  ✓ Independent deployment (deploy user service without touching orders)
  ✓ Independent scaling (scale the hot service, not everything)
  ✓ Technology freedom (Python for ML, Go for API, Rust for performance)
  ✓ Team autonomy (team owns a service end-to-end)
  ✓ Blast radius containment (user service bug doesn't crash payments)

What you pay:
  ✗ Network is unreliable (timeouts, retries, circuit breakers)
  ✗ Distributed transactions (sagas, eventual consistency)
  ✗ Data joins across services (no SQL JOIN across databases)
  ✗ Distributed debugging (tracing, log correlation)
  ✗ Integration testing is hard (must run many services together)
  ✗ Operational overhead (N deployments, N databases, N monitoring)
  ✗ Service discovery, load balancing, health checking per service
  ✗ API versioning and backward compatibility
  ✗ Reduced availability (serial dependency: 99.9%^10 = 99%)
```

### When Microservices Make Sense

**Team size**: Multiple teams (5+ engineers each) that would step on
each other in a shared codebase. Each team owns 1-3 services.

**Scale heterogeneity**: One part of the system needs 100x the
resources of another. Scale independently.

**Technology requirements**: Different parts genuinely need different
tech (ML pipeline in Python, low-latency API in Go, data pipeline in
Spark).

**Deployment independence**: You need to deploy the payment service 5
times a day without touching the rest of the system.

**Rule of thumb**: Start with a monolith (or modular monolith). Extract
services only when you have a concrete reason. Amazon, Netflix, and
Uber all started as monoliths and extracted services incrementally.

---

## Service Communication Patterns

### Synchronous (Request-Response)

```
Service A → HTTP/gRPC request → Service B → response → Service A

Pros: Simple. Immediate result. Easy to reason about.
Cons: Coupling. A depends on B being available. Latency accumulates.
      A is blocked waiting for B.
```

### Asynchronous (Event-Driven)

```
Service A → publishes event to queue → Service B consumes later

Pros: Decoupled. A doesn't wait. B can process at its own pace.
      B can be down temporarily (queue buffers).
Cons: Eventual consistency. Harder to debug. No immediate response.
```

### When to Use Each

```
Synchronous:
  - User-facing requests that need an immediate response
  - Reads that require current data (GET /user/42)
  - Operations that need immediate confirmation (payment charge)

Asynchronous:
  - Notifications (email, push, SMS)
  - Analytics and logging
  - Heavy processing (image resize, report generation)
  - Cross-service data synchronization
  - Any operation where "accepted for processing" is an acceptable response
```

**Hybrid** (common production pattern):

```
User → API Gateway → Order Service (sync: create order)
  Order Service → Kafka (async: "OrderCreated" event)
    → Payment Service (charges card)
    → Inventory Service (reserves items)
    → Notification Service (sends confirmation email)
    → Analytics Service (records event)

The user gets an immediate response ("order received").
Downstream processing happens asynchronously.
If payment fails, the saga compensates.
```

---

## API Gateway

A single entry point that routes requests to the appropriate service.

```
Client → [API Gateway] → /auth/*   → Auth Service
                       → /users/*  → User Service
                       → /orders/* → Order Service
                       → /search/* → Search Service
```

### Responsibilities

```
Routing:          Path/host-based routing to services
Authentication:   Validate JWT/API key once at the gateway
Rate limiting:    Per-client or per-API rate limits
TLS termination:  HTTPS at the edge, HTTP internally
Request/Response transformation: Aggregate multiple service calls
Caching:          Cache common responses
Logging:          Centralized access logging
CORS:             Handle cross-origin headers
Protocol translation: REST to gRPC, WebSocket upgrade
```

### BFF (Backend for Frontend)

Instead of one generic API gateway, create specialized gateways per
client type:

```
Mobile app  → [Mobile BFF]  → services (optimized for mobile: smaller payloads,
                                         aggregated responses, offline support)
Web app     → [Web BFF]     → services (optimized for web: SSR data,
                                         CDN-friendly responses)
Admin panel → [Admin BFF]   → services (optimized for admin: bulk operations,
                                         detailed data)
```

Each BFF tailors the API to its client's needs. The mobile BFF
might aggregate 3 service calls into 1 response to reduce round
trips over a cellular network.

---

## Service Discovery

In a dynamic environment (containers, auto-scaling), service addresses
change. Service discovery provides a registry of where services are.

### Client-Side Discovery

```
Service A:
  1. Query service registry: "Where is User Service?"
  2. Registry returns: [10.0.1.1:8080, 10.0.1.2:8080, 10.0.1.3:8080]
  3. Client picks one (with load balancing logic)
  4. Sends request directly to chosen instance

Registry: Consul, etcd, ZooKeeper
  Services register themselves on startup.
  Health checks remove unhealthy instances.

Pros: No extra hop. Client can implement smart routing.
Cons: Client library must be implemented in every language.
```

### Server-Side Discovery

```
Service A:
  1. Sends request to load balancer: GET http://user-service/users/42
  2. Load balancer queries registry
  3. Routes to a healthy instance

Pros: Client is simple (just a hostname). Language-agnostic.
Cons: Extra network hop through the load balancer.

This is what Kubernetes does:
  Service name → ClusterIP → kube-proxy → Pod

  "user-service" resolves to a virtual IP via DNS.
  kube-proxy (or iptables/IPVS) routes to a healthy Pod.
  The calling service just uses the hostname.
```

### DNS-Based Discovery

```
Service name → DNS SRV record → IP:port of healthy instances

_user._tcp.service.consul.  SRV  10 0 8080 user-1.node.consul.
_user._tcp.service.consul.  SRV  10 0 8080 user-2.node.consul.

Pros: Universal (every language has DNS).
Cons: DNS TTL caching can cause stale routing.
      DNS doesn't support health checking natively
      (requires integration with Consul/etc.)
```

---

## Service Mesh

A dedicated infrastructure layer for service-to-service communication.
Instead of implementing retries, timeouts, circuit breakers, mTLS, and
observability in every service's code, these concerns are handled by
a **sidecar proxy** that runs alongside each service.

```
Without service mesh:
  Each service implements: retries, timeouts, circuit breaker,
  mTLS, tracing, metrics, service discovery, load balancing
  → duplicated across every service, every language

With service mesh:
  ┌─────────────────────────┐  ┌─────────────────────────┐
  │ Service A               │  │ Service B               │
  │ (application code only) │  │ (application code only) │
  │                         │  │                         │
  │ ┌─────────────────────┐ │  │ ┌─────────────────────┐ │
  │ │ Sidecar Proxy       │◄├──┤►│ Sidecar Proxy       │ │
  │ │ (Envoy)             │ │  │ │ (Envoy)             │ │
  │ │ - mTLS              │ │  │ │ - mTLS              │ │
  │ │ - retries           │ │  │ │ - retries           │ │
  │ │ - circuit breaker   │ │  │ │ - circuit breaker   │ │
  │ │ - tracing           │ │  │ │ - tracing           │ │
  │ │ - metrics           │ │  │ │ - metrics           │ │
  │ │ - load balancing    │ │  │ │ - load balancing    │ │
  │ └─────────────────────┘ │  │ └─────────────────────┘ │
  └─────────────────────────┘  └─────────────────────────┘

  Control Plane (Istio/Linkerd):
    - Pushes configuration to all sidecar proxies
    - Certificate authority for mTLS
    - Collects telemetry from all proxies
    - Defines traffic policies (canary, retries, rate limits)
```

**How the sidecar proxy works (Envoy):**

All traffic to/from the service is intercepted by the sidecar via
iptables rules (or eBPF in newer implementations). The application
thinks it's talking directly to the destination, but the sidecar
handles TLS, load balancing, retries, and observability transparently.

```
Application sends: HTTP to user-service:8080
  → iptables redirects to local Envoy on port 15001
  → Envoy resolves "user-service" via control plane config
  → Envoy establishes mTLS to destination Envoy
  → Destination Envoy forwards to local Service B
  → Response flows back the same path
```

**When a service mesh is worth it:**
- 20+ services with multiple teams
- Need consistent mTLS, observability, traffic policies
- Polyglot environment (can't standardize on one library)
- Need canary deployments, traffic splitting, fault injection

**When it's not worth it:**
- < 10 services (overhead exceeds benefit)
- Single team (just use a shared library)
- Latency-sensitive (sidecar adds ~1-3ms per hop)

---

## Migration Patterns

### Strangler Fig

Incrementally migrate from a monolith to services by routing
traffic based on the request:

```
Phase 1: Everything goes to monolith
  Client → [Router] → Monolith

Phase 2: Extract user service, route user requests to it
  Client → [Router] → /users/*  → User Service (NEW)
                    → /*         → Monolith (everything else)

Phase 3: Extract order service
  Client → [Router] → /users/*  → User Service
                    → /orders/* → Order Service (NEW)
                    → /*         → Monolith (shrinking)

Phase N: Monolith is empty, decommission it
```

Named after the strangler fig tree, which grows around a host tree
and eventually replaces it. The monolith is never "rewritten" — it
slowly shrinks as functionality is extracted.

**Rules:**
- Never rewrite from scratch (Joel Spolsky's "Things You Should Never Do")
- Extract one module at a time
- Route at the API gateway/proxy level
- The monolith and new services coexist during migration
- Rollback is trivial (change the routing rule)

### Database Decomposition

The hardest part of extracting a service: separating its data.

```
Step 1: Service extracted, but shared database
  User Service → [shared DB, users table]
  Order Service → [shared DB, orders table]
  Problem: services can still read each other's tables. Coupling.

Step 2: Logical separation (separate schemas, same DB)
  User Service → [DB, user_schema.users]
  Order Service → [DB, order_schema.orders]
  Enforce: services can't access other schemas. Cross-schema
  queries replaced by API calls.

Step 3: Physical separation (separate databases)
  User Service → [User DB]
  Order Service → [Order DB]
  Full independence. But: no cross-database joins.
  Order Service needs user data → calls User Service API.
```

**The join problem:**
```
Monolith query:
  SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id
  → One query, one database, sub-millisecond.

Microservices equivalent:
  Order Service: get orders → [order_id, user_id, amount]
  For each unique user_id: call User Service → get name
  Merge in memory.
  → N+1 network calls, milliseconds each. 100x slower.

Solutions:
  1. Denormalize: store user_name in the orders table
     (update via event: User Service publishes "UserNameChanged")
  2. CQRS: maintain a read-optimized view with pre-joined data
  3. API composition: Order Service calls User Service with batch API
     (POST /users/batch with list of IDs → single network call)
```

---

## Observability

In a distributed system, observability is mandatory, not optional.
Three pillars:

### Logs

Structured, correlated logs across services:

```json
{
    "timestamp": "2024-06-15T14:30:00.123Z",
    "level": "info",
    "service": "order-service",
    "trace_id": "abc123def456",
    "span_id": "789ghi",
    "user_id": "user_42",
    "message": "Order created",
    "order_id": "ord_999",
    "amount": 99.99,
    "duration_ms": 45
}
```

The `trace_id` correlates logs across services. Filter by trace_id
to see the entire request flow.

### Metrics

Numerical measurements aggregated over time:

```
Four golden signals (Google SRE):
  1. Latency:     how long requests take (p50, p95, p99)
  2. Traffic:     how many requests per second
  3. Errors:      error rate (5xx / total)
  4. Saturation:  how full the system is (CPU, memory, queue depth)

RED method (for request-driven services):
  Rate, Errors, Duration

USE method (for resources — CPU, memory, disk):
  Utilization, Saturation, Errors
```

### Distributed Tracing

Track a request as it flows through multiple services:

```
Trace: abc123def456

├─ API Gateway (5ms)
│  ├─ Auth Service: validate token (2ms)
│  └─ Order Service: create order (40ms)
│     ├─ DB: INSERT order (3ms)
│     ├─ Inventory Service: reserve (15ms)
│     │  └─ DB: UPDATE inventory (5ms)
│     └─ Kafka: publish OrderCreated (2ms)

Total: 45ms (API Gateway end-to-end)
Bottleneck: Inventory Service (15ms)
```

Each service adds a **span** (a unit of work with start time, end
time, and metadata). Spans are linked by the trace ID (propagated
via headers: `traceparent: 00-abc123-789ghi-01`).

Tools: Jaeger, Zipkin, AWS X-Ray, Datadog APM.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Start monolith, extract services when forced | Amazon, Netflix, Uber all started as monoliths. Don't distribute prematurely |
| Modular monolith: 80% of microservice benefits, 20% of the cost | Module boundaries, separate schemas, public interfaces. Shopify's approach |
| Microservices trade code complexity for operational complexity | Network failure, distributed transactions, debugging, N pipelines, reduced availability |
| 99.9%^10 = 99.0% — serial dependencies reduce availability | Each network hop is a failure point. Minimize the critical path |
| Sync for user-facing reads, async for processing and events | Hybrid: immediate response + async downstream processing via queue |
| API Gateway: single entry point for routing, auth, rate limiting | BFF pattern: specialized gateways per client type (mobile, web, admin) |
| Service discovery: client-side (Consul) or server-side (Kubernetes) | Kubernetes DNS-based discovery is the production standard |
| Service mesh: sidecar proxy handles retries, mTLS, observability | Worth it at 20+ services. Adds ~1-3ms latency per hop. Istio/Linkerd |
| Strangler fig: incrementally extract from monolith | Never rewrite from scratch. Route at the gateway, one module at a time |
| Database decomposition is the hardest part of service extraction | Shared DB → separate schema → separate DB. Denormalize or use CQRS for joins |
| Observability: logs (trace_id), metrics (4 golden signals), traces | Distributed tracing is mandatory in microservices. Not optional |
