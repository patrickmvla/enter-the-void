# Password Hashing — From First Principles

## The Core Problem

You need to verify identity through a shared secret (password), but you can't
store the secret. If your storage is compromised, every identity is compromised.
Encryption doesn't solve this — it's reversible, and the key must be accessible
to your application, which means it's accessible to an attacker who owns your
application server.

You need a **one-way function**: easy to compute forward, infeasible to reverse.

---

## What Makes a Hash Function One-Way

A cryptographic hash function has three properties that matter here:

### 1. Preimage Resistance
Given a hash output `h`, it's computationally infeasible to find any input `m`
such that `hash(m) = h`. "Computationally infeasible" means it would take
longer than the heat death of the universe with current hardware.

This is what makes it "one-way." The function destroys information during
computation — multiple inputs map to the same output (pigeonhole principle),
and the internal state transformations are designed so you can't work backward
through them.

### 2. Second Preimage Resistance
Given an input `m1`, it's infeasible to find a different input `m2` where
`hash(m1) = hash(m2)`. You can't find a different password that produces the
same hash.

### 3. Collision Resistance
It's infeasible to find *any* two inputs `m1` and `m2` where
`hash(m1) = hash(m2)`. This is stronger than second preimage resistance —
the attacker gets to choose both inputs.

**For password hashing, preimage resistance is what matters most.** The attacker
has the hash and wants to find the password.

### How the One-Way Property Works (Simplified)

SHA-256 as an example:

1. The input is padded and split into 512-bit blocks
2. Each block goes through 64 rounds of operations
3. Each round applies: bitwise rotations, XOR, AND, modular addition
4. The output of each round feeds into the next
5. The final 256-bit state is the hash

The key insight: **modular addition and bitwise operations are lossy**. When you
add `a + b mod 2^32 = c`, knowing `c` doesn't tell you `a` and `b` — there
are 2^32 possible pairs. After 64 rounds of these lossy operations composed
together, the relationship between input and output is astronomically complex.
There's no mathematical shortcut to invert it.

---

## Why Fast Hash Functions Fail for Passwords

SHA-256 was designed for **data integrity** — verifying file downloads,
checksumming network packets, building Merkle trees. Speed is a feature there.

The numbers on modern hardware (2024):

| Hardware | MD5 | SHA-256 | bcrypt (cost=10) |
|----------|-----|---------|-------------------|
| Single CPU core | ~500M/s | ~200M/s | ~10/s |
| Modern GPU (RTX 4090) | ~60B/s | ~8B/s | ~150K/s |
| Dedicated ASIC | ~200B/s | ~30B/s | N/A (memory-bound) |

The average human password has ~30-40 bits of entropy (dictionary words,
predictable substitutions). That's roughly 1-2 billion possibilities.

**Time to exhaust the keyspace of a typical password:**

| Algorithm | GPU time |
|-----------|----------|
| MD5 | < 1 second |
| SHA-256 | < 1 second |
| bcrypt (cost=10) | ~2 hours |
| bcrypt (cost=14) | ~30 hours |
| Argon2id (64MB, t=3) | weeks+ (memory bottleneck) |

This is the entire argument for slow hashing. The only variable you control
is how expensive each guess is.

---

## Salting — Deeper Than "Random String Prepended"

### What a Salt Actually Is

A salt is a unique, random value generated per-password-hash operation. It's
stored alongside the hash in plaintext. It is **not secret**.

### What It Defeats

**Rainbow tables**: A rainbow table is a precomputed time-memory tradeoff.
Instead of storing every hash (too much storage), it stores chains:

```
password → hash → reduce → password' → hash' → reduce → password'' → ...
```

The "reduce" function maps a hash back to a password-space string (not the
original — just a new candidate). You store only the start and end of each
chain. To crack a hash, you apply the reduce function repeatedly and check if
you hit any chain endpoint. If you do, you replay that chain from its start
to find the input.

A rainbow table for 8-character alphanumeric passwords with SHA-256 is ~100GB
and takes days to generate. But once built, it cracks any unsalted hash in
seconds.

