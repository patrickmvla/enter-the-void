# Authorization Models — Who Can Do What, and How the System Decides

## The Fundamental Question

Authentication answers: **"Who are you?"**
Authorization answers: **"What are you allowed to do?"**

These are separate concerns. A system can know exactly who you are and still
need to decide whether you're allowed to read this file, delete that record,
or modify that configuration. Authorization is the decision engine.

Every authorization system, no matter how complex, resolves to one function:

```
authorize(subject, action, resource) → allow | deny
```

- **Subject**: Who is requesting (user, service, role, group)
- **Action**: What they want to do (read, write, delete, execute)
- **Resource**: What they want to do it to (file, database row, API endpoint)

The models differ in **how they encode the rules** that drive this decision.

---

## ACLs — Access Control Lists

The simplest model. Each resource has a list of who can do what to it.

### How It Works

```
Resource: /documents/budget.xlsx
ACL:
  - alice: read, write
  - bob: read
  - finance-group: read, write, delete
  - *: (no access)
```

The authorization check: look up the resource's ACL, find the subject,
check if the action is listed.

### Real Implementation: POSIX File Permissions

Linux file permissions are a compressed ACL with three fixed categories:

```
-rwxr-x--- 1 alice finance budget.xlsx
 │││ │││ │││
 │││ │││ └── other: no access (---)
 │││ └──── group (finance): read + execute (r-x)
 └────── owner (alice): read + write + execute (rwx)
```

9 bits encoding 3 permissions (rwx) for 3 categories (owner, group, other).

**The kernel checks permissions in order:**
1. If the process's effective UID matches the file owner → use owner bits
2. If the process's effective GID matches the file group → use group bits
3. Otherwise → use other bits

**Only one category applies.** If alice is the owner but also in the finance
group, and the owner bits say `rw-` but group bits say `rwx`, alice gets
`rw-` (owner), not `rwx` (group). The first match wins. This catches people
off guard — being the owner can actually give you *fewer* permissions than
the group.

### Extended ACLs (POSIX ACLs)

The 3-category model is too rigid for real-world use. Extended ACLs add
per-user and per-group entries:

```bash
$ getfacl budget.xlsx
# owner: alice
# group: finance
user::rwx
user:bob:r--
group::r-x
group:engineering:---
mask::r-x
other::---
```

The **mask** is a maximum permission boundary — it caps the effective
permissions of all named users and groups (except the owner). If the mask
is `r-x` and bob has `rw-`, his effective permission is `r--` (the
intersection).

**Evaluation order:**
1. Owner → use owner entry
2. Named user → use that user's entry, intersected with mask
3. Owning group or named group → use the matching group entry, intersected
   with mask
4. Other → use other entry

### ACL Limitations

- **Doesn't scale**: With thousands of users and millions of resources,
  maintaining per-resource ACLs is unmanageable
- **No policy abstraction**: Rules are scattered across resources, not
  centralized. "Remove bob's access to everything" requires scanning every
  resource
- **No context**: Can't express "allow access only during business hours"
  or "allow only from the corporate network"
- **Permission explosion**: As the number of subjects × resources grows,
  the number of ACL entries grows multiplicatively

---

## RBAC — Role-Based Access Control

Instead of assigning permissions directly to users, assign them to
**roles**, then assign roles to users.

### The Model

```
Users         Roles              Permissions
──────        ──────             ────────────
alice    ──>  admin         ──>  users:read
bob      ──>  editor        ──>  users:write
charlie  ──>  viewer        ──>  users:delete
                                 posts:read
alice    ──>  editor             posts:write
                                 posts:delete
                                 settings:read
                                 settings:write
```

The indirection through roles is the key insight. You don't ask "can alice
delete users?" You ask "does alice have a role that includes the
users:delete permission?"

```
authorize(alice, delete, users):
    roles = getRoles(alice)                    // [admin, editor]
    for role in roles:
        permissions = getPermissions(role)     // admin → [users:*, posts:*, settings:*]
        if "users:delete" in permissions:
            return allow
    return deny
```

### Core RBAC Components (NIST RBAC Model)

The NIST standard (INCITS 359-2004) defines four levels:

