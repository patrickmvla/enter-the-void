# DNS — The Internet's Naming System

## The Problem DNS Solves

Humans remember names. Machines use numbers. DNS (Domain Name System)
translates between the two — mapping human-readable domain names
(`example.com`) to IP addresses (`93.184.216.34`).

But DNS is far more than a phone book. It's a **globally distributed,
hierarchical, eventually-consistent database** that handles:

- Name-to-IP resolution (A/AAAA records)
- Mail routing (MX records)
- Service discovery (SRV records)
- Domain ownership verification (TXT records)
- Load balancing and failover (multiple records, TTLs)
- Content delivery network routing (GeoDNS, anycast)
- Security (DNSSEC, CAA, SPF, DKIM, DMARC via TXT)

DNS is queried before almost every internet connection. Your browser,
your API calls, your email — they all start with a DNS lookup. A DNS
outage takes down everything that depends on it (Dyn attack, 2016 —
took down Twitter, GitHub, Netflix, Reddit, all at once).

---

## The DNS Hierarchy

DNS is organized as a tree. Each node in the tree is a zone, and each
zone is authoritative for the names under it.

```
                        . (root)
                        │
        ┌───────────────┼───────────────┐
        │               │               │
       com             org             net
        │               │               │
    ┌───┼───┐       ┌───┘           ┌───┘
    │   │   │       │               │
 google  example  wikipedia      cloudflare
    │                               │
   www                             www
   mail
   maps
```

**The root zone** (`.`) is the top of the hierarchy. It knows where to
find the TLD (Top-Level Domain) servers. There are 13 root server
**identities** (a.root-servers.net through m.root-servers.net), but
these are anycast addresses — there are actually ~1,700+ physical root
server instances worldwide. Any packet sent to a root server IP reaches
the nearest instance.

**TLD servers** (`.com`, `.org`, `.net`, `.io`, etc.) know which
authoritative nameservers handle each domain under them. Verisign
operates `.com` and `.net`. The `.com` zone alone has ~160 million
registered domains.

**Authoritative nameservers** hold the actual DNS records for a domain.
When you buy `example.com` from a registrar, you point it at nameservers
(e.g., `ns1.cloudflare.com`) that host your zone file — the set of
records for your domain.

---

## The DNS Message Format

Every DNS query and response follows the same binary format defined in
RFC 1035. Understanding the wire format reveals how DNS actually works.

### Header (12 bytes, always present)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              ID               |QR| Opcode|AA|TC|RD|RA| Z|AD|CD| RCODE |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           QDCOUNT             |           ANCOUNT             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           NSCOUNT             |           ARCOUNT             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key fields:**

- **ID** (16 bits): Transaction ID. The client generates a random ID, the
  server echoes it in the response. This is how the client matches responses
  to queries. **Security critical**: If an attacker can guess the transaction
  ID, they can forge responses (Kaminsky attack — discussed below).

- **QR** (1 bit): 0 = query, 1 = response

- **AA** (1 bit): Authoritative Answer. Set when the responding server is
  the authority for the queried domain (not a cached answer).

- **TC** (1 bit): Truncation. Response was too large for UDP (>512 bytes
  traditionally, >~1232 bytes with EDNS0). Client should retry over TCP.

- **RD** (1 bit): Recursion Desired. Client asking the server to resolve
  recursively (follow the chain) rather than just returning what it knows.

- **RA** (1 bit): Recursion Available. Server supports recursive queries.

- **AD** (1 bit): Authenticated Data. DNSSEC validation succeeded.

- **CD** (1 bit): Checking Disabled. Client wants the answer even if DNSSEC
  validation fails.

