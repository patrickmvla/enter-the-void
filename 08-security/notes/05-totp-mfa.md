# TOTP & Multi-Factor Authentication — From the Math Up

## Why Passwords Aren't Enough

A password is a single factor: **something you know**. If an attacker learns
it — phishing, breach, shoulder surfing, keylogger, social engineering — your
account is fully compromised. No amount of password complexity or hashing
sophistication changes this fundamental weakness.

Multi-factor authentication requires proving identity through **independent
categories** of evidence:

| Factor | Category | Examples |
|--------|----------|---------|
| Something you **know** | Knowledge | Password, PIN, security question |
| Something you **have** | Possession | Phone, hardware key, smart card |
| Something you **are** | Inherence | Fingerprint, face, iris, voice |

The factors must be from **different categories** to be true MFA. A password
+ security question is two knowledge factors — if the attacker has one
(phishing), they likely have both. A password + TOTP code is knowledge +
possession — the attacker needs your password AND physical access to your
device.

---

## HOTP — The Foundation (RFC 4226)

Before understanding TOTP, you must understand HOTP (HMAC-based One-Time
Password). TOTP is a specific application of HOTP where the counter is
derived from time.

### The Inputs

```
HOTP(K, C) → one-time password

K = shared secret (the key)
C = counter (an incrementing integer)
```

Both the server and the client know `K` and `C`. Each time a code is
generated, `C` increments.

### The Algorithm — Step by Step

**Step 1: HMAC-SHA1**

```
hmac_result = HMAC-SHA1(K, C)
```

`C` is an 8-byte big-endian integer. `K` is typically 20 bytes (160 bits,
matching SHA-1's output size). The HMAC output is 20 bytes (160 bits).

We covered HMAC internals in topic 03 — the double-hash with XOR padding
that prevents length extension. The same construction applies here, using
SHA-1 as the underlying hash.

**Why SHA-1?** The RFC was written in 2005. SHA-1's collision resistance
is broken (SHAttered attack, 2017), but HMAC-SHA1 is still secure. HMAC's
security depends on the hash function's **PRF (pseudorandom function)
property**, not its collision resistance. Finding an HMAC-SHA1 collision
requires a different (and much harder) attack than finding a SHA-1 collision.
The RFC specifies SHA-1, and no practical attacks exist against HMAC-SHA1.

**Step 2: Dynamic Truncation**

The 20-byte HMAC output is too large for a human to type. We need to extract
a manageable number from it — deterministically, so both sides compute the
same result.

```
// The last 4 bits of the HMAC determine the offset
offset = hmac_result[19] & 0x0F    // value 0-15

// Extract 4 bytes starting at that offset
binary_code = (hmac_result[offset]     & 0x7F) << 24   // mask high bit (sign)
            | (hmac_result[offset + 1] & 0xFF) << 16
            | (hmac_result[offset + 2] & 0xFF) << 8
            | (hmac_result[offset + 3] & 0xFF)
```

The `& 0x7F` on the first byte clears the sign bit so the result is always
a positive 31-bit integer (0 to 2^31 - 1).

**Why dynamic truncation instead of just taking the first 4 bytes?** If an
attacker can observe multiple OTPs, a fixed extraction point would give them
information about a predictable portion of the HMAC output. Dynamic
truncation, where the position depends on the HMAC output itself, distributes
the extraction across the full 20 bytes, making analysis harder.

**Step 3: Reduce to N digits**

```
otp = binary_code % 10^digits

// For 6 digits:
otp = binary_code % 1000000
```

If the result is less than 6 digits, left-pad with zeros: `42` → `000042`.

### Concrete Example

```
Secret (K): "12345678901234567890" (20 bytes ASCII, for illustration)
Counter (C): 0 (8 bytes: 0x0000000000000000)

1. HMAC-SHA1(K, C) = cc93cf18508d94934c64b65d8ba7667fb7cde4b0
                      (20 bytes in hex)

2. Last byte: 0xb0 → offset = 0xb0 & 0x0F = 0

3. Bytes at offset 0-3: cc 93 cf 18
   binary_code = (0xcc & 0x7F) << 24 | 0x93 << 16 | 0xcf << 8 | 0x18
               = 0x4c93cf18
               = 1284755224

4. 1284755224 % 1000000 = 755224

OTP = "755224"
```

Both the authenticator app and the server compute this independently. If
they get the same result, the user has proven they possess the shared secret.

### Counter Synchronization Problem

HOTP has a practical issue: the counter must stay synchronized.

If the user generates a code but doesn't submit it (opens the app, sees the
code, closes it), the client's counter advances but the server's doesn't.
After several skipped codes, the counters diverge and authentication fails.

