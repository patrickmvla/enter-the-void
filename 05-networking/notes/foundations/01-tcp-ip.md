# TCP/IP — Reliable Delivery Over an Unreliable Network

## The Problem TCP Solves

IP (Internet Protocol) delivers packets between machines. But IP makes
**no guarantees**:

- Packets can be **lost** (router queue overflow, bit errors, congestion)
- Packets can arrive **out of order** (different routes through the network)
- Packets can be **duplicated** (retransmission by intermediate routers)
- Packets can be **corrupted** (bit flips in transit)
- There's no concept of a **connection** — each packet is independent

IP is a **best-effort** delivery service. It tries, but it doesn't promise.

TCP (Transmission Control Protocol) builds **reliable, ordered, error-checked,
connection-oriented** communication on top of this unreliable foundation.
It solves every problem IP doesn't.

---

## The IP Packet

Before TCP, understand what it rides on.

### IPv4 Header (20 bytes minimum)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key fields:**

- **Version** (4 bits): 4 for IPv4, 6 for IPv6
- **IHL** (4 bits): Internet Header Length in 32-bit words. Minimum 5
  (20 bytes), maximum 15 (60 bytes with options)
- **Total Length** (16 bits): Entire packet size including header and data.
  Maximum 65,535 bytes. In practice, limited by MTU (typically 1500 bytes
  on Ethernet)
- **TTL** (8 bits): Time to Live. Decremented by each router. When it
  reaches 0, the packet is dropped and an ICMP "Time Exceeded" is sent
  back. Prevents packets from looping forever. Default: 64 or 128.
  `traceroute` works by sending packets with TTL=1, 2, 3... and collecting
  the ICMP responses from each hop
- **Protocol** (8 bits): Which transport protocol. 6 = TCP, 17 = UDP,
  1 = ICMP
- **Source/Destination Address** (32 bits each): IPv4 addresses.
  2^32 = ~4.3 billion possible addresses — the reason we're running out
  and need IPv6 (128-bit addresses, 2^128 ≈ 3.4 × 10^38)

### IPv6 Header (40 bytes, fixed)

IPv6 was designed to fix IPv4's limitations. The header is paradoxically
**simpler** despite addressing a harder problem.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                         Source Address                         |
+                          (128 bits)                            +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                      Destination Address                      |
+                          (128 bits)                            +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**What changed from IPv4:**

- **No Header Checksum**: IPv4 routers recompute the checksum at every hop
  (because TTL changes). This is expensive. IPv6 eliminates it entirely —
  link-layer (Ethernet CRC) and transport-layer (TCP/UDP checksum) already
  catch corruption. Removing it speeds up router forwarding.

- **No Fragmentation by Routers**: IPv4 routers can fragment oversized packets.
  IPv6 routers don't — they send an ICMPv6 "Packet Too Big" message back to
  the sender, which must reduce its packet size. This moves fragmentation
  complexity to the endpoints (Path MTU Discovery) and keeps routers simple.

- **Fixed Header Size**: IPv4's IHL field exists because options make the header
  variable-length. IPv6 is always 40 bytes. Options are handled through
  **extension headers** — a chain of headers, each with a "Next Header" field
  pointing to the next one:

  ```
  IPv6 Header (Next Header: TCP)        → TCP Header → Data
  IPv6 Header (Next Header: Fragment)   → Fragment Header (Next Header: TCP) → TCP → Data
  IPv6 Header (Next Header: Hop-by-Hop) → Hop-by-Hop (Next Header: Routing) → Routing (Next Header: TCP) → TCP → Data
  ```

- **Flow Label** (20 bits): Identifies packets belonging to the same flow so
  routers can fast-path them without inspecting deeper headers. Designed for
  QoS but underused in practice.

- **128-bit Addresses**: 2^128 ≈ 3.4 × 10^38 addresses. Every grain of sand
  on Earth could have billions of addresses. Notation: eight groups of four hex
  digits (`2001:0db8:85a3:0000:0000:8a2e:0370:7334`), with leading zeros and
  consecutive zero-groups compressible (`2001:db8:85a3::8a2e:370:7334`).

**IPv6 Adoption Reality**: As of 2025, ~45% of Google's traffic is IPv6.
Most networks run **dual-stack** — both IPv4 and IPv6 simultaneously. The
transition has taken decades because NAT (Network Address Translation)
extended IPv4's life by allowing thousands of devices to share a single
public IP.

### IP Routing

Every router maintains a **routing table** — a mapping from IP prefixes
to next-hop addresses:

```
Destination        Next Hop        Interface
10.0.0.0/8         10.0.0.1        eth0
172.16.0.0/12      172.16.0.1      eth1
0.0.0.0/0          203.0.113.1     eth2      (default route)
```

When a packet arrives, the router finds the **longest prefix match** for
the destination IP. If 10.0.5.42 arrives, it matches both `10.0.0.0/8`
and the default route `0.0.0.0/0` — the /8 prefix is longer (more
specific), so it wins.

**BGP (Border Gateway Protocol)** is how routers across the internet
exchange routing information. Every ISP, cloud provider, and large
organization runs BGP to announce which IP prefixes they own and which
paths exist to reach them. The global BGP routing table has ~1 million
entries. A BGP misconfiguration can take down large portions of the
internet — Facebook's 2021 outage was caused by a BGP withdrawal that
made Facebook's DNS servers unreachable.

---

## The TCP Segment

### TCP Header (20 bytes minimum)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key fields:**

