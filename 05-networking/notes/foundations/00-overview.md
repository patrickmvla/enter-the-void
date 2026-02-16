# Networking — How Data Moves Between Machines

## The Core Problem

Two computers need to exchange data. They might be in the same room or
on opposite sides of the planet. Between them: copper wires, fiber optic
cables, radio waves, routers, switches, firewalls, load balancers, proxies,
and the entire chaotic infrastructure of the internet.

Networking is the set of protocols and systems that make this work
reliably, efficiently, and securely — despite hardware failures, packet
loss, congestion, latency, and adversaries.

---

## The Mental Model: Layers

Networking is organized in layers. Each layer solves one problem and
provides a clean abstraction to the layer above. The layer below handles
the messy details.

### The TCP/IP Model (What the Internet Actually Uses)

```
┌─────────────────────┐
│  Application Layer   │  HTTP, DNS, TLS, WebSocket, gRPC, SMTP
│                     │  "What data do I want to send?"
├─────────────────────┤
│  Transport Layer     │  TCP, UDP
│                     │  "How do I get it there reliably (or fast)?"
├─────────────────────┤
│  Internet Layer      │  IP (IPv4, IPv6), ICMP
│                     │  "How do I route it across networks?"
├─────────────────────┤
│  Link Layer          │  Ethernet, WiFi (802.11), ARP
│                     │  "How do I send bits to the next hop?"
└─────────────────────┘
```

### Why Not the OSI Model?

The OSI (Open Systems Interconnection) model has 7 layers. It's taught
in every networking course but doesn't match how the internet works:

```
OSI                          TCP/IP (reality)
───                          ────
7. Application  ─┐
6. Presentation  ├────────→  Application
5. Session      ─┘
4. Transport    ──────────→  Transport
3. Network      ──────────→  Internet
2. Data Link    ─┐
1. Physical     ─┘────────→  Link
```

OSI separates "presentation" (encoding, encryption) and "session"
(connection management) into their own layers. In practice, these are
handled within the application layer (TLS does encryption, HTTP/2 does
multiplexing, applications manage their own sessions). The 7-layer model
is useful vocabulary ("layer 7 load balancer," "layer 4 firewall") but
the 4-layer TCP/IP model describes what actually happens.

---

## What Happens When You Make an HTTP Request

The full stack in action — every layer involved when your browser
requests `https://example.com/api/data`:

```
APPLICATION LAYER
  Your code:       fetch('https://example.com/api/data')
  TLS:             Encrypt the HTTP request
  HTTP:            GET /api/data HTTP/1.1\r\nHost: example.com\r\n\r\n

TRANSPORT LAYER
  TCP:             Split into segments, add port numbers (src:54321, dst:443)
                   Add sequence numbers for ordering
                   Add checksum for integrity

INTERNET LAYER
  IP:              Add source IP (192.168.1.42) and destination IP (93.184.216.34)
                   Fragment if necessary (MTU)

LINK LAYER
  Ethernet:        Add source MAC and destination MAC (next-hop router)
  Physical:        Convert to electrical signals / light / radio waves

─── ON THE WIRE ───

  Arrives at your router → strips link layer, reads IP → routes to ISP
  ISP router → routes toward destination (BGP routing tables)
  ... possibly 10-20 hops ...
  Arrives at destination's network → link layer to the server's NIC
  Server's kernel: reassemble → TCP → TLS decrypt → HTTP parse → your app
```

### Encapsulation

Each layer wraps the data from the layer above with its own header:

```
[Ethernet Header][IP Header][TCP Header][TLS Record][HTTP Request][Data]
 14 bytes         20 bytes   20 bytes    5+ bytes    variable      variable
 └── Link ──────┘└─ Internet ┘└ Transport ┘└──── Application ─────────────┘
```

The receiving side strips headers in reverse order. Each layer reads only
its own header, processes it, and passes the payload up.

**Maximum Transmission Unit (MTU)**: Ethernet frames have a max payload
of 1500 bytes. If an IP packet exceeds this, it's either fragmented (IPv4)
or the sender is told to send smaller packets (IPv6, which doesn't allow
fragmentation). TCP handles this by negotiating the Maximum Segment Size
(MSS) during the handshake — typically 1460 bytes (1500 MTU - 20 IP header
- 20 TCP header).

---

## The Topics — Bottom Up

```
Start here
│
├── 01-tcp-ip.md
│   The transport and internet layers. How TCP provides reliable,
│   ordered delivery over an unreliable network. The 3-way handshake,
│   congestion control (slow start, CUBIC), flow control (sliding
│   window), and why UDP exists.
│
├── 02-dns.md
│   How "example.com" becomes 93.184.216.34. The resolution chain
│   (recursive → root → TLD → authoritative), caching, record types,
│   DNSSEC, and DNS-over-HTTPS.
│
├── 03-tls.md
│   How TLS secures the connection. The handshake (ClientHello →
│   ServerHello → key exchange → encrypted), certificate chains,
│   cipher suites, TLS 1.3 improvements. Ties directly to the
│   cryptographic primitives from security/authentication.
│
└── Then: application-layer/
    ├── 01-http.md          HTTP/1.1 → HTTP/2 → HTTP/3
    ├── 02-websockets.md    Real-time: WebSockets, SSE, long polling
    └── 03-apis.md          REST vs gRPC vs GraphQL — protocol-level
```

Each topic builds on the previous. TCP/IP is the foundation everything
rides on. DNS resolves names before connections are made. TLS encrypts the
TCP connection. HTTP structures the application data. WebSockets and APIs
are patterns built on top of HTTP/TLS/TCP.