**With a 16-byte salt, the table would need to be built for each of the 2^128
possible salt values.** This is physically impossible. The table becomes
useless.

**Batch attacks**: Without salts, if 10,000 users have the hash
`5e884898da28...`, the attacker knows all 10,000 use "password". With salts,
each must be attacked individually.

### What It Does NOT Defeat

A targeted attack on a single user. The attacker has the salt (it's stored
in plaintext next to the hash). They prepend it and brute-force as usual.
Salting only removes shortcuts — it doesn't slow down the per-guess cost.

### Salt Requirements

- **Length**: At least 16 bytes (128 bits). Must be large enough that
  collisions are negligible across all users and all time.
- **Randomness**: Must come from a CSPRNG (cryptographically secure
  pseudorandom number generator), not `Math.random()`.
- **Uniqueness**: A new salt for every hash operation, including when a user
  changes their password. Never reuse salts.

---

## Where Randomness Comes From

When you call `crypto.randomBytes(16)` in Node.js, here's the full chain:

### Linux: `/dev/urandom`

1. **Hardware entropy sources**: CPU thermal noise, interrupt timing jitter,
   keyboard/mouse timing (on desktop), disk I/O timing, network packet
   arrival times.

2. **RDRAND/RDSEED**: Modern Intel/AMD CPUs have a hardware random number
   generator built into the chip. It uses thermal noise in silicon circuits.
   Linux mixes this into its pool but doesn't trust it exclusively (in case
   the CPU is compromised — see Intel ME concerns).

3. **Entropy pool**: The kernel maintains an internal entropy pool (ChaCha20-
   based CSPRNG since Linux 4.8). Hardware events are mixed in using a
   cryptographic hash. The pool is continually re-seeded.

4. **`/dev/urandom` vs `/dev/random`**: Historic distinction — `/dev/random`
   would block when the entropy estimate was low. Since Linux 5.6, they're
   effectively identical. Both use the same CSPRNG. The only blocking happens
   at boot before the pool is initialized.

5. **Node.js `crypto.randomBytes()`** → calls OpenSSL's `RAND_bytes()` →
   reads from `/dev/urandom` (or `getrandom()` syscall on modern kernels).

### Why `Math.random()` Is Not Safe

`Math.random()` uses xorshift128+ in V8. It's a fast PRNG but:
- The internal state is 128 bits and deterministic from the seed
- The seed is based on a weak source (often just timestamp)
- Given enough outputs, you can reconstruct the internal state and predict
  all future and past outputs
- It is not entropy-pooled, not re-seeded, not cryptographically secure

Using `Math.random()` for salts means an attacker who can observe some
outputs can predict all your salts.

---

## bcrypt Internals

bcrypt is built on the **Blowfish** block cipher, specifically its expensive
key schedule (the `eksblowfish` setup).

### Blowfish Key Schedule (Normal)

Blowfish has internal state: an array P (18 32-bit subkeys) and four S-boxes
(each 256 entries of 32 bits). The key schedule initializes these:

1. P-array initialized with hex digits of pi (nothing-up-my-sleeve constant)
2. XOR the key bytes into the P-array (cycling through the key)
3. Encrypt the all-zero block with the current state → output becomes P[0],P[1]
4. Encrypt that output again → becomes P[2],P[3]
5. Continue until all P entries and all S-box entries are set
6. This takes 521 encryptions total

### Expensive Key Schedule (eksblowfish) — What Makes bcrypt Slow

bcrypt modifies this process:

```
eksblowfish(cost, salt, key):
    // Phase 1: Standard-ish initialization
    P, S = InitState()
    P, S = ExpandKey(P, S, salt, key)

    // Phase 2: The expensive part
    repeat 2^cost times:
        P, S = ExpandKey(P, S, 0, key)    // re-key with password
        P, S = ExpandKey(P, S, 0, salt)   // re-key with salt

    return (P, S)
```

