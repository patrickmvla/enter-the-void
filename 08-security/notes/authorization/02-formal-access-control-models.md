# Formal Access Control Models — The Theory That Governs the Practice

## Why Formal Models Matter

The practical models (RBAC, ABAC, ReBAC) describe how to **implement**
authorization. Formal models describe the **mathematical properties** that
an authorization system must satisfy to be provably secure.

Without formal models, you're building authorization by intuition — and
intuition misses edge cases. Formal models give you:

1. **Provable guarantees**: "Information classified as Top Secret can never
   flow to a user with Secret clearance" — not as a hope, but as a
   mathematical invariant.
2. **Vocabulary for security properties**: Confidentiality, integrity,
   availability are not vague concepts — they have precise definitions
   with precise enforcement rules.
3. **A basis for evaluation**: Common Criteria (ISO 15408), the framework
   used to certify security products for government use, is built on
   formal access control models.

---

## DAC vs MAC — The Fundamental Split

Every access control system falls into one of two paradigms, or a
combination of both. This is the most important distinction in access
control theory.

### DAC — Discretionary Access Control

**The owner of a resource decides who can access it.**

The word "discretionary" means the access decision is at the owner's
discretion. If alice owns a file, alice decides who can read, write, or
execute it. The system enforces alice's decisions but doesn't override
them.

```
alice creates document.txt
alice grants bob read access to document.txt
alice revokes charlie's access to document.txt
→ All at alice's discretion. The system enforces but doesn't dictate.
```

**Every system we've discussed so far is DAC:**
- Unix file permissions: the file owner sets rwx bits
- ACLs: the resource owner manages the access list
- RBAC (as typically implemented): administrators assign roles at their
  discretion
- Google Drive sharing: the document owner shares with whoever they want

**The fundamental weakness of DAC:**

DAC cannot control **information flow**. Alice can grant bob read access.
Bob can then copy the contents to a new file he owns and share that with
charlie — even though alice never authorized charlie.

```
alice → shares document with bob (authorized)
bob   → reads document, creates copy
bob   → shares copy with charlie (alice has no control)
charlie → reads alice's data (alice never authorized this)
```

The system enforced alice's access control correctly at every step. But
the information still flowed to an unauthorized party. DAC controls
**access to containers** (files, records), not **flow of information**.

This is not a theoretical concern. It's the model behind every data
leak where an authorized user copies data to an unauthorized location —
emailing a spreadsheet, downloading to a personal device, pasting into
a chat.

**DAC is sufficient when you trust all authorized users.** It fails when
insiders are adversaries or when information flow must be controlled
(classified systems, healthcare, financial compliance).

### MAC — Mandatory Access Control

**The system enforces access policy that no user can override — not even
the resource owner.**

"Mandatory" means the policy is imposed by the system, not chosen by
users. A user with Secret clearance cannot grant access to a Top Secret
document, even if they somehow obtain it. The system prevents it
regardless of the user's intent.

```
System policy: Top Secret data can only be accessed by Top Secret-cleared users
alice (Top Secret) creates classified_report.txt → labeled Top Secret
alice CANNOT share classified_report.txt with bob (Secret clearance)
→ The system refuses, regardless of alice's wishes
→ alice cannot downgrade the classification label
```

**MAC controls information flow, not just access to containers.** The
system tracks the sensitivity of data and ensures it never flows to a
less-privileged context.

**Where MAC is used:**
- Military and intelligence systems (MLS — Multi-Level Security)
- SELinux (Security-Enhanced Linux) — MAC enforcement in the Linux kernel
- AppArmor — MAC via application profiles
- iOS/Android app sandboxing — apps can't access each other's data
  regardless of user intent
- macOS Gatekeeper + TCC — system enforces camera/microphone access
  policies that apps can't bypass

### DAC + MAC Together

Most secure systems layer both:

```
Layer 1 (MAC): System enforces mandatory policy
  → "This process can only access files labeled 'web-server'"
  → Cannot be overridden by any user

Layer 2 (DAC): Within MAC boundaries, owners control access
  → "alice can read/write, bob can read, others can't access"
  → Owner's discretion, but constrained by MAC
```

SELinux is the canonical example. A process running as root (which
traditionally bypasses all DAC checks in Unix) is still constrained by
SELinux MAC policies. Root can set file permissions however it wants
(DAC), but if the SELinux policy says the `httpd_t` process type can't
read files labeled `user_home_t`, root's DAC permissions are irrelevant.