- **Source/Destination Port** (16 bits each): Identifies the application.
  Port 443 = HTTPS, 80 = HTTP, 22 = SSH. Range 0-65535. Ports 0-1023 are
  "well-known" (require root on Unix). The combination of
  (src_ip, src_port, dst_ip, dst_port) uniquely identifies a TCP connection.

- **Sequence Number** (32 bits): The byte offset of the first byte of data
  in this segment within the stream. If the sequence number is 1000 and
  the segment carries 500 bytes, the next segment should have sequence
  number 1500. This is how TCP orders segments and detects gaps.

- **Acknowledgment Number** (32 bits): The next sequence number the sender
  expects to receive. "I've received everything up to byte 1500, send me
  1500 next." This is how TCP confirms delivery.

- **Flags** (6 bits):
  - **SYN**: Synchronize — initiates a connection
  - **ACK**: Acknowledgment — confirms received data
  - **FIN**: Finish — initiates graceful close
  - **RST**: Reset — abruptly terminates the connection
  - **PSH**: Push — tells the receiver to deliver data immediately (don't buffer)
  - **URG**: Urgent — rarely used, marks urgent data

- **Window** (16 bits): How many bytes the sender is willing to receive
  before the other side must wait. This is the **flow control** mechanism
  (explained below). With window scaling option, effectively up to 30 bits
  (~1 GB).

- **Checksum** (16 bits): Covers the header, data, and a pseudo-header
  (source/destination IP, protocol). Detects corruption but not
  malicious modification (not cryptographic — that's TLS's job).

---

## The Three-Way Handshake

Every TCP connection begins with this exchange. It synchronizes sequence
numbers and negotiates parameters.

```
Client                                    Server
  │                                         │
  │ SYN (seq=x)                             │
  │ ───────────────────────────────────────> │
  │                                         │
  │ SYN-ACK (seq=y, ack=x+1)               │
  │ <─────────────────────────────────────── │
  │                                         │
  │ ACK (seq=x+1, ack=y+1)                 │
  │ ───────────────────────────────────────> │
  │                                         │
  │ Connection established                  │
```

### Step by Step

**Step 1: Client sends SYN**

```
TCP Header:
  src_port: 54321 (random ephemeral port)
  dst_port: 443
  seq: 1000 (random Initial Sequence Number)
  ack: 0
  flags: SYN
  window: 65535
  options: MSS=1460, Window Scale=7, SACK Permitted, Timestamps
```

The client picks a random Initial Sequence Number (ISN). **The ISN must be
unpredictable.** If an attacker can predict the ISN, they can inject
packets into an established connection (TCP sequence prediction attack).
Modern OSes use a combination of a clock-based counter and a
cryptographic hash of the connection tuple + a secret key.

The options negotiate capabilities:
- **MSS (Maximum Segment Size)**: "My MTU supports segments up to 1460
  bytes." Prevents IP fragmentation.
- **Window Scale**: "Multiply the window field by 2^7 = 128." Allows
  windows larger than 65535 bytes (up to ~1 GB).
- **SACK (Selective ACK)**: "I support acknowledging non-contiguous ranges."
  Critical for performance over lossy links.
- **Timestamps**: Used for round-trip time measurement and PAWS (Protection
  Against Wrapped Sequences) for high-speed connections.

**Step 2: Server sends SYN-ACK**

```
TCP Header:
  src_port: 443
  dst_port: 54321
  seq: 5000 (server's random ISN)
  ack: 1001 (client's ISN + 1, confirming receipt)
  flags: SYN, ACK
  window: 65535
  options: MSS=1460, Window Scale=7, SACK Permitted, Timestamps
```

The server picks its own ISN and acknowledges the client's by setting
`ack = client_ISN + 1`. The SYN consumes one sequence number (even though
it carries no data).

**Step 3: Client sends ACK**

```
TCP Header:
  seq: 1001
  ack: 5001 (server's ISN + 1)
  flags: ACK
  window: 65535
```

Connection is established. Both sides know each other's ISN and agreed-upon
options. Data can flow in both directions.

### Why Three Steps and Not Two?

Two messages (SYN → SYN-ACK) would leave the server without confirmation
that the client received its ISN. The server wouldn't know if the client
is ready. The third ACK confirms both sides are synchronized.

**Without the third step**, the server would allocate resources for a
connection the client may never complete. This is exactly what a **SYN
flood attack** exploits — sending millions of SYNs without completing
the handshake, exhausting the server's connection table.

### SYN Flood Defense: SYN Cookies

Instead of storing state for half-open connections, the server encodes
the connection parameters into the ISN itself:

```
server_ISN = hash(src_ip, src_port, dst_ip, dst_port, secret, timestamp)
```

The server sends the SYN-ACK and **forgets about the connection**. When
the client's ACK arrives with `ack = server_ISN + 1`, the server
recalculates the hash and verifies it matches. If it does, the connection
is legitimate — allocate resources now.

**Tradeoff**: SYN cookies can't carry TCP options (MSS, window scale,
SACK) because there's nowhere to store them between the SYN-ACK and ACK.
Some implementations encode the MSS in a few bits of the cookie at the
cost of precision.

---

## The TCP State Machine

Every TCP connection is a finite state machine with 11 states. Understanding
these states is essential for debugging connection issues, reading `netstat`
output, and understanding why connections hang.

