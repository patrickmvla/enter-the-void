# TLS — Securing the Channel

## The Problem TLS Solves

TCP delivers data reliably. DNS resolves names. But neither provides:

- **Confidentiality**: Anyone on the network path can read TCP data in
  plaintext — ISPs, coffee shop WiFi operators, compromised routers,
  government surveillance taps.
- **Integrity**: An attacker can modify TCP segments in transit (inject
  malicious JavaScript into an HTTP response, redirect a download to
  malware). TCP's checksum detects accidental corruption, not intentional
  modification.
- **Authentication**: TCP connects to an IP address, but nothing proves
  that IP belongs to the server you intended. A DNS spoofing or BGP
  hijacking attack can redirect your connection to an attacker.

TLS (Transport Layer Security) solves all three. It wraps a TCP connection
in a cryptographic layer that ensures:

1. Only the intended parties can read the data (confidentiality)
2. The data hasn't been tampered with (integrity)
3. You're talking to who you think you're talking to (authentication)

TLS sits between TCP and the application layer. The application sees a
stream of plaintext; TLS handles all the cryptography transparently.

```
Without TLS:  Application → TCP → IP → Wire
With TLS:     Application → TLS → TCP → IP → Wire
```

Every HTTPS connection, every secure API call, every encrypted email relay
uses TLS. It's the single most important security protocol on the internet.

---

## TLS Record Protocol

All TLS communication is structured as **records**. The record protocol
frames the data and provides the basic structure that the handshake and
application data ride on.

### Record Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Content Type  |     Version   |            Length             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                    Fragment (≤ 16384 bytes)                    |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Content Type** (1 byte):
  - 20 = ChangeCipherSpec
  - 21 = Alert
  - 22 = Handshake
  - 23 = Application Data

- **Version** (2 bytes): `0x0303` = TLS 1.2, `0x0303` = TLS 1.3 (TLS 1.3
  records lie about their version for compatibility — they say 1.2 in the
  record layer to avoid middlebox interference. The actual version is
  negotiated inside the handshake via the `supported_versions` extension).

- **Length** (2 bytes): Size of the fragment. Maximum 2^14 = 16384 bytes
  (before compression/encryption). After encryption, up to 16384 + 256
  bytes (encryption overhead).

### Record Types in Practice

```
Handshake:           ClientHello, ServerHello, Certificate, etc.
                     Multiple handshake messages can be packed into
                     one record, or one message can span multiple records.

Application Data:    Encrypted HTTP requests/responses, encrypted
                     anything. After the handshake, all data is type 23.

Alert:               Warnings and fatal errors. close_notify (graceful
                     shutdown), bad_record_mac (integrity failure),
                     handshake_failure, certificate_expired, etc.

ChangeCipherSpec:    TLS 1.2 only. Signals the switch from plaintext to
                     encrypted. TLS 1.3 eliminates this (encryption starts
                     immediately after ServerHello).
```

---

## The TLS 1.3 Handshake

TLS 1.3 (RFC 8446, published 2018) is the current version. It's a
significant simplification over TLS 1.2 — fewer round trips, fewer
cipher suites, fewer things that can go wrong.

### The Full Handshake (1-RTT)

```
Client                                              Server

ClientHello
  + supported_versions
  + key_share (ECDHE public key)
  + signature_algorithms
  + server_name (SNI)
  + psk_key_exchange_modes
  + [pre_shared_key]
───────────────────────────────────────────────────>
                                                    ServerHello
                                                      + supported_versions
                                                      + key_share (ECDHE public key)
                                              {EncryptedExtensions}
                                              {CertificateRequest*}
                                              {Certificate}
                                              {CertificateVerify}
                                              {Finished}
<───────────────────────────────────────────────────
{Certificate*}
{CertificateVerify*}
{Finished}
───────────────────────────────────────────────────>
[Application Data]          <────────────────────>  [Application Data]

Key:
  + = extension in that message
  {} = encrypted with handshake keys
  [] = encrypted with application keys
  * = optional (only if client auth is requested)
```

**One round trip** to establish a fully encrypted connection. Compare to
TLS 1.2's two round trips. This matters — on a 100ms RTT connection,
TLS 1.3 saves 100ms per new connection.

