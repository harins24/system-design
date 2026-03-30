# TCP vs UDP

**Phase 2 — Networking Fundamentals | Topic 6 of 8**

---

## The Core Difference

```
TCP:  I care that every byte arrives correctly, in order.
      I'll do whatever it takes to guarantee that.
      Slower, but reliable.

UDP:  I care about speed. Send and forget.
      Some packets lost? Fine.
      Faster, but unreliable.
```

Both are Layer 4 (Transport) protocols.

---

## TCP — Transmission Control Protocol

TCP provides **reliable, ordered, error-checked** delivery of a stream of bytes.

### The Three-Way Handshake

Before any data is sent, TCP establishes a connection:

```
Client                    Server
  │──── SYN ───────────────►│  "I want to connect, my seq=100"
  │◄─── SYN-ACK ────────────│  "OK, my seq=200, I got your 100"
  │──── ACK ───────────────►│  "Got your 200, connection open"
  │  [Data flows now]       │
  │──── FIN ───────────────►│  "I'm done sending"
  │◄─── ACK + FIN ──────────│  Connection closed
```

This handshake adds 1.5 round trips of latency before any data flows. For a US-to-Europe connection (~100ms RTT), that's 150ms just to establish the connection.

### What TCP Guarantees

- **Reliable delivery** — every packet is acknowledged; unacknowledged packets are retransmitted
- **Ordered delivery** — packets are numbered; out-of-order packets are buffered and reordered
- **Flow control** — receiver advertises buffer space; sender doesn't overwhelm receiver
- **Congestion control** — TCP detects network congestion and reduces sending rate

### TCP Head-of-Line Blocking

```
Stream of packets: 1, 2, 3, 4, 5
Packet 2 is lost:  1, ✗, 3, 4, 5

TCP behavior: Hold 3, 4, 5 in buffer
              Wait for retransmission of 2
              Deliver everything only after 2 arrives

Result: 3, 4, 5 are ready but blocked waiting for 2
```

HTTP/3 solves this by using QUIC over UDP.

---

## UDP — User Datagram Protocol

UDP provides **connectionless, unreliable, fast** delivery of independent packets called datagrams.

```
Sender sends packet → done.
No handshake, no ACK, no retransmit.
Receiver gets it or doesn't → sender doesn't know or care.
```

### What UDP Gives Up (And Gains)

```
No connection setup     → no handshake latency
No acknowledgments      → no retransmit delay
No ordering             → packets may arrive out of order
No flow control         → sender can overwhelm receiver
No congestion control   → sender can overwhelm network
No head-of-line block   → each packet independent

Result: Lower latency, higher throughput, less CPU overhead
        But: packets can be lost, duplicated, or reordered
```

### When Packet Loss Is Acceptable

```
Video streaming:  Losing a frame → slight visual artifact
                  Waiting for retransmit → unwatchable buffering

Voice call:       Losing 20ms → tiny gap, brain fills it in
                  Waiting for retransmit → conversation impossible

Gaming:           Position update from 50ms ago is useless
                  Need current position now, even if slightly wrong

DNS:              Small request, small response
                  If lost, just send the whole request again
```

---

## Side by Side Comparison

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Required (handshake) | None |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Guaranteed in-order | No guarantee |
| Error checking | Yes + retransmit | Checksum only |
| Flow control | Yes | No |
| Speed | Slower | Faster |
| Header size | 20-60 bytes | 8 bytes |
| Use when | Data must arrive complete and correct | Speed > reliability |

---

## Where Each Is Used

**TCP:**
- HTTP/HTTPS, gRPC, Kafka, PostgreSQL, SSH, SMTP

**UDP:**
- DNS, video calls (Zoom, WebRTC), live streaming, online gaming, NTP, DHCP

---

## Building Reliability on UDP

Many modern protocols are built on UDP but add their own reliability mechanisms:

**QUIC (HTTP/3):**
- Built on UDP with connection IDs, stream multiplexing, TLS 1.3 built in
- Independent per-stream reliability — packet loss in stream 1 doesn't block stream 2
- Faster connection establishment, better on mobile (connection survives IP change)

**WebRTC (video calls):**
- Built on UDP with SRTP encryption, RTCP quality feedback, jitter buffers
- Acceptable to lose some packets; retransmit makes voice/video worse

**Pattern:** Use UDP when you need low latency and want control over exactly what reliability mechanisms make sense for your application.

---

## Kafka and TCP

Kafka is built entirely on TCP — and for good reason:

```
Message durability requirements:
  1M+ messages/day
  99.9% uptime
  Zero data loss allowed

TCP gives:
  ✅ Guaranteed delivery (every message reaches broker)
  ✅ Ordered delivery (messages stay in partition order)
  ✅ Error detection (corrupted packets retransmitted)
  ✅ Flow control (producers don't overwhelm brokers)
```

Kafka's high throughput comes from sequential disk writes, zero-copy transfers, and batching — all over TCP. The protocol is efficient, not the transport.

---

## Key Takeaways

```
TCP:
  Reliable, ordered, connection-based
  Three-way handshake before data flows
  Retransmits lost packets automatically
  Flow + congestion control built in
  Use: HTTP, Kafka, databases, SSH, anything needing correctness

UDP:
  Unreliable, connectionless, fast
  No handshake, no ACKs, no retransmit
  Packets may be lost, duplicated, reordered
  Use: DNS, video calls, gaming, live streaming

Key insight:
  UDP doesn't mean "no reliability"
  QUIC, WebRTC, gaming protocols build
  custom reliability on UDP — tuned to their needs

TCP head-of-line blocking:
  Packet loss stalls all subsequent packets
  QUIC (HTTP/3) solves this by using UDP with per-stream loss handling

Kafka uses TCP:
  High throughput from batching + sequential writes
  Not from using UDP — reliability is non-negotiable
```