```
                              ┌───────────┐
                              │  CLOSED   │
                              └─────┬─────┘
                   ┌────────────────┴────────────────┐
                   │ (passive open)     (active open) │
                   ▼                                  ▼
             ┌──────────┐                       ┌───────────┐
             │  LISTEN  │                       │ SYN_SENT  │
             └────┬─────┘                       └─────┬─────┘
          rcv SYN │                          rcv SYN-ACK │
          snd SYN-ACK                           snd ACK │
                  ▼                                     ▼
           ┌──────────────┐                    ┌──────────────┐
           │ SYN_RECEIVED │───────────────────>│ ESTABLISHED  │
           └──────────────┘    rcv ACK         └──────┬───────┘
                                              ┌───────┴───────┐
                                    (close)   │               │  rcv FIN
                                    snd FIN   │               │  snd ACK
                                              ▼               ▼
                                       ┌────────────┐  ┌────────────┐
                                       │ FIN_WAIT_1 │  │ CLOSE_WAIT │
                                       └──────┬─────┘  └─────┬──────┘
                                  ┌───────────┼───────┐      │ (close)
                           rcv ACK│    rcv FIN│       │rcv   │ snd FIN
                                  │    snd ACK│       │FIN+  │
                                  ▼           ▼       │ACK   ▼
                           ┌────────────┐ ┌────────┐  │ ┌──────────┐
                           │ FIN_WAIT_2 │ │CLOSING │  │ │ LAST_ACK │
                           └──────┬─────┘ └───┬────┘  │ └────┬─────┘
                         rcv FIN  │    rcv ACK│       │  rcv ACK │
                         snd ACK  │           │       │          │
                                  ▼           ▼       ▼          ▼
                              ┌─────────────────────────┐  ┌────────┐
                              │       TIME_WAIT         │  │ CLOSED │
                              │     (wait 2×MSL)        │  └────────┘
                              └───────────┬─────────────┘
                                          │ timeout
                                          ▼
                                     ┌────────┐
                                     │ CLOSED │
                                     └────────┘
```

### The 11 States Explained

| State | Meaning | Common Issue |
|-------|---------|-------------|
| **CLOSED** | No connection | Initial/final state |
| **LISTEN** | Server waiting for SYN | Server is running, waiting for clients |
| **SYN_SENT** | Client sent SYN, waiting for SYN-ACK | Firewall blocking? Server down? |
| **SYN_RECEIVED** | Server got SYN, sent SYN-ACK, waiting for ACK | SYN flood if thousands accumulate |
| **ESTABLISHED** | Connection active, data flowing | Normal operation |
| **FIN_WAIT_1** | Sent FIN, waiting for ACK | Initiator of close |
| **FIN_WAIT_2** | Got ACK of our FIN, waiting for peer's FIN | Peer hasn't closed yet (half-closed) |
| **CLOSING** | Both sides sent FIN simultaneously | Rare, simultaneous close |
| **TIME_WAIT** | Sent final ACK, waiting for stale packets to expire | Thousands = normal on busy servers |
| **CLOSE_WAIT** | Received FIN, haven't sent our own FIN | **Bug alert**: application isn't calling `close()` |
| **LAST_ACK** | Sent our FIN, waiting for final ACK | Brief transitional state |

**Debugging with states**: If you see thousands of CLOSE_WAIT sockets, the
application has a bug — it's receiving connection closes but not closing its
end. If you see thousands of TIME_WAIT, that's normal for a server handling
many short-lived connections (see TIME_WAIT discussion in Connection
Termination). If you see SYN_SENT accumulating, the server is unreachable or
a firewall is dropping packets.

```bash
# See connection states on Linux
ss -tan state established | wc -l        # count established connections
ss -tan state time-wait | wc -l          # count TIME_WAIT
ss -tan state close-wait | wc -l         # count CLOSE_WAIT (potential bug)
netstat -an | awk '/^tcp/ {print $6}' | sort | uniq -c | sort -rn  # all states
```

---

## Reliable Delivery

### Sequence Numbers and ACKs

Every byte in the TCP stream has a sequence number. The receiver
acknowledges by telling the sender the next byte it expects:

```
Sender                              Receiver
  │ [seq=1000, 500 bytes]            │
  │ ────────────────────────────────>│
  │                                  │
  │              [ack=1500]          │  "Got bytes 1000-1499, send 1500"
  │ <────────────────────────────────│
  │                                  │
  │ [seq=1500, 500 bytes]            │
  │ ────────────────────────────────>│
  │                                  │
  │              [ack=2000]          │  "Got bytes 1500-1999, send 2000"
  │ <────────────────────────────────│
```

### What Happens When a Packet Is Lost

```
Sender                              Receiver
  │ [seq=1000, 500 bytes]            │
  │ ────────────────────────────────>│  ✓ received
  │                                  │
  │ [seq=1500, 500 bytes]            │
  │ ─────────── X (lost) ──────────  │  ✗ never arrives
  │                                  │
  │ [seq=2000, 500 bytes]            │
  │ ────────────────────────────────>│  received, but gap at 1500
  │                                  │
  │              [ack=1500]          │  "I still need 1500"
  │ <────────────────────────────────│
  │              [ack=1500]          │  duplicate ACK
  │ <────────────────────────────────│
  │              [ack=1500]          │  third duplicate ACK
  │ <────────────────────────────────│
  │                                  │
  │ Sender sees 3 duplicate ACKs     │
  │ → Fast Retransmit seq=1500       │
  │ ────────────────────────────────>│  ✓ gap filled
  │                                  │
  │              [ack=2500]          │  "Got everything up to 2500"
  │ <────────────────────────────────│
```

**Three duplicate ACKs** triggers **Fast Retransmit** — the sender
retransmits the missing segment immediately without waiting for a timeout.
This is much faster than the retransmission timeout (RTO), which can be
seconds.

### Selective Acknowledgment (SACK)