**Solution: look-ahead window.** The server doesn't just check counter `C` —
it checks `C`, `C+1`, `C+2`, ... up to `C+s` (where `s` is a configurable
window, typically 5-10). If any match, it updates the server's counter to
that value.

This is functional but inelegant. The larger the window, the more vulnerable
to brute force (an attacker gets `s` chances per guess). This is one
motivation for TOTP.

---

## TOTP — Time-Based OTP (RFC 6238)

TOTP eliminates the counter synchronization problem by deriving the counter
from the current time.

### The Algorithm

```
TOTP = HOTP(K, T)

T = floor((current_unix_time - T0) / X)

T0 = Unix epoch (0) — the reference time
X  = time step in seconds (default: 30)
```

That's it. TOTP is literally HOTP where the counter `C` is replaced by the
number of time steps since epoch.

```
At Unix time 1708012800 (Feb 15, 2024 16:00:00 UTC):
T = floor(1708012800 / 30) = 56933760

TOTP = HOTP(K, 56933760)
```

30 seconds later, `T` increments to 56933761, and a new code is generated.
The authenticator app shows a countdown timer indicating when the current
code expires.

### Why 30 Seconds?

The time step balances two forces:

- **Shorter step (e.g., 10s)**: More secure (smaller window for an attacker
  to use a stolen code), but frustrating for users who can't type fast enough.
- **Longer step (e.g., 60s)**: More user-friendly, but a stolen code is
  valid for up to 60 seconds.

30 seconds is the consensus. It gives most users enough time to read and
type the code while limiting the replay window.

### Clock Drift

The server and client clocks must be reasonably synchronized. Unlike HOTP's
counter synchronization, time sync is handled by the operating system
(NTP on servers, network time on phones). But imperfect sync means the
server should accept codes from adjacent time steps.

```
Server checks:
  T-1  (previous 30-second window)
  T    (current window)
  T+1  (next window)
```

This means a code is effectively valid for ~90 seconds (the 30-second
window it was generated in, plus one window before and after). Some
implementations check T-2 through T+2, giving ~150 seconds. Wider windows
reduce lockouts from clock drift but increase the replay window.

**Tracking the last used timestamp prevents replay**: Store the most recent
successfully-used `T` value. Reject any code with a `T` value equal to or
less than the last used one. This prevents an attacker from reusing a code
within its validity window.

---

## The Shared Secret — Generation, Exchange, and Storage

### Generating the Secret

The shared secret must be generated by a CSPRNG. Typical length: 20 bytes
(160 bits) for SHA-1 based TOTP. The RFC recommends secrets at least as
long as the HMAC output (20 bytes for SHA-1, 32 for SHA-256, 64 for
SHA-512).

```javascript
const secret = crypto.randomBytes(20);
```

### Encoding: Base32

The secret needs to be transferable to an authenticator app. It's encoded
as Base32 (RFC 4648), not Base64 or hex.

```
Base32 alphabet: A-Z, 2-7  (32 characters)

Raw bytes:  48 65 6c 6c 6f  ("Hello")
Base32:     JBSWY3DP
```

**Why Base32?**
- Case-insensitive (A-Z only, no lowercase) — users can type it manually
  without case ambiguity
- No confusing characters (no 0/O or 1/I/l ambiguity, since 0 and 1 aren't
  in the alphabet)
- Easily represented in QR codes

### The `otpauth://` URI

The standard way to transfer the secret to an authenticator app:

```
otpauth://totp/MyApp:user@example.com?
    secret=JBSWY3DPEHPK3PXP&
    issuer=MyApp&
    algorithm=SHA1&
    digits=6&
    period=30
```

| Parameter | Meaning | Default |
|-----------|---------|---------|
| `secret` | Base32-encoded shared secret | (required) |
| `issuer` | Service name (shown in app) | (recommended) |
| `algorithm` | Hash function: SHA1, SHA256, SHA512 | SHA1 |
| `digits` | Number of digits in OTP | 6 |
| `period` | Time step in seconds | 30 |