---

## The Bell-LaPadula Model (1973)

The foundational formal model for **confidentiality**. Developed by David
Elliott Bell and Leonard LaPadula at MITRE for the U.S. Department of
Defense.

### The Setup

Every subject (user/process) has a **clearance level**.
Every object (file/resource) has a **classification level**.

The levels form a **lattice** (a partially ordered set):

```
Top Secret (TS)
     │
  Secret (S)
     │
Confidential (C)
     │
Unclassified (U)
```

In practice, the lattice also includes **compartments** (categories):

```
(Top Secret, {Nuclear, SIGINT})
(Top Secret, {Nuclear})
(Secret, {SIGINT})
(Unclassified, {})
```

A clearance **dominates** a classification if the clearance level is ≥ the
classification level AND the clearance compartments are a superset of the
classification compartments.

```
(TS, {Nuclear, SIGINT}) dominates (S, {Nuclear})    ✓
(TS, {Nuclear})         dominates (S, {SIGINT})      ✗ (missing SIGINT)
(S, {Nuclear, SIGINT})  dominates (TS, {Nuclear})    ✗ (S < TS)
```

### The Two Rules

**Simple Security Property (No Read Up):**

A subject can read an object only if the subject's clearance **dominates**
the object's classification.

```
Subject: alice (Secret, {Nuclear})
Object:  report.txt (Confidential, {Nuclear})

Secret ≥ Confidential ✓
{Nuclear} ⊇ {Nuclear} ✓
→ alice CAN read report.txt

Object:  briefing.txt (Top Secret, {Nuclear})
Secret < Top Secret ✗
→ alice CANNOT read briefing.txt
```

A Secret-cleared analyst cannot read Top Secret documents. Information
flows down (to lower clearances through reading), which is prevented.

**Star Property (\*-Property) (No Write Down):**

A subject can write to an object only if the object's classification
**dominates** the subject's clearance.

```
Subject: alice (Secret, {Nuclear})
Object:  public_notes.txt (Unclassified, {})

Unclassified < Secret ✗
→ alice CANNOT write to public_notes.txt

Object:  classified_log.txt (Top Secret, {Nuclear, SIGINT})
Top Secret ≥ Secret ✓
{Nuclear, SIGINT} ⊇ {Nuclear} ✓
→ alice CAN write to classified_log.txt
```

**Why no write down?** If alice has Secret clearance and has read Secret
documents, her knowledge is at the Secret level. If she writes to an
Unclassified file, she might (intentionally or accidentally) include
Secret information. The system prevents this by blocking the write
entirely.

**The combined effect:** Information can only flow **upward** in the
lattice. Data can be read from lower levels and written to higher levels,
but never the reverse. This creates a one-way valve for information flow.

```
Top Secret ← can receive information from all levels
     ↑
  Secret   ← can receive from Confidential and Unclassified
     ↑
Confidential ← can receive from Unclassified
     ↑
Unclassified ← can only receive from Unclassified
```

### The Tranquility Properties

**Strong tranquility**: Security labels never change. Once a document is
classified as Top Secret, it stays Top Secret forever. Simplest to
implement, most restrictive.

**Weak tranquility**: Labels can change but never in a way that violates
the security policy. A document can be **upgraded** (Confidential → Secret)
but not **downgraded** without explicit declassification by a trusted
authority.

### Limitations

1. **No integrity protection**: Bell-LaPadula only addresses confidentiality.
   A low-clearance subject can write to a high-clearance object (the
   \*-property allows writing up). An attacker with Unclassified clearance
   could write garbage to a Top Secret file — corrupting it. The model
   protects against information leaking down but not against corruption
   flowing up.

2. **Covert channels**: The model prevents direct information flow through
   read/write operations. But information can leak through **side channels**:
   timing of operations, resource consumption, existence or absence of
   files, error messages. Covert channel analysis is a separate discipline.

3. **Rigid hierarchy**: Real-world information sharing doesn't always follow
   strict hierarchies. A Top Secret report might need to be shared with a
   Secret-cleared analyst for a specific operation. Bell-LaPadula requires
   either upgrading the analyst's clearance or declassifying the report —
   both heavyweight operations.

4. **The \*-property is impractical in pure form**: If Top Secret users can
   never write to Secret files, they can't send email to Secret-cleared
   colleagues or save documents to shared drives. Real systems implement
   "trusted subjects" that are exempt from the \*-property — essentially
   punching holes in the model for usability.

