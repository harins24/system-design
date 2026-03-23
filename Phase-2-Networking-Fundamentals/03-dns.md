# Domain Name System (DNS)

**Phase 2 — Networking Fundamentals | Topic 3 of 8**

---

## What is DNS?

DNS is the phone book of the internet. It translates human-readable domain names into IP addresses that machines use to communicate.

```
You type:    www.google.com
DNS returns: 142.250.80.46
Browser connects to 142.250.80.46
```

DNS is one of the oldest and most critical infrastructure components of the internet. It's also a masterclass in distributed system design — handling trillions of queries daily with high availability and low latency.

---

## The DNS Hierarchy

DNS is a distributed hierarchical system — no single server knows everything:

```
                    . (Root)
                    │
        ┌───────────┼───────────┐
       .com        .org        .io       ← Top Level Domains (TLD)
        │
    google.com                           ← Authoritative Name Server
        │
   ┌────┴────┐
  www    mail.google.com                 ← Subdomains
```

- **Root servers** — 13 sets managed by ICANN/various orgs
- **TLD servers** — .com managed by Verisign, .org by PIR, etc.
- **Authoritative servers** — managed by you or your DNS provider (Route 53, Cloudflare)

---

## How DNS Resolution Works — Step by Step

When you type `blog.algomaster.io` in your browser:

```
1. Browser cache
   "Have I looked this up recently?" → if yes, use cached IP

2. OS cache / hosts file
   Check /etc/hosts → if found, use it

3. Recursive Resolver (your ISP or 8.8.8.8)
   Your machine asks its configured DNS resolver
   This resolver does the heavy lifting

4. Root Name Server
   Resolver asks: "Who handles .io?"
   Root says: "Ask the .io TLD server at 65.22.161.17"

5. TLD Name Server (.io)
   Resolver asks: "Who handles algomaster.io?"
   TLD says: "Ask algomaster's authoritative server at ns1.cloudflare.com"

6. Authoritative Name Server
   Resolver asks: "What is the IP for blog.algomaster.io?"
   Authoritative says: "It's 104.21.56.78"

7. Recursive Resolver returns 104.21.56.78 to your browser
   Browser caches it, connects to 104.21.56.78
```

This whole process typically takes 20-120ms the first time. After caching, it's essentially instant.

---

## DNS Record Types

| Record | Purpose |
|--------|---------|
| **A** | Maps domain to IPv4 address |
| **AAAA** | Maps domain to IPv6 address |
| **CNAME** | Maps domain to another domain (alias) — cannot be used on root domain |
| **MX** | Mail exchange — specifies email servers |
| **TXT** | Arbitrary text — used for SPF, DKIM, domain ownership verification |
| **NS** | Specifies authoritative name servers |
| **SRV** | Specifies location of services (port + hostname) — used by Kafka, SIP |

---

## TTL — Time To Live

Every DNS record has a TTL — how long resolvers and browsers should cache the result.

```
blog.algomaster.io → 104.21.56.78  TTL: 3600 (1 hour)
```

| TTL | Pros | Cons |
|-----|------|------|
| Low (60s) | Changes propagate quickly, good for blue-green deployments | More DNS queries, slightly higher latency |
| High (86400s) | Heavily cached, fast resolution, low DNS load | Changes take 24h to propagate, bad for migrations |

> **Production tip:** Before migrating a server, lower TTL to 60 seconds a day in advance. After migration is stable, raise TTL back up.

---

## DNS in System Design

### Load Balancing via DNS (Round Robin)
DNS can return multiple A records — different clients get different IPs:

```
api.yoursite.com → 10.0.1.1  (returned to client A)
api.yoursite.com → 10.0.1.2  (returned to client B)
api.yoursite.com → 10.0.1.3  (returned to client C)
```

Problem: Clients cache IPs; DNS doesn't know if a server is unhealthy. **Better: Use a proper load balancer and point DNS to it.**

### Latency-Based Routing (Route 53)
```
User in Tokyo    → api.yoursite.com → routed to ap-northeast-1
User in New York → api.yoursite.com → routed to us-east-1
User in London   → api.yoursite.com → routed to eu-west-1
```

### Health Check + Failover
```
Route 53 health checks us-east-1 every 30 seconds
us-east-1 goes down
Route 53 automatically updates DNS to point to us-west-2
→ Automatic geographic failover
```

### Service Discovery via DNS (Kubernetes)
```
Your Order Service calls:
http://payment-service:8080/pay

Kubernetes DNS resolves:
payment-service → 10.96.45.23 (ClusterIP of Payment Service)

When Payment Service pods restart and get new IPs,
the ClusterIP stays constant → DNS record stays valid
```

---

## DNS Caching Layers

Understanding all the places DNS gets cached helps diagnose "why isn't my DNS change working?":

```
1. Browser cache          (Chrome: chrome://net-internals/#dns)
2. OS cache               (flush: ipconfig /flushdns on Windows)
3. Router cache
4. ISP recursive resolver (may ignore TTL and cache longer)
5. Corporate DNS server
```

**Change propagation time = max TTL at any of these layers + ISP stubbornness = "up to 48 hours" (worst case)**

---

## Key Takeaways

```
DNS:    Translates domain names → IP addresses
        Hierarchical, distributed, cached at every layer

Resolution order:
  Browser cache → OS cache → Recursive Resolver →
  Root → TLD → Authoritative → back to you

Key records:
  A     → domain to IPv4
  CNAME → domain alias to another domain
  MX    → mail servers
  TXT   → verification, SPF/DKIM
  NS    → authoritative name servers

TTL:    Low = fast propagation, more queries
        High = cached, slow propagation
        Lower TTL before migrations!

System design uses:
  Route 53 latency routing → direct users to nearest region
  Health check failover    → automatic geographic failover
  Kubernetes DNS           → service discovery within cluster

Never hardcode IPs in config — always use DNS names
```
