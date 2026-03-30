# IP Addresses

**Phase 2 — Networking Fundamentals | Topic 2 of 8**

---

## What is an IP Address?

An IP address is a unique numerical label assigned to every device on a network that identifies it and enables communication. Think of it as a postal address for a machine.

```
Without IP addresses:
"Send this data to... somewhere on the internet" → impossible

With IP addresses:
"Send this data to 142.250.80.46" → routes directly to Google
```

---

## IPv4

The original and still dominant version. A 32-bit number written as four decimal octets:

```
192.168.1.100
 │   │  │  │
 │   │  │  └── 4th octet (0-255)
 │   │  └───── 3rd octet (0-255)
 │   └──────── 2nd octet (0-255)
 └──────────── 1st octet (0-255)

32 bits total → 2³² = ~4.3 billion possible addresses
```

The problem: 4.3 billion addresses sounded like enough in 1983. With smartphones, IoT devices, and billions of users, we ran out. IPv4 exhaustion is real — which is why NAT and IPv6 exist.

---

## IPv6

128-bit addresses written in hexadecimal, separated by colons:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334

Shortened form:
2001:db8:85a3::8a2e:370:7334

128 bits → 2¹²⁸ = 340 undecillion addresses
(enough for every atom on Earth to have an address)
```

IPv6 adoption is growing but IPv4 still dominates. In interviews, acknowledge IPv6 exists but design around IPv4 unless specifically asked.

---

## Public vs Private IP Addresses

**Public IPs** are globally unique and routable on the internet. Your cloud servers have public IPs.

**Private IPs** are reserved ranges not routable on the public internet. Used within private networks:

```
Private IP ranges (memorize these):
10.0.0.0    – 10.255.255.255     (10.x.x.x)
172.16.0.0  – 172.31.255.255     (172.16-31.x.x)
192.168.0.0 – 192.168.255.255    (192.168.x.x)

Your home router: 192.168.1.1
Your laptop:      192.168.1.100
AWS VPC:          10.0.0.0/16
```

Private IPs are how you get around IPv4 exhaustion — millions of networks each use the same private ranges internally, but only need a few public IPs facing the internet.

---

## NAT — Network Address Translation

NAT sits at the boundary between your private network and the internet, translating private IPs to a public IP and back.

```
Your laptop (192.168.1.100) requests google.com

Router (NAT):
  Outgoing: Replace 192.168.1.100 → 73.45.123.10 (your public IP)
  Incoming: Replace 73.45.123.10  → 192.168.1.100

Google only ever sees 73.45.123.10 — your public IP.
Your private IP is hidden behind it.
```

In AWS, instances in private subnets use a NAT Gateway to access the internet — they have private IPs internally but their outbound traffic appears to come from the NAT Gateway's public IP.

---

## CIDR Notation — Subnetting

CIDR (Classless Inter-Domain Routing) notation specifies a range of IP addresses:

```
10.0.0.0/16

          10.0.0.0  = network address
          /16       = first 16 bits are fixed (the network part)
                      remaining 16 bits are variable (host part)

→ 2¹⁶ = 65,536 addresses in this range
   from 10.0.0.0 to 10.0.255.255
```

Common CIDR blocks:

```
/32  → 1 address    (single host — used in security group rules)
/24  → 256 addresses    (10.0.1.0 – 10.0.1.255)
/16  → 65,536 addresses (10.0.0.0 – 10.0.255.255)
/8   → 16M addresses    (10.0.0.0 – 10.255.255.255)
```

A common AWS VPC setup:

```
VPC:              10.0.0.0/16  (65,536 addresses total)
Public Subnet 1:  10.0.1.0/24  (256 addresses, us-east-1a)
Public Subnet 2:  10.0.2.0/24  (256 addresses, us-east-1b)
Private Subnet 1: 10.0.3.0/24  (256 addresses, us-east-1a)
Private Subnet 2: 10.0.4.0/24  (256 addresses, us-east-1b)
```

---

## Static vs Dynamic IP Addresses

```
Static IP — permanently assigned, never changes.
Use for: Servers, databases, load balancers
         DNS records point to these
         Firewall rules whitelist these
         In AWS: Elastic IP (EIP)

Dynamic IP — assigned by DHCP, can change.
Use for: Client devices, temporary instances
         Your home internet connection
         EC2 instances on start (get new IP each time unless EIP)
```

> **Design rule:** Never hardcode IPs in config — always use DNS names. DNS can be updated when IPs change.

---

## Special Addresses

```
127.0.0.1    → Loopback (localhost) — refers to the machine itself

0.0.0.0      → All interfaces — when a server binds to this,
               it listens on every network interface

255.255.255.255 → Broadcast — sends to all devices on local network

169.254.x.x  → Link-local — auto-assigned when DHCP fails
               (AWS uses 169.254.169.254 for instance metadata)
```

---

## IP Addresses in System Design

```
Load Balancer design:
Users hit Load Balancer's public IP
LB forwards to backend servers' private IPs
Backend servers never expose public IPs
→ Security: attackers can't directly reach your servers

Database security:
DB instance has only private IP
Only accessible from within the VPC
Application servers in same VPC can reach it
Public internet cannot
→ This is the correct architecture

Microservices in Kubernetes:
Each Pod gets a private IP within the cluster network
Services get a stable ClusterIP (virtual IP)
Pods come and go, but Service IP stays constant
→ Always talk to the Service IP, never Pod IPs directly

Kafka broker addressing:
Brokers advertise their IP/hostname to clients
In cloud environments, use private IPs within VPC for
internal traffic — lower latency, higher security, no egress costs
```

---

## Key Takeaways

```
IPv4:    32-bit, 4.3B addresses, still dominant
IPv6:    128-bit, practically unlimited, growing adoption

Public IP:   Globally unique, routable on internet
Private IP:  Local network only (10.x, 172.16-31.x, 192.168.x)

NAT:     Translates private → public IP at network boundary
CIDR:    /24 = 256 addresses, /16 = 65K, /8 = 16M

Static IP:   For servers, DBs, load balancers
Dynamic IP:  For clients, temporary instances

Design rule: Never hardcode IPs — always use DNS names
             Put DBs and backends on private IPs only
             Users only reach your public-facing load balancer
```