### Step by Step

**1. ClientHello**

The client sends everything the server needs in a single message:

```
ClientHello:
  client_random: 32 bytes of cryptographic randomness
  legacy_session_id: 32 bytes (compatibility, ignored in TLS 1.3)
  cipher_suites: [
    TLS_AES_256_GCM_SHA384,        (0x1302)
    TLS_AES_128_GCM_SHA256,        (0x1301)
    TLS_CHACHA20_POLY1305_SHA256   (0x1303)
  ]
  extensions:
    supported_versions: [TLS 1.3, TLS 1.2]
    key_share: [
      x25519: <32-byte public key>,
      secp256r1: <65-byte public key>     (fallback)
    ]
    signature_algorithms: [
      ecdsa_secp256r1_sha256,
      rsa_pss_rsae_sha256,
      ed25519
    ]
    server_name: "example.com"            (SNI)
    application_layer_protocol_negotiation: ["h2", "http/1.1"]  (ALPN)
```

**Key insight**: The client **speculatively sends key shares** for its
preferred groups. In TLS 1.2, the key exchange algorithm was negotiated
first, then keys were exchanged in a second round trip. TLS 1.3
eliminates this by having the client guess which key exchange the server
will choose and send its public key immediately.

If the server doesn't support any of the client's key share groups, it
sends a `HelloRetryRequest` asking for a different group — adding an
extra round trip. But this is rare in practice because x25519 and
secp256r1 cover nearly all servers.

**SNI (Server Name Indication)**: The client tells the server which
hostname it's connecting to, in plaintext, before encryption is
established. This allows one IP address to host multiple TLS sites
(virtual hosting). But it also means anyone watching the connection can
see which domain you're connecting to. **ECH (Encrypted Client Hello)**,
still being standardized, encrypts the SNI using a public key published
in DNS.

**ALPN (Application-Layer Protocol Negotiation)**: Negotiates the
application protocol (HTTP/2, HTTP/1.1, etc.) during the TLS handshake
instead of after. This avoids the extra round trip that HTTP/2's upgrade
mechanism would require.

**2. ServerHello**

The server chooses a cipher suite and key exchange group, and sends
its own key share:

```
ServerHello:
  server_random: 32 bytes
  cipher_suite: TLS_AES_256_GCM_SHA384
  extensions:
    supported_versions: TLS 1.3
    key_share: x25519: <32-byte public key>
```

At this point, both sides can compute the shared secret:

```
shared_secret = ECDHE(client_private_key, server_public_key)
              = ECDHE(server_private_key, client_public_key)
              (same value — this is the Diffie-Hellman property)
```

**Everything after ServerHello is encrypted.** TLS 1.3 derives
handshake encryption keys immediately from the shared secret and
encrypts the remaining handshake messages. In TLS 1.2, the certificate
and other handshake data were sent in plaintext — revealing the
server's certificate chain to passive observers.

**3. Server's Encrypted Messages**

All encrypted with handshake traffic keys derived from the shared secret:

- **EncryptedExtensions**: Non-cryptographic extensions that couldn't be
  in ServerHello (which must remain compatible with TLS 1.2 middleboxes).
  Includes ALPN result.

- **Certificate**: The server's certificate chain (see Certificate Chain
  section below). In TLS 1.3, this is encrypted — a passive observer
  can't see which certificate the server presents.

- **CertificateVerify**: A signature over the entire handshake transcript
  (hash of all messages so far) using the server's private key. This
  proves the server holds the private key for the certificate. The
  transcript signature also prevents transcript manipulation — any
  change to any handshake message would invalidate the signature.

- **Finished**: A MAC over the handshake transcript using a key derived
  from the handshake secret. This confirms key agreement succeeded and
  the handshake wasn't tampered with.

**4. Client's Response**

- **Finished**: The client's MAC over the transcript. After this, both
  sides switch to application traffic keys.

Application data can now flow in both directions, encrypted and
authenticated.

### Key Derivation: HKDF

TLS 1.3 uses HKDF (HMAC-based Key Derivation Function, RFC 5869) to
derive all keys from the shared secret. The key schedule is a carefully
designed chain:

