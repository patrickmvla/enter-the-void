# Scopes & Token-Based Authorization — Where Auth Meets Authz

## The Bridge

Authentication (topics 01-05) establishes **who** the user is and produces
a token. Authorization (topics 01-02) defines **what** they can do. Scopes
are the mechanism that bridges these two systems — they embed authorization
constraints **inside** the authentication token.

This topic ties together OAuth scopes (from authentication/04-oauth2.md),
JWT claims (from authentication/03-jwts.md), and the authorization models
(from 01-authorization-models.md).

---

## How Scopes Become Authorization Decisions

### The Lifecycle of a Scope

```
1. Client requests scope:
   GET /authorize?scope=read:repos write:repos

2. Authorization server checks:
   → Does this client have permission to request these scopes?
   → Does the user consent to granting these scopes?

3. User consents (possibly to a subset):
   → User approves read:repos but denies write:repos

4. Token is issued with granted scopes:
   { "scope": "read:repos", "sub": "42" }

5. Client presents token to resource server:
   GET /repos
   Authorization: Bearer <token>

6. Resource server enforces:
   → Is the token valid? (authentication)
   → Does the token's scope include "read:repos"? (authorization)
   → Does user 42 have access to this specific repo? (deeper authorization)
```

**Scopes are a ceiling, not a floor.** A token with `read:repos` means the
client can read repos that **the user already has access to**. It doesn't
grant access to repos the user can't see. Scopes limit what the client
can do on behalf of the user — they don't expand the user's own
permissions.

```
User's permissions:        can read repos A, B, C. Can write to repo A.
Token scope:               read:repos
Effective authorization:   can read repos A, B, C. Cannot write to anything.

Token scope:               write:repos
Effective authorization:   can write to repo A. (scope allows write,
                           user has write on A only)
```

This is the **intersection model**:

```
Effective permissions = User's permissions ∩ Token scopes
```

The same model as AWS IAM's policy intersection (identity policies ∩
permission boundaries ∩ SCPs).

---

## Scope Enforcement at the Resource Server

### The Common Mistake: Checking Only the Token

```javascript
// BAD — checks scope but not user permissions
app.delete('/repos/:id', (req, res) => {
    if (!req.token.scope.includes('delete:repos')) {
        return res.status(403).json({ error: 'Insufficient scope' });
    }
    // Deletes the repo without checking if the user owns it
    db.repos.delete(req.params.id);
});
```

This is a **confused deputy** (from 01-authorization-models.md). The token
says the client can delete repos, but can this user delete THIS repo?
Scope enforcement without user-level authorization is incomplete.

### The Correct Pattern: Layered Enforcement

```javascript
// GOOD — checks scope AND user permissions
app.delete('/repos/:id', (req, res) => {
    // Layer 1: Scope check (what the token allows)
    if (!req.token.scope.includes('delete:repos')) {
        return res.status(403).json({ error: 'Insufficient scope' });
    }

    // Layer 2: User authorization (what the user is allowed to do)
    const repo = db.repos.findById(req.params.id);
    if (repo.ownerId !== req.token.sub) {
        return res.status(403).json({ error: 'Not authorized' });
    }

    // Layer 3: Business rules (additional constraints)
    if (repo.isProtected) {
        return res.status(403).json({ error: 'Protected repository' });
    }

    db.repos.delete(req.params.id);
});
```

Three layers:
1. **Scope**: Can the token do this type of operation?
2. **User authorization**: Can this user do this to this resource?
3. **Business rules**: Are there additional constraints?

### Middleware Pattern

In practice, scope enforcement is middleware that runs before route
handlers:

```javascript
function requireScope(...requiredScopes) {
    return (req, res, next) => {
        const tokenScopes = req.token.scope?.split(' ') || [];
        const hasAll = requiredScopes.every(s => tokenScopes.includes(s));
        if (!hasAll) {
            return res.status(403).json({
                error: 'insufficient_scope',
                required: requiredScopes,
                provided: tokenScopes
            });
        }
        next();
    };
}

// Route registration
app.get('/repos',      requireScope('read:repos'),   listRepos);
app.post('/repos',     requireScope('write:repos'),  createRepo);
app.delete('/repos/:id', requireScope('delete:repos'), deleteRepo);
```

The `insufficient_scope` error (with WWW-Authenticate header) is defined
in RFC 6750:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope",
    scope="delete:repos",
    error_description="The token does not have the required scope"
```

---

## Scopes vs Permissions vs Roles

These are often conflated. They're different:

| Concept | Belongs to | Controls | Granularity |
|---------|-----------|----------|-------------|
| **Scope** | Token (OAuth) | What the **client application** can do on behalf of the user | Per API action type |
| **Permission** | User (RBAC/ABAC) | What the **user** can do | Per resource instance |
| **Role** | User (RBAC) | A named bundle of permissions | Per role definition |

```
Example: GitHub API

Scopes (in the OAuth token):
  read:repos         — client can read repositories
  write:repos        — client can create/update repositories
  delete:repos       — client can delete repositories