**Level 1 — Flat RBAC:**
- Users assigned to roles
- Roles assigned to permissions
- Users can have multiple roles
- Many-to-many: a user can have multiple roles, a role can have multiple users

**Level 2 — Hierarchical RBAC:**
Roles inherit from other roles:

```
           admin
          /     \
     editor     moderator
          \     /
          viewer
```

An admin inherits all permissions of editor, moderator, and viewer. You
define permissions at the lowest applicable level — `viewer` gets `read`,
`editor` adds `write`, `admin` adds `delete`. No duplication.

**Inheritance direction matters.** Senior roles inherit from junior roles.
`admin` inherits `viewer`'s permissions, not the other way around.

**Level 3 — Constrained RBAC:**
Adds constraints, most importantly **Separation of Duties (SoD)**:

- **Static SoD**: A user cannot be assigned to conflicting roles. Example:
  no user can be both `accounts-payable` and `accounts-receivable` (fraud
  prevention). The constraint is checked at role assignment time.

- **Dynamic SoD**: A user can hold conflicting roles but cannot activate
  them simultaneously. Example: a user can be both `developer` and
  `code-reviewer` but cannot review their own pull request (the system
  prevents activating both roles on the same resource).

**Level 4 — Symmetric RBAC:**
Adds permission-to-role review. Administrators can audit: "which roles have
the `users:delete` permission?" and "which users have the `admin` role?"
Bidirectional querying.

### RBAC in Practice: Database Schema

```sql
CREATE TABLE roles (
    id    SERIAL PRIMARY KEY,
    name  TEXT UNIQUE NOT NULL    -- 'admin', 'editor', 'viewer'
);

CREATE TABLE permissions (
    id       SERIAL PRIMARY KEY,
    resource TEXT NOT NULL,        -- 'users', 'posts', 'settings'
    action   TEXT NOT NULL,        -- 'read', 'write', 'delete'
    UNIQUE(resource, action)
);

CREATE TABLE role_permissions (
    role_id       INT REFERENCES roles(id),
    permission_id INT REFERENCES permissions(id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id INT REFERENCES users(id),
    role_id INT REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);

-- Authorization check:
SELECT EXISTS (
    SELECT 1
    FROM user_roles ur
    JOIN role_permissions rp ON ur.role_id = rp.role_id
    JOIN permissions p ON rp.permission_id = p.id
    WHERE ur.user_id = $1        -- subject
      AND p.resource = $2        -- resource
      AND p.action = $3          -- action
) AS authorized;
```

This is an O(1) lookup (with proper indexing) regardless of the number of
users, roles, or permissions. The join-based resolution is the core engine
of RBAC.

### RBAC Limitations