```
PSK (or 0)
  │
  ▼
HKDF-Extract ──────────────────> Early Secret
  │                                 │
  │                          Derive-Secret(., "ext binder", "")
  │                          Derive-Secret(., "res binder", "")
  │                          Derive-Secret(., "c e traffic", ClientHello)
  │                          Derive-Secret(., "e exp master", ClientHello)
  │
  ▼
(EC)DHE shared secret
  │
  ▼
HKDF-Extract ──────────────────> Handshake Secret
  │                                 │
  │                          Derive-Secret(., "c hs traffic", CH..SH)
  │                                 → client_handshake_traffic_secret
  │                          Derive-Secret(., "s hs traffic", CH..SH)
  │                                 → server_handshake_traffic_secret
  │
  ▼
HKDF-Extract ──────────────────> Master Secret
                                    │
                             Derive-Secret(., "c ap traffic", CH..SF)
                                    → client_application_traffic_secret
                             Derive-Secret(., "s ap traffic", CH..SF)
                                    → server_application_traffic_secret
                             Derive-Secret(., "exp master", CH..SF)
                                    → exporter_master_secret
                             Derive-Secret(., "res master", CH..CF)
                                    → resumption_master_secret
```

Each `Derive-Secret` call includes a **transcript hash** — the hash of
all handshake messages up to that point. This means every derived key is
cryptographically bound to the entire handshake. If any message is
altered, all derived keys change, and the Finished MACs won't match.

**Why this complexity?** Each key has a specific purpose and scope:
- Handshake keys encrypt the handshake itself
- Application keys encrypt the data
- The resumption secret enables 0-RTT on reconnection
- The exporter secret allows external protocols to derive TLS-bound keys

**Forward secrecy**: The (EC)DHE shared secret is ephemeral — both sides
generate fresh key pairs for every connection and discard the private keys
after deriving the master secret. Even if the server's long-term private
key (from its certificate) is later compromised, past sessions can't be
decrypted because the ephemeral keys are gone. This is **forward secrecy**
(also called "perfect forward secrecy"). TLS 1.3 mandates it — there's
no option for static key exchange.

---

## 0-RTT Resumption

When a client reconnects to a server it has visited before, TLS 1.3 can
send application data in the very first message — zero round trips of
handshake latency.

### How It Works

```
First connection (normal 1-RTT):
  Client ←→ Server handshake
  Server sends NewSessionTicket containing:
    - ticket (encrypted session state or reference)
    - ticket_lifetime (max 7 days)
    - ticket_age_add (obfuscation for age)
    - max_early_data_size

Reconnection (0-RTT):
  Client sends:
    ClientHello
      + pre_shared_key (ticket from previous session)
      + early_data indication
      + key_share (still do ECDHE for forward secrecy)
    [Early application data encrypted with early traffic keys]
  ─────────────────────────────────────────────────────>
                                          Server decrypts early data
                                          immediately, starts processing
```