Roles (in GitHub's system):
  owner              — full control of the organization
  maintainer         — manage repos, can't delete org
  member             — read access, contribute to assigned repos

Permissions (per-repo):
  alice has "admin" on repo X
  alice has "read" on repo Y
  bob has "write" on repo X

Authorization check for "delete repo X via OAuth app":
  1. Token scope includes delete:repos?       ✓ (scope check)
  2. User alice has admin permission on X?     ✓ (permission check)
  3. alice's org role allows repo deletion?    ✓ (role check)
  → ALLOW

Authorization check for "delete repo Y via OAuth app":
  1. Token scope includes delete:repos?       ✓
  2. User alice has admin permission on Y?     ✗ (only read)
  → DENY (despite having the scope)
```

---

## Scope Design Patterns

### Hierarchical Scopes

```
repo                    — full access to repositories
repo:read               — read-only access
repo:write              — read + write access
repo:admin              — read + write + delete + settings

Does "repo" include "repo:read"?
```

**Two approaches:**

**Flat (GitHub, most APIs)**: Each scope is independent. Having `repo`
doesn't imply `repo:read`. The client must request each scope explicitly.
Simpler to implement and audit.

**Hierarchical (some Google APIs)**: A broader scope implies narrower
ones. `https://www.googleapis.com/auth/drive` implies
`https://www.googleapis.com/auth/drive.readonly`. More convenient for
clients but harder to audit (implicit permissions).

### Action-Based vs Resource-Based

**Action-based** (most common):
```
read:repos
write:repos
delete:repos
admin:org
```

Scopes describe what operations are allowed, regardless of which specific
resource. Fine-grained resource authorization happens at the application
layer.

**Resource-based** (less common):
```
repo:owner/name:read
repo:owner/name:write
```

Scopes reference specific resources. More precise but creates a
combinatorial explosion of scopes for users with access to many resources.
GitHub's fine-grained personal access tokens use this approach.

### Dynamic Scopes and Downscoping

A token can be **downscoped** — exchanging a broad token for a narrower
one:

```http
POST /token HTTP/1.1
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
subject_token=BROAD_TOKEN
scope=read:repos
```

**Use case**: A backend service receives a user's token with broad scopes.
Before passing it to a less-trusted microservice, it downscopes to only
what that service needs. This is the **principle of least privilege**
applied to token delegation — and it's the OAuth equivalent of macaroon
attenuation (from 01-authorization-models.md).

---

## JWT Claims as Authorization Data

Beyond scopes, JWTs can carry authorization-relevant claims:

```json
{
    "sub": "42",
    "scope": "read:repos write:repos",
    "role": "admin",
    "tenant_id": "acme-corp",
    "permissions": ["users:read", "users:write", "billing:read"],
    "groups": ["engineering", "platform-team"]
}
```

### The Tradeoff: Fat Tokens vs Thin Tokens

**Fat token** (authorization data in the JWT):
```json
{
    "sub": "42",
    "permissions": ["users:read", "users:write", "posts:read",
                    "posts:write", "posts:delete", "billing:read",
                    "billing:write", "admin:settings"]
}
```

Pros: Resource server has everything it needs. No additional lookups.
Cons: Token is large. Permissions are stale for the token's lifetime.
Adding a permission requires a new token. Can exceed header size limits.

**Thin token** (minimal identity, look up permissions):
```json
{
    "sub": "42",
    "role": "admin",
    "tenant_id": "acme-corp"
}
```

Pros: Token is small. Permissions are always current (looked up in
real-time). Revocation is instant (change the permission, next lookup
reflects it).
Cons: Every request requires a permission lookup (database, cache, or
policy engine call).

**The hybrid approach** (most common):
Put **stable, coarse-grained** authorization data in the JWT (role, tenant,
basic scopes). Look up **dynamic, fine-grained** permissions at the
resource server. The token answers "what category of user is this?" and
the server answers "can this category of user do this specific thing?"

---

## Token Authorization in Microservices

In a microservice architecture, a user's request may traverse multiple
services. How does authorization propagate?

### Token Forwarding

```
User → API Gateway → Service A → Service B → Service C
       (validates     (forwards   (forwards
        token)         token)      token)
```

Each service receives the same user token and makes its own authorization
decision. Service B checks the token's scope AND applies its own RBAC/ABAC
rules.

**Problem**: If Service B needs to call Service C on behalf of Service A
(not the user), whose identity should the token carry?

### Token Exchange (RFC 8693)

A service can exchange the user's token for a new token with:
- A different audience (targeted for the downstream service)
- Narrower scopes (least privilege)
- The service's own identity as the actor

```http
POST /token
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
subject_token=USER_TOKEN
subject_token_type=urn:ietf:params:oauth:token-type:access_token
audience=service-c
scope=read:data
```

Response:
```json
{
    "access_token": "NEW_TOKEN_FOR_SERVICE_C",
    "token_type": "Bearer",
    "issued_token_type": "urn:ietf:params:oauth:token-type:access_token"
}
```

The new token has `aud: "service-c"` and `scope: "read:data"` — Service C
knows this request is authorized for reading data only, even if the
original token had broader scopes. If Service C is compromised, the
leaked token is only valid for Service C with minimal scopes.

### The Phantom Token Pattern

API Gateway receives an opaque (reference) token from the client, exchanges
it for a JWT internally, and forwards the JWT to microservices:

```
Client → API Gateway → Microservices
         (opaque→JWT)   (verify JWT locally)

External: opaque token (can't be decoded, must be introspected)
Internal: JWT (self-contained, verifiable without network call)
```

Benefits:
- External tokens reveal nothing if leaked (no readable claims)
- Internal services get the performance benefit of JWT verification
- Token introspection happens once at the gateway, not in every service
- The gateway can enrich the JWT with additional claims from the user
  profile or permission system

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| Scopes are a ceiling, not a floor | `write:repos` doesn't grant access to repos the user can't already write to |
| Effective permissions = user permissions ∩ token scopes | Same intersection model as AWS IAM |
| Scope enforcement alone is insufficient | Must also check user-level authorization per resource |
| Scopes ≠ permissions ≠ roles | Scopes control the client, permissions control the user, roles bundle permissions |
| Fat tokens carry stale authorization | Permissions embedded in JWTs can be outdated for the token's lifetime |
| Token exchange enables least privilege in microservices | Downscope and retarget tokens for each downstream service |
| The phantom token pattern separates external from internal tokens | Opaque externally, JWT internally. Best of both worlds. |
