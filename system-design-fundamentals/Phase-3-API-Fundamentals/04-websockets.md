# WebSockets

**Phase 3 — API Fundamentals | Topic 4 of 8**

---

## The Problem WebSockets Solve

HTTP is request-response — the client always initiates. The server can never push data to the client unprompted. For most web interactions that's fine. But some applications need the server to send data to the client instantly when something happens.

```
HTTP polling (naive solution):
Client: "Any new messages?" → Server: "No"  (every 1 second)
Client: "Any new messages?" → Server: "No"
Client: "Any new messages?" → Server: "No"
Client: "Any new messages?" → Server: "Yes! Here they are"

Problems:
→ Massive wasted requests (99% return nothing)
→ Latency = polling interval (up to 1 second delay)
→ Server hammered with empty requests
→ Battery drain on mobile

Long polling (slightly better):
Client: "Any new messages?"
Server: holds connection open... (up to 30 seconds)
        message arrives → responds immediately
Client: immediately sends next "Any new messages?"

Better latency, but:
→ Still HTTP overhead per message
→ Connection held open consumes server resources
→ Proxies and load balancers may kill long connections
→ Not truly bidirectional
```

WebSockets solve this cleanly.

---

## What is a WebSocket?

A WebSocket is a **persistent, full-duplex communication channel** over a single TCP connection. Once established, both client and server can send messages to each other at any time, with minimal overhead.

```
HTTP:
Client ──request──► Server
Client ◄──response── Server
(connection closes or is reused for next request)

WebSocket:
Client ◄══════════════════════════════► Server
         persistent bidirectional channel
         either side can send anytime
         low overhead per message
```

### The WebSocket Handshake

WebSocket connections start as HTTP then upgrade:

```
1. Client sends HTTP Upgrade request:
GET /ws HTTP/1.1
Host: api.yoursite.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

2. Server responds with 101 Switching Protocols:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. HTTP connection upgraded to WebSocket
   TCP connection stays open
   Both sides now communicate in WebSocket frames
   (not HTTP anymore)
```

After the handshake, messages travel as lightweight frames — as small as 2 bytes of overhead vs hundreds of bytes for HTTP headers.

### WebSocket Message Format

```
WebSocket Frame Structure:
┌─────────────────────────────────────────┐
│ FIN │ RSV │ Opcode │ Mask │ Payload Len │
│  1  │  3  │   4    │  1   │    7 bits   │
├─────────────────────────────────────────┤
│         Extended Payload Length          │
│              (if needed)                │
├─────────────────────────────────────────┤
│            Masking Key                  │
│           (if Mask=1)                   │
├─────────────────────────────────────────┤
│            Payload Data                 │
└─────────────────────────────────────────┘

Opcodes:
0x1 = Text frame   (UTF-8 string, typically JSON)
0x2 = Binary frame (raw bytes)
0x8 = Close frame  (graceful disconnect)
0x9 = Ping frame   (keepalive)
0xA = Pong frame   (keepalive response)

Minimum overhead: 2 bytes vs ~500 bytes for HTTP headers
→ Dramatic bandwidth reduction for frequent small messages
```

---

## WebSockets in Practice

### Client Side (Browser)

```javascript
// Connect
const ws = new WebSocket('wss://api.yoursite.com/ws');

// Connection established
ws.onopen = () => {
    console.log('Connected');
    ws.send(JSON.stringify({
        type: 'SUBSCRIBE',
        channel: 'order-updates',
        orderId: '456'
    }));
};

// Receive message from server
ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.type === 'ORDER_UPDATE') {
        updateOrderStatus(data.orderId, data.status);
    }
};

// Handle disconnect
ws.onclose = (event) => {
    console.log('Disconnected:', event.code, event.reason);
    // Reconnect with exponential backoff
    setTimeout(reconnect, backoff);
};

// Handle errors
ws.onerror = (error) => {
    console.error('WebSocket error:', error);
};
```