`ExpandKey` is the full Blowfish key schedule — 521 block encryptions. At
cost=10, this runs 2^10 = 1024 times, meaning 2 × 1024 × 521 = **1,067,008
Blowfish encryptions** just to set up the key.

After the key schedule, bcrypt encrypts the string `"OrpheanBeholderScryDoubt"`
64 times with the final state. The output is the hash.

### Why This Resists Hardware Acceleration

Each `ExpandKey` call modifies 4KB of S-box state that the next call reads.
The operations are sequential — iteration N depends on the output of iteration
N-1. You can't parallelize within a single hash computation. GPUs can run many
hashes in parallel, but each individual hash still takes the full time.

This is why the table above shows ~150K bcrypt/s on a top GPU vs ~8B SHA-256/s.
The per-hash cost is what matters.

---

## scrypt — Adding Memory Hardness

bcrypt's weakness: the 4KB state fits in L1 cache, and custom ASICs could
potentially integrate that small amount of memory. scrypt was designed by
Colin Percival (2009) specifically to require **large amounts of RAM**.

### How scrypt Works

1. **PBKDF2** generates an initial key from password + salt
2. **ROMix**: Fills a large array (N blocks, each 128×r bytes) sequentially,
   where each block depends on the previous one
3. Then accesses that array in a **pseudorandom order** determined by the
   data itself (data-dependent memory access pattern)
4. Final PBKDF2 derives the output

With N=2^14 and r=8, this requires ~16MB of RAM. The pseudorandom access
pattern means you can't predict which block you need next — you must keep
the entire array in memory.

**Time-memory tradeoff**: An attacker could use less memory by recomputing
blocks on demand, but this increases computation exponentially. Using half
the memory squares the computation time.

### The ASIC Problem

GPUs have limited memory per core (~48KB shared memory per SM on NVIDIA).
To run scrypt with 16MB per hash, you can only run a few instances per GPU,
destroying the parallelism advantage. ASICs would need to integrate large
amounts of expensive SRAM.

---

## Argon2 — Current State of the Art

Winner of the Password Hashing Competition (PHC, 2015). Designed with
lessons learned from bcrypt, scrypt, and attacks on both.

### Variants

- **Argon2d**: Data-dependent memory access. Maximum resistance to GPU/ASIC
  attacks. Vulnerable to side-channel attacks (cache timing) because the
  memory access pattern depends on the password.

- **Argon2i**: Data-independent memory access. Resistant to side-channel
  attacks. Slightly weaker against GPU/ASIC (because the access pattern is
  predictable, an attacker could optimize memory usage).

- **Argon2id**: Hybrid. First half of the first pass uses Argon2i (side-
  channel safe during the most vulnerable phase), rest uses Argon2d (GPU/ASIC
  resistant). **This is what you should use.**

### Parameters

- **m (memory)**: Memory cost in KB. Recommended: at least 64MB (65536 KB)
  for server-side, 256MB+ for offline key derivation.
- **t (iterations)**: Time cost. More iterations = more time even with fixed
  memory. Recommended: at least 3.
- **p (parallelism)**: Lanes of computation. Set to number of available cores.
  Note: this doesn't make it faster for attackers — it lets *you* use your
  server's cores efficiently.

### Internal Structure

Argon2 fills a 2D memory matrix (p lanes × m/p blocks per lane):

1. Each block is 1KB
2. Blocks are filled using the Blake2b hash function
3. Each block depends on a previous block in the same lane AND a block
   from a (possibly different) lane
4. In Argon2d, which previous block is chosen depends on the current block's
   content (data-dependent)
5. In Argon2i, the indices are generated independently from a counter

The cross-lane dependencies mean even the parallel lanes can't be computed
fully independently, creating a complex dependency graph that resists
optimization.

### Tuning Guide

The goal: make hashing take 0.5-1 second on your server under expected load.

1. Set p = number of CPU cores dedicated to hashing
2. Set m as high as your server can afford (memory per concurrent login ×
   max concurrent logins must fit in RAM)