- **Role explosion**: As requirements get specific ("editor who can only
  edit posts in the marketing category on weekdays"), you create more and
  more roles. A large organization can end up with thousands of roles,
  many with only slight differences.
- **No context**: RBAC is static. It can't express "allow if the user
  created the resource" or "allow if the request comes from a trusted IP."
- **Coarse-grained**: Permissions are typically per resource *type*
  (`posts:write`), not per resource *instance* ("write to post #42").
  Per-instance RBAC requires a row-level mechanism.

---

## ABAC — Attribute-Based Access Control

Instead of pre-defined roles, authorization decisions are based on
**attributes** of the subject, resource, action, and environment.

### The Model

```
Policy: allow if
    subject.department == "engineering" AND
    resource.classification != "top-secret" AND
    action == "read" AND
    environment.time.hour >= 9 AND environment.time.hour <= 17

Attributes:
    subject:     { department: "engineering", clearance: "secret", role: "developer" }
    resource:    { type: "document", classification: "confidential", owner: "alice" }
    action:      { type: "read" }
    environment: { time: "2024-02-15T14:30:00Z", ip: "10.0.0.42", device: "managed" }
```

The policy engine evaluates attributes at decision time. No pre-computed
role assignments — the decision is dynamic.

### XACML — The Standard (Conceptually)

XACML (eXtensible Access Control Markup Language) is the OASIS standard for
ABAC. The architecture defines:

```
                    ┌──────────────┐
                    │   PAP        │  Policy Administration Point
                    │ (write       │  Where policies are authored
                    │  policies)   │
                    └──────┬───────┘
                           │ policies
                           ▼
┌─────────┐    request  ┌──────────────┐    query   ┌──────────────┐
│   PEP   │ ──────────> │   PDP        │ ─────────> │   PIP        │
│ (enforce)│            │ (decide)     │            │ (fetch       │
│          │ <────────  │              │ <─────────  │  attributes) │
└─────────┘   decision  └──────────────┘   attrs    └──────────────┘

PEP: Policy Enforcement Point — intercepts the request, asks for a decision
PDP: Policy Decision Point — evaluates policies against attributes
PIP: Policy Information Point — fetches attributes from external sources
PAP: Policy Administration Point — where admins write policies
```

The separation is important: the enforcement (PEP) is decoupled from the
decision logic (PDP). You can change policies without changing application
code. The PDP can pull attributes from LDAP, databases, APIs — anywhere.

### ABAC vs RBAC

| | RBAC | ABAC |
|---|------|------|
| Decision basis | Role membership | Attribute evaluation |
| Granularity | Per role (coarse) | Per attribute combination (fine) |
| Context-aware | No | Yes (time, location, device, risk) |
| Policy location | Scattered (role-permission tables) | Centralized (policy engine) |
| Complexity | Simple to implement | Complex policies, complex engine |
| Performance | Fast (pre-computed joins) | Slower (runtime evaluation) |
| Auditability | Easy (list roles + permissions) | Harder (policies can interact) |

**In practice, most systems use RBAC as the foundation with ABAC-like rules
for specific cases.** "The user must have the `editor` role (RBAC) AND must
be the resource owner (attribute) AND the resource must not be archived
(attribute)."

---

## ReBAC — Relationship-Based Access Control

Authorization based on the **relationship** between the subject and the
resource, or between resources.

### The Insight

Many authorization decisions are naturally about relationships:
- "Can alice view this document?" → "Is alice in a group that has viewer
  access to this document's folder?"
- "Can bob edit this file?" → "Is bob a member of the team that owns the
  project that contains this file?"

These are **graph traversals**, not role lookups or attribute evaluations.

### Google Zanzibar — The Foundational Paper

Google's Zanzibar (2019) is the authorization system behind Google Drive,
YouTube, Cloud IAM, and other Google services. It handles authorization
checks at **millions of QPS** with **<10ms latency** for systems with
**trillions of access control relationships**.

#### The Data Model: Relation Tuples

Everything is stored as tuples:

```
<object>#<relation>@<subject>

doc:readme#viewer@user:alice          // alice is a viewer of doc:readme
doc:readme#owner@user:bob             // bob is the owner of doc:readme
folder:eng#viewer@group:engineering#member  // members of engineering group
                                            // are viewers of folder:eng
group:engineering#member@user:alice    // alice is a member of engineering
group:engineering#member@user:charlie  // charlie is a member of engineering
```

The subject can be a **userset** — a set of users defined by a relationship
on another object. `group:engineering#member` means "all users who are
members of the engineering group."

#### Authorization as Graph Traversal

```
Check: can alice view doc:readme?

1. Direct check: is there a tuple (doc:readme#viewer@user:alice)?
   → YES. Authorized.

Check: can charlie view doc:readme?

1. Direct check: (doc:readme#viewer@user:charlie)?
   → No.
2. Expand: doc:readme#viewer includes folder:eng#viewer (inheritance)
   → Check: (folder:eng#viewer@user:charlie)?
3. Expand: folder:eng#viewer includes group:engineering#member
   → Check: (group:engineering#member@user:charlie)?
   → YES. Authorized.
```

The check traverses a graph of relationships. The depth of the traversal
depends on the relationship hierarchy (folders containing folders, groups
containing groups, etc.).

#### Performance at Scale

Zanzibar achieves its performance through:

- **Leopard indexing**: Precomputed transitive closure for common relationship
  patterns. Instead of traversing the graph on every check, periodically
  materialize the effective permissions. Trade storage for latency.
- **Request hedging**: Send the same check to multiple servers, take the
  first response. Reduces tail latency.
- **Namespace caching**: Cache relationship tuples by namespace (all tuples
  for `doc:readme`), sharded across a distributed cache.
- **Zookies (consistency tokens)**: After a write (e.g., sharing a doc with
  someone), return a token that ensures subsequent reads see the write.
  Prevents the "I just shared this but they can't see it yet" problem
  without requiring global consistency on every read.

#### Open Source Implementations

- **SpiceDB** (AuthZed): Direct Zanzibar implementation, gRPC API, strongly
  consistent.
- **OpenFGA** (Auth0/Okta): Zanzibar-inspired, simpler API, good for
  getting started.
- **Keto** (Ory): Zanzibar implementation in Go, part of the Ory ecosystem.

### Zanzibar Schema Example

```
definition user {}

definition group {
    relation member: user | group#member
}

definition folder {
    relation owner: user
    relation viewer: user | group#member
    relation parent: folder

    permission view = viewer + owner + parent->view
}

definition document {
    relation owner: user
    relation editor: user | group#member
    relation viewer: user | group#member
    relation parent: folder

    permission edit = editor + owner
    permission view = viewer + editor + owner + parent->view
}
```

`parent->view` means "anyone who can view the parent can also view this."
This is computed permission — the system traverses the parent relationship
and checks the `view` permission recursively. This is how Google Drive
folder permissions cascade to documents within them.

---

## Policy Engines

### OPA — Open Policy Agent

A general-purpose policy engine. Policies are written in **Rego**, a
purpose-built declarative language.

```rego
# policy.rego

package authz

import rego.v1

default allow := false

# Admins can do anything
allow if {
    input.subject.role == "admin"
}

# Users can read their own data
allow if {
    input.action == "read"
    input.resource.owner == input.subject.id
}

# Editors can write to non-archived resources
allow if {
    input.action == "write"
    input.subject.role == "editor"
    not input.resource.archived
}

# Anyone can read public resources
allow if {
    input.action == "read"
    input.resource.visibility == "public"
}
```

```json
// Input (the authorization request):
{
    "subject": { "id": "alice", "role": "editor" },
    "action": "write",
    "resource": { "type": "document", "id": "doc-42", "archived": false, "owner": "bob" }
}

// Output: { "allow": true }  (matched rule 3)
```

**How OPA evaluates:**

Rego rules are not if/else chains — they're **logical rules that are all
evaluated independently**. Any rule that produces `allow = true` is
sufficient. If no rule produces `true`, the `default allow := false`
applies.

Each rule body is a conjunction (AND): all statements in the body must be
true. Multiple rules for the same output are a disjunction (OR): any one
being true is sufficient.

**OPA deployment patterns:**
- **Sidecar**: OPA runs alongside each service (as a container in the pod).
  Authorization decisions are a local HTTP call (~1ms). No network dependency
  on a central service.
- **Library**: OPA compiled as a Go library, embedded in the application.
  Eliminates even the local HTTP call. Fastest option.
- **Central service**: Single OPA instance that all services query. Simplest
  to operate but introduces a network hop and a single point of failure.

**Policy distribution**: OPA periodically polls a **bundle server** for
updated policies. Policies are compiled to an intermediate representation
(plan) for fast evaluation. Updates propagate to all OPA instances within
the polling interval (typically seconds).

### Cedar — AWS's Authorization Language

Developed by AWS for Amazon Verified Permissions and other services.
Designed to be analyzable — you can mathematically prove properties about
your policies (e.g., "no policy ever allows a non-admin to delete a user").

```cedar
// Admins can do anything
permit(
    principal in Group::"admins",
    action,
    resource
);

// Users can read their own profiles
permit(
    principal,
    action == Action::"read",
    resource
) when {
    resource.owner == principal
};

// Explicitly deny access to archived resources
forbid(
    principal,
    action,
    resource
) when {
    resource.archived == true
};
```

**Cedar's evaluation model:**

```
1. Collect all policies that match the request
2. If ANY forbid policy matches → DENY (explicit deny always wins)
3. If ANY permit policy matches → ALLOW
4. If no policy matches → DENY (default deny)
```

This is the same model as AWS IAM. The key property: **explicit deny
overrides everything.** You can grant broad permissions with a permit
policy and carve out exceptions with forbid policies. This is safer than
the reverse (trying to enumerate everything that's allowed).

**Cedar's formal verification**: Cedar policies can be analyzed by an
SMT (Satisfiability Modulo Theories) solver to answer questions like:
- "Can any non-admin ever delete a resource?" (safety property)
- "Are policies A and B equivalent?" (refactoring verification)
- "Does this policy change grant access to anyone who didn't have it
  before?" (change impact analysis)

This is unique among practical authorization systems and is why AWS chose
to build Cedar rather than adopt OPA.

---

## Real-World Authorization Systems

### AWS IAM — Policy Documents

AWS IAM is ABAC implemented as JSON policy documents:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:ListBucket"],
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "203.0.113.0/24"
                },
                "StringEquals": {
                    "s3:prefix": ["home/${aws:username}/"]
                }
            }
        },
        {
            "Effect": "Deny",
            "Action": "s3:*",
            "Resource": "*",
            "Condition": {
                "Bool": {
                    "aws:MultiFactorAuthPresent": "false"
                }
            }
        }
    ]
}
```

**Policy evaluation logic:**

```
1. Gather all policies (identity-based, resource-based, SCPs,
   permission boundaries, session policies)