### Server Side — Spring Boot

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatHandler(), "/ws/chat")
                .setAllowedOrigins("*");

        registry.addHandler(orderHandler(), "/ws/orders")
                .setAllowedOrigins("*");
    }
}

@Component
public class OrderWebSocketHandler extends TextWebSocketHandler {

    // Track active connections
    private final Map<String, WebSocketSession> sessions =
        new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = extractUserId(session);
        sessions.put(userId, session);
        log.info("User {} connected. Total: {}", userId, sessions.size());
    }

    @Override
    protected void handleTextMessage(WebSocketSession session,
                                     TextMessage message) throws Exception {
        OrderRequest request = objectMapper.readValue(
            message.getPayload(), OrderRequest.class);
        subscribeToOrder(session, request.getOrderId());
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session,
                                      CloseStatus status) {
        String userId = extractUserId(session);
        sessions.remove(userId);
    }

    // Push update to specific user
    public void sendOrderUpdate(String userId, OrderUpdate update) {
        WebSocketSession session = sessions.get(userId);
        if (session != null && session.isOpen()) {
            try {
                session.sendMessage(new TextMessage(
                    objectMapper.writeValueAsString(update)));
            } catch (IOException e) {
                sessions.remove(userId);
            }
        }
    }
}
```

### STOMP Over WebSockets — Spring's Preferred Approach

Raw WebSockets are low-level — you manage message routing yourself. STOMP (Simple Text Oriented Messaging Protocol) adds a messaging layer on top, like HTTP on top of TCP.

```java
@Configuration
@EnableWebSocketMessageBroker
public class StompConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Client subscribes to these prefixes
        registry.enableSimpleBroker("/topic", "/queue");
        // Client sends to these prefixes
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS(); // fallback for older browsers
    }
}

@Controller
public class OrderController {

    private final SimpMessagingTemplate messagingTemplate;

    // Client sends to /app/order.update
    @MessageMapping("/order.update")
    public void updateOrder(OrderMessage message) {
        orderService.update(message);

        // Broadcast to all subscribers of /topic/orders
        messagingTemplate.convertAndSend(
            "/topic/orders/" + message.getOrderId(),
            new OrderStatusUpdate(message.getOrderId(), "SHIPPED"));

        // Send to specific user's private queue
        messagingTemplate.convertAndSendToUser(
            message.getUserId(),
            "/queue/notifications",
            new Notification("Your order has shipped!"));
    }
}
```

STOMP gives you pub/sub semantics, user-specific queues, and message routing — all over WebSockets.

---

## WebSocket Scaling Challenges

### The Stickiness Problem

```
User A connected to Server 1
Message for User A arrives → which server handles it?

Server 2 receives message for User A
Server 2 doesn't have User A's WebSocket connection
Server 2 can't deliver the message

Solution 1: Sticky sessions
  Load balancer always routes User A to Server 1
  Problem: loses benefit of horizontal scaling
           Server 1 failure = all its users disconnected

Solution 2: Message broker (correct approach)
  All servers subscribe to a shared pub/sub system (Redis)
  Any server receives message → publishes to Redis
  Redis delivers to correct server → server delivers to user
```

### Scaling with Redis Pub/Sub

```
User A ──WebSocket──► Server 1
User B ──WebSocket──► Server 2
User C ──WebSocket──► Server 3

Message: "Update for User A"
                ↓
Any server receives it
                ↓
        [Redis Pub/Sub]
        /       |       \
   Server 1  Server 2  Server 3
     ↓
Server 1 has User A's connection
Server 1 delivers to User A
```

```java
// Spring Boot + Redis pub/sub for WebSocket scaling
@Service
public class WebSocketMessageRelay {

    private final RedisTemplate<String, String> redisTemplate;
    private final OrderWebSocketHandler wsHandler;