---

## The Biba Model (1977)

The **integrity** counterpart to Bell-LaPadula. Developed by Kenneth Biba.
Where Bell-LaPadula prevents unauthorized **disclosure**, Biba prevents
unauthorized **modification**.

### The Insight

Confidentiality and integrity have **dual** requirements:

| | Confidentiality (Bell-LaPadula) | Integrity (Biba) |
|---|------|------|
| Concern | Unauthorized reading | Unauthorized modification |
| Prevent | Information flowing down | Corruption flowing up |
| Read rule | No read up | No read down |
| Write rule | No write down | No write up |

The rules are **inverted**. This makes sense intuitively:

- **Confidentiality**: Don't let secrets flow to uncleared people
  (prevent downward flow)
- **Integrity**: Don't let untrusted data corrupt trusted systems
  (prevent upward flow)

### The Setup

Every subject has an **integrity level**.
Every object has an **integrity level**.

```
High Integrity (verified, audited, trusted)
     │
Medium Integrity (internal, partially trusted)
     │
Low Integrity (external, untrusted)
```

### The Two Rules

**Simple Integrity Property (No Read Down):**

A subject can read an object only if the object's integrity level
**dominates** the subject's integrity level.

```
Subject: critical_service (High Integrity)
Object:  user_input.txt (Low Integrity)

Low < High ✗
→ critical_service CANNOT read user_input.txt
```

**Why?** If a high-integrity process reads low-integrity data, it might
make decisions based on that data — effectively corrupting its behavior.
A banking transaction processor (High) should not read unvalidated user
input (Low) directly.

**Integrity Confinement Property (No Write Up):**

A subject can write to an object only if the subject's integrity level
**dominates** the object's integrity level.

```
Subject: untrusted_script (Low Integrity)
Object:  system_config.txt (High Integrity)

Low < High ✗
→ untrusted_script CANNOT write to system_config.txt
```

**Why?** Low-integrity code modifying high-integrity data is the
definition of corruption. An unvalidated user script should not be able
to modify system configuration.

### The Combined Effect

Information flows **downward** in the integrity lattice — the inverse
of Bell-LaPadula's upward flow.

```
High Integrity → produces trusted output, can write to High and below
     │
     ↓ (information flows down)
     │
Medium Integrity → can receive from High, produce for Medium and below
     │
     ↓
     │
Low Integrity → can receive from all, can only write to Low
```

### Real-World Biba: Input Validation

You implement Biba every time you validate user input:

```javascript
// User input is Low Integrity
const rawInput = req.body.query;  // Low

// Validation is the integrity upgrade mechanism
const validated = sanitize(rawInput);  // Upgraded to Medium
const parameterized = db.prepare(validated);  // Upgraded to High

// Now safe to use in a High Integrity context
const result = parameterized.execute();
```

The **sanitization/validation boundary** is Biba's "trusted subject" — the
component authorized to upgrade data from a lower integrity level to a
higher one. Without this boundary, low-integrity data flows directly into
high-integrity operations (SQL injection, XSS, command injection).

Every injection vulnerability is a Biba violation: untrusted input
(Low Integrity) reaching a trusted interpreter (High Integrity) without
passing through a validation/sanitization gate.

### Real-World Biba: Software Supply Chain

```
Integrity levels:
  High:   Your production code (audited, reviewed, tested)
  Medium: Direct dependencies (partially trusted, version-locked)
  Low:    Transitive dependencies (unknown authors, unknown code)

Biba says:
  Your High-integrity code should not execute Low-integrity code
  without validation (auditing, pinning, SRI hashes).

Supply chain attacks violate Biba:
  Attacker compromises a Low-integrity transitive dependency
  → It executes in your High-integrity production environment
  → Corruption flows upward: Low → High
```

Package managers don't enforce Biba. `npm install` pulls transitive
dependencies at whatever integrity level they happen to be. Lock files
and hash verification (package-lock.json, yarn.lock with integrity
hashes) are partial Biba enforcement — they ensure the code hasn't been
tampered with since you last audited it, but they don't verify the code
is trustworthy in the first place.

---

## The Clark-Wilson Model (1987)

A formal model for **commercial integrity** — designed for business
systems rather than military systems. Where Biba is about preventing
corruption through information flow, Clark-Wilson is about ensuring
**well-formed transactions** and **separation of duties**.