- **RCODE** (4 bits): Response code.
  - 0 = NOERROR (success)
  - 2 = SERVFAIL (server failure)
  - 3 = NXDOMAIN (domain doesn't exist)
  - 5 = REFUSED (policy rejection)

- **QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT**: Number of entries in the Question,
  Answer, Authority, and Additional sections.

### Question Section

```
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     QNAME                        |  (variable length)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     QTYPE                        |  (16 bits)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     QCLASS                       |  (16 bits)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

**QNAME encoding**: Domain names are encoded as a sequence of labels, each
preceded by its length byte, terminated by a zero byte:

```
www.example.com →  03 77 77 77  07 65 78 61 6d 70 6c 65  03 63 6f 6d  00
                   ↑  w  w  w   ↑  e  x  a  m  p  l  e   ↑  c  o  m   ↑
                   3 bytes      7 bytes                    3 bytes      end
```

Each label can be up to 63 bytes (length fits in 6 bits — the top 2 bits
are reserved for **name compression**, a pointer to a previously seen name
in the message, reducing packet size for repeated domain suffixes).

**QTYPE**: What kind of record is being queried.

| Type | Value | Meaning |
|------|-------|---------|
| A | 1 | IPv4 address |
| AAAA | 28 | IPv6 address |
| CNAME | 5 | Canonical name (alias) |
| MX | 15 | Mail exchange |
| NS | 2 | Nameserver |
| TXT | 16 | Arbitrary text (SPF, DKIM, domain verification) |
| SRV | 33 | Service location (host + port) |
| SOA | 6 | Start of Authority (zone metadata) |
| PTR | 12 | Reverse DNS (IP → name) |
| CAA | 257 | Certificate Authority Authorization |

### Answer Section

Each answer is a **Resource Record (RR)**:

```
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     NAME                         |  (compressed or full)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     TYPE                         |  (16 bits)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     CLASS                        |  (16 bits, usually IN=1)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     TTL                          |  (32 bits, seconds)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     RDLENGTH                     |  (16 bits)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     RDATA                        |  (variable)
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

**TTL (Time To Live)**: How long this answer can be cached (in seconds).
A TTL of 300 means "cache this for 5 minutes." After that, re-query.
Low TTLs (30-60s) enable fast failover but increase DNS traffic. High
TTLs (3600-86400) reduce traffic but changes propagate slowly.

**RDATA**: The actual record data. For an A record, this is 4 bytes
(the IPv4 address). For AAAA, 16 bytes. For CNAME, a compressed domain
name. For MX, a 16-bit preference value followed by a domain name.

---

## The Resolution Process

When your application calls `getaddrinfo("example.com")`, here's the
full chain of events.

### Step 0: Local Cache and Hosts File

Before any network query:

1. **Application cache**: Browsers maintain their own DNS cache
   (Chrome: `chrome://net-internals/#dns`). Typically 1 minute.
2. **OS resolver cache**: `systemd-resolved` on Linux, `dscacheutil`
   on macOS. Honors TTLs from the DNS response.
3. **Hosts file**: `/etc/hosts` (Linux/macOS) or
   `C:\Windows\System32\drivers\etc\hosts`. Static mappings that
   override DNS. `127.0.0.1 localhost` lives here.

If any of these have the answer, no network query is made.

### Step 1: Stub Resolver → Recursive Resolver

Your OS sends the query to a **recursive resolver** (also called a
"caching resolver" or "full-service resolver"). This is the server
configured in `/etc/resolv.conf` or via DHCP — typically your ISP's
resolver, or a public one like:

- `8.8.8.8` / `8.8.4.4` (Google Public DNS)
- `1.1.1.1` / `1.0.0.1` (Cloudflare)
- `9.9.9.9` (Quad9 — blocks malicious domains)

The stub resolver is simple — it sends the query with the RD
(Recursion Desired) flag set and expects a complete answer back. All
the complexity is in the recursive resolver.

### Step 2: Recursive Resolution

If the recursive resolver doesn't have the answer cached, it walks
the DNS tree top-down:

```
Recursive Resolver                    Root Server (.)
  │                                     │
  │ "Where is example.com?"             │
  │ ───────────────────────────────────>│
  │                                     │
  │ "I don't know, but .com is at       │
  │  a.gtld-servers.net (192.5.6.30)"  │
  │ <───────────────────────────────────│
  │
  │                                   .com TLD Server
  │ "Where is example.com?"             │
  │ ───────────────────────────────────>│
  │                                     │
  │ "I don't know, but example.com's    │
  │  NS is ns1.example.com (93.x.x.x)" │
  │ <───────────────────────────────────│
  │
  │                                   Authoritative NS for example.com
  │ "What is example.com's A record?"   │
  │ ───────────────────────────────────>│
  │                                     │
  │ "example.com A 93.184.216.34        │
  │  TTL 3600"                          │
  │ <───────────────────────────────────│
  │
  │ (cache the answer for 3600 seconds)
  │
  │ Return 93.184.216.34 to client
```

**Three queries** to resolve a name cold (no cache). In practice, the
recursive resolver almost always has the root and TLD answers cached
(their TTLs are long — 48 hours for root, hours for TLD NS records).
So most resolutions require only 1-2 external queries.

### The Glue Record Problem

Notice something circular? The `.com` TLD server says "example.com's
nameserver is `ns1.example.com`." But to find `ns1.example.com`'s IP,
we'd need to query... `example.com`'s nameserver. Chicken-and-egg.

**Glue records** solve this. When a nameserver's name is within the zone
it serves, the parent zone includes the IP address directly in the
**Additional section** of the response:

```
Authority Section:
  example.com  NS  ns1.example.com

Additional Section (glue):
  ns1.example.com  A  93.184.216.34
```

The recursive resolver gets the NS name AND its IP in the same response.
No circular dependency. Glue records are required when nameservers are
"in-bailiwick" (within the zone they serve).

---

## DNS Record Types — Deep Dive

### A and AAAA Records

The fundamental records. Map names to IP addresses.

```
example.com.    300   IN   A      93.184.216.34
example.com.    300   IN   AAAA   2606:2800:220:1:248:1893:25c8:1946
```

Multiple A records for the same name = **round-robin DNS**. The resolver
returns all records, and the client picks one (often the first). This is
the simplest form of load balancing but has no health checking — if one
server dies, clients still get its IP until the TTL expires.

**Happy Eyeballs (RFC 8305)**: When both A and AAAA records exist, modern
clients race IPv4 and IPv6 connections simultaneously. Whichever connects
first wins. This avoids the latency penalty of trying IPv6 first and
falling back to IPv4 when it fails (which was the original behavior and
caused bad user experience on networks with broken IPv6).

### CNAME Records

An alias. "This name is actually that other name."

```
www.example.com.    300   IN   CNAME   example.com.
```

A query for `www.example.com` returns the CNAME, and the resolver must
then resolve `example.com` to get the actual IP. This adds an extra
resolution step.

**CNAME restrictions:**
- A CNAME cannot coexist with any other record type for the same name.
  This means you **cannot** put a CNAME at the zone apex (`example.com`
  itself) because the apex already has SOA and NS records. This is why
  many DNS providers offer proprietary "ALIAS" or "ANAME" records that
  flatten the CNAME at query time.
- CNAME chains (CNAME pointing to another CNAME) are allowed but
  discouraged — each link adds a resolution step.

### MX Records

Mail routing. When someone sends email to `user@example.com`, the
sender's mail server queries the MX records for `example.com`:

```
example.com.    300   IN   MX   10   mail1.example.com.
example.com.    300   IN   MX   20   mail2.example.com.
```

The number is **priority** (lower = preferred). The sender tries
`mail1.example.com` first (priority 10). If it's unreachable, fall
back to `mail2.example.com` (priority 20). Same priority = random
selection (basic load balancing).

### TXT Records

Arbitrary text. Originally intended for human-readable notes, TXT
records are now the workhorse of domain verification and email security:

```
example.com.  300  IN  TXT  "v=spf1 include:_spf.google.com ~all"
example.com.  300  IN  TXT  "google-site-verification=abc123..."
```

**SPF (Sender Policy Framework)**: Lists which servers are allowed to
send email for your domain. The receiving mail server checks if the
sending IP matches the SPF record. `~all` = soft fail (mark as spam),
`-all` = hard fail (reject).

**DKIM (DomainKeys Identified Mail)**: The sending server signs the
email with a private key. The public key is published as a TXT record
at `selector._domainkey.example.com`. The receiver fetches the key
and verifies the signature. This proves the email wasn't tampered with
in transit and came from an authorized sender.

**DMARC (Domain-based Message Authentication, Reporting & Conformance)**:
Policy that tells receivers what to do when SPF and DKIM both fail.
Published at `_dmarc.example.com`:

```
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

`p=reject` = reject emails that fail both SPF and DKIM. `rua` = send
aggregate reports to this address (so you can see who's spoofing your
domain).

**SPF + DKIM + DMARC** together form the email authentication triad.
Without all three, your domain is trivially spoofable.

### SRV Records

Service discovery. "Where is the XMPP server for this domain?"

```
_xmpp._tcp.example.com.  300  IN  SRV  10  5  5222  xmpp.example.com.
                                        ↑   ↑   ↑     ↑
                                      pri  wt  port  target
```

Format: `_service._protocol.domain`. Priority and weight allow
load balancing. Used by SIP, XMPP, LDAP, Minecraft, and Kubernetes
service discovery. Not used by HTTP (which has its own mechanisms).

### SOA Record

Start of Authority. Every zone has exactly one SOA record containing
zone metadata:

```
example.com.  86400  IN  SOA  ns1.example.com. admin.example.com. (
    2024010101  ; serial   — incremented on every zone change
    3600        ; refresh  — how often secondaries check for updates
    900         ; retry    — retry interval if refresh fails
    604800      ; expire   — secondaries stop serving if no refresh for this long
    86400       ; minimum  — negative cache TTL (NXDOMAIN caching)
)
```

The **serial number** is how secondary nameservers detect zone changes.
When a secondary's serial is lower than the primary's, it initiates a
**zone transfer** (AXFR for full, IXFR for incremental) to sync.

The **minimum field** was repurposed by RFC 2308 to mean the negative
cache TTL — how long resolvers cache NXDOMAIN (name doesn't exist)
responses. A high value means "if you queried this name and it didn't
exist, don't ask again for this many seconds."

### CAA Records

Certificate Authority Authorization. Specifies which CAs are allowed
to issue certificates for your domain:

```
example.com.  86400  IN  CAA  0  issue  "letsencrypt.org"
example.com.  86400  IN  CAA  0  issuewild  "letsencrypt.org"
example.com.  86400  IN  CAA  0  iodef  "mailto:security@example.com"
```

Before issuing a certificate, the CA must check CAA records and refuse
if it's not listed. `issue` controls regular certs, `issuewild` controls
wildcards, `iodef` specifies where to report violations.

---

## DNS Caching

Caching is what makes DNS performant. Without caching, every web page
load would require 3+ DNS round trips (root → TLD → authoritative)
adding 50-200ms of latency.

### Where Caching Happens

```
Browser Cache    →  OS Cache       →  Recursive Resolver Cache  →  Authoritative
(~1 minute)        (honors TTL)       (honors TTL)                 (source of truth)
```

Each layer caches answers for the TTL specified in the response.

### TTL Strategy

| TTL | Use Case | Tradeoff |
|-----|----------|----------|
| 30-60s | Active failover, blue-green deploys | High query volume, more latency |
| 300s (5 min) | General web services | Good balance |
| 3600s (1 hour) | Stable services | Low query volume, slow changes |
| 86400s (1 day) | Infrastructure records (NS, MX) | Minimal traffic, very slow changes |

**Pre-migration TTL lowering**: Before migrating a service to a new IP,
lower the TTL to 60s 24-48 hours in advance. This ensures all caches
have the low TTL by migration time, so the IP change propagates in ~60s
instead of hours. After migration is stable, raise the TTL back.

### Negative Caching (RFC 2308)

NXDOMAIN responses are cached too. If you query `nonexistent.example.com`
and get NXDOMAIN, that's cached for the SOA minimum TTL. This prevents
repeated queries for names that don't exist — important because typos and
misconfigured software generate a lot of NXDOMAIN traffic.

**The problem**: If you create a new subdomain and someone queried it
before it existed, their resolver has a negative cache entry. They won't
see the new record until the negative TTL expires. This is why SOA
minimum values shouldn't be too high.

---

## DNS Transport — UDP, TCP, and Beyond

### UDP: The Default

DNS traditionally uses UDP port 53. Queries are small (<512 bytes was
the original limit), responses are usually small, and the
request-response pattern doesn't need TCP's connection overhead. A DNS
query over UDP is a single round trip.

**The 512-byte limit** comes from the minimum reassembly buffer size
guaranteed by IPv4. Larger UDP packets might be fragmented, and DNS
over UDP was designed to avoid fragmentation. But modern responses
(DNSSEC signatures, large TXT records, many A records) often exceed
512 bytes.

### EDNS0 (Extension Mechanisms for DNS, RFC 6891)

EDNS0 extends the DNS protocol by adding an OPT pseudo-record in the
Additional section. The key extension: a **UDP payload size** field
that allows larger UDP responses (typically 1232 bytes — chosen to
avoid IPv6 fragmentation issues with the ~1280-byte minimum MTU).

```
; OPT record (not a real record — a protocol extension)
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 1232
```

The `do` flag means "DNSSEC OK" — the resolver supports DNSSEC and
wants signed responses.

### TCP: The Fallback

If the response is truncated (TC bit set), the client retries over TCP
port 53. TCP is also used for:
- Zone transfers (AXFR/IXFR) between primary and secondary nameservers
- DNSSEC responses that exceed even EDNS0 UDP limits
- Connections where UDP is blocked by firewalls

**RFC 7766** mandated that all DNS servers MUST support TCP. It's not
optional.

### DNS-over-HTTPS (DoH, RFC 8484)

DNS queries sent as HTTPS requests to a resolver:

```http
GET /dns-query?dns=AAABAAAB... HTTP/2
Host: dns.cloudflare.com
Accept: application/dns-message
```

Or POST with the binary DNS message as the body.

**Why it exists**: Traditional DNS is unencrypted. Anyone on the network
path (ISP, coffee shop WiFi, corporate firewall) can see which domains
you're resolving. DoH encrypts DNS queries inside HTTPS, making them
indistinguishable from regular web traffic.

**The controversy**: ISPs and enterprises lose visibility into DNS
traffic. They can't block malicious domains via DNS anymore (their
DNS filters are bypassed). Network administrators can't enforce
security policies. Privacy advocates say this is a feature, not a bug.
Network operators say it breaks legitimate security tools.

### DNS-over-TLS (DoT, RFC 7858)

DNS over a TLS connection on port 853. Simpler than DoH (no HTTP
framing), but easily blockable by firewalls (they can see traffic
on port 853 and block it). DoH on port 443 blends in with HTTPS
traffic.

### DNS-over-QUIC (DoQ, RFC 9250)

DNS over QUIC on port 853. Combines the privacy of encryption with
QUIC's 0-RTT connection establishment. A DNS query can potentially
be answered in a single round trip including the handshake (vs DoT's
TCP+TLS handshake overhead). Still relatively new and not widely deployed.

---

## DNSSEC — Authenticating DNS Responses

### The Problem: DNS Spoofing

Standard DNS has no authentication. When you receive a DNS response,
you trust it based on... the transaction ID (16 bits) and the source
port matching. That's it. An attacker who can:

1. Observe your DNS query (to learn the transaction ID), OR
2. Predict the transaction ID (if it's sequential or poorly randomized)

...can send a forged response that arrives before the legitimate one.
Your resolver accepts the forged answer and caches it — now everyone
using that resolver gets sent to the attacker's IP. This is **DNS
cache poisoning**.

### The Kaminsky Attack (2008)

Dan Kaminsky discovered that the 16-bit transaction ID provided far
less protection than anyone assumed.

**The classic attack** required the attacker to race the legitimate
response for a specific query. If they lost the race, they had to wait
for the TTL to expire before trying again (the cached legitimate answer
blocks the forged one).

**Kaminsky's insight**: Don't attack the target name directly. Query
random nonexistent subdomains:

```
Attacker queries: random1234.example.com
→ Resolver doesn't have it cached (it doesn't exist), so it queries
  the authoritative server