2. If any explicit Deny matches → DENY
3. If no policy allows → DENY (implicit deny)
4. If any Allow matches AND no Deny matches → ALLOW
```

**Policy types stack:**

```
Effective permissions =
    Identity policies ∩ Permission boundaries ∩ SCPs ∩ Resource policies

(Only actions allowed by ALL applicable policy types are permitted)
```

- **Identity policies**: Attached to IAM users/roles/groups
- **Resource policies**: Attached to the resource itself (S3 bucket policy)
- **Permission boundaries**: Maximum permissions for an identity (ceiling)
- **SCPs (Service Control Policies)**: Maximum permissions for an AWS account
  within an Organization (organizational ceiling)

The intersection model means each layer can only restrict, never expand
beyond what the layer above allows. An SCP denying S3 access means no
identity policy in that account can grant S3 access.

### PostgreSQL Row-Level Security (RLS)

Authorization at the database row level:

```sql
-- Enable RLS on the table
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Users can only see documents they own or that are public
CREATE POLICY select_documents ON documents
    FOR SELECT
    USING (
        owner_id = current_setting('app.user_id')::int
        OR visibility = 'public'
    );

-- Users can only update documents they own
CREATE POLICY update_documents ON documents
    FOR UPDATE
    USING (owner_id = current_setting('app.user_id')::int)
    WITH CHECK (owner_id = current_setting('app.user_id')::int);