### Why a Different Integrity Model?

Biba's "no read down, no write up" is too rigid for business:

- An accountant MUST read unvalidated invoices (read down) to process them
- A payment system MUST modify account balances based on external input
  (write up based on lower-integrity data)

Business integrity isn't about preventing all cross-level flow — it's
about ensuring that data modifications follow **correct procedures** and
are performed by **authorized personnel**.

### Core Concepts

**Constrained Data Items (CDIs):**
Data that must maintain integrity. Account balances, inventory counts,
medical records, financial transactions. These can only be modified
through approved procedures.

**Unconstrained Data Items (UDIs):**
Data that hasn't been validated yet. User input, imported files, external
API responses. These have no integrity guarantee.

**Transformation Procedures (TPs):**
The only way to modify CDIs. Well-defined operations that maintain
integrity invariants. "Transfer funds" is a TP — it debits one account
and credits another atomically, ensuring the total balance is preserved.

**Integrity Verification Procedures (IVPs):**
Checks that confirm CDIs are in a valid state. "Total debits = total
credits." Run periodically or after TPs to detect integrity violations.

### The Rules

**Certification Rules (what must be true about the system):**

```
C1: All IVPs must ensure CDIs are in a valid state when the IVP runs.
    (The auditing procedures actually work.)

C2: All TPs must take CDIs from one valid state to another valid state.
    (Every operation preserves invariants. A fund transfer can't create
    or destroy money.)

C3: Separation of duty — the set of TPs that a user can execute must
    be chosen to ensure no single user can compromise the integrity of
    a CDI. (The person who creates a purchase order can't approve it.)

C4: All TPs must log their operations to an append-only CDI (audit log).
    (Everything is traceable.)

C5: Any TP that takes a UDI as input must validate it and transform it
    into a CDI (or reject it). UDIs cannot directly become CDIs.
    (All external input must be validated before it enters the system.)
```

**Enforcement Rules (what the system must enforce):**

```
E1: The system must maintain the list of (user, TP, CDI) triples —
    which users can execute which TPs on which data items.
    (The authorization matrix.)

E2: The system must ensure that users can only execute TPs on CDIs
    for which they are authorized per E1.
    (Enforcement of the authorization matrix.)

E3: The system must authenticate the identity of each user attempting
    to execute a TP.
    (Authentication before authorization — tying back to our auth topics.)

E4: Only the certifier (security officer, not the TP author) can modify
    the (user, TP, CDI) authorization list.
    (Separation of duty in managing the authorization system itself.)
```

### Clark-Wilson in Practice

Every well-designed business application implements Clark-Wilson, often
without knowing it:

```
Online Banking System:

CDIs:
  - Account balances
  - Transaction records
  - Audit logs

UDIs:
  - User-submitted transfer requests
  - Imported bank statements

TPs:
  - transfer_funds(from, to, amount)
    → Validates amount > 0
    → Checks from.balance >= amount
    → Debits from, credits to (atomic)
    → Logs the transaction (C4)
  - approve_wire(request_id)
    → Can only be executed by a manager (C3: separation of duty)
    → The person who created the wire request cannot approve it

IVPs:
  - daily_reconciliation()
    → Sums all account balances
    → Compares to known total
    → Flags discrepancies

Authorization triples (E1):
  (teller, transfer_funds, checking_accounts)
  (teller, transfer_funds, savings_accounts)
  (manager, approve_wire, all_accounts)
  (auditor, daily_reconciliation, all_accounts)
  ✗ (teller, approve_wire, any)  — separation of duty
```

### Clark-Wilson vs Biba

| | Biba | Clark-Wilson |
|---|------|-------------|
| Focus | Information flow | Transaction correctness |
| Approach | Prevent low→high contamination | Ensure procedures are correct and authorized |
| Flexibility | Rigid lattice | Procedures can cross integrity levels if well-formed |
| Separation of duty | Not addressed | Core requirement (C3) |
| Audit | Not addressed | Core requirement (C4) |
| Input validation | Implicit (no read down) | Explicit (C5: UDI→CDI transformation) |
| Best for | Systems where contamination = failure | Business systems with processes and workflows |

---

## How the Formal Models Map to Modern Systems

The formal models aren't relics — they're the **theory underneath** every
practical system:

### Bell-LaPadula → Data Classification