The `totp` in `otpauth://totp/` distinguishes from `hotp`. The label
(`MyApp:user@example.com`) is display text in the app.

### QR Code Enrollment

The `otpauth://` URI is embedded in a QR code. The enrollment flow:

```
1. User enables 2FA on the website
2. Server generates secret: crypto.randomBytes(20)
3. Server stores the secret (encrypted) in the user's record
4. Server constructs the otpauth:// URI
5. Server renders a QR code containing the URI
6. User scans QR code with authenticator app
7. Authenticator app stores the secret locally
8. User enters the current TOTP code to verify enrollment
9. Server verifies the code — if correct, 2FA is active
```

**Step 8 is critical.** Without verification, you'd enable 2FA on an
account where the user may have failed to scan the QR code. They'd be
immediately locked out.

### Secret Storage on the Server

The server must store the shared secret to verify codes later. This is a
significant security concern — unlike password hashes, the secret must be
stored in a recoverable form (you need the actual bytes to compute HMAC).

**Options:**
- **Encrypted at rest**: AES-256-GCM with a key from an HSM or KMS. The
  most common approach.
- **HSM (Hardware Security Module)**: The secret never leaves the HSM.
  HMAC computation happens inside the hardware. Most secure, most expensive.
- **Plaintext in database**: Terrible. A database breach exposes all TOTP
  secrets — attacker can generate valid codes for every account.

**This is fundamentally different from passwords.** Password hashes are
one-way — a breach exposes hashes, not passwords. TOTP secrets must be
reversible — a breach exposes the actual secret, and the attacker can
generate valid codes.

This asymmetry is why hardware security keys (WebAuthn) are superior —
the server stores only a public key, not a shared secret.

### Secret Storage on the Client

Authenticator apps store secrets locally:

- **Google Authenticator**: Originally stored in app sandbox (no backup,
  no sync). Losing your phone meant losing all secrets. Now offers encrypted
  cloud backup (since 2023).
- **Authy**: Encrypted cloud backup from the start. The encryption key is
  derived from a "backups password" the user sets.
- **1Password, Bitwarden**: Store TOTP secrets alongside passwords. Convenient
  but arguably reduces to single-factor — if the password manager is
  compromised, the attacker has both the password and the TOTP secret.

---

## Why 6 Digits?

A 6-digit code has 1,000,000 possible values. An attacker guessing randomly
has a 1-in-1,000,000 chance per attempt. This sounds weak, but it's combined
with:

1. **Rate limiting**: Lock the account after 3-5 failed attempts
2. **Time window**: The code changes every 30 seconds — the attacker has
   limited time
3. **Second factor**: The attacker already needs the password. TOTP isn't
   the only defense, it's an additional layer.

### The Math of Brute Force

Without rate limiting:
```
1,000,000 possible codes / 30 seconds per window = ~33,333 guesses needed
at 1 guess per attempt, with 30 seconds per window, that's ~1000 minutes

But: if the server accepts T-1, T, T+1 (3 windows):
each guess has 3/1,000,000 chance = 1/333,333
```

With rate limiting (5 attempts per 30-second window):
```
5 guesses × (3/1,000,000) = 15/1,000,000 = 0.0015% chance per window
At one window per 30 seconds: ~22,222 windows = ~7.7 days average
But the account locks after 5 failures, so: game over.
```

**8-digit codes** (used by some services) give 100,000,000 possibilities.
Diminishing returns — the rate limiting is the real defense, not the code
length.

---

## Recovery Codes

When the user loses their authenticator device, they need a backup method.
Recovery codes are one-time-use codes generated at enrollment time.

### How They Work

```
1. At 2FA enrollment, server generates 8-10 recovery codes:
   ["a1b2c3d4e5", "f6g7h8i9j0", "k1l2m3n4o5", ...]

2. Each code: crypto.randomBytes(5).toString('hex') → 10 hex chars
   (40 bits of entropy per code)

3. Codes are displayed ONCE to the user: "Save these somewhere safe"

4. Server stores hashed versions (bcrypt or SHA-256 with per-code salt)
   Just like passwords — one-way hash, don't store plaintext

5. When user loses their device:
   - Enter recovery code instead of TOTP
   - Server hashes the input and compares against stored hashes
   - If match: mark that code as used (delete the hash), log the user in
   - Prompt user to set up new 2FA immediately
```