The client derives early traffic keys from the pre-shared key (PSK,
from the previous session's resumption secret) and encrypts application
data before the handshake completes.

### The Replay Problem

0-RTT data is **not forward-secret** (it's encrypted with keys derived
from the PSK, not from the new ECDHE exchange) and is **replayable**.

An attacker who captures the ClientHello + early data can replay it:

```
Legitimate client sends:
  ClientHello + 0-RTT: POST /transfer?amount=1000&to=alice

Attacker captures and replays:
  Same ClientHello + 0-RTT: POST /transfer?amount=1000&to=alice
  → Server processes the transfer AGAIN
```

The server can't distinguish the replay from the original because the
0-RTT data is encrypted with the same key (derived from the same PSK).

**Mitigations:**
- Servers should only accept 0-RTT for **idempotent** requests (GET, HEAD)
- Servers can maintain a replay cache (reject duplicate ClientHellos) but
  this doesn't work across distributed server clusters
- The `max_early_data_size` limits how much 0-RTT data can be sent
- Applications must be designed to tolerate replayed 0-RTT

**In practice**: Most deployments accept 0-RTT only for safe HTTP methods.
Cloudflare and major CDNs support 0-RTT for GET requests, saving ~30-100ms
on page loads for returning visitors.

---

## TLS 1.2 vs TLS 1.3

Understanding TLS 1.2 matters because it's still widely deployed and
because TLS 1.3's design decisions make more sense when you see what
it replaced.

### TLS 1.2 Handshake (2-RTT)

```
Client                                              Server

ClientHello (version, random, cipher suites)
───────────────────────────────────────────────────>
                                                    ServerHello
                                                    Certificate
                                                    ServerKeyExchange
                                                    ServerHelloDone
<───────────────────────────────────────────────────
ClientKeyExchange
ChangeCipherSpec
Finished
───────────────────────────────────────────────────>
                                                    ChangeCipherSpec
                                                    Finished
<───────────────────────────────────────────────────
[Application Data]          <────────────────────>  [Application Data]
```

**Two full round trips** before application data flows (three if you
count TCP's handshake). On a 100ms RTT connection: 200ms of TLS
overhead vs TLS 1.3's 100ms.

### What TLS 1.3 Removed

**Static RSA key exchange**: In TLS 1.2, the client could encrypt the
pre-master secret with the server's RSA public key (from the certificate).
No ephemeral keys, no forward secrecy. If the server's private key is
later compromised, all recorded sessions can be decrypted. The NSA's
"collect now, decrypt later" strategy exploits this. TLS 1.3 eliminates
static RSA — ECDHE is mandatory.

**CBC cipher suites**: CBC mode with MAC-then-encrypt led to a series of
devastating attacks:
- **BEAST** (2011): Predictable IVs in TLS 1.0 CBC allowed chosen-plaintext
  attacks
- **Lucky Thirteen** (2013): Timing side channel in CBC padding
  verification leaked plaintext byte by byte
- **POODLE** (2014): SSL 3.0's CBC padding wasn't deterministic, allowing
  padding oracle attacks

TLS 1.3 uses only AEAD ciphers (AES-GCM, ChaCha20-Poly1305) which
combine encryption and authentication in a single operation, eliminating
the encrypt-then-MAC vs MAC-then-encrypt footgun.

**Compression**: TLS-level compression enabled the CRIME attack (2012) —
by observing the compressed size of requests containing a mix of known
and unknown data, an attacker could recover secrets (like session cookies)
byte by byte. TLS 1.3 forbids compression entirely.

**Renegotiation**: TLS 1.2 allowed renegotiating the connection parameters
mid-stream. This led to the renegotiation attack (2009) where an attacker
could inject data at the start of a renegotiated connection. TLS 1.3
removes renegotiation — if you want new keys, use KeyUpdate.

**Custom Diffie-Hellman parameters**: TLS 1.2 allowed servers to choose
arbitrary DH group parameters, enabling Logjam (2015) — many servers used
weak 512-bit or 1024-bit groups. TLS 1.3 only allows named groups
(x25519, secp256r1, etc.) with known-good parameters.

### TLS 1.3 Cipher Suites

TLS 1.3 has exactly 5 cipher suites (vs TLS 1.2's 300+):

```
TLS_AES_128_GCM_SHA256         (0x1301)  — most common
TLS_AES_256_GCM_SHA384         (0x1302)  — stronger AES
TLS_CHACHA20_POLY1305_SHA256   (0x1303)  — fast on mobile (no AES-NI)
TLS_AES_128_CCM_SHA256         (0x1304)  — IoT/constrained devices
TLS_AES_128_CCM_8_SHA256       (0x1305)  — IoT (shorter auth tag)
```

The cipher suite only specifies the AEAD algorithm and hash. Key exchange
(always ECDHE) and signature algorithm are negotiated separately via
extensions. This dramatic simplification eliminates hundreds of weak
combinations that existed in TLS 1.2.

**ChaCha20-Poly1305**: Designed by Daniel Bernstein. A stream cipher +
MAC combination that's fast in software without hardware acceleration.
Mobile devices without AES-NI instructions are 3-5x faster with
ChaCha20 than AES-GCM. Servers typically prefer AES-GCM (they have
AES-NI) but negotiate ChaCha20 when the client prefers it (mobile).

---

## The Certificate Chain

### X.509 Certificates

A TLS certificate is an X.509 document containing:

```
Certificate:
    Data:
        Version: 3
        Serial Number: 04:00:00:00:00:01:15:4b:5a:c3:94
        Issuer: C=US, O=Let's Encrypt, CN=R3
        Validity:
            Not Before: Jan  1 00:00:00 2024 GMT
            Not After:  Mar 31 23:59:59 2024 GMT
        Subject: CN=example.com
        Subject Public Key Info:
            Algorithm: ECDSA (P-256)
            Public Key: 04:a1:b2:c3:... (65 bytes, uncompressed point)
        Extensions:
            Subject Alternative Name (SAN):
                DNS: example.com
                DNS: www.example.com
                DNS: *.example.com
            Key Usage: Digital Signature
            Extended Key Usage: TLS Web Server Authentication
            Authority Information Access:
                OCSP: http://ocsp.letsencrypt.org
                CA Issuers: http://cert.letsencrypt.org/r3
            Certificate Policies:
                2.23.140.1.2.1 (Domain Validated)
            SCT (Signed Certificate Timestamp):
                [Certificate Transparency log entries]
    Signature Algorithm: SHA256withRSA
    Signature: 3a:4b:5c:...
```

**Key fields:**
- **Subject/SAN**: Which domains this certificate is valid for. Modern
  certificates use the SAN extension (Subject Alternative Name), not the
  Subject CN. A single certificate can cover multiple domains and wildcards.
- **Validity period**: Let's Encrypt issues 90-day certificates (encouraging
  automation). Traditional CAs issue 1-year (previously up to 3 years, now
  capped at 398 days by browser policy, moving toward 90 days).
- **Public key**: The server's public key that the client uses to verify
  the CertificateVerify signature.
- **Signature**: The CA's signature over the certificate data. This is what
  makes the certificate trustworthy — the CA vouches for the binding between
  the domain name and the public key.

### The Chain of Trust

```
Root CA Certificate (self-signed)
  │  Stored in OS/browser trust store (~150 root CAs)
  │  Never sent by the server (client already has it)
  │  Key kept offline in HSMs
  │
  │  Signs ▼
  │
Intermediate CA Certificate
  │  Signed by the Root CA
  │  Server sends this in the Certificate message
  │  Key online but in HSMs with strict access controls
  │
  │  Signs ▼
  │
Leaf Certificate (server's certificate)
     Signed by the Intermediate CA
     Server sends this in the Certificate message
     Contains the server's public key and domain name
```

**Why intermediates?** The root CA's private key is incredibly valuable —
if compromised, an attacker can issue certificates for any domain. Root
keys are kept in air-gapped HSMs (Hardware Security Modules) and used
only to sign intermediate CA certificates (infrequent operation). The
intermediate CA does the day-to-day signing.

If an intermediate CA is compromised, only its certificates need to be
revoked — the root can issue a new intermediate. If a root is compromised,
the entire trust chain collapses and the root must be removed from all
trust stores worldwide (this has happened — DigiNotar, 2011).

### Certificate Validation

When the client receives the server's certificate chain, it must:

1. **Build the chain**: Link the leaf certificate to a trusted root
   through intermediates, using the Issuer/Subject fields.

2. **Verify signatures**: Each certificate's signature is verified
   using the signing certificate's public key. The root is self-signed
   and trusted because it's in the trust store.

3. **Check validity dates**: `Not Before ≤ now ≤ Not After`

4. **Check domain name**: The requested hostname (from SNI) must match
   the certificate's SAN entries. Wildcards (`*.example.com`) match
   one label only (`www.example.com` yes, `sub.www.example.com` no).

5. **Check revocation**: Is the certificate revoked?
   - **CRL (Certificate Revocation List)**: A list of revoked serial
     numbers published by the CA. Must download the entire list. Slow,
     doesn't scale.
   - **OCSP (Online Certificate Status Protocol)**: Real-time query to
     the CA. "Is certificate serial X still valid?" Fast but leaks
     browsing history to the CA (they see which sites you visit).
   - **OCSP Stapling**: The server queries OCSP itself and includes the
     signed response in the TLS handshake. Client gets freshness proof
     without contacting the CA. Stapled responses are signed by the CA
     and have a validity period (typically 7 days).
   - **CRLite** (Firefox): A compressed bloom filter of all revoked
     certificates, synced periodically. Local check, no network request,
     complete coverage.

6. **Check Certificate Transparency**: Modern browsers require SCTs
   (Signed Certificate Timestamps) proving the certificate was logged
   in a public CT log. This makes mis-issued certificates detectable.

### Certificate Transparency (CT)

CT is a system of public, append-only logs that record every certificate
issued by participating CAs. The insight: if all certificates are public,
mis-issuance is detectable.

```
CA issues cert for example.com
  │
  ├──> Submits to CT Log 1 ──> returns SCT (Signed Certificate Timestamp)
  ├──> Submits to CT Log 2 ──> returns SCT
  │
  ▼
Certificate includes SCTs as proof of logging

Domain owner monitors CT logs for example.com
  → Alert if an unexpected certificate appears
  → "Someone got a cert for my domain from a CA I didn't use!"
```

**Real catches:**
- Symantec issued unauthorized test certificates for google.com (2015).
  Caught by CT monitoring. Led to Symantec's removal from browser trust
  stores.
- Let's Encrypt certificates for phishing domains are visible in CT logs,
  enabling proactive takedowns.

**Browsers enforce CT**: Chrome requires 2-3 SCTs from independent logs
for Extended Validation certificates, and increasingly for all certificates.

---

## AEAD Encryption — How Data Is Actually Protected

After the handshake, every TLS record is encrypted using an AEAD
(Authenticated Encryption with Associated Data) cipher.

### AES-128-GCM (Most Common)

```
Input:
  Key:    16 bytes (from key schedule)
  Nonce:  12 bytes (8-byte sequence number XORed with 4-byte IV from key schedule)
  AAD:    TLS record header (content type, version, length)
  Plaintext: application data + content type byte (TLS 1.3 inner content type)

Output:
  Ciphertext: same length as plaintext
  Tag: 16 bytes (authentication tag)

TLS Record:
  [record header][nonce (implicit)][ciphertext][tag]
```

**GCM (Galois/Counter Mode)** combines CTR mode encryption with GHASH
authentication:

1. **CTR mode**: Generate a keystream by encrypting successive counter
   values (nonce || counter). XOR the keystream with plaintext. This is
   parallelizable — each block can be encrypted independently.

2. **GHASH**: A polynomial hash over the ciphertext and AAD using
   multiplication in GF(2^128). Produces the authentication tag. Any
   modification to the ciphertext or AAD changes the tag.

**Nonce uniqueness is critical.** Reusing a nonce with the same key
in GCM is catastrophic — it reveals the XOR of two plaintexts AND
allows the attacker to forge authentication tags (recover the GHASH
key). TLS 1.3 prevents this by using a per-record sequence number
(starting at 0, incrementing by 1) XORed with a random IV. The sequence
number is never reused because TLS connections are sequential.

### ChaCha20-Poly1305

```
ChaCha20: Stream cipher. 256-bit key, 96-bit nonce, 32-bit counter.
          20 rounds of quarter-round operations on a 4×4 state matrix.
          Pure ARX (Add, Rotate, XOR) — no S-boxes, no lookup tables.
          → Immune to cache timing attacks (constant-time by design).

Poly1305: One-time authenticator. Takes a 256-bit one-time key (derived
          from ChaCha20's first block) and computes a 128-bit tag.
          Based on polynomial evaluation modulo 2^130 - 5.
```

**Why ChaCha20 exists alongside AES**: AES-GCM is fast on CPUs with
AES-NI hardware instructions (~1 cycle/byte). Without AES-NI (older
CPUs, many ARM mobile chips), AES-GCM drops to ~10-15 cycles/byte.
ChaCha20-Poly1305 runs at ~5 cycles/byte in software regardless of
hardware support. Google deployed ChaCha20 specifically for Android
devices where AES was a battery drain.

---

## The Key Exchange — Elliptic Curve Diffie-Hellman

### The Problem

Two parties need to agree on a shared secret over a public channel.
Anyone watching can see everything they exchange. How do they end up
with a secret that the observer doesn't know?

### Diffie-Hellman Over Elliptic Curves (ECDHE)

An elliptic curve is a set of points (x, y) satisfying:

```
y² = x³ + ax + b    (over a finite field F_p, where p is a large prime)
```

The key operation is **point multiplication**: given a point G (the
generator) and a scalar n, compute n × G by repeated point addition.
This is efficient. But given G and n × G, recovering n (the **elliptic
curve discrete logarithm problem**) is computationally infeasible for
sufficiently large curves.

```
Curve: x25519 (Curve25519, by Daniel Bernstein)
       y² = x³ + 486662x² + x   (mod 2^255 - 19)
       ~128-bit security level
       32-byte keys (compact)

Client:
  a = random 32-byte scalar (private key)
  A = a × G (public key)      → sent in ClientHello key_share

Server:
  b = random 32-byte scalar (private key)
  B = b × G (public key)      → sent in ServerHello key_share

Shared secret:
  Client computes: a × B = a × (b × G) = ab × G
  Server computes: b × A = b × (a × G) = ab × G
  → Same point. The x-coordinate is the shared secret.

Attacker sees: A and B (public keys)
  Needs to compute: ab × G
  Has: a × G and b × G
  → Cannot recover a or b (ECDLP is hard)
  → Cannot compute ab × G from a × G and b × G (CDH assumption)
```

**Why x25519?** Designed to be misuse-resistant:
- Every 32-byte string is a valid private key (no validation needed)
- Constant-time implementation is natural (no branching on secret data)
- Immune to timing attacks, invalid curve attacks
- Fast: ~125,000 operations/second on modern CPUs

**secp256r1 (P-256)**: NIST's curve. Widely deployed (older TLS stacks,
hardware tokens). 32-byte scalars, 64-byte uncompressed public keys.
Correct implementation is harder than x25519 (must validate points,
careful about side channels). Some distrust its NIST origin (concerns
about potential backdoors, though no evidence exists).

---

## Common TLS Attacks and Failures

### Downgrade Attacks

An active attacker (man-in-the-middle) modifies the ClientHello to
remove TLS 1.3 support, forcing a fallback to TLS 1.2 (or worse,
SSL 3.0) where weaker cipher suites are available.

**TLS 1.3 defense**: The server's Finished message includes a MAC
over the entire handshake transcript, including the ClientHello. If
the attacker modified the ClientHello, the MAC won't verify and the
connection fails. Additionally, TLS 1.3 defines a special sentinel
value in the ServerHello random field (`44 4F 57 4E 47 52 44 01` =
"DOWNGRD\x01") that the client checks if it receives a TLS 1.2
ServerHello despite offering 1.3. The FREAK and Logjam attacks
exploited TLS 1.2's vulnerability to downgrade.

### Heartbleed (CVE-2014-0160)

Not a TLS protocol flaw — a buffer over-read bug in OpenSSL's
implementation of the TLS Heartbeat extension.

```
Heartbeat request:
  "Echo back 'hello' (5 bytes)"
  → Server echoes "hello" ✓

Malicious request:
  "Echo back 'hello' (65535 bytes)"
  → Server reads 5 bytes of "hello" + 65530 bytes of adjacent memory
  → Adjacent memory contains: private keys, session tokens, user
    passwords, other users' requests — whatever's in the process heap
```

OpenSSL trusted the client-specified length without checking it against
the actual payload size. A 1-line bounds check fix. But the damage:
estimated 17% of TLS-enabled servers were vulnerable, including major
sites. Private keys had to be assumed compromised — mass certificate
reissuance.

### Certificate Misissuance

The trust model's weakness: any of ~150 root CAs can issue a certificate
for any domain. A compromised or malicious CA can issue a valid certificate
for google.com:

- **DigiNotar (2011)**: Iranian hackers compromised DigiNotar and issued
  fraudulent certificates for google.com, used to intercept Gmail traffic
  of Iranian users. DigiNotar went bankrupt.
- **CNNIC (2015)**: A CNNIC subsidiary issued unauthorized certificates.
  Google and Mozilla removed CNNIC from their trust stores.

**Defenses:**
- Certificate Transparency (makes misissuance publicly visible)
- CAA records (CAs must check before issuing)
- Certificate pinning (HPKP — deprecated because it was too dangerous;
  a configuration mistake could brick a domain for months)
- Shorter certificate lifetimes (limits exposure window)

---

## TLS in Practice

### Where TLS Termination Happens

```
Client ──TLS──> Load Balancer/CDN ──plaintext──> Application Server
                (terminates TLS)                 (never sees TLS)
```

Most production deployments terminate TLS at the edge (nginx, HAProxy,
Cloudflare, AWS ALB). Benefits:
- Centralized certificate management
- Hardware acceleration (AES-NI on the load balancer)
- The application server doesn't deal with TLS complexity
- HTTP/2 negotiation happens at the edge

The internal link (load balancer → app server) is often plaintext within
a trusted network. Zero trust architectures use mTLS (mutual TLS) even
internally — both sides present certificates, proving identity in both
directions.

### mTLS (Mutual TLS)

Standard TLS authenticates only the server. mTLS authenticates both:

```
Standard TLS:    Client verifies server's certificate
Mutual TLS:      Client verifies server's certificate
                 Server verifies client's certificate
```

Used for:
- Service-to-service authentication in microservices (Istio, Linkerd)
- API authentication (client certificate instead of API key)
- Zero trust networks (every connection is authenticated, even internal)

The server sends a CertificateRequest during the handshake, and the
client responds with its own Certificate and CertificateVerify.

### TLS Performance

**Handshake cost**: The dominant cost is the public-key cryptography
during the handshake (ECDHE + signature verification). Application data
encryption (AES-GCM with AES-NI) is ~1 cycle/byte — negligible.

```
Operation                    Time (modern hardware)
ECDHE x25519 key exchange    ~0.05ms per side
RSA-2048 signature verify    ~0.03ms
ECDSA P-256 signature verify ~0.08ms
AES-128-GCM encrypt 1 KB     ~0.001ms
Full TLS 1.3 handshake        ~1ms CPU + 1 RTT network
```

**Session resumption** (PSK or tickets) avoids the certificate
verification, saving ~0.1ms of CPU per connection. At scale (millions
of new connections per second), this matters.

**Key rotation**: Application traffic keys can be updated mid-connection
with a KeyUpdate message. Both sides derive new keys from the existing
ones. This limits the amount of data encrypted under a single key
(reducing the impact of a key compromise to a window rather than the
entire connection).

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| TLS provides confidentiality + integrity + authentication | Wraps TCP transparently. Application sees plaintext |
| TLS 1.3 handshake is 1-RTT (vs 1.2's 2-RTT) | Client speculatively sends key shares in ClientHello |
| Everything after ServerHello is encrypted in 1.3 | Certificate is hidden from passive observers (unlike 1.2) |
| HKDF key schedule binds all keys to the transcript | Any handshake modification changes all derived keys |
| Forward secrecy is mandatory in TLS 1.3 | Ephemeral ECDHE only — no static RSA key exchange |
| 0-RTT saves a round trip but is replayable | Only use for idempotent requests. Not forward-secret |
| TLS 1.3 removed CBC, compression, renegotiation, static RSA | Each one had produced real-world attacks (POODLE, CRIME, etc.) |
| Only 5 cipher suites in TLS 1.3 | AES-GCM and ChaCha20-Poly1305 are the ones that matter |
| GCM nonce reuse is catastrophic | TLS prevents it with sequential record numbers XORed with random IV |
| x25519 is the preferred key exchange | Misuse-resistant, constant-time, fast, compact |
| Certificate chain: leaf → intermediate → root | Root offline in HSMs. Intermediate does day-to-day signing |
| CT logs make misissuance detectable | All certificates are public. Monitor for unauthorized issuance |
| OCSP stapling avoids privacy leak to CA | Server fetches OCSP response and staples it to the handshake |
| SNI leaks the destination domain (plaintext in ClientHello) | ECH (Encrypted Client Hello) is the solution, still being standardized |
| Heartbleed was implementation, not protocol | Bounds check failure in OpenSSL's Heartbeat extension |
| mTLS authenticates both sides | Standard for service mesh (Istio/Linkerd) and zero trust |