```
Formal:    Security labels on objects, clearances on subjects
Modern:    AWS Macie classifying S3 objects as PII/financial/confidential
           GitHub secret scanning detecting API keys in commits
           DLP (Data Loss Prevention) systems blocking emails with SSNs

Formal:    No read up
Modern:    IAM policies restricting access by data classification tag

Formal:    No write down
Modern:    DLP blocking copy of classified data to USB drives
           Cloud DLP blocking sensitive data from leaving a VPC
```

### Biba → Input Validation and Supply Chain Security

```
Formal:    Integrity levels, no read down, no write up
Modern:    Input sanitization before database queries (prevent SQL injection)
           CSP (Content Security Policy) blocking inline scripts (prevent XSS)
           Package lock files with integrity hashes
           Code signing for deployment artifacts
           Container image signing (cosign, Notary)

Formal:    Trusted subjects that can upgrade integrity
Modern:    Validation middleware, sanitization libraries
           CI/CD pipelines (untrusted code → build → test → verified artifact)
```

### Clark-Wilson → Business Logic Authorization

```
Formal:    Transformation Procedures on Constrained Data Items
Modern:    API endpoints that modify data through validated operations
           Database stored procedures with integrity constraints
           Workflow engines with approval steps

Formal:    Separation of duty (C3)
Modern:    Pull request approvals (author ≠ reviewer)
           Payment authorization (requester ≠ approver)
           AWS SCPs preventing account admins from modifying audit trails

Formal:    Integrity Verification Procedures (IVPs)
Modern:    Database CHECK constraints, foreign key constraints
           Reconciliation jobs, consistency checkers
           Terraform plan/apply (plan = IVP, apply = TP)

Formal:    Audit logging (C4)
Modern:    AWS CloudTrail, database audit logs, application event logs
           Append-only audit tables with RLS preventing modification
```

### MAC → Sandboxing and Process Isolation

```
Formal:    System-enforced policy that users cannot override
Modern:    SELinux: httpd_t can only access httpd_sys_content_t files
           Docker: seccomp profiles restricting system calls
           iOS: apps cannot access other apps' data
           Chromium: renderer processes can't access the filesystem
           WebAssembly: modules execute in a sandboxed memory space

The key MAC property:
  Even if the application is compromised (DAC is bypassed),
  MAC limits what the compromised process can do.
  RCE in a web server ≠ access to the database if MAC is enforced.
```

---

## The Lattice Model — Unifying Framework

Bell-LaPadula and Biba are both instances of a more general **lattice-
based access control** model (Denning, 1976).

A **lattice** is a partially ordered set where every pair of elements has
a unique least upper bound (join) and greatest lower bound (meet).

```
For security labels (level, compartments):

Join (least upper bound):
  (Secret, {A}) ⊔ (Top Secret, {B}) = (Top Secret, {A, B})
  → The label that is high enough to dominate both

Meet (greatest lower bound):
  (Secret, {A}) ⊓ (Top Secret, {B}) = (Secret, {})
  → The label dominated by both

This forms a lattice because every pair has a unique join and meet.
```

**The information flow rule**: Information can flow from label L1 to label
L2 only if L1 ≤ L2 in the lattice (for confidentiality) or L1 ≥ L2
(for integrity).

The lattice model provides a **single mathematical framework** for
reasoning about both confidentiality and integrity, and for proving that
a system's access control policy prevents unauthorized information flow.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| DAC vs MAC | DAC: owner decides. MAC: system enforces. DAC can't control information flow. |
| Bell-LaPadula | Confidentiality model. No read up, no write down. Information flows upward only. |
| The \*-property prevents data leaking down | A Top Secret user can't write to an Unclassified file. |
| Biba | Integrity model. No read down, no write up. Corruption can't flow upward. |
| Every injection attack is a Biba violation | Untrusted input reaching a trusted interpreter without validation. |
| Clark-Wilson | Commercial integrity. Well-formed transactions, separation of duties, audit logging. |
| TPs are the only way to modify CDIs | Data changes must go through validated procedures, not direct writes. |
| Formal models are not academic abstractions | They're the theory underneath input validation, sandboxing, DLP, code signing, and every authorization decision. |
| The lattice model unifies both | Confidentiality and integrity are dual problems on the same lattice structure. |
| MAC prevents damage from compromised DAC | Even if root is compromised, SELinux limits what the attacker can do. |
