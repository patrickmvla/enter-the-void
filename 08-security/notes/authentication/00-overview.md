# Authentication — Proving Identity

## What Authentication Is

Authentication is the process of verifying that an entity is who they
claim to be. Before a system can decide what you're allowed to do
(authorization), it must first establish who you are.

At the most fundamental level, authentication is a **challenge-response
protocol**: the system challenges the claimant to produce evidence that
only the legitimate entity could produce, and the claimant responds.

---

## The Three Factors

Every authentication method falls into one of three categories. These
are not arbitrary groupings — they represent fundamentally different
types of evidence, each with distinct attack surfaces.

### Something You Know (Knowledge)

A secret that exists in the claimant's memory.

**Examples**: Passwords, PINs, security questions, passphrases.

**Attack surface**: The secret can be guessed (brute force, dictionary
attacks), stolen (phishing, keyloggers, shoulder surfing), or leaked
(database breaches, reuse across services). The secret exists in at
least two places — the user's mind and the server's storage — creating
two points of compromise.

**Fundamental weakness**: Knowledge can be shared, forgotten, and
extracted. A human can be tricked into revealing it. Once compromised,
there's no physical barrier to exploitation — the attacker uses it from
anywhere.

### Something You Have (Possession)

A physical object the claimant possesses.

**Examples**: Phone (TOTP app, push notification), hardware security key
(YubiKey), smart card, SIM card (SMS — weakest form).

**Attack surface**: The object can be stolen, cloned (SIM swapping), or
remotely compromised (malware on phone). But the attacker must obtain or
interact with the physical object — remote exploitation alone isn't
sufficient.

**Fundamental strength**: Physical objects are scarce. An attacker can
guess a million passwords per second from across the world, but they
can't duplicate a hardware security key remotely. The physics of the
object limits the attack.

### Something You Are (Inherence)

A biometric characteristic of the claimant's body.

**Examples**: Fingerprint, facial geometry, iris pattern, voice print,
typing cadence (behavioral biometric).

**Attack surface**: Biometrics can't be changed if compromised. Your
fingerprint is left on every surface you touch. High-resolution photos
reveal iris patterns. Voice can be synthesized. Biometric systems have
false-positive rates (accepting impostors) and false-negative rates
(rejecting legitimate users).

**Fundamental weakness**: Biometrics are not secrets — they're public
physical characteristics. They work as authentication because the
*sensor* is trusted (your phone's fingerprint reader is tamper-resistant),
not because the biometric itself is secret. Move the authentication to
an untrusted sensor and biometrics collapse.

**Fundamental strength**: Biometrics can't be forgotten or left at home.
They're always with you.

---

## Why Multi-Factor

Each factor has a different attack model:

- **Knowledge** is vulnerable to **remote attacks** (phishing, brute force,
  breach reuse)
- **Possession** is vulnerable to **physical attacks** (theft, SIM swap)
- **Inherence** is vulnerable to **sensor attacks** (spoofed biometrics)

Combining factors from different categories forces the attacker to execute
**multiple independent attack types simultaneously**. An attacker who
phishes your password still needs your phone. An attacker who steals your
phone still needs your password.

The factors must be **independent**. Two knowledge factors (password +
security question) fail together under the same attack (phishing extracts
both). A password + TOTP code requires both phishing and physical access
— fundamentally different operations.

---

## Authentication Methods — The Landscape

### Direct Authentication

The user proves identity directly to your system.

| Method | Factor(s) | How It Works |
|--------|-----------|-------------|
| Password | Knowledge | User provides a shared secret → [01-password-hashing.md](01-password-hashing.md) |
| Password + TOTP | Knowledge + Possession | Password then time-based code from authenticator → [05-totp-mfa.md](05-totp-mfa.md) |
| Password + WebAuthn | Knowledge + Possession | Password then hardware key challenge-response → [05-totp-mfa.md](05-totp-mfa.md) |
| Passkey (WebAuthn) | Possession + Inherence | Hardware key + biometric unlock. No password. |
| Client certificate (mTLS) | Possession | TLS handshake proves possession of private key |

### Delegated Authentication

Your system trusts another system to authenticate the user.

| Method | How It Works |
|--------|-------------|
| OAuth 2.0 / OIDC | User authenticates with a provider (Google, GitHub), your app receives proof → [04-oauth2.md](04-oauth2.md) |
| SAML | Enterprise SSO. XML-based assertions from an Identity Provider. |
| Kerberos | Ticket-based. User authenticates once with KDC, receives tickets for services. |

### Session Maintenance

After initial authentication, the system remembers the user.

| Method | How It Works |
|--------|-------------|
| Session cookies | Server-side state, client holds opaque ID → [02-sessions.md](02-sessions.md) |
| JWTs | Client holds signed token with identity claims → [03-jwts.md](03-jwts.md) |
| API keys | Long-lived tokens for service-to-service auth. Not user authentication. |

---

## How the Topics Connect

```
User wants to access a protected resource
│
├─ How does the system verify the password?
│  └─ 01-password-hashing.md
│     Hashing, salting, bcrypt/Argon2, timing attacks
│
├─ How does the system remember the user after login?
│  ├─ 02-sessions.md
│  │  Server-side state, cookies, Redis, session attacks
│  │
│  └─ 03-jwts.md
│     Client-side tokens, signing algorithms, revocation tradeoff
│
├─ How does "Login with Google/GitHub" work?
│  └─ 04-oauth2.md
│     Authorization code flow, PKCE, OIDC, scopes
│
└─ How does 2FA add a second layer?
   └─ 05-totp-mfa.md
      HOTP/TOTP internals, WebAuthn, phishing resistance
```

Each topic builds on the previous. Password hashing establishes the
cryptographic primitives. Sessions and JWTs show two approaches to
maintaining authenticated state. OAuth shows how to delegate
authentication to a third party. TOTP/WebAuthn adds a second factor
on top of any of the above.

---

## The Authentication Decision

| Scenario | Recommended Approach |
|----------|---------------------|
| Traditional web app, single service | Password + session cookies + optional TOTP |
| SPA with API backend | Password + JWT (access/refresh) + TOTP |
| Microservice architecture | OIDC for user auth, JWT propagation between services |
| Consumer app with social login | OAuth 2.0 / OIDC (Google, Apple, GitHub) |
| Enterprise / B2B | SAML or OIDC with company IdP, enforce MFA |
| High security (banking, gov) | Password + WebAuthn (hardware key), short sessions, step-up auth |
| Machine-to-machine | OAuth client credentials or mTLS |
| Passwordless | WebAuthn/Passkeys or magic links (email-based) |

There is no single "best" authentication method. The choice depends on
your threat model, user base, and the value of what you're protecting.
