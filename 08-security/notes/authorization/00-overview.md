# Authorization — Controlling Access

## What Authorization Is

Authorization is the process of determining whether an authenticated
entity is **permitted** to perform a requested action on a resource.
Authentication answers "who are you?" Authorization answers "what can
you do?"

These are decoupled concerns. You can be fully authenticated (the system
knows exactly who you are) and still be denied access (you don't have
permission). Conversely, some systems grant access without authentication
(public endpoints, anonymous access) — authorization without identity.

---

## The Core Abstraction

Every authorization decision, regardless of the model or implementation,
reduces to:

```
authorize(subject, action, resource) → allow | deny
```

- **Subject**: The entity requesting access. A user, a service, a role,
  a group, a token.
- **Action**: What they want to do. Read, write, delete, execute, approve,
  transfer.
- **Resource**: What they want to do it to. A file, a database row, an API
  endpoint, a feature, a configuration.

The **authorization model** defines how the rules governing this decision
are structured, stored, and evaluated.

---

## Why Authorization Is Harder Than Authentication

Authentication is a binary question: is this person who they claim to be?
Yes or no.

Authorization is combinatorial. Consider a simple system with:
- 1,000 users
- 50 actions
- 100,000 resources

The authorization matrix has 1,000 × 50 × 100,000 = **5 billion** possible
decisions. You can't enumerate them all. You need a model that expresses
the rules compactly and evaluates them efficiently.

Additionally:
- Rules change frequently (new employees, role changes, resource creation)
- Context matters (time of day, location, device, risk score)
- Relationships are hierarchical (folders contain files, teams contain
  members, organizations contain teams)
- Rules conflict (one rule allows, another denies — who wins?)
- Audit requirements demand explainability (why was this access granted?)

---

## The Models

Each model is a different strategy for compressing the authorization matrix
into manageable rules.

| Model | Strategy | Deep Dive |
|-------|----------|-----------|
| **ACL** | List who can do what, per resource | [01-authorization-models.md](01-authorization-models.md#acls--access-control-lists) |
| **RBAC** | Group permissions into roles, assign roles to users | [01-authorization-models.md](01-authorization-models.md#rbac--role-based-access-control) |
| **ABAC** | Evaluate attributes of subject, resource, action, and environment at runtime | [01-authorization-models.md](01-authorization-models.md#abac--attribute-based-access-control) |
| **ReBAC** | Derive permissions from relationships between entities (graph-based) | [01-authorization-models.md](01-authorization-models.md#rebac--relationship-based-access-control) |
| **Capability-based** | Token IS the permission — no identity lookup | [01-authorization-models.md](01-authorization-models.md#capability-based-security) |

These are not mutually exclusive. Production systems combine them:
- **RBAC + ABAC**: "Editors can write posts" (RBAC) + "only their own
  posts, during business hours" (ABAC)
- **RBAC + ReBAC**: "Admins can manage the org" (RBAC) + "members of
  a team can access team repos" (ReBAC)

---

## The Principles

### Default Deny

If no rule explicitly grants access, access is denied. Never default to
allow. A missing rule should lock things down, not open them up.

### Least Privilege

Grant the minimum permissions necessary. No more. An intern doesn't need
admin access. A read-only service doesn't need write permissions. A
deployment script doesn't need access to production secrets it doesn't
use.

### Separation of Duties

No single person should have end-to-end control over a critical process.
The person who writes code shouldn't approve their own deployment. The
person who requests a payment shouldn't approve it.

### Fail Closed

If the authorization system is unavailable (network partition, service
crash, timeout), deny access. Don't fall back to "allow everything until
the system recovers." Availability failures should not become security
failures.

---

## Where Authorization Lives

```
                        ┌─────────────────────┐
  API Gateway           │  Coarse-grained      │  "Is this user authenticated?"
  (edge)                │  rate limiting,       │  "Is this endpoint public?"
                        │  IP blocking          │
                        └──────────┬────────────┘
                                   │
                        ┌──────────▼────────────┐
  Application Layer     │  Business logic        │  "Can this user edit THIS post?"
  (service code)        │  authorization         │  "Is this their own resource?"
                        │  (RBAC, ABAC, ReBAC)   │  "Are they in the right tenant?"
                        └──────────┬────────────┘
                                   │
                        ┌──────────▼────────────┐
  Data Layer            │  Row-level security    │  "Filter rows by tenant_id"
  (database)            │  column-level security │  "Hide salary column from
                        │  (PostgreSQL RLS)      │   non-HR roles"
                        └───────────────────────┘
```

Defense in depth: authorization should be enforced at **multiple layers**.
A bug in the application layer that skips an authorization check should be
caught by RLS at the data layer. A misconfigured API gateway should be
caught by application-level checks.

The application layer is where most authorization logic lives. The other
layers are safety nets.