-- Admins can do anything
CREATE POLICY admin_all ON documents
    FOR ALL
    USING (
        EXISTS (
            SELECT 1 FROM user_roles
            WHERE user_id = current_setting('app.user_id')::int
            AND role = 'admin'
        )
    );
```

**How it works internally:**

RLS policies are appended to every query as additional WHERE clauses. When
a user runs `SELECT * FROM documents`, the engine actually executes:

```sql
SELECT * FROM documents
WHERE (
    -- select_documents policy
    (owner_id = current_setting('app.user_id')::int OR visibility = 'public')
    -- OR admin_all policy
    OR EXISTS (SELECT 1 FROM user_roles WHERE ...)
);
```

Multiple policies for the same command are combined with OR (any policy
granting access is sufficient). Policies for different commands (SELECT vs
UPDATE) are independent.

**`USING` vs `WITH CHECK`:**
- `USING`: Filters which existing rows are visible (for SELECT, UPDATE,
  DELETE). Rows not matching are silently invisible — not an error, just
  absent from results.
- `WITH CHECK`: Validates new or modified data (for INSERT, UPDATE). Rows
  not matching cause an error — the operation is rejected.

This distinction is critical: `USING` can hide data silently, while
`WITH CHECK` actively rejects invalid writes. An UPDATE policy with both
means "you can only update rows you can see (USING), and the result must
still satisfy the policy (WITH CHECK)."

**The bypass:** Table owners and superusers bypass RLS by default. To
enforce RLS on the table owner: `ALTER TABLE documents FORCE ROW LEVEL
SECURITY;`

**Performance:** RLS policies are evaluated per-row. Complex policies with
subqueries can degrade query performance. The query planner incorporates
RLS predicates into its optimization, but joins in USING clauses may not
be optimized as well as joins in the main query.

---

## The Confused Deputy Problem

A fundamental authorization vulnerability that affects any system with
delegated authority.

### The Classic Example

```
User → Service A → Service B