**Why hash them like passwords?** If the database is breached, the attacker
shouldn't be able to use recovery codes to bypass 2FA. Hashed recovery codes
require brute-forcing, which at 40 bits of entropy is feasible for a targeted
attack but not at scale across all users (each code has a unique salt).

Some services use longer codes (128+ bits) that are infeasible to brute
force even without rate limiting.

---

## What TOTP Does NOT Protect Against

Understanding the limitations is as important as understanding the mechanism.

### Real-Time Phishing (Man-in-the-Middle)

```
User              Attacker's Site           Real Server
 │                (looks like real site)        │
 │ enters email    │                            │
 │ + password      │                            │
 │────────────────>│                            │
 │                 │ forwards credentials       │
 │                 │───────────────────────────>│
 │                 │                            │
 │                 │ "enter your 2FA code"      │
 │                 │<───────────────────────────│
 │ "enter 2FA"    │                            │
 │<────────────────│                            │
 │                 │                            │
 │ enters TOTP     │                            │
 │────────────────>│                            │
 │                 │ forwards TOTP code         │
 │                 │───────────────────────────>│
 │                 │                            │
 │                 │ session token              │
 │                 │<───────────────────────────│
 │                 │                            │
 │  "logged in"   │ attacker now has session    │
 │<────────────────│                            │
```

The attacker proxies the entire login flow in real time. The TOTP code is
valid — the user typed it within the 30-second window. The attacker relays
it immediately. The server sees a valid password + valid TOTP = successful
login. The attacker gets the session token.

**Tools like Evilginx2 and Modlishka automate this.** They act as a reverse
proxy, serving a pixel-perfect copy of the target site, relaying all traffic
in real time. From the server's perspective, the login is legitimate.

**TOTP cannot prevent this.** The code is a shared secret that the user
types — if the user is tricked into typing it on the wrong site, it's
compromised. This is the fundamental limitation of all OTP-based 2FA.

### SIM Swapping (SMS 2FA)

SMS-based "2FA" sends codes via text message. An attacker:
1. Social-engineers the victim's phone carrier
2. Convinces them to transfer the victim's number to a new SIM
3. Receives all SMS messages, including 2FA codes
4. Logs in with stolen password + intercepted SMS code

This isn't an attack on TOTP — it's specific to SMS. But it's worth
understanding because many services still offer SMS as a 2FA option and
users treat it as equivalent to TOTP. It's not.

**SMS is the weakest second factor.** NIST SP 800-63B (2017) deprecated SMS
as a second factor due to SIM swapping and SS7 (Signaling System 7)
vulnerabilities. SS7, the protocol phone carriers use to route calls and
texts, has known flaws that allow interception of SMS messages with access
to the telecom network.

### Session Hijacking After Authentication

TOTP protects the **login** — it doesn't protect the **session** after login.
If an attacker steals the session cookie (XSS, malware, physical access),
they have a fully authenticated session. TOTP has already been verified and
is no longer involved.

**Defense**: Re-require TOTP for sensitive actions (password change, fund
transfer, API key generation). This is "step-up authentication" — elevating
the authentication level for high-risk operations.

---

## WebAuthn / FIDO2 — Why TOTP Is Being Superseded

### The Phishing Problem Solved

WebAuthn (Web Authentication API) uses public-key cryptography bound to the
**origin** (domain) of the website.

```
1. Registration:
   - Browser generates a key pair (private + public)
   - Private key stored on the device (hardware security key, TPM, phone)
   - Public key sent to the server
   - The key pair is bound to the specific origin (e.g., "github.com")

2. Authentication:
   - Server sends a random challenge
   - Browser asks the authenticator device to sign the challenge
   - Authenticator checks the origin: "Is this github.com?"
   - If the origin matches: signs the challenge with the private key
   - If the origin doesn't match (phishing site): refuses to sign
   - Server verifies the signature with the stored public key
```

**The origin check is performed by the authenticator, not the user.** A
phishing site at `github-login.com` will fail because the authenticator
only has a key registered for `github.com`. The user doesn't need to
notice the domain mismatch — the cryptography enforces it.

