# Proxy vs Reverse Proxy

**Phase 2 — Networking Fundamentals | Topic 4 of 8**

---

## The Core Distinction

```
Forward Proxy:  sits in front of CLIENTS
                clients → proxy → internet
                server doesn't know who the real client is

Reverse Proxy:  sits in front of SERVERS
                internet → proxy → servers
                client doesn't know which real server it hits
```

The "forward" and "reverse" refer to which side of the communication the proxy represents.

---

## Forward Proxy

A forward proxy acts on behalf of **clients**. The client sends its request to the proxy, the proxy forwards it to the destination server.

```
Without proxy:
[Client] ──────────────────────► [Server]
 IP visible to server

With forward proxy:
[Client] ──► [Forward Proxy] ──► [Server]
              proxy IP visible
              client IP hidden
```

### What Forward Proxies Do

- **Anonymity / Privacy** — destination server sees the proxy's IP, not the client's
- **Content Filtering** — corporate networks block access to certain websites
- **Caching** — if 100 employees request the same resource, proxy fetches it once
- **Access Control** — force all outbound traffic through one point
- **Bypassing Restrictions** — VPNs route traffic through a proxy in another country

**Real Examples:** Squid Proxy (corporate), VPN (technically a forward proxy), Tor network

---

## Reverse Proxy

A reverse proxy acts on behalf of **servers**. Clients send requests thinking it IS the server.

```
Without reverse proxy:
[Client] ──────────────────────► [Server A]
                                 [Server B]
                                 [Server C]
Client needs to know about all servers

With reverse proxy:
[Client] ──► [Reverse Proxy] ──► [Server A]
             single endpoint     [Server B]
             client sees only    [Server C]
             the proxy
```

### What Reverse Proxies Do

**Load Balancing**
```
All requests → nginx (reverse proxy)
nginx distributes:
  Request 1 → Server A
  Request 2 → Server B
  Request 3 → Server C
  Request 4 → Server A (round robin)
```

**SSL Termination**
```
Client ──HTTPS──► [Reverse Proxy] ──HTTP──► [Backend Servers]
                  decrypts here              plain traffic inside
                                             private network
```
Performance win (SSL processing is CPU intensive) + security win (SSL cert management in one place).

**Caching** — Cache responses from backend servers. Backend never contacted for cached content.

**Compression** — Compress responses before sending to clients.

**Security / Protection** — Hides backend server IPs. DDoS attacks hit the proxy, not your application servers.

**Request Routing**
```
/api/users   → User Service
/api/orders  → Order Service
/api/payment → Payment Service
/            → Frontend Service
```

**Real Examples:** Nginx, HAProxy, AWS ALB, AWS CloudFront, Traefik, Kong

---

## Side by Side Comparison

| | Forward Proxy | Reverse Proxy |
|--|--------------|--------------|
| Sits in front of | Clients | Servers |
| Represents | Clients | Servers |
| Client knows | Proxy exists | Nothing (thinks proxy IS server) |
| Server knows | Proxy's IP only | Nothing |
| Primary use | Privacy, filtering, access control | Load balancing, SSL termination, security, routing |
| Who configures | Client/IT admin | Server/DevOps team |
| Examples | Corporate proxy, VPN | Nginx, ALB, CloudFront |

---

## Nginx as Reverse Proxy

```nginx
upstream backend_servers {
    server 10.0.1.1:8080;   # Spring Boot instance 1
    server 10.0.1.2:8080;   # Spring Boot instance 2
    server 10.0.1.3:8080;   # Spring Boot instance 3
}

server {
    listen 443 ssl;
    server_name api.yoursite.com;

    # SSL termination here
    ssl_certificate /etc/ssl/certs/yoursite.crt;
    ssl_certificate_key /etc/ssl/private/yoursite.key;

    location /api/users {
        proxy_pass http://backend_servers;    # load balance
    }

    location /api/orders {
        proxy_pass http://order_service:8081; # route to specific service
    }

    # Cache static content
    location /static {
        proxy_cache my_cache;
        proxy_pass http://backend_servers;
    }
}
```

One Nginx config doing load balancing, SSL termination, request routing, and caching simultaneously.

---

## In Your Architecture at Your Company

```
Internet
    ↓
AWS ALB (Reverse Proxy — Layer 7)
    → SSL termination
    → Routes /api/* to Spring Boot services
    → Health checks backend instances
    ↓
Spring Boot Microservices (multiple instances)
    ↓
Kafka (internal — no proxy needed, direct broker connection)
    ↓
Databases (private subnet — no direct internet access)
```

---

## Key Takeaways

```
Forward Proxy:
  → Sits in front of clients
  → Client's IP hidden from server
  → Use: privacy, content filtering, VPN, corporate access control

Reverse Proxy:
  → Sits in front of servers
  → Server IPs hidden from clients
  → Use: load balancing, SSL termination, caching,
         compression, security, request routing

Real systems: Nginx, HAProxy, AWS ALB, CloudFront, Traefik

In every system design interview:
  → Always put a reverse proxy in front of your backend
  → Terminate SSL at the reverse proxy
  → Backend servers on private IPs only
```