3. Increase t until you hit your target time

For a server handling 100 concurrent logins with 32GB RAM:
- m = 64MB means 6.4GB for hashing — reasonable
- t = 3, p = 4 might give ~500ms on modern hardware

---

## PBKDF2 — Why It's Weaker (But Still Everywhere)

**Password-Based Key Derivation Function 2** (RFC 2898, year 2000).

```
PBKDF2(password, salt, iterations, keyLength, hashFunction):
    U1 = HMAC(password, salt || blockNumber)
    U2 = HMAC(password, U1)
    U3 = HMAC(password, U2)
    ...
    return U1 XOR U2 XOR U3 XOR ... XOR U_iterations
```

It just applies HMAC repeatedly. No memory hardness, no complex state. A GPU
that's fast at SHA-256 is equally fast at PBKDF2-SHA256 per iteration.

**Still used in**: LUKS disk encryption, WPA2 Wi-Fi (only 4096 iterations —
crackable), many legacy systems, Apple's Keychain, .NET's `Rfc2898DeriveBytes`.

With 600,000+ iterations (OWASP 2023 recommendation for SHA-256), it's
acceptable. But Argon2id at equivalent time cost provides much stronger
protection due to memory hardness.

---

## Timing Attacks — The Implementation Detail That Breaks Everything

### The Vulnerability

String comparison in most languages short-circuits:

```
function equals(a, b):
    for i in 0..length:
        if a[i] != b[i]:
            return false    // EXITS HERE — time reveals position
    return true
```

If the first byte differs: ~1 comparison, fast return.
If only the last byte differs: ~N comparisons, slower return.

The difference is nanoseconds. But over thousands of requests, statistical
analysis can detect it. An attacker compares response times for different
first characters until they find the one with slightly longer response time
— that means the first character matched. Then they move to the second
character. They reconstruct the hash byte by byte.

### Real-World Feasibility

- **Local network**: Demonstrated with ~1000 requests per character position
- **Over the internet**: Harder due to network jitter, but demonstrated on
  specific setups using statistical filtering (Crosby et al., 2009)
- **Most practical against APIs with consistent, low-latency responses**

### Constant-Time Comparison

```
function constantTimeEquals(a, b):
    if length(a) != length(b):
        return false
    result = 0
    for i in 0..length:
        result |= a[i] XOR b[i]   // accumulate differences
    return result == 0              // check ONCE at the end
```

Every byte is compared regardless. XOR produces 0 for matching bytes, non-zero
for mismatches. OR accumulates any non-zero into `result`. The loop runs to
completion every time. The final check is a single comparison.

**Important**: the length check at the top technically leaks length information,
but for password hashes the length is always fixed (e.g., always 60 chars for
bcrypt), so this isn't exploitable.

---

## Pepper — The Extra Layer

A pepper is a secret value mixed into the hash that's stored **outside the
database** — in an environment variable, HSM (Hardware Security Module), or
a separate secrets management system.

```
hash = argon2id(password + pepper, salt, params)
```

### Threat Model

If the attacker gets only the database (SQL injection, backup theft, leaked
dump), they have salts and hashes but NOT the pepper. They can't even begin
brute-forcing because every guess would need the pepper prepended.

If the attacker gets the application server too, they have the pepper, and
it provides no additional protection.

### Key-Based Approach (Better Than String Concatenation)

Instead of prepending a pepper, encrypt the hash output with a symmetric key:

```
hash = argon2id(password, salt, params)
stored = AES-256-GCM(hash, pepper_key)
```

Advantage: you can **rotate the pepper** without requiring users to re-enter
passwords. Decrypt with old key, re-encrypt with new key. With concatenation,
you'd need every user to log in again.

---

## Real Breaches — What Went Wrong

### RockYou (2009)
- **32 million passwords stored in plaintext**
- SQL injection exposed the full database
- No hashing at all
- This dataset became the standard wordlist for password cracking tools
- Proved that people use predictable passwords at massive scale

