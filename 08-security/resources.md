# Security — Resources

Curated references worth revisiting. Not a link dump — each entry is here
because it explains something at a depth that matters.

---

## RFCs & Standards

These are the actual specifications. Read them when you want to know exactly
how something works, not someone's interpretation.

- **RFC 4226** — HOTP: HMAC-Based One-Time Password Algorithm
  https://datatracker.ietf.org/doc/html/rfc4226
- **RFC 6238** — TOTP: Time-Based One-Time Password Algorithm
  https://datatracker.ietf.org/doc/html/rfc6238
- **RFC 6749** — OAuth 2.0 Authorization Framework
  https://datatracker.ietf.org/doc/html/rfc6749
- **RFC 7519** — JSON Web Token (JWT)
  https://datatracker.ietf.org/doc/html/rfc7519
- **RFC 7636** — PKCE (Proof Key for Code Exchange)
  https://datatracker.ietf.org/doc/html/rfc7636
- **RFC 7662** — OAuth 2.0 Token Introspection
  https://datatracker.ietf.org/doc/html/rfc7662
- **RFC 6265** — HTTP State Management Mechanism (Cookies)
  https://datatracker.ietf.org/doc/html/rfc6265
- **RFC 2898** — PBKDF2 (Password-Based Key Derivation Function 2)
  https://datatracker.ietf.org/doc/html/rfc2898
- **OpenID Connect Core 1.0**
  https://openid.net/specs/openid-connect-core-1_0.html
- **OAuth 2.1 Draft** — Consolidates best practices, removes implicit + ROPC
  https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1
- **NIST SP 800-63B** — Digital Identity Guidelines (Authentication & Lifecycle)
  https://pages.nist.gov/800-63-3/sp800-63b.html
- **INCITS 359-2004** — NIST RBAC Standard
  https://csrc.nist.gov/projects/role-based-access-control

---

## Papers

- **Zanzibar: Google's Consistent, Global Authorization System (2019)**
  https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/
  The paper behind Google Drive, YouTube, and Cloud IAM authorization.
  Read for: relation tuples, Leopard indexing, Zookies, the performance
  model for trillions of relationships at <10ms.

- **Macaroons: Cookies with Contextual Caveats (2014)**
  https://research.google/pubs/macaroons-cookies-with-contextual-caveats-for-decentralized-authorization-in-the-cloud/
  Capability tokens with HMAC-chained caveats. Read for: attenuated
  delegation, why capabilities solve the confused deputy problem.

- **The Password Hashing Competition (PHC)**
  https://www.password-hashing.net/
  How Argon2 was selected. Includes the Argon2 specification and analysis
  of all candidates.

- **scrypt: A new key derivation function (Colin Percival, 2009)**
  https://www.tarsnap.com/scrypt/scrypt.pdf
  The original scrypt paper. Read for: memory-hard functions, ROMix,
  time-memory tradeoff analysis.

---

## Deep Technical References

- **How bcrypt Works (Provos & Mazieres, 1999)**
  https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf
  The original bcrypt paper. Eksblowfish key schedule, cost factor design,
  why 4KB state resists parallelization.

- **CrackStation — Salted Password Hashing**
  https://crackstation.net/hashing-security.htm
  Practical guide with the right level of depth. Good reference for why
  each defense exists.

- **Auth0 — OAuth 2.0 Authorization Code Flow with PKCE**
  https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce
  Clear walkthrough with diagrams. Useful for seeing the flow in practice.

- **JWT.io**
  https://jwt.io/
  Decode and inspect JWTs. The debugger is useful for understanding token
  structure. The introduction page explains the format well.

- **OWASP Cheat Sheet Series**
  https://cheatsheetseries.owasp.org/
  Particularly:
  - Password Storage Cheat Sheet
  - Session Management Cheat Sheet
  - JSON Web Token Cheat Sheet
  - OAuth 2.0 Cheat Sheet
  - Cross-Site Request Forgery Prevention Cheat Sheet
  - Authorization Cheat Sheet

- **Web Authentication (WebAuthn) Spec**
  https://www.w3.org/TR/webauthn-3/
  The W3C specification for FIDO2 browser authentication. Dense but
  definitive.

---

## Authorization Systems — Documentation

- **OPA (Open Policy Agent)**
  https://www.openpolicyagent.org/docs/latest/
  Rego language reference, deployment patterns, integration guides.

- **Cedar Language**
  https://www.cedarpolicy.com/
  AWS's authorization policy language. Read the specification and the
  formal verification docs.

- **SpiceDB (AuthZed)**
  https://authzed.com/docs
  Zanzibar implementation. The schema language docs are the clearest
  explanation of ReBAC modeling.

- **OpenFGA**
  https://openfga.dev/docs
  Zanzibar-inspired, simpler API. Good modeling guides.

- **PostgreSQL Row-Level Security**
  https://www.postgresql.org/docs/current/ddl-rowsecurity.html
  Official docs. Read alongside the CREATE POLICY reference.

- **AWS IAM Policy Evaluation Logic**
  https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
  The canonical reference for how AWS resolves allow/deny across policy types.

---

## Breach Analysis & Attack Research

- **Have I Been Pwned**
  https://haveibeenpwned.com/
  Troy Hunt's breach database. Useful for understanding the scale and
  frequency of credential breaches.

- **Hashcat**
  https://hashcat.net/hashcat/
  GPU-accelerated password cracking. The benchmark page shows real-world
  hash rates per algorithm per GPU — the numbers cited in the notes.

- **Evilginx**
  https://github.com/kgretzky/evilginx2
  Real-time phishing proxy that bypasses TOTP. Understanding the tool
  is understanding the threat model TOTP can't defend against.

- **SHAttered — SHA-1 Collision**
  https://shattered.io/
  The first practical SHA-1 collision (2017). Relevant context for why
  HMAC-SHA1 is still safe (different attack model) while SHA-1 for
  signatures is not.

---

## Books

- **Serious Cryptography** — Jean-Philippe Aumasson
  Covers hash functions, MACs, authenticated encryption, and public-key
  crypto at the right depth. No hand-waving, no unnecessary math.

- **Designing Data-Intensive Applications** — Martin Kleppmann
  Chapter 4 (Encoding), Chapter 9 (Consistency & Consensus) for
  understanding the distributed systems context behind OAuth,
  session replication, and Zanzibar's consistency model.

- **The Web Application Hacker's Handbook** — Stuttard & Pinto
  Authentication and session management attack taxonomy. Dated but the
  fundamentals haven't changed.

- **OAuth 2 in Action** — Justin Richer & Antonio Sanso
  The most thorough treatment of OAuth implementation details, including
  the security considerations most tutorials skip.