Without SACK, the receiver can only say "I've received everything up to
X." If bytes 1000-1499 and 2000-2499 are received but 1500-1999 is
missing, the ACK says "send 1500" — the sender doesn't know that 2000-2499
was received and might retransmit it unnecessarily.

With SACK, the ACK includes the ranges that were received:

```
ACK: 1500
SACK blocks: [2000-2500]
```

"I need 1500-1999, but I already have 2000-2499 — don't resend that."

This dramatically reduces unnecessary retransmissions on lossy connections
(mobile, satellite, WiFi). SACK is negotiated during the handshake and
supported by virtually all modern TCP stacks.

---

## Retransmission Timer — Jacobson's Algorithm

The RTO (Retransmission Timeout) determines how long TCP waits before
retransmitting a segment. Too short → spurious retransmissions, wasting
bandwidth. Too long → unnecessary waiting, poor performance.

### The Original Algorithm (RFC 793)

```
SRTT = α × SRTT + (1 - α) × RTT_sample     (α = 0.9)
RTO  = β × SRTT                              (β = 2)
```

Simple exponential moving average. Problem: it couldn't adapt to high
variance in RTT. On a network with RTT varying between 50ms and 200ms,
the smoothed average might be ~100ms, and RTO = 200ms. But a sample of
180ms (normal variance) would trigger a spurious retransmission.

### Jacobson's Algorithm (RFC 6298)

Van Jacobson (1988) added a variance estimator. This is the algorithm
used by all modern TCP stacks:

```
On each RTT measurement:

  RTTVAR = (1 - β) × RTTVAR + β × |SRTT - RTT_sample|     (β = 1/4)
  SRTT   = (1 - α) × SRTT   + α × RTT_sample              (α = 1/8)
  RTO    = SRTT + max(G, 4 × RTTVAR)

  G = clock granularity (typically 1ms on modern systems)

Initial values (before any measurement):
  RTO = 1 second (RFC 6298)

After first measurement:
  SRTT   = RTT_sample
  RTTVAR = RTT_sample / 2
  RTO    = SRTT + max(G, 4 × RTTVAR) = SRTT + 2 × RTT_sample
```

**Why this works**: The `4 × RTTVAR` term means the timeout adapts to
network jitter. On a stable network (low variance), RTO stays close to
SRTT. On a jittery network, RTO expands to accommodate the variance.

**Exponential backoff on timeout**: If a retransmitted segment also times
out, double the RTO:

```
First timeout:  RTO = calculated value
Second timeout: RTO = 2 × RTO
Third timeout:  RTO = 4 × RTO
...capped at 60 seconds (RFC 6298)
```

**Karn's Algorithm**: Don't update SRTT/RTTVAR from retransmitted segments.
If a segment is retransmitted and then ACKed, you can't tell if the ACK is
for the original or the retransmission — so the RTT sample is ambiguous.
TCP Timestamps (RFC 7323) solve this by echoing the sender's timestamp
in the ACK, allowing RTT measurement even for retransmitted segments.

### Worked Example

```
Network with RTT bouncing between 80ms and 120ms:

Initial: RTO = 1000ms

First sample: RTT = 100ms
  SRTT   = 100ms
  RTTVAR = 50ms
  RTO    = 100 + max(1, 4 × 50) = 100 + 200 = 300ms

Second sample: RTT = 80ms
  RTTVAR = 0.75 × 50 + 0.25 × |100 - 80| = 37.5 + 5 = 42.5ms
  SRTT   = 0.875 × 100 + 0.125 × 80 = 87.5 + 10 = 97.5ms
  RTO    = 97.5 + max(1, 4 × 42.5) = 97.5 + 170 = 267.5ms

Third sample: RTT = 120ms
  RTTVAR = 0.75 × 42.5 + 0.25 × |97.5 - 120| = 31.875 + 5.625 = 37.5ms
  SRTT   = 0.875 × 97.5 + 0.125 × 120 = 85.3 + 15 = 100.3ms
  RTO    = 100.3 + max(1, 4 × 37.5) = 100.3 + 150 = 250.3ms
```

The RTO stabilizes around 250ms — comfortably above the 80-120ms RTT range
but not wastefully high.

---

## Flow Control — The Sliding Window

### The Problem

The sender might transmit faster than the receiver can process. Without
control, the receiver's buffer overflows and packets are dropped — wasting
network bandwidth on data that will just be thrown away.

### How It Works

The receiver advertises a **window size** — how much buffer space it has
available. The sender can send up to that many bytes without waiting for
an ACK.

```
Receiver's window: 4000 bytes

Sender's view of the byte stream:
[already ACKed][  sent, unACKed  ][    can send    ][ can't send yet ]
     ...       [1000............3000][3000......5000][5000...........
               └── 2000 bytes in flight ──┘└─ 2000 bytes window ─┘
```

As the receiver processes data, it sends ACKs with an updated window:

```
[ack=2000, window=4000]  → "I've processed up to 2000, I have 4000 free"
[ack=3000, window=2000]  → "Processed up to 3000, but I'm slower — only 2000 free"
[ack=3000, window=0]     → "STOP SENDING — my buffer is full"
```

**Window = 0** means the receiver is overwhelmed. The sender stops and
periodically sends a 1-byte **window probe** to check if space has opened
up. This prevents deadlock (the receiver might never send an update if the
sender never sends data, and the sender never sends data if the window
is 0).

### Window Scaling

The window field is 16 bits (max 65535). For high-bandwidth, high-latency
connections (intercontinental, data center to data center), 65KB in flight
is not enough.