### LinkedIn (2012)
- **6.5 million SHA-1 hashes, unsalted**
- No salt meant identical passwords had identical hashes
- Within hours of the dump going public, 60% were cracked using rainbow
  tables and dictionary attacks
- The remaining 40% fell within days to GPU-accelerated brute force
- LinkedIn switched to bcrypt after the breach

### Adobe (2013)
- **153 million accounts**
- Used 3DES encryption (not hashing) in ECB mode
- ECB mode encrypts identical inputs to identical outputs — same as unsalted
  hashing in practice
- Password hints were stored in plaintext next to the encrypted passwords
- Attackers used the hints to guess passwords, then found all other accounts
  with the same encrypted blob
- The hint for the most common encrypted value included: "password",
  "pwd", "my password is password"

### Dropbox (2012, disclosed 2016)
- **68 million bcrypt hashes**
- Actually did it mostly right — bcrypt with cost=10
- First half of the dump used SHA-1 (migrated mid-way)
- The bcrypt hashes remain largely uncracked years later
- **This is what proper password hashing buys you** — even after a breach,
  the passwords are protected

### The Pattern

Every major password breach falls into one of:
1. Plaintext storage (RockYou) — instant compromise
2. Fast hash, no salt (LinkedIn) — cracked in hours
3. Encryption instead of hashing (Adobe) — cracked through patterns
4. Proper slow hash (Dropbox bcrypt) — attacker still can't extract passwords

---

## The Full Login Flow — Every Layer

```
CLIENT                          SERVER                          DATABASE
  │                               │                               │
  │ POST /login {email, pass}     │                               │
  │ ─────────────────────────────>│                               │
  │         (over TLS)            │                               │
  │                               │ SELECT hash FROM users        │
  │                               │   WHERE email = $1            │
  │                               │ ─────────────────────────────>│
  │                               │                               │
  │                               │ {hash: "$argon2id$v=19$..."}  │
  │                               │ <─────────────────────────────│
  │                               │                               │
  │                               │ argon2id.verify(              │
  │                               │   submitted_password,         │
  │                               │   stored_hash                 │
  │                               │ )                             │
  │                               │ // internally:                │
  │                               │ // 1. parse salt, params      │
  │                               │ //    from stored_hash        │
  │                               │ // 2. hash submitted_password │
  │                               │ //    with same salt & params │
  │                               │ // 3. constant-time compare   │
  │                               │                               │
  │                               │ if match:                     │
  │                               │   create session (next topic) │
  │  200 OK + session token       │                               │
  │ <─────────────────────────────│                               │
  │                               │                               │
  │                               │ if no match:                  │
  │  401 "invalid credentials"    │                               │
  │ <─────────────────────────────│                               │
  │  (never reveal WHICH          │                               │
  │   field was wrong)            │                               │
```

### Security Details in This Flow

1. **TLS** — the password travels encrypted on the wire. Without TLS, anyone
   on the network path can read it.

2. **Parameterized query** (`$1`) — prevents SQL injection. Never interpolate
   user input into SQL strings.

3. **Generic error message** — "invalid credentials" not "user not found" or
   "wrong password." Revealing which field failed tells an attacker whether
   an email is registered (account enumeration).

4. **Timing consistency** — even when the user doesn't exist, you should still
   run a dummy hash computation to prevent timing-based account enumeration.
   If "user not found" returns in 1ms but "wrong password" returns in 500ms
   (because it ran argon2), the attacker can distinguish them.

---

## Decision Matrix

| Situation | Use |
|-----------|-----|
| New application, full control | Argon2id (m=64MB, t=3, p=cores) |
| Can't use Argon2 (platform limitation) | bcrypt (cost=12+) |
| Legacy system, can't change hash function | PBKDF2-SHA256 (600,000+ iterations) |
| Key derivation from password (encryption) | Argon2id or scrypt |
| Never, under any circumstance | MD5, SHA-1, SHA-256 alone, any unsalted hash |

---

## Credential Stuffing — The Attack That Bypasses All Hashing

