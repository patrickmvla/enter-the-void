# API Protocols — REST vs gRPC vs GraphQL

## What This Is About

An API (Application Programming Interface) defines how two systems
communicate. This topic covers the three dominant API paradigms at
the **protocol level** — how they encode data, what transport they use,
how they handle errors, and the engineering tradeoffs behind each.

These aren't just "different ways to call a function over the network."
Each makes fundamentally different choices about schema definition,
serialization, transport, streaming, and coupling between client and
server. Those choices cascade into real architectural constraints.

---

## REST — Representational State Transfer

### What REST Actually Is

REST (Fielding, 2000) is an architectural style, not a protocol. Most
"REST APIs" are really "HTTP APIs with JSON." True REST is defined by
constraints:

1. **Client-Server**: Separation of concerns.
2. **Stateless**: Each request contains all information needed to process
   it. No server-side session state between requests.
3. **Cacheable**: Responses must declare themselves cacheable or not.
4. **Uniform Interface**: Resources are identified by URIs. Representations
   (JSON, HTML) are transferred. Self-descriptive messages. HATEOAS.
5. **Layered System**: Intermediaries (proxies, CDNs, load balancers) can
   be inserted transparently.
6. **Code on Demand** (optional): Server can send executable code.

**HATEOAS** (Hypermedia As The Engine Of Application State): The
response should contain links to related actions, so the client
discovers the API by following links rather than hardcoding URLs:

```json
{
    "id": 42,
    "name": "alice",
    "links": {
        "self": "/users/42",
        "posts": "/users/42/posts",
        "delete": { "href": "/users/42", "method": "DELETE" }
    }
}
```

Almost no production API implements HATEOAS. In practice, REST means
"JSON over HTTP with resource-oriented URLs." And that's fine — the
HTTP constraints (methods, status codes, caching, content negotiation)
provide most of REST's value without HATEOAS's complexity.

### REST in Practice

**Resource-oriented URL design:**
```
GET    /users           → list users
POST   /users           → create a user
GET    /users/42        → get user 42
PUT    /users/42        → replace user 42
PATCH  /users/42        → partially update user 42
DELETE /users/42        → delete user 42
GET    /users/42/posts  → list user 42's posts
```

**URL design principles:**
- Nouns, not verbs (`/users/42`, not `/getUser?id=42`)
- Plural resource names (`/users`, not `/user`)
- Nested resources for clear ownership (`/users/42/posts`)
- Query parameters for filtering, sorting, pagination
  (`/users?role=admin&sort=-created_at&page=2&limit=20`)
- No trailing slashes (or be consistent)

**Pagination patterns:**

```json
// Offset-based (simple but has consistency issues on inserts/deletes)
GET /users?offset=20&limit=10

// Cursor-based (stable, efficient, can't jump to page N)
GET /users?cursor=eyJpZCI6NDJ9&limit=10
{
    "data": [...],
    "cursors": {
        "next": "eyJpZCI6NTJ9",
        "prev": "eyJpZCI6NDN9"
    }
}

// Keyset pagination (cursor under the hood, uses indexed columns)
GET /users?created_after=2024-01-01T00:00:00Z&limit=10
```

**Offset-based** breaks when items are inserted or deleted between pages
(you skip or duplicate items). **Cursor-based** encodes the position in
an opaque token and is stable regardless of mutations. Most production
APIs use cursor-based pagination.

### REST Weaknesses