**Bandwidth-Delay Product (BDP)**: The amount of data that can be "in the
pipe" at once. For a 1 Gbps link with 50ms RTT:

```
BDP = 1,000,000,000 bits/sec × 0.050 sec = 50,000,000 bits = 6.25 MB
```

You need a 6.25 MB window to fully utilize the link. 65KB would use only
~1% of the bandwidth.

The **Window Scale option** (negotiated in the SYN) specifies a shift
count. A shift of 7 means multiply the window field by 2^7 = 128:

```
Window field: 65535 × 128 = 8,388,480 bytes (~8 MB) — enough for the link
```

---

## Congestion Control — Not Overwhelming the Network

### The Problem

Flow control prevents overwhelming the **receiver**. But what about the
**network**? The path between sender and receiver may traverse links with
different capacities. A fast sender can overwhelm a slow intermediate link,
causing queues to build up and packets to be dropped.

TCP's congestion control algorithms are the reason the internet doesn't
collapse under load. Without them, every connection would blast data at
maximum speed, overwhelming routers, causing massive packet loss, triggering
retransmissions, causing more congestion — a **congestion collapse**.

### The Congestion Window (cwnd)

In addition to the receiver's window (rwnd), the sender maintains a
**congestion window (cwnd)** — its estimate of how much data the network
can handle.

```
Effective sending rate = min(cwnd, rwnd)
```

The sender can send whichever is smaller: what the receiver can handle or
what the network can handle.

### Slow Start

Despite the name, slow start is actually **exponential growth**.

```
Initial cwnd: 1 MSS (or 10 MSS on modern systems, per RFC 6928)

Round 1: send 1 segment     → receive 1 ACK → cwnd = 2
Round 2: send 2 segments    → receive 2 ACKs → cwnd = 4
Round 3: send 4 segments    → receive 4 ACKs → cwnd = 8
Round 4: send 8 segments    → receive 8 ACKs → cwnd = 16
...

cwnd doubles every RTT. Growth is exponential: 1 → 2 → 4 → 8 → 16 → ...
```

Slow start continues until:
1. **cwnd reaches ssthresh** (slow start threshold) → switch to Congestion
   Avoidance (linear growth)
2. **Packet loss is detected** → cut cwnd, adjust ssthresh

**Why start slow?** The sender doesn't know the network's capacity. Starting
at full speed would immediately congest the network. Exponential growth
quickly discovers the available bandwidth while being cautious.

**Initial window**: Historically 1 MSS (~1.4 KB). Modern stacks use 10 MSS
(~14 KB) per RFC 6928. This means small responses (< 14 KB) can be sent in
a single round trip. This is why keeping your initial page load under 14 KB
matters for performance — it fits in the first congestion window.

### Congestion Avoidance

Once cwnd reaches ssthresh, growth switches to **linear** (additive
increase):

```
Each RTT: cwnd += 1 MSS

cwnd: 16 → 17 → 18 → 19 → 20 → ...
```

This probes for more bandwidth cautiously. The connection gradually
uses more of the available capacity without spiking.

### Loss Detection and Response

**Timeout (RTO)**: No ACK received within the retransmission timeout.
Indicates severe congestion. Response:
```
ssthresh = cwnd / 2
cwnd = 1 MSS
→ Back to slow start (exponential growth from 1)
```

This is the most aggressive response — the connection drops to minimum
and slowly rebuilds.

**Three duplicate ACKs (Fast Retransmit)**: Indicates a single lost packet,
not severe congestion (subsequent packets still arrived). Response depends
on the specific algorithm:

### TCP Reno (Classic)

```
On 3 duplicate ACKs:
  ssthresh = cwnd / 2
  cwnd = ssthresh     (halve the window)
  → Enter Fast Recovery (linear growth from the halved window)

On timeout:
  ssthresh = cwnd / 2
  cwnd = 1 MSS
  → Slow start
```

### TCP CUBIC (Linux default since 2.6.19)

CUBIC is the most widely deployed congestion control algorithm. Instead of
linear growth during congestion avoidance, it uses a **cubic function** of
time since the last congestion event.

```
W(t) = C × (t - K)³ + W_max

W_max: cwnd at the time of the last loss event
K:     ∛(W_max × β / C)  (time to reach W_max again)
C:     scaling constant (0.4)
β:     multiplicative decrease factor (0.7 — less aggressive than Reno's 0.5)
t:     time since last congestion event
```