Password hashing protects against **database breaches**. It does nothing
against **credential stuffing** — where the attacker already has the
plaintext password from a different breach.

### How It Works

1. Attacker obtains a credential dump (email + plaintext password pairs)
   from a breached service (RockYou, Collection #1-5, etc.)
2. Attacker writes a bot that tries each email/password pair against your
   login endpoint
3. Because users reuse passwords, ~0.5-2% of attempts succeed
4. On a dump of 1 billion credentials, that's 5-20 million valid logins

Your Argon2id hashing is irrelevant — the attacker isn't cracking hashes.
They're submitting valid passwords through your front door.

### Defenses

- **Rate limiting**: Cap login attempts per IP and per account. Slowdown
  or lockout after N failures.
- **Credential screening**: On registration and password change, check the
  password against the Have I Been Pwned API (k-anonymity model — you send
  the first 5 chars of the SHA-1 hash, they return all matching hashes,
  you check locally). Reject known-breached passwords.
- **Bot detection**: CAPTCHA, device fingerprinting, behavioral analysis.
  Automated stuffing tools submit thousands of logins per minute from
  rotating IPs — detect the pattern, not just the IP.
- **Breached password notification**: Periodically check your users'
  password hashes against known breach databases and force password resets.

---

## Password Entropy — How Attackers Build Wordlists

Brute force doesn't mean trying every possible string. Real attackers use
**smart wordlists** ordered by probability.

### The Hierarchy of Attacks

```
1. Dictionary attack: Try common passwords
   "password", "123456", "qwerty", "letmein"
   → Cracks ~5% of any large database instantly

2. Dictionary + rules: Apply transformations
   "password" → "Password", "p@ssword", "password1", "Password!"
   Hashcat rules apply: capitalization, leet speak, appended numbers/symbols
   → Cracks ~30-40% with a good ruleset

3. Hybrid: Dictionary + brute force suffix
   "password" + 4-digit number → "password0000" through "password9999"
   → Catches the "just add numbers" pattern

4. Markov chain / probabilistic: Generate candidates based on character
   frequency and position statistics from known password dumps
   → "qaz2wsx" ranks higher than "xyzabc" because keyboard patterns
   are more common

5. Full brute force: Try every combination for a given length/charset
   → Last resort. 8 chars alphanumeric = 2.8 trillion combinations.
```

### Password Strength Estimation (zxcvbn)

Dropbox's zxcvbn library estimates password entropy by identifying patterns:

```
"Tr0ub4dor&3"
Patterns found:
  "troubador" → dictionary word (rank ~50,000) + l33t substitutions
  "&" → special char
  "3" → single digit

Estimated guesses: ~60,000 (dictionary rank × substitution variants)
Time to crack at 10K guesses/sec: ~6 seconds

vs.

"correct horse battery staple"
Patterns found:
  4 common dictionary words (ranks ~1000, ~2000, ~5000, ~20000)

Estimated guesses: ~2 × 10^16
Time to crack at 10K guesses/sec: ~63,000 years
```

The second password is longer, easier to remember, and exponentially harder
to crack. This is because password security comes from **unpredictability**
(entropy), not from complexity rules. A short password with special
characters is predictable (people substitute the same way). A long
passphrase of random words has genuine entropy.

**Why complexity rules fail**: Requiring uppercase + lowercase + number +
symbol doesn't increase security proportionally — it increases it by a
constant factor (maybe 10-100x) while annoying users into writing passwords
down. Four random words multiply the keyspace by ~10^16.

---

### Migration Strategy

If you inherit a system using weak hashing:

1. On next login, verify against old hash
2. If valid, re-hash with Argon2id and overwrite
3. Users who never log in again keep old hashes — set a deadline, force
   password reset after that date
4. Alternatively: wrap the old hash — `argon2id(old_sha256_hash, new_salt)`
   This upgrades security immediately without waiting for logins, though the
   inner SHA-256 still limits entropy to 256 bits (fine for passwords)