User asks Service A to access a resource on Service B.
Service A has its own credentials to Service B (it's a trusted service).
Service A makes the request to Service B using its own credentials.
Service B sees Service A (a trusted service) and allows the request.

But: did the User have permission to ask for that resource?
Service A didn't check. Service B can't check (it doesn't know about the user).
```

Service A is the "confused deputy" — it was tricked into using its own
authority on behalf of an unauthorized user.

### Real-World Instances

**CSRF is a confused deputy attack.** The browser (deputy) has the user's
cookies. A malicious site tricks the browser into making a request. The
server sees valid cookies and processes it. The browser is confused about
who is actually requesting the action.

**S3 bucket access via Lambda.** A Lambda function has an IAM role with
broad S3 access. A user calls the Lambda with a request for a specific
file. The Lambda fetches the file using its own IAM role. Did the user
have permission to that specific file? If the Lambda doesn't check, it's
a confused deputy.

### Prevention

1. **Pass the user's identity through the chain.** Service A forwards the
   user's token to Service B, not its own. Service B checks the user's
   permissions directly. (OAuth token forwarding, JWT propagation)

2. **Check authorization at the deputy.** Service A verifies the user is
   allowed to request the resource before using its own credentials.
   (Application-level authorization checks)

3. **Capability-based tokens.** Instead of identity-based access ("alice
   can read X"), use capabilities ("this token grants read on X"). The
   token itself is the authorization — whoever holds it can access the
   resource. No confused deputy because there's no ambient authority to
   misuse. (S3 presigned URLs, macaroons)

---

## Capability-Based Security

An alternative paradigm to identity-based access control.

### Identity-Based (What We've Discussed So Far)

```
Request: "I'm alice, I want to read document 42"
System: looks up alice's permissions → allowed
```

The system has ambient authority — it knows who everyone is and what they
can do. Any request from alice is authorized based on her identity.

### Capability-Based

```
Request: "Here is a token that grants read access to document 42"
System: validates the token → allowed (doesn't care who is presenting it)
```

A **capability** is an unforgeable token that bundles a reference to a
resource with the permissions to access it. If you have the capability, you
have the access. No identity lookup, no ambient authority.

### Examples in Practice

**S3 Presigned URLs:**
```
https://my-bucket.s3.amazonaws.com/secret-doc.pdf?
    X-Amz-Algorithm=AWS4-HMAC-SHA256&
    X-Amz-Credential=AKIA.../s3/aws4_request&
    X-Amz-Expires=3600&
    X-Amz-Signature=abc123...
```

Anyone with this URL can download the file for 1 hour. No AWS credentials
needed. The URL itself is the capability. Share it and you share the access.

**Macaroons** (Google, 2014):
Tokens with embedded caveats (restrictions) that can be **attenuated** by
anyone in the chain — you can add restrictions but never remove them.

```
Original macaroon: access to all docs in folder X
  → Add caveat: "only read" → now read-only access to folder X
  → Add caveat: "before 2024-03-01" → read-only, expires March 1
  → Add caveat: "from IP 10.0.0.0/8" → read-only, expires, internal only
```

Each caveat is cryptographically chained (HMAC of the previous signature +
the caveat string). You can verify all caveats without contacting the
issuer. You can add caveats without contacting the issuer. You cannot
remove caveats.

**Advantage**: Delegated attenuation. A service can take a broad capability
and narrow it before passing it to a less-trusted service, without going
back to the authorization server.

---

## Permission Resolution — When Rules Conflict

Every authorization system must answer: what happens when one rule says
"allow" and another says "deny"?

### Resolution Strategies

**Deny overrides (most common):**
```
If ANY applicable rule says deny → deny
Else if ANY applicable rule says allow → allow
Else → deny (default deny)
```

Used by: AWS IAM, Cedar, most enterprise systems. The safest default —
a single deny rule can't be overridden by any number of allow rules.

**Allow overrides:**
```
If ANY applicable rule says allow → allow
Else if ANY applicable rule says deny → deny
Else → deny
```

Rare in practice. A single misconfigured allow rule could bypass all deny
rules. Generally considered unsafe.

**First match:**
```
Evaluate rules in order. First matching rule wins.
```

Used by: Firewalls (iptables), some web servers (nginx location blocks).
Order-dependent — moving a rule up or down changes the outcome. Fragile
in complex configurations.

**Most specific wins:**
```
More specific rules override more general rules.
```

Used by: CSS specificity, some RBAC systems. A rule for "alice on
document #42" overrides "editors on all documents." Requires a clear
definition of "more specific" — can be ambiguous.

### The Principle of Least Privilege

Regardless of resolution strategy, the foundational principle:

**Grant the minimum permissions necessary for the task. No more.**

- Start with zero access (default deny)
- Add permissions as needed
- Use time-limited grants where possible
- Regularly audit and revoke unused permissions
- Separate high-privilege operations (admin functions) from normal operations

---

## Multi-Tenancy Authorization

Most SaaS applications serve multiple tenants (organizations, teams,
workspaces) from a single deployment. Authorization must ensure that
**Tenant A can never access Tenant B's data**, even through bugs,
misconfigurations, or clever API manipulation.

### Tenant Isolation Strategies

**Row-level tenant column:**

```sql
CREATE TABLE documents (
    id         UUID PRIMARY KEY,
    tenant_id  UUID NOT NULL REFERENCES tenants(id),
    title      TEXT,
    content    TEXT
);

-- EVERY query must include tenant_id
SELECT * FROM documents WHERE tenant_id = $1 AND id = $2;

-- RLS enforces this even if the application forgets:
CREATE POLICY tenant_isolation ON documents
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

**The danger**: A single missing `WHERE tenant_id = ?` clause and you have
a cross-tenant data leak. RLS is the safety net — it enforces the filter
at the database level regardless of application code.

**Schema-per-tenant:**

```sql
-- Each tenant gets their own schema
CREATE SCHEMA tenant_abc123;
CREATE TABLE tenant_abc123.documents (...);

-- The application sets the search_path per request
SET search_path TO tenant_abc123, public;
SELECT * FROM documents;  -- automatically hits tenant_abc123.documents
```

Stronger isolation but harder to manage at scale (migrations must run
across all schemas). Works well up to ~1000 tenants. Beyond that, the
operational overhead becomes significant.

**Database-per-tenant:**

Strongest isolation. Each tenant's data is in a completely separate
database. A bug can't possibly leak data across tenants because the data
isn't co-located. But: expensive to operate, connection pooling is complex,
and schema migrations must be applied to every database.

### The Authorization Stack for Multi-Tenant Apps

```
Request arrives with:
  - User identity (JWT or session)
  - Tenant context (from subdomain, header, or token claim)

Layer 1: Tenant resolution
  → Which tenant is this request for?
  → Does this user belong to this tenant?

Layer 2: Role within tenant
  → What role does this user have IN THIS TENANT?
  → (A user might be admin in Tenant A but viewer in Tenant B)

Layer 3: Resource authorization
  → Can this role perform this action on this resource?
  → Is this resource within this tenant? (defense in depth)

Layer 4: Data layer enforcement
  → RLS policies filter rows by tenant_id
  → Even if layers 1-3 fail, the database refuses cross-tenant data
```

### Tenant-Scoped Roles

In multi-tenant systems, roles are scoped to a tenant — not global:

```sql
CREATE TABLE tenant_memberships (
    user_id   UUID REFERENCES users(id),
    tenant_id UUID REFERENCES tenants(id),
    role      TEXT NOT NULL,  -- 'owner', 'admin', 'member', 'viewer'
    PRIMARY KEY (user_id, tenant_id)
);

-- Alice is admin in Acme Corp, viewer in Globex
-- Bob is owner of Globex, has no role in Acme Corp
```

A common mistake: using global roles (a single `role` column on the users
table) for multi-tenant apps. This means a user has the same role in every
tenant, which is almost never the correct model.

---

## Temporal Authorization

Some permissions should only be valid for a limited time.

### Time-Bounded Access

```
"Contractor X can access the codebase until March 31, 2024"
"On-call engineer has admin access for the next 8 hours"
"Audit access granted for 30 days per compliance request #1234"
```

Implementation:

```sql
CREATE TABLE role_assignments (
    user_id    UUID,
    role       TEXT,
    resource   TEXT,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ,  -- NULL = permanent
    granted_by UUID,         -- who approved this
    reason     TEXT          -- audit trail
);

-- Authorization check includes time:
SELECT EXISTS (
    SELECT 1 FROM role_assignments
    WHERE user_id = $1
      AND role = $2
      AND (expires_at IS NULL OR expires_at > now())
) AS authorized;
```

### Just-In-Time (JIT) Access

No standing privileges. Users request access, it's granted for a short
window, then automatically revoked.

```
1. Engineer requests "admin" access to production database
2. Request goes to approval queue (manager or automated policy)
3. If approved: grant for 4 hours with full audit logging
4. After 4 hours: automatically revoke. No human action needed.
5. All actions during the window are logged for later audit
```

**Why**: Standing privileges (always-on admin access) are a risk. An
attacker who compromises an admin account has permanent access. JIT access
means even a compromised admin account only has elevated privileges during
the approved window. Google's BeyondCorp and HashiCorp's Boundary implement
this pattern.

### Break-Glass Access

Emergency override that bypasses normal authorization. A doctor needs
patient records during an emergency. The system allows access but logs
everything and triggers an immediate audit review.

```
1. User requests break-glass access
2. System grants access immediately (no approval delay)
3. Sends alert to security team
4. Logs every action with break-glass flag
5. Requires justification retroactively
6. Unjustified break-glass use → policy violation
```

This exists because authorization systems must handle the case where denying
access causes greater harm than granting it. The key is making unauthorized
break-glass use detectable and accountable.

---

## Choosing a Model

| System | Best Fit | Examples |
|--------|----------|---------|
| ACL | Simple, few users, few resources | File systems, small apps |
| RBAC | Organizational hierarchy, well-defined roles | Enterprise apps, SaaS |
| ABAC | Complex context-dependent rules | Government, healthcare, finance |
| ReBAC | Sharing, collaboration, nested hierarchies | Google Drive, GitHub, Notion |
| Capabilities | Delegation, cross-service, no ambient authority | APIs, presigned URLs, IoT |

**Most real systems combine models:**
- GitHub: RBAC (owner, maintainer, contributor roles) + ReBAC (org → team
  → repo relationships) + ABAC (branch protection rules based on file
  patterns)
- AWS: ABAC (IAM policies with conditions) + RBAC (IAM roles) + Capabilities
  (presigned URLs, STS temporary credentials)
- Supabase: RBAC (Postgres roles) + ABAC/RLS (row-level security policies
  with runtime attributes from JWTs)

The model you choose should match the complexity of your authorization
requirements. Don't build Zanzibar for a blog app. Don't use ACLs for
a multi-tenant SaaS.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| `authorize(subject, action, resource) → allow/deny` | Every model resolves to this function |
| ACLs are per-resource, don't scale | Fine for file systems, not for applications |
| RBAC uses roles as indirection | Users → Roles → Permissions. Hierarchical RBAC enables inheritance |
| RBAC breaks down at granularity | "Editor who can only edit their own posts on weekdays" = role explosion |
| ABAC evaluates attributes at runtime | Context-aware but complex to implement and audit |
| ReBAC models authorization as a graph | Zanzibar handles trillions of relationships at <10ms latency |
| Policy engines decouple authorization from code | OPA/Rego for general purpose, Cedar for formal verification |
| Deny always overrides allow | AWS IAM, Cedar, most enterprise systems. Safest default. |
| The confused deputy is a fundamental problem | Any system with delegated authority. Pass user identity, don't use ambient authority. |
| Capabilities invert the model | The token IS the authorization. No identity lookup. Unforgeable, attenuable (macaroons). |
| PostgreSQL RLS is authorization at the data layer | Policies become WHERE clauses. USING filters reads, WITH CHECK validates writes. |
| Multi-tenancy needs defense in depth | Tenant column + RLS + application checks. One missing WHERE = data leak. |
| Temporal access reduces blast radius | JIT access, time-bounded grants, break-glass with audit. No standing privileges. |
| Least privilege is the foundational principle | Default deny. Add permissions minimally. Audit regularly. |