**Over-fetching**: `GET /users/42` returns everything about the user
(name, email, avatar, preferences, activity_log, ...) when the client
only needs the name. No way to specify which fields you want
(some APIs add `?fields=name,email` but it's not standardized).

**Under-fetching**: To display a user profile with their posts and
follower count, the client needs:
```
GET /users/42            → user data
GET /users/42/posts      → their posts
GET /users/42/followers  → follower count
```
Three round trips for one screen. On a 200ms RTT mobile connection,
that's 600ms of latency minimum.

**No schema contract**: REST APIs typically don't have a machine-readable
schema. Clients are built against documentation (which may be wrong or
stale). Breaking changes aren't caught until runtime.

**Versioning**: No standard approach. Options:
```
URL:     /v1/users, /v2/users        (most common, simple, but URL proliferation)
Header:  Accept: application/vnd.api.v2+json  (more "RESTful" but harder to use)
Query:   /users?version=2            (easy but feels wrong)
```

### OpenAPI (Swagger)

The de facto standard for REST API schemas. A YAML/JSON file describing
endpoints, parameters, request/response bodies, and authentication:

```yaml
openapi: 3.0.0
paths:
  /users/{id}:
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found
```

OpenAPI enables code generation (client SDKs, server stubs), automated
testing, documentation generation (Swagger UI), and contract-first
development. It retrofits some of the schema benefits that gRPC and
GraphQL have natively.

---

## gRPC — Remote Procedure Calls with Protocol Buffers

### What gRPC Is

gRPC (Google Remote Procedure Call) is a high-performance RPC framework.
The client calls a method on the server as if it were a local function.
Under the hood: HTTP/2 transport, Protocol Buffers serialization, and
code-generated client/server stubs.

```
REST:   "Send an HTTP request to a URL, parse the JSON response"
gRPC:   "Call this function on the server, get a typed response"
```

### Protocol Buffers (protobuf)

Protobuf is the serialization format. You define messages in `.proto`
files, and the protobuf compiler generates code in your target language.

```protobuf
syntax = "proto3";

package users;

service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListUsersRequest) returns (stream User);
    rpc CreateUser (CreateUserRequest) returns (User);
    rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
    int32 id = 1;
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    repeated string roles = 4;
    google.protobuf.Timestamp created_at = 5;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}
```

### Protobuf Binary Encoding

Protobuf encodes data as a sequence of field-tag + value pairs. Each
field has a tag encoding the field number and wire type:

```
Tag = (field_number << 3) | wire_type

Wire types:
  0 = Varint (int32, int64, bool, enum)
  1 = 64-bit (fixed64, double)
  2 = Length-delimited (string, bytes, embedded messages, repeated)
  5 = 32-bit (fixed32, float)
```

**Varint encoding**: Variable-length integer. Small numbers use fewer
bytes. Each byte uses 7 bits for data and 1 bit (MSB) to indicate if
more bytes follow:

```
Value: 150
Binary: 10010110 (doesn't fit in 7 bits)

Encoded: 10010110 00000001
         ↑                ↑
         MSB=1 (more)     MSB=0 (done)
         data: 0010110    data: 0000001

Decode: 0000001 0010110 = 10010110 = 150

The integer 1 encodes as: 00000001 (1 byte)
The integer 150 encodes as: 10010110 00000001 (2 bytes)
The integer 300 encodes as: 10101100 00000010 (2 bytes)
```

**Example encoding of a User message:**

```
User { id: 42, name: "alice" }

Field 1 (id=42):
  Tag: (1 << 3) | 0 = 0x08    (field 1, varint)
  Value: 0x2A                   (42 as varint)

Field 2 (name="alice"):
  Tag: (2 << 3) | 2 = 0x12    (field 2, length-delimited)
  Length: 0x05                  (5 bytes)
  Value: 61 6C 69 63 65        ("alice" in UTF-8)

Wire bytes: 08 2A 12 05 61 6C 69 63 65  (9 bytes)

JSON equivalent: {"id":42,"name":"alice"}  (24 bytes)
```

Protobuf is **2-5x smaller** than JSON for typical payloads. No field
names on the wire (just numbers), no quoting, no delimiters, compact
integer encoding. For high-throughput services (10K+ requests/second),
this saves significant bandwidth and CPU (parsing binary is faster than
parsing JSON).

### gRPC Streaming Modes

gRPC supports four communication patterns over HTTP/2:

**1. Unary (standard request-response):**
```
Client → GetUser(id=42) → Server
Client ← User{...}      ← Server
```

**2. Server streaming:**
```
Client → ListUsers(page_size=100) → Server
Client ← User{...}                ← Server
Client ← User{...}                ← Server
Client ← User{...}                ← Server
Client ← (end of stream)          ← Server
```

The server sends a stream of responses. Used for: pagination without
multiple round trips, live feeds, large result sets.

**3. Client streaming:**
```
Client → UploadChunk{...} → Server
Client → UploadChunk{...} → Server
Client → UploadChunk{...} → Server
Client → (end of stream)  → Server
Client ← UploadResult{...} ← Server
```

The client sends a stream of requests, server responds once. Used for:
file uploads, batched writes, sensor data ingestion.

**4. Bidirectional streaming:**
```
Client → ChatMessage{...} → Server
Client ← ChatMessage{...} ← Server
Client → ChatMessage{...} → Server
Client → ChatMessage{...} → Server
Client ← ChatMessage{...} ← Server
```

Both sides send streams independently. Used for: chat, real-time
collaboration, interactive protocols.

All four modes are first-class in gRPC. Streaming in REST requires
workarounds (SSE, WebSockets, chunked transfer).

### gRPC on the Wire

gRPC frames messages over HTTP/2:

```
HTTP/2 Request:
  :method: POST
  :path: /users.UserService/GetUser
  :scheme: https
  content-type: application/grpc
  grpc-encoding: gzip
  te: trailers

  [Length-Prefixed Message]:
    [1 byte: compressed flag][4 bytes: message length][protobuf bytes]

HTTP/2 Response:
  :status: 200
  content-type: application/grpc

  [Length-Prefixed Message]:
    [1 byte: compressed flag][4 bytes: message length][protobuf bytes]

  Trailers:
    grpc-status: 0
    grpc-message: OK
```

**Key details:**
- Every RPC is an HTTP/2 POST (method semantics live in the proto
  definition, not the HTTP method)
- Path is `/{package}.{Service}/{Method}`
- Status is in HTTP/2 **trailers** (sent after the body), not in the
  HTTP status code (which is always 200 for gRPC). This allows
  streaming — the server doesn't know the final status until the
  stream is complete
- Messages are length-prefixed (5-byte header: compression flag + 4-byte
  big-endian length)

### gRPC Status Codes

gRPC has its own status code system (separate from HTTP):

| Code | Name | Meaning |
|------|------|---------|
| 0 | OK | Success |
| 1 | CANCELLED | Client cancelled |
| 2 | UNKNOWN | Unknown error |
| 3 | INVALID_ARGUMENT | Bad input (like HTTP 400) |
| 4 | DEADLINE_EXCEEDED | Timeout (like HTTP 504) |
| 5 | NOT_FOUND | Resource not found (like HTTP 404) |
| 7 | PERMISSION_DENIED | Forbidden (like HTTP 403) |
| 8 | RESOURCE_EXHAUSTED | Rate limited (like HTTP 429) |
| 12 | UNIMPLEMENTED | Method not implemented (like HTTP 501) |
| 13 | INTERNAL | Server error (like HTTP 500) |
| 14 | UNAVAILABLE | Service down (like HTTP 503) |
| 16 | UNAUTHENTICATED | Not authenticated (like HTTP 401) |

### gRPC Weaknesses

**Browser support**: Browsers can't make raw HTTP/2 requests with
trailers. gRPC-Web is a proxy-based workaround — a proxy (Envoy)
translates between gRPC-Web (browser-friendly) and native gRPC. Adds
complexity and a dependency.

**Human readability**: Binary protobuf is not debuggable with curl or
browser dev tools. You need gRPC-specific tools (`grpcurl`, `grpcui`,
Postman's gRPC support). REST + JSON is debuggable with any HTTP tool.

**Tight coupling**: The proto file is a shared contract. Changes require
regenerating client code. Proto schema evolution rules (don't reuse field
numbers, add new fields as optional) prevent breaking existing clients,
but the development workflow is heavier than REST's "just change the JSON."

**Load balancing**: gRPC uses long-lived HTTP/2 connections. Traditional
L4 (TCP) load balancers distribute connections, not requests — one
connection carrying 1000 RPCs looks like one connection. You need L7
(application-aware) load balancers that understand HTTP/2 streams, or
client-side load balancing (the gRPC client maintains connections to
multiple servers and distributes requests itself).

---

## GraphQL — Query Language for APIs

### What GraphQL Is

GraphQL (Facebook, 2015) is a query language that lets the client
specify exactly what data it needs. The server exposes a schema, and
the client writes queries against it. One endpoint, one request, exactly
the data you asked for.

### The Schema

GraphQL APIs are defined by a type system:

```graphql
type User {
    id: ID!
    name: String!
    email: String!
    posts(first: Int, after: String): PostConnection!
    followers: [User!]!
    createdAt: DateTime!
}

type Post {
    id: ID!
    title: String!
    body: String!
    author: User!
    comments: [Comment!]!
}

type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
}

type PostEdge {
    node: Post!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    endCursor: String
}

type Query {
    user(id: ID!): User
    users(first: Int, after: String): UserConnection!
    post(id: ID!): Post
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
}

type Subscription {
    messageReceived(channelId: ID!): Message!
}
```

`!` means non-nullable. The schema is the contract — it's introspectable
(clients can query the schema itself) and validated at both compile time
(codegen) and runtime (the server rejects invalid queries).

### Queries

The client asks for exactly what it needs:

```graphql
# Solves the under-fetching problem: one request, all data
query UserProfile {
    user(id: "42") {
        name
        email
        posts(first: 5) {
            edges {
                node {
                    title
                    comments {
                        body
                        author {
                            name
                        }
                    }
                }
            }
            pageInfo {
                hasNextPage
                endCursor
            }
        }
        followers {
            name
        }
    }
}
```

This single query replaces 3+ REST requests and returns only the
requested fields (no over-fetching).

### How It Works on the Wire

GraphQL is transport-agnostic but almost always runs over HTTP:

```http
POST /graphql HTTP/1.1
Content-Type: application/json

{
    "query": "query { user(id: \"42\") { name email } }",
    "variables": { "id": "42" },
    "operationName": "GetUser"
}
```

Response:
```json
{
    "data": {
        "user": {
            "name": "alice",
            "email": "alice@example.com"
        }
    }
}
```

**Always POST** (queries are in the body). GET is possible for simple
queries (query in query parameter) and enables HTTP caching, but POST
is the norm because queries can be arbitrarily long.

**Single endpoint**: `/graphql`. No resource-oriented URLs. The query
itself determines what data is accessed. This means HTTP caching
doesn't work out of the box (every request is POST to the same URL).
CDN caching requires application-level strategies (persisted queries,
`@cacheControl` directives).

### Resolvers — How the Server Processes Queries

The server maps each field in the schema to a **resolver function**:

```javascript
const resolvers = {
    Query: {
        user: (parent, { id }, context) => {
            return context.db.users.findById(id);
        }
    },
    User: {
        posts: (user, { first, after }, context) => {
            return context.db.posts.findByAuthor(user.id, { first, after });
        },
        followers: (user, args, context) => {
            return context.db.follows.findFollowers(user.id);
        }
    },
    Post: {
        author: (post, args, context) => {
            return context.db.users.findById(post.authorId);
        },
        comments: (post, args, context) => {
            return context.db.comments.findByPost(post.id);
        }
    }
};
```

The GraphQL engine traverses the query tree, calling resolvers at each
level. The result is assembled into the response matching the query
shape.

### The N+1 Problem

Consider the query:
```graphql
{
    users(first: 10) {
        name
        posts { title }
    }
}
```

Naive execution:
```
1 query: SELECT * FROM users LIMIT 10
10 queries: SELECT * FROM posts WHERE author_id = ?  (one per user)
= 11 queries (N+1 pattern)
```

**DataLoader** solves this by batching and caching:

```javascript
const postLoader = new DataLoader(async (userIds) => {
    // Single query: SELECT * FROM posts WHERE author_id IN (?, ?, ...)
    const posts = await db.posts.findByAuthorIds(userIds);
    // Return posts grouped by userId, in the same order as userIds
    return userIds.map(id => posts.filter(p => p.authorId === id));
});

// Resolver uses the loader
User: {
    posts: (user) => postLoader.load(user.id)
}
```

DataLoader collects all `.load(id)` calls within a single tick of the
event loop and batches them into one database query. Result: 2 queries
instead of 11.

### GraphQL Weaknesses

**Complexity cost queries**: A malicious or poorly written query can
request deeply nested data that generates enormous database load:

```graphql
# Dangerous: exponential expansion
{
    users {
        followers {
            followers {
                followers {
                    followers {
                        name
                    }
                }
            }
        }
    }
}
```

**Defenses:**
- **Query depth limiting**: Reject queries deeper than N levels
- **Query complexity analysis**: Assign costs to fields (`followers`
  costs more than `name`) and reject queries exceeding a budget
- **Persisted queries**: Only allow pre-approved query hashes — the
  client sends a hash, the server looks up the corresponding query.
  Eliminates arbitrary queries entirely
- **Rate limiting per query complexity**: Instead of rate limiting per
  request, limit by estimated cost

**Caching difficulty**: REST's URL-based caching (CDN caches GET
`/users/42`) doesn't work for POST `/graphql`. Solutions:
- Client-side normalized caching (Apollo Client, urql) — stores entities
  by type + ID and updates all queries referencing them
- Persisted queries enable GET requests with query hashes, making CDN
  caching possible
- `@cacheControl` directives per field

**File uploads**: GraphQL has no native file upload mechanism. The
multipart request specification (used by Apollo) is a community
standard, not part of the GraphQL spec. Many teams use REST for
uploads alongside GraphQL for data.

**Error handling**: GraphQL always returns HTTP 200 (the GraphQL request
itself succeeded, even if the resolvers errored). Errors are in the
response body:

```json
{
    "data": { "user": null },
    "errors": [{
        "message": "User not found",
        "locations": [{ "line": 2, "column": 3 }],
        "path": ["user"],
        "extensions": { "code": "NOT_FOUND" }
    }]
}
```

This breaks HTTP-level error monitoring (all responses are 200) and
requires GraphQL-aware tooling.

---

## Head-to-Head Comparison

### Wire Efficiency

| Protocol | Serialization | Typical Payload | Schema |
|----------|--------------|-----------------|--------|
| REST | JSON (text) | 100% (all fields) | Optional (OpenAPI) |
| gRPC | Protobuf (binary) | ~40-60% of JSON equivalent | Required (.proto) |
| GraphQL | JSON (text) | Variable (only requested fields) | Required (SDL) |

gRPC wins on per-message size (binary encoding). GraphQL wins when
the client only needs a few fields from a large entity. REST sends
everything whether the client needs it or not.

### Latency

| Scenario | REST | gRPC | GraphQL |
|----------|------|------|---------|
| Single resource fetch | 1 round trip | 1 round trip | 1 round trip |
| Related resources | N round trips | N RPCs (or custom aggregate) | 1 round trip |
| Streaming | SSE/WebSocket (separate) | Native (HTTP/2 streams) | Subscriptions (WebSocket) |
| Connection reuse | Keep-alive | HTTP/2 multiplexing | Keep-alive |

GraphQL eliminates REST's multi-request problem. gRPC's binary
encoding and HTTP/2 multiplexing give lowest absolute latency for
high-throughput scenarios.

### Developer Experience

| Factor | REST | gRPC | GraphQL |
|--------|------|------|---------|
| Debugging | curl, browser, any HTTP tool | grpcurl, specialized tools | GraphiQL, Playground |
| Code generation | Optional (OpenAPI) | Required (protoc) | Optional but recommended |
| Documentation | Swagger/OpenAPI | Proto files ARE the docs | Schema IS the docs + introspection |
| Learning curve | Low | Medium-high | Medium |
| Browser support | Native | Requires gRPC-Web proxy | Native (it's HTTP + JSON) |
| Caching | HTTP caching works naturally | No built-in caching | Complex (no URL-based caching) |
| Error handling | HTTP status codes | gRPC status codes | Always 200, errors in body |

### When to Use What

**REST when:**
- Public APIs (broadest compatibility, easiest to consume)
- Simple CRUD operations
- HTTP caching is important
- You need the simplest possible implementation
- Browser clients without a GraphQL client library

**gRPC when:**
- Internal service-to-service communication (microservices)
- High throughput, low latency requirements
- Streaming (bidirectional real-time data)
- Strong schema enforcement with code generation
- Polyglot environments (proto files generate code in any language)

**GraphQL when:**
- Mobile clients (minimize over-fetching, reduce data transfer)
- Complex data relationships (social graphs, nested resources)
- Multiple client types with different data needs (mobile vs web)
- Rapid frontend iteration (add fields without backend changes)
- Teams where frontend and backend develop independently

### Hybrid Architectures

Production systems often combine protocols:

```
Mobile App ──GraphQL──> API Gateway ──gRPC──> Microservice A
                                    ──gRPC──> Microservice B
                                    ──REST──> Legacy Service C

Web App ──REST──> Same API Gateway ──gRPC──> ...
```

- **GraphQL at the edge**: Aggregation layer that satisfies complex
  client queries by fanning out to backend services
- **gRPC internally**: Efficient, strongly-typed service-to-service
  communication
- **REST for public/legacy**: Broadest compatibility for external
  consumers and systems that can't be changed

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| REST = HTTP + JSON + resources | Not a protocol — an architectural style. Most APIs violate REST constraints (no HATEOAS) |
| REST over/under-fetching is a real problem | Multiple round trips for related data. All fields returned even when unwanted |
| Protobuf is 2-5x smaller than JSON | Varint encoding, field numbers instead of names, no delimiters |
| gRPC uses HTTP/2 with trailers for status | All RPCs are POST. Status is in trailers, not HTTP status code |
| gRPC has 4 streaming modes natively | Unary, server-stream, client-stream, bidirectional. REST can't match this |
| gRPC needs L7 load balancing | Long-lived HTTP/2 connections make L4 balancers ineffective |
| GraphQL: one endpoint, client specifies fields | Solves over/under-fetching. But POST always returns 200, breaks HTTP caching |
| N+1 problem is GraphQL's biggest performance trap | DataLoader batches per-tick. Without it, query cost explodes |
| Query complexity limits prevent abuse | Depth limits, cost analysis, persisted queries |
| No single protocol wins everywhere | REST for public APIs, gRPC for internal services, GraphQL for complex client needs |