**The shape**: After a loss, cwnd drops to 0.7 × W_max (30% cut instead
of Reno's 50%). Growth is initially fast (steep cubic curve), slows as it
approaches W_max (probing cautiously near the previous limit), then
accelerates again beyond W_max (exploring higher bandwidth).

```
cwnd
 ▲
 │                           ╱
 │         W_max ─ ─ ─ ─ ─╱─ ─ ─
 │                       ╱╱
 │                     ╱╱
 │                  ╱╱
 │               ╱╱      cubic growth: fast→slow→fast
 │             ╱╱
 │           ╱╱
 │     ╱╱╱╱╱╱
 │ ╱╱╱╱
 └──────────────────────────────> time
   loss                    steady state
   event
```

**Why cubic?** The cubic function is **independent of RTT**. Reno's
additive increase (1 MSS per RTT) means high-RTT connections grow slower
and get less bandwidth. CUBIC's growth is a function of **time**, not
round trips — making it fairer for connections with different latencies.

### BBR (Bottleneck Bandwidth and Round-trip propagation time)

Developed by Google (2016). A fundamentally different approach — instead
of using packet loss as the congestion signal, BBR actively measures the
network's bandwidth and RTT.

**The insight**: Traditional algorithms (Reno, CUBIC) increase cwnd until
packets are lost. But packet loss means a buffer somewhere is full — the
algorithm already overshot the optimal rate. BBR instead tries to find the
**Kleinrock operating point**: maximum bandwidth with minimum delay (no
queue buildup).

```
BBR maintains two estimates:
  BtlBw: Bottleneck Bandwidth (max delivery rate observed)
  RTprop: Round-trip propagation time (min RTT observed)

Optimal sending rate = BtlBw × RTprop

BBR cycles through four phases:
  1. Startup:     Exponential growth (like slow start) to find BtlBw
  2. Drain:       Reduce rate to drain queues built during startup
  3. ProbeBW:     Cycle between slightly above and below BtlBw to track changes
  4. ProbeRTT:    Periodically reduce cwnd to minimum to get a fresh RTprop
```

**BBR vs CUBIC**: BBR achieves higher throughput on lossy networks (it
doesn't interpret every loss as congestion). But BBR can be unfair to
CUBIC flows — it may take more than its share of bandwidth. BBRv2
addresses some of these fairness issues.

### ECN (Explicit Congestion Notification)

Traditional congestion control relies on **packet loss** as the congestion
signal. But loss means a buffer somewhere is already full — the algorithm
overshot. ECN lets routers signal congestion **before** dropping packets.

**How it works:**

```
IP Header:  Two ECN bits in the Traffic Class / Type of Service field
            00 = Not ECN-Capable
            01 = ECN-Capable Transport (ECT(1))
            10 = ECN-Capable Transport (ECT(0))
            11 = Congestion Experienced (CE)

TCP Header: Two flags negotiated in the SYN handshake
            ECE (ECN-Echo):    "I received a CE-marked packet"
            CWR (Congestion Window Reduced): "I've responded to your ECE"
```

```
1. Sender marks outgoing packets ECT(0) → "This packet supports ECN"
2. Router detects queue building up (before overflow)
3. Router marks packet CE → "I'm getting congested"
   (instead of dropping the packet)
4. Receiver sees CE-marked packet → sets ECE flag in ACK
5. Sender receives ECE → reduces cwnd (same as loss response)
6. Sender sets CWR flag → "I've reduced, stop sending ECE"
```

**Why it matters**: ECN allows congestion response without retransmission.
The data arrives intact — only the congestion signal is different. This is
especially important for:
- Low-latency applications where retransmission adds unacceptable delay
- BBR and other model-based algorithms that can use ECN as an early signal
- Data center networks (DCTCP) where precise congestion feedback enables
  near-zero-loss operation at high utilization

**DCTCP (Data Center TCP)**: Uses ECN differently — instead of treating
any CE mark as "halve the window," DCTCP measures the **fraction** of
CE-marked packets and reduces the window proportionally. If 10% of
packets are CE-marked, reduce cwnd by 10%, not 50%. This allows data
center networks to maintain very low queue depths while running at
near 100% link utilization.

---

## Head-of-Line Blocking — TCP's Fundamental Limitation

This is the problem that motivates HTTP/2's multiplexing and ultimately
HTTP/3's move to QUIC/UDP.

### The Problem

TCP guarantees **in-order delivery**. If byte 1000 is lost but bytes
1001-5000 arrive, the kernel buffers 1001-5000 and waits for 1000 to be
retransmitted. The application sees nothing until the gap is filled.

```
Stream: [1][2][3][4][5][6][7][8][9][10]
         ✓  ✓  ✗  ✓  ✓  ✓  ✓  ✓  ✓  ✓
                ↑
                lost — everything after is blocked

Application recv() returns: [1][2].........(waiting)........[3][4][5][6][7][8][9][10]
```

This is fine for a single stream of data (downloading a file). But
consider HTTP/2, which multiplexes **multiple independent requests**
over a single TCP connection:

```
TCP stream: [Req A][Req B][Req A][Req C][Req B][Req A]
             ✓      ✗      ✓      ✓      ✓      ✓
                    ↑
                    This packet belongs to Request B

Request A: has data ready but is blocked waiting for B's packet
Request C: has data ready but is blocked waiting for B's packet
```

A single lost packet from one request blocks **all** requests on the
connection. This is **worse** than HTTP/1.1's approach of opening
separate TCP connections per request — at least there, a loss on one
connection doesn't block others.

### Why TCP Can't Fix This

Head-of-line blocking is intrinsic to TCP's design. The byte stream
abstraction guarantees ordered delivery — there's no way to tell the
kernel "deliver these bytes out of order, they belong to independent
streams." The TCP stack has no concept of application-level multiplexing.

### The Solution: QUIC

QUIC (the transport protocol under HTTP/3) runs over UDP and implements
its own reliability per-stream:

```
QUIC connection:
  Stream 1: [A][A][A]  ← independent ordering
  Stream 2: [B][X][B]  ← loss here only blocks Stream 2
  Stream 3: [C][C][C]  ← unaffected by Stream 2's loss
```

Each QUIC stream has its own sequence numbers and retransmission. A loss
on Stream 2 doesn't block Streams 1 or 3. This is why HTTP/3 exists —
it's HTTP/2's multiplexing without TCP's head-of-line blocking.

---

## Connection Termination

### Graceful Close (Four-Way Handshake)

```
Client                                    Server
  │                                         │
  │ FIN (seq=x)                             │
  │ ───────────────────────────────────────> │  "I'm done sending"
  │                                         │
  │ ACK (ack=x+1)                           │
  │ <─────────────────────────────────────── │  "Got your FIN"
  │                                         │
  │ (server may continue sending data)      │
  │                                         │
  │ FIN (seq=y)                             │
  │ <─────────────────────────────────────── │  "I'm done sending too"
  │                                         │
  │ ACK (ack=y+1)                           │
  │ ───────────────────────────────────────> │  "Got your FIN"
  │                                         │
  │ (wait 2×MSL)                            │
  │ Connection closed                       │
```

**Why four steps?** TCP is full-duplex — each direction closes
independently. The client's FIN closes client→server. The server's FIN
closes server→client. Between the two FINs, the server can still send
data (it might be finishing a response).

**TIME_WAIT state**: After sending the final ACK, the client enters
TIME_WAIT for 2 × MSL (Maximum Segment Lifetime, typically 60 seconds).
Purpose:

1. If the final ACK was lost, the server will retransmit its FIN, and
   the client needs to be able to re-ACK it
2. Ensures all packets from this connection have expired before the port
   is reused — preventing old packets from being misinterpreted as part
   of a new connection

**TIME_WAIT in practice**: A server handling many short-lived connections
can accumulate thousands of TIME_WAIT sockets. Each one holds a port for
60+ seconds. Mitigations:
- `SO_REUSEADDR`: Allow reuse of local addresses in TIME_WAIT
- `SO_REUSEPORT`: Allow multiple sockets to bind to the same port
- `tcp_tw_reuse`: Kernel option to reuse TIME_WAIT sockets for new
  outgoing connections (safe because timestamps prevent old packet confusion)

### Abrupt Close (RST)

A RST immediately terminates the connection without the four-way handshake.
Used when:
- The receiver gets a packet for a connection that doesn't exist
- The application crashes or explicitly aborts (`SO_LINGER` with timeout=0)
- A firewall or IDS kills a connection

RST doesn't require an ACK. The connection is dead immediately. Any data
in flight is lost.

---

## TCP Fast Open (TFO)

The three-way handshake costs one round trip before data can flow. For
short requests (HTTP GET), this handshake latency dominates total request
time.

**TCP Fast Open** allows data in the SYN packet:

```
First connection (normal handshake + cookie exchange):
  Client → SYN + Cookie Request
  Server → SYN-ACK + Cookie
  Client → ACK

Subsequent connections (data in SYN):
  Client → SYN + Cookie + HTTP GET /index.html
  Server → SYN-ACK + HTTP Response (already processing!)
  Client → ACK
```

The server issues a cryptographic cookie on the first connection. On
subsequent connections, the client includes this cookie in the SYN along
with request data. The server validates the cookie and processes the
request immediately — saving a full RTT.

**Limitation**: TFO is vulnerable to replay attacks. An attacker who
captures a SYN+data packet can replay it, causing the server to process
the request again. This is safe for idempotent requests (GET) but
dangerous for non-idempotent ones (POST). Applications must be aware.

---

## UDP — When You Don't Need TCP

UDP (User Datagram Protocol) is the anti-TCP. No connection, no ordering,
no reliability, no congestion control. Just source port, destination port,
length, checksum, and data.

### UDP Header (8 bytes)

```
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**8 bytes** vs TCP's 20+ bytes. No sequence numbers, no ACKs, no window,
no state. Each datagram is independent.

### When UDP Wins

**Real-time media** (VoIP, video streaming, gaming):
A lost packet in a voice call is better handled by skipping forward (brief
glitch) than by retransmitting (delay accumulates). TCP's retransmission
and ordering guarantees add latency that's worse than the data loss.

**DNS queries**:
A single request-response pair. The overhead of a TCP handshake (1 RTT)
doubles the latency. If the UDP packet is lost, the application just
retries. (DNS does use TCP for zone transfers and large responses > 512
bytes.)

**QUIC / HTTP/3**:
Built on UDP but implements its own reliability, ordering, and congestion
control in userspace. This lets QUIC innovate on transport mechanisms
without waiting for OS kernel updates. More in the HTTP topic.

**DHCP**:
The client doesn't have an IP address yet, so it can't establish a TCP
connection. UDP broadcast works without prior IP configuration.

### The Misconception: "UDP Is Faster"

UDP itself isn't faster — it just does less. If your application needs
reliable, ordered delivery, building it on UDP means reimplementing
what TCP already provides. The real advantage of UDP is **control** —
you choose exactly which guarantees you need and implement only those,
avoiding TCP's one-size-fits-all approach.

---

## Nagle's Algorithm and Delayed ACKs

Two performance optimizations that interact badly.

### Nagle's Algorithm

When the application writes small amounts of data (keystroke by keystroke
in SSH, small API responses), Nagle's algorithm buffers them:

```
Rule: If there is unacknowledged data in flight, buffer small writes
      and send them together when the ACK arrives.

Without Nagle: send("H"), send("e"), send("l"), send("l"), send("o")
  → 5 packets, each with 40 bytes of headers and 1 byte of data
  → 200 bytes of headers for 5 bytes of data (97.5% overhead)

With Nagle: send("H") immediately, buffer "ello" until H is ACKed
  → 2 packets: "H" then "ello"
  → 80 bytes of headers for 5 bytes of data (better)
```

### Delayed ACKs

The receiver doesn't ACK every segment immediately. It waits up to
~40-200ms hoping to piggyback the ACK on a data response:

```
Without delay: receive segment → immediately send empty ACK
  → Extra packet just for the ACK

With delay: receive segment → wait up to 200ms → if we have data to send,
            attach ACK to data packet. If timer expires, send empty ACK.
```

### The Interaction Problem

```
1. App sends small request (via Nagle: sent immediately, nothing in flight)
2. Server receives, starts processing
3. App writes a second small piece (Nagle: buffer it, first is unACKed)
4. Server's delayed ACK timer: waiting up to 200ms to piggyback ACK
5. Nagle on client: waiting for ACK to send buffered data
6. → DEADLOCK for up to 200ms
```

The client waits for an ACK to send more data. The server waits for data
to piggyback an ACK. Both are waiting. Eventually the delayed ACK timer
fires and the ACK is sent, releasing the Nagle buffer. But you've added
200ms of latency to every interaction.

**The fix**: `TCP_NODELAY` socket option disables Nagle's algorithm. Most
application protocols (HTTP, database drivers, game servers) set this
because they manage their own buffering and small writes are
latency-sensitive.

---

## How TCP Maps to Application Code

### The Socket API

The TCP protocol is accessed through the **Berkeley Sockets API**, the
interface between application code and the kernel's TCP stack:

```
Server:
  socket()     → Create a socket (file descriptor)
  bind()       → Attach to a local address and port
  listen()     → Mark as passive (willing to accept connections)
  accept()     → Block until a client connects; return new socket
  recv()/send() → Read/write data
  close()      → Initiate FIN handshake

Client:
  socket()     → Create a socket
  connect()    → Initiate three-way handshake to server
  send()/recv() → Read/write data
  close()      → Initiate FIN handshake
```

**`listen()` backlog**: The parameter to `listen(backlog)` controls the
size of the kernel's queue for completed connections waiting to be
`accept()`ed. If the queue is full, the kernel either drops the SYN or
sends RST (behavior varies by OS). This is distinct from the SYN queue
(half-open connections waiting for the third ACK).

**File descriptors**: Each socket is a file descriptor. The OS has a
per-process limit (`ulimit -n`, often 1024 by default) and a system-wide
limit (`/proc/sys/fs/file-max`). A server handling 10,000 concurrent
connections needs at least 10,000 file descriptors. Production servers
set this to 65535 or higher.

### Connection Handling Models

**Process-per-connection (Apache prefork)**:
`accept()` returns a new socket, `fork()` a child process to handle it.
Simple but expensive — each process is ~1-10 MB. 10,000 connections =
10-100 GB of RAM.

**Thread-per-connection**:
Lighter than processes (~1 MB per thread stack) but still limited. 10,000
threads consume ~10 GB of stack space, plus context-switching overhead.

**Event-driven (epoll/kqueue — Node.js, nginx)**:
A single thread monitors thousands of sockets using the OS's event
notification system:

```c
// Linux epoll
int epfd = epoll_create(1);
epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &event);

while (true) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, timeout);
    for (int i = 0; i < n; i++) {
        if (events[i].events & EPOLLIN) {
            handle_read(events[i].data.fd);
        }
    }
}
```

`epoll_wait()` blocks until any of the monitored sockets have data. The
kernel maintains a ready list — O(1) for adding sockets, O(ready) for
waiting (only returns sockets that actually have events). This is how a
single Node.js process handles 10,000+ concurrent connections with minimal
RAM.

**`epoll` vs `select` vs `poll`**:
- `select`: O(n) — scans all file descriptors every call. Limit of 1024 fds.
- `poll`: O(n) — same scanning behavior, no fd limit.
- `epoll`: O(1) add, O(ready) wait. Kernel maintains the interest list.
  Scales to millions of connections.
- `kqueue` (BSD/macOS): Similar to epoll. Unified interface for sockets,
  files, signals, timers.

---

## Key Takeaways

| Concept | What You Must Know |
|---------|--------------------|
| IP is unreliable, unordered, connectionless | TCP builds reliability on top |
| IPv6 simplified the IP header | No checksum, no router fragmentation, fixed 40 bytes, extension headers for options |
| TCP has 11 states | CLOSE_WAIT accumulation = application bug. TIME_WAIT accumulation = normal |
| Three-way handshake syncs ISNs | SYN → SYN-ACK → ACK. ISN must be unpredictable |
| SYN cookies defend against SYN floods | Encode state in the ISN, don't allocate until ACK |
| Sequence numbers track byte positions | ACKs confirm receipt. 3 duplicate ACKs trigger fast retransmit |
| SACK acknowledges non-contiguous ranges | Prevents unnecessary retransmission on lossy links |
| RTO uses Jacobson's algorithm | SRTT + 4×RTTVAR adapts to network jitter. Karn's algorithm: don't sample retransmissions |
| Flow control: receiver's window (rwnd) | Prevents overwhelming the receiver. Window=0 means stop |
| Congestion control: sender's cwnd | Prevents overwhelming the network. Effective rate = min(cwnd, rwnd) |
| Slow start is exponential, congestion avoidance is linear | CUBIC uses a cubic function of time (RTT-independent) |
| BBR measures bandwidth and RTT directly | Targets the optimal operating point without filling buffers |
| ECN signals congestion without dropping packets | Router marks CE, receiver echoes ECE, sender reduces cwnd |
| Head-of-line blocking is TCP's fundamental limit | One lost packet blocks all multiplexed streams. QUIC solves this |
| TIME_WAIT lasts 2×MSL (~60s) | Prevents old packets from corrupting new connections |
| TCP_NODELAY disables Nagle's algorithm | Prevents latency from Nagle + delayed ACK interaction |
| epoll/kqueue enable C10K+ per thread | Event-driven I/O is how Node.js/nginx handle massive concurrency |
| UDP is not "faster" — it does less | Use UDP when you need control over reliability tradeoffs |