This is what makes WebAuthn **phishing-resistant** — a property TOTP
fundamentally cannot have.

### No Shared Secret

With TOTP, both the server and the client hold the same secret. A server
breach exposes the secret.

With WebAuthn, the server holds only the **public key**. A server breach
gives the attacker nothing useful — they can't sign challenges with a
public key. The private key never leaves the authenticator device.

### Types of Authenticators

- **Roaming authenticators**: Hardware security keys (YubiKey, Titan). USB
  or NFC. Dedicated hardware, most secure, but requires carrying a physical
  device.
- **Platform authenticators**: Built into the device. Touch ID, Face ID,
  Windows Hello, Android biometrics. Convenient, no extra hardware, but
  tied to one device.
- **Passkeys**: Platform authenticators that sync across devices via cloud
  (iCloud Keychain, Google Password Manager). Combines the convenience of
  platform authenticators with multi-device availability. The private key
  is synced encrypted — the cloud provider can't read it (in theory).

### Why TOTP Still Exists

- **Universal support**: Every service supports TOTP. WebAuthn adoption is
  growing but not universal.
- **No hardware required**: Any smartphone can be a TOTP authenticator.
  WebAuthn requires either a hardware key or a recent device with platform
  authenticator support.
- **Cross-platform**: TOTP works everywhere. Passkey sync is fragmented
  across ecosystems (Apple, Google, Microsoft).
- **Offline**: TOTP works without network connectivity. WebAuthn
  registration requires the server, but authentication can work offline
  if the relying party caches the public key.

**TOTP is "good enough" for most threat models.** WebAuthn is strictly
superior for phishing resistance, but TOTP + a strong password + rate
limiting protects against the vast majority of real-world attacks. The
attacks that bypass TOTP (real-time phishing proxies) are sophisticated
and targeted — most users won't face them.

---

## Implementation Checklist

What a correct TOTP implementation requires:

```
Secret generation:
  ✓ CSPRNG, 20+ bytes
  ✓ Stored encrypted (AES-256-GCM with KMS-managed key)
  ✓ Base32 encoded for transport

Enrollment:
  ✓ Generate secret
  ✓ Display QR code with otpauth:// URI
  ✓ Also display Base32 secret as text (for manual entry)
  ✓ Require the user to enter a valid code before activating 2FA
  ✓ Generate and display recovery codes
  ✓ Store recovery codes hashed (bcrypt or salted SHA-256)

Verification:
  ✓ Accept T-1, T, T+1 (configurable window)
  ✓ Store last used T value — reject codes with T ≤ last used
  ✓ Rate limit: lock after N failed attempts
  ✓ Constant-time comparison of code strings
  ✓ Log all 2FA events (success, failure, recovery code use)

Session management:
  ✓ Regenerate session after successful 2FA (topic 02)
  ✓ Mark session as 2FA-verified
  ✓ Require 2FA re-verification for sensitive actions (step-up)

Recovery:
  ✓ Recovery codes as alternative to TOTP
  ✓ One-time use — delete after successful use
  ✓ Option to regenerate recovery codes (invalidates old ones)
  ✓ Support for disabling 2FA (with re-authentication)
```

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| TOTP = HOTP with time-derived counter | `T = floor(unix_time / 30)`, fed into HMAC-SHA1 + dynamic truncation |
| Dynamic truncation | Last nibble selects offset, extract 4 bytes, mask sign bit, mod 10^digits |
| HMAC-SHA1 is still secure for TOTP | Collision attacks on SHA-1 don't affect HMAC's PRF security |
| The secret is stored reversibly | Unlike passwords — server needs the actual secret for HMAC. Encrypt it. |
| 30-second window with ±1 tolerance | Effectively ~90 seconds of validity. Track last used T to prevent replay |
| Rate limiting is the real defense | 6 digits (1M combinations) is enough when combined with lockout |
| TOTP does NOT prevent real-time phishing | Attacker proxies the code in real time. The code is valid. |
| SMS is not TOTP | SIM swapping and SS7 make SMS the weakest second factor |
| WebAuthn is cryptographically phishing-resistant | Origin-bound public-key auth. Server never holds the secret. |
| Recovery codes are passwords | Hash them. One-time use. Generated at enrollment. |