    // When this server needs to send to a user on another server
    public void relayMessage(String userId, OrderUpdate update) {
        String channel = "user:" + userId;
        redisTemplate.convertAndSend(channel,
            objectMapper.writeValueAsString(update));
    }

    // Listen for messages from other servers
    @RedisListener(channels = "user:*")
    public void onMessage(String message, String channel) {
        String userId = channel.replace("user:", "");
        wsHandler.sendOrderUpdate(userId,
            objectMapper.readValue(message, OrderUpdate.class));
    }
}
```

### Connection Limits

```
Each WebSocket = persistent TCP connection = file descriptor
OS default: ~1024 file descriptors per process

1 server = ~65,000 connections maximum (with tuning)
1 million concurrent users = ~16 servers minimum

Tuning:
  ulimit -n 1000000  (increase file descriptors)
  net.ipv4.tcp_max_syn_backlog = 65536
  net.core.somaxconn = 65536
```

---

## WebSocket vs Server-Sent Events (SSE)

```
WebSocket:
  Full duplex — client AND server can send
  Binary or text
  Custom protocol over TCP
  More complex to scale
  Use: chat, gaming, collaborative editing, trading

SSE (Server-Sent Events):
  One direction only — server pushes to client
  Text only (UTF-8)
  Built on HTTP — works with HTTP/2 naturally
  Easier to scale (stateless-ish, works with CDN)
  Automatic reconnection built into browser
  Use: live feeds, notifications, dashboards, progress updates
```

```java
// SSE is simpler when you only need server→client:
@GetMapping(value = "/orders/{id}/stream",
            produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<OrderUpdate> streamOrderUpdates(@PathVariable String id) {
    return orderService.getUpdateStream(id); // Reactor Flux
}
```

---

## When to Use WebSockets

```
Use WebSockets when:
✅ Real-time bidirectional communication required
   → Chat applications (WhatsApp, Slack)
   → Collaborative editing (Google Docs)
   → Multiplayer gaming
   → Live trading dashboards
   → Real-time order tracking

Use SSE when:
✅ Server-to-client only (read-heavy real-time)
   → Live sports scores
   → News feeds
   → Progress bars
   → Log streaming

Use Long Polling when:
✅ Simplicity over performance, infrequent updates
   → Legacy systems, simple notifications

Use HTTP polling when:
✅ Updates are infrequent (every few minutes)
   → Dashboard refresh every 5 minutes
   → Not worth WebSocket complexity
```

---

## WebSockets in System Design Interviews

When designing a chat system or real-time feature:

> "For real-time messaging I'd use WebSockets. Each client maintains a persistent WebSocket connection to a connection server. When a message is sent, it's published to Kafka. A message delivery service consumes from Kafka and delivers to the recipient's connection server via Redis pub/sub — that server pushes to the recipient's WebSocket. This decouples message persistence (Kafka) from message delivery (WebSocket), and Redis pub/sub allows any connection server to deliver to any user regardless of which server they're connected to."

That answer demonstrates: WebSocket mechanics, horizontal scaling solution, separation of concerns, and Kafka integration — all in one flow.

---

## Key Takeaways

```
WebSocket: Persistent full-duplex TCP connection
           Client OR server can send anytime
           Minimal per-message overhead (2 bytes vs 500 bytes HTTP)

Handshake: HTTP Upgrade → 101 Switching Protocols
           Connection stays open, no more HTTP

vs Polling:     WebSocket = instant push, minimal overhead
vs Long Poll:   WebSocket = true bidirectional, lower overhead
vs SSE:         WebSocket = bidirectional; SSE = server→client only

Scaling challenge: Connections are stateful
Solution:         Redis pub/sub relay between servers
                  Any server can deliver to any user

Spring Boot:
  Raw WebSocket: WebSocketHandler
  STOMP:         @EnableWebSocketMessageBroker, pub/sub built in
  SSE:           Flux with TEXT_EVENT_STREAM_VALUE

Use when:
  Chat, gaming, collaboration, live trading, real-time tracking
```
