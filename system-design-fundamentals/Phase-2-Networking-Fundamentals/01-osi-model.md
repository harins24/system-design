# The OSI Model

**Phase 2 — Networking Fundamentals | Topic 1 of 8**

---

## What is the OSI Model?

OSI stands for Open Systems Interconnection. It's a conceptual framework that standardizes how different network systems communicate by dividing the communication process into 7 distinct layers.

Think of it as a contract — any two systems that follow this model can talk to each other, regardless of hardware, OS, or vendor.

```
Sender                          Receiver
─────────────────────────────────────────
7. Application  ──────────────► Application
6. Presentation ──────────────► Presentation
5. Session      ──────────────► Session
4. Transport    ──────────────► Transport
3. Network      ──────────────► Network
2. Data Link    ──────────────► Data Link
1. Physical     ──────────────► Physical
─────────────────────────────────────────
Data travels DOWN on sender, UP on receiver
```

Each layer has a specific responsibility and only communicates with the layers directly above and below it.

---

## The 7 Layers — Top to Bottom

### Layer 7 — Application
**What it does:** The layer closest to the user. Provides network services directly to applications.

- **Protocols:** HTTP, HTTPS, FTP, SMTP, DNS, WebSocket, gRPC
- Your Spring Boot REST endpoints live here
- Your Kafka producer/consumer API lives here

### Layer 6 — Presentation
**What it does:** Data translation, encryption, and compression.

- **Responsibilities:** TLS/SSL encryption, JSON/XML/Protobuf/Avro serialization, gzip/snappy compression
- When Kafka compresses messages with Snappy, that's Layer 6
- When HTTPS encrypts your payload, that's Layer 6

### Layer 5 — Session
**What it does:** Manages sessions — establishing, maintaining, and terminating connections between applications.

- Session establishment and teardown
- Authentication and authorization at connection level
- When your Spring Boot service maintains a persistent connection to PostgreSQL, the session layer manages that connection lifecycle
- In practice, Layers 5 and 6 are often handled by the application or transport layer in modern protocols

### Layer 4 — Transport
**What it does:** End-to-end communication between processes. Handles reliability, flow control, and segmentation.

- **Protocols:** TCP, UDP
- TCP → reliable, ordered, connection-oriented (used by HTTP, HTTPS, Kafka, PostgreSQL)
- UDP → unreliable, connectionless, fast (used by DNS, video streaming, gaming)
- **Port numbers** identify which process on a machine
- When your Spring Boot app listens on port 8080, that's a Layer 4 concept

### Layer 3 — Network
**What it does:** Logical addressing and routing. Gets packets from source to destination across multiple networks.

- **Protocols:** IP (IPv4, IPv6), ICMP, routing protocols (BGP, OSPF)
- IP addresses identify machines globally
- When a request travels from a user's browser in Chicago to your server in AWS us-east-1, Layer 3 routing determines the path

### Layer 2 — Data Link
**What it does:** Node-to-node communication on the same network. Handles physical addressing.

- **Protocols:** Ethernet, WiFi (802.11), MAC
- MAC addresses identify network cards physically
- Switches operate at this layer

### Layer 1 — Physical
**What it does:** Raw bit transmission over physical medium.

- Ethernet cables, fiber optic cables, WiFi radio waves
- This layer doesn't understand data — just bits: 0s and 1s

---

## Data Encapsulation — How Data Actually Travels

As data moves down the stack on the sender, each layer wraps it with its own header:

```
Application data:   [HTTP Request]
After Layer 4:      [TCP Header][HTTP Request]
After Layer 3:      [IP Header][TCP Header][HTTP Request]
After Layer 2:      [Ethernet Header][IP Header][TCP Header][HTTP Request][Ethernet Tail]
Layer 1:            01010101010101... (bits over wire)
```

On the receiver, each layer strips its header and passes the rest up.

---

## Why OSI Matters in System Design

### Load Balancers:
```
Layer 4 Load Balancer: routes based on IP + port
→ Fast, no inspection of content
→ Used for TCP/UDP traffic
→ AWS NLB

Layer 7 Load Balancer: routes based on HTTP content
→ Can route /api/users to User Service
→ Can route /api/orders to Order Service
→ Can do SSL termination, cookie-based routing
→ AWS ALB
```

### Firewalls:
```
Layer 3/4 firewall: blocks by IP address and port
Layer 7 firewall (WAF): inspects HTTP content, blocks SQL injection
```

### Where TLS/SSL sits:
```
TLS is between Layer 4 and Layer 7 (sometimes called Layer 4.5 or Layer 6)
SSL termination at load balancer = decrypt at LB, plain HTTP to backend
End-to-end TLS = encrypted all the way to the application
```

### Kafka and OSI:
```
Layer 7: Kafka protocol (produce/consume API)
Layer 6: Message compression (Snappy, LZ4, Gzip)
Layer 4: TCP (Kafka runs over TCP for reliability)
Layer 3: IP routing between brokers and clients
```

---

## The Practical Cheat Sheet

| Layer | Name | Key Protocols | Real World |
|-------|------|--------------|------------|
| 7 | Application | HTTP, gRPC, Kafka | Your code |
| 6 | Presentation | TLS, JSON, Avro | Encryption, serialization |
| 5 | Session | (handled by app/TCP) | DB connections |
| 4 | Transport | TCP, UDP | Ports, reliability |
| 3 | Network | IP | Routing, IP addresses |
| 2 | Data Link | Ethernet, WiFi | MAC addresses, switches |
| 1 | Physical | Cables, radio | Actual wire/fiber |

**Memory trick:** "All People Seem To Need Data Processing" (Application, Presentation, Session, Transport, Network, Data Link, Physical) — top to bottom.

---

## Key Takeaways

```
OSI: 7-layer model standardizing network communication

Most important layers for system design:
  Layer 7 → Where your application code lives (HTTP, gRPC, Kafka)
  Layer 4 → TCP vs UDP, ports, reliability
  Layer 3 → IP addresses, routing
  Layer 2 → MAC addresses, local network

L4 Load Balancer → routes by IP/port → fast, dumb
L7 Load Balancer → routes by HTTP content → smart, flexible

TLS → Layer 6 encryption → terminate at LB or end-to-end
```