→ Attacker floods forged responses for random1234.example.com, each
  with a different guessed transaction ID
→ If the forged response also includes a poisoned authority section:
    "example.com NS ns1.evil.com"
  ...then the resolver caches the DELEGATION — all future queries for
  anything under example.com go to the attacker's nameserver.
```

Key: each random subdomain generates a fresh query with a new
transaction ID to guess. The attacker gets unlimited attempts per second
(no TTL to wait for). With 16-bit transaction IDs, the expected
number of attempts is ~32,768 — trivial at network speeds.

**The mitigation**: Source port randomization (Hubert-Vixie, RFC 5452).
Randomize the UDP source port of queries, adding ~16 bits of entropy
on top of the transaction ID. Now the attacker must guess both the
transaction ID (16 bits) and the source port (~16 bits) = ~32 bits of
entropy = ~2 billion guesses. This makes the attack impractical (but
not impossible — NAT devices that de-randomize ports re-introduce the
vulnerability).

**The real fix**: DNSSEC.

### How DNSSEC Works

DNSSEC adds **digital signatures** to DNS records. The authoritative
server signs its records with a private key, and resolvers verify the
signatures with the corresponding public key.

**The chain of trust:**

```
Root Zone (.)
  │  Signs the root DNSKEY with the Root KSK (Key Signing Key)
  │  Root KSK fingerprint is hardcoded in resolvers (the "trust anchor")
  │
  │  Root zone contains DS record for .com
  │  (DS = hash of .com's KSK — "I vouch for .com's key")
  │
  ▼
.com Zone
  │  Signs .com records with .com's ZSK (Zone Signing Key)
  │  .com's KSK signs .com's ZSK (KSK→ZSK delegation)
  │
  │  .com zone contains DS record for example.com
  │
  ▼
example.com Zone
  │  Signs example.com records with its ZSK
  │  example.com's KSK signs its ZSK
  │
  ▼
example.com.  A  93.184.216.34  +  RRSIG (signature over the A record)
```

**Record types added by DNSSEC:**

| Record | Purpose |
|--------|---------|
| **RRSIG** | Signature over a set of records (e.g., the A records for a name) |
| **DNSKEY** | Public key for the zone (both KSK and ZSK) |
| **DS** | Delegation Signer — hash of child zone's KSK, stored in parent |
| **NSEC/NSEC3** | Authenticated denial of existence (proof that a name DOESN'T exist) |

**KSK vs ZSK**: Two keys per zone. The KSK (Key Signing Key) signs only
the DNSKEY records. The ZSK (Zone Signing Key) signs everything else.
This separation allows the ZSK to be rotated frequently (it's used
constantly) without changing the DS record in the parent zone (which
requires coordination with the parent). Only KSK rotation requires
updating the parent's DS record.

### NSEC and Authenticated Denial

DNSSEC must prove not just that records ARE authentic, but that
records that DON'T EXIST truly don't exist. Otherwise an attacker
could simply strip DNSSEC signatures and forge NXDOMAIN.

**NSEC (Next Secure)**: Lists the next name in the zone in canonical
order. If you query `b.example.com` and it doesn't exist, the server
returns:

```
a.example.com  NSEC  c.example.com  A AAAA RRSIG NSEC
```

"The next name after `a` is `c` — there's nothing between them, so `b`
doesn't exist." This is signed, so it can't be forged.

**Problem: zone walking**. An attacker can enumerate every name in the
zone by following the NSEC chain: query `a`, get NSEC pointing to `c`,
query `c`, get NSEC pointing to `d`, etc. This reveals the entire zone
contents.

**NSEC3 (RFC 5155)**: Hashes the names before ordering them. Instead
of `a.example.com NSEC c.example.com`, it's
`H(a.example.com) NSEC3 H(c.example.com)`. An attacker sees hashes,
not names. They'd need to brute-force the hash to recover names (and
NSEC3 uses a salt + iterations to slow this down). Not perfect — short
names and common subdomains like `www` are still easily crackable — but
it raises the bar significantly.

### DNSSEC Deployment Reality

As of 2025, ~30% of domains are DNSSEC-signed, but only ~25% of clients
validate. Major barriers:
- Operational complexity (key management, rotation, DS record coordination)
- Increased response sizes (signatures are large, ~100+ bytes per RRSIG)
- Risk of breaking resolution if signatures expire or keys are misconfigured
- Some argue DoH/DoT provide sufficient security for most use cases

---

## DNS-Based Attacks

### DDoS Amplification

DNS is a powerful amplification vector. A small query can produce a
much larger response:

```
Query:    ~60 bytes   "What are ALL the records for example.com?" (type ANY)
Response: ~3000 bytes  (all records + DNSSEC signatures)

Amplification factor: ~50x
```

The attacker spoofs the source IP of the query (using the victim's IP).
The DNS server sends the large response to the victim. With thousands
of open resolvers as amplifiers, a modest botnet can generate enormous
traffic volumes.

**Mitigations:**
- Rate limiting on open resolvers
- BCP 38 (network ingress filtering) to prevent IP spoofing
- Response Rate Limiting (RRL) on authoritative servers
- Disabling `ANY` queries (Cloudflare returns empty, RFC 8482)

### DNS Rebinding

Bypass same-origin policy in browsers by manipulating DNS TTLs:

```
1. Victim visits attacker.com
2. DNS for attacker.com returns attacker's IP (1.2.3.4), TTL=0
3. JavaScript on attacker.com makes XHR to attacker.com
4. DNS TTL has expired — browser re-resolves attacker.com
5. DNS now returns 192.168.1.1 (victim's internal router)
6. Browser sends request to 192.168.1.1 with Origin: attacker.com
7. The router doesn't check Origin headers — serves internal pages
8. JavaScript reads the response (same-origin: attacker.com)
```

The attacker's JavaScript can now scan and interact with the victim's
internal network. Defenses: DNS rebinding protection in resolvers
(block private IPs in public DNS responses), `Host` header validation
on internal services.

### DNS Tunneling

DNS queries can carry arbitrary data in the subdomain label:

```
base64encodeddata.tunnel.attacker.com  TXT
```

The attacker runs an authoritative nameserver for `tunnel.attacker.com`.
The encoded data in the subdomain is the payload. The TXT response
carries data back. This creates a covert, low-bandwidth communication
channel that bypasses most firewalls (DNS is almost never blocked).

Used by malware for command-and-control, data exfiltration through
networks that block all outbound traffic except DNS. Detection:
unusually long subdomain labels, high query volume, entropy analysis.

---

## DNS in Practice — What Developers Need to Know

### DNS and Application Latency

DNS resolution adds latency to every new connection. Strategies to
minimize impact:

1. **Connection reuse** (HTTP keep-alive, connection pooling):
   DNS is only queried once per connection, not once per request.
2. **DNS prefetching**: `<link rel="dns-prefetch" href="//cdn.example.com">`
   Resolves the domain before the browser needs it.
3. **Minimize domains**: Each unique domain requires a separate DNS
   resolution. CDN, API, analytics, fonts — each adds latency on first
   request.
4. **Use long TTLs** for stable services. Short TTLs mean more frequent
   resolution.

### The "It's Always DNS" Problem

DNS is the root cause of an outsized proportion of outages because:
- It's the first thing that happens and everything depends on it
- TTL caching means changes propagate unpredictably
- Negative caching means deleted records persist
- Different resolvers have different caching behavior
- Registrar, nameserver, and zone file are three separate systems that
  can each fail independently

**Real outages:**
- **Dyn (2016)**: Mirai botnet DDoS against Dyn's DNS infrastructure
  took down Twitter, GitHub, Netflix, Reddit, Spotify — all because
  they used Dyn as their DNS provider. Single point of failure.
- **Facebook (2021)**: BGP withdrawal made Facebook's authoritative DNS
  servers unreachable. Facebook, Instagram, WhatsApp down for ~6 hours.
  Internal tools also used Facebook DNS, so engineers couldn't even
  access systems to fix the problem.
- **Cloudflare (2022)**: Registrar disabled cloudflare.com domain
  briefly, causing resolution failures.

### Debugging DNS

```bash
# Basic resolution
dig example.com A
dig example.com AAAA
dig example.com MX

# Trace the full resolution chain (root → TLD → authoritative)
dig +trace example.com

# Query a specific nameserver
dig @8.8.8.8 example.com A

# See all records
dig example.com ANY

# Check DNSSEC
dig +dnssec example.com A

# Reverse DNS
dig -x 93.184.216.34

# Short answer only
dig +short example.com A

# Check TTL (query twice, watch TTL decrease)
dig example.com A | grep -A1 "ANSWER SECTION"
sleep 10
dig example.com A | grep -A1 "ANSWER SECTION"

# Check authoritative nameservers
dig example.com NS

# Check SOA (zone metadata)
dig example.com SOA
```

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| DNS is a hierarchical, distributed database | Root → TLD → Authoritative. ~1700 root instances via anycast |
| DNS messages are 12-byte header + sections | ID field is security-critical (Kaminsky). TC bit triggers TCP fallback |
| QNAME encoding: length-prefixed labels | `www.example.com` = `03 77 77 77 07 65 78 61 6d 70 6c 65 03 63 6f 6d 00` |
| Recursive resolver does the heavy lifting | Walks the tree, caches answers. Stub resolver just asks and waits |
| Glue records break circular dependencies | Parent zone includes IP for in-bailiwick nameservers |
| TTL controls cache duration and propagation speed | Lower TTL before migrations. Negative caching traps exist |
| CNAME can't coexist with other types at same name | Zone apex can't be CNAME — use ALIAS/ANAME for CDN at apex |
| SPF + DKIM + DMARC = email authentication | All three via TXT records. Without them, your domain is spoofable |
| DNSSEC signs the chain: root → TLD → zone | KSK signs DNSKEY, ZSK signs everything else. DS record links parent to child |
| Kaminsky attack: unlimited cache poisoning attempts | Mitigated by source port randomization. Solved by DNSSEC |
| NSEC3 prevents zone enumeration via hashed names | NSEC chains leak all names. NSEC3 hashes them (imperfect but raises the bar) |
| DNS amplification: 50x factor for DDoS | Small query → large response. Source IP spoofing + open resolvers |
| DNS rebinding bypasses same-origin policy | TTL=0 → re-resolve to internal IP → JS reads internal resources |
| DoH/DoT encrypt DNS queries | DoH blends with HTTPS traffic. DoT on port 853 is blockable |
| "It's always DNS" | DNS is the first thing and everything depends on it. Dyn, Facebook, Cloudflare outages prove it |
