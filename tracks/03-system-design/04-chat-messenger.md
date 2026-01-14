# 04. Chat / Messenger

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- 1:1 messaging
- Group chats (up to 500 members)
- Online/offline status (presence)
- Read receipts
- Media sharing (images, files)
- Message history & search

### Нефункциональные
- Real-time delivery (< 500ms)
- High availability (99.99%)
- Message ordering guaranteed
- Scale: 500M DAU

---

## Capacity Estimation

```
Messages:
- 500M DAU × 40 messages/user/day = 20B messages/day
- QPS = 20B / 86400 ≈ 230,000 QPS

Connections:
- 500M concurrent WebSocket connections (peak)
- Each connection: ~10KB memory
- Total: 5 PB RAM (distributed across servers)

Storage:
- Message: ~200 bytes average
- 20B × 200B = 4TB/day
- 5 years retention: 7.3 PB
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Client  │◄───▶│  Load Balancer  │◄───▶│  Chat Server     │
│  (App)   │     │  (WebSocket)    │     │  (WebSocket Hub) │
└──────────┘     └─────────────────┘     └────────┬─────────┘
                                                  │
          ┌───────────────────┬───────────────────┼───────────────────┐
          │                   │                   │                   │
    ┌─────▼─────┐      ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
    │  Presence │      │   Message   │     │   Message   │     │    Media    │
    │  Service  │      │   Service   │     │    Queue    │     │   Service   │
    └─────┬─────┘      └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
          │                   │                   │                   │
    ┌─────▼─────┐      ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
    │   Redis   │      │  Cassandra  │     │    Kafka    │     │     S3      │
    │  (Cache)  │      │ (Messages)  │     │             │     │             │
    └───────────┘      └─────────────┘     └─────────────┘     └─────────────┘
```

---

## API Design

### WebSocket Events

```javascript
// Client → Server
{
    "type": "send_message",
    "data": {
        "conversation_id": "conv_123",
        "content": "Hello!",
        "type": "text"
    }
}

{
    "type": "typing",
    "data": {
        "conversation_id": "conv_123",
        "is_typing": true
    }
}

{
    "type": "mark_read",
    "data": {
        "conversation_id": "conv_123",
        "message_id": "msg_456"
    }
}

// Server → Client
{
    "type": "new_message",
    "data": {
        "message_id": "msg_789",
        "conversation_id": "conv_123",
        "sender_id": "user_456",
        "content": "Hi there!",
        "timestamp": 1623456789
    }
}

{
    "type": "presence_update",
    "data": {
        "user_id": "user_456",
        "status": "online",
        "last_seen": 1623456789
    }
}
```

### REST API

```
GET  /api/v1/conversations
GET  /api/v1/conversations/{id}/messages?before={timestamp}&limit=50
POST /api/v1/conversations/{id}/messages
POST /api/v1/media/upload
```

---

## Data Model

### Messages (Cassandra)
```sql
-- Partition by conversation_id for sequential reads
CREATE TABLE messages (
    conversation_id UUID,
    message_id      TIMEUUID,
    sender_id       UUID,
    content         TEXT,
    message_type    TEXT,  -- text, image, file
    media_url       TEXT,
    created_at      TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

### Conversations (PostgreSQL)
```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY,
    type            VARCHAR(20),  -- direct, group
    name            VARCHAR(255),
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);

CREATE TABLE conversation_members (
    conversation_id UUID REFERENCES conversations(id),
    user_id         UUID,
    role            VARCHAR(20),  -- admin, member
    joined_at       TIMESTAMP,
    last_read_at    TIMESTAMP,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX idx_user_conversations ON conversation_members(user_id);
```

### User Presence (Redis)
```
# Online status
SET user:123:presence online EX 60

# Last seen
SET user:123:last_seen 1623456789

# Active connections
SADD user:123:connections server_456
```

---

## Deep Dive

### 1. WebSocket Connection Management

```
┌─────────────────────────────────────────────────────────────┐
│                    WebSocket Gateway                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Connection Manager                      │    │
│  │  user_123 → [conn_a, conn_b]  (multiple devices)    │    │
│  │  user_456 → [conn_c]                                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                            │
                  ┌─────────┴─────────┐
                  │   Redis Pub/Sub   │
                  │ (cross-server)    │
                  └───────────────────┘
```

```python
class ConnectionManager:
    def __init__(self):
        self.connections = {}  # user_id -> set of WebSocket

    async def connect(self, user_id: str, websocket: WebSocket):
        await websocket.accept()
        if user_id not in self.connections:
            self.connections[user_id] = set()
        self.connections[user_id].add(websocket)

        # Notify presence service
        await redis.sadd(f"user:{user_id}:connections", server_id)
        await redis.publish("presence", json.dumps({
            "user_id": user_id,
            "status": "online"
        }))

    async def send_to_user(self, user_id: str, message: dict):
        # Local connections
        if user_id in self.connections:
            for ws in self.connections[user_id]:
                await ws.send_json(message)

        # Remote connections (via Redis)
        servers = await redis.smembers(f"user:{user_id}:connections")
        for server in servers:
            if server != server_id:
                await redis.publish(f"server:{server}", json.dumps({
                    "user_id": user_id,
                    "message": message
                }))
```

### 2. Message Delivery Flow

```
Sender                  Server                    Recipient
  │                       │                           │
  │──── Send Message ────▶│                           │
  │                       │                           │
  │                       │── Save to Cassandra      │
  │                       │                           │
  │◀── ACK (sent) ────────│                           │
  │                       │                           │
  │                       │── Push via WebSocket ────▶│
  │                       │                           │
  │                       │◀── ACK (delivered) ───────│
  │                       │                           │
  │◀── Delivered Status ──│                           │
```

```python
async def handle_message(sender_id: str, message: dict):
    conversation_id = message['conversation_id']
    msg_id = generate_timeuuid()

    # 1. Save to database
    await cassandra.execute("""
        INSERT INTO messages (conversation_id, message_id, sender_id, content, created_at)
        VALUES (%s, %s, %s, %s, %s)
    """, (conversation_id, msg_id, sender_id, message['content'], datetime.utcnow()))

    # 2. Send ACK to sender
    await send_to_user(sender_id, {
        "type": "message_sent",
        "message_id": str(msg_id)
    })

    # 3. Fan-out to recipients
    members = await get_conversation_members(conversation_id)
    for member_id in members:
        if member_id != sender_id:
            await send_to_user(member_id, {
                "type": "new_message",
                "message_id": str(msg_id),
                "conversation_id": conversation_id,
                "sender_id": sender_id,
                "content": message['content']
            })
```

### 3. Ordering & Consistency

**Problem:** Messages can arrive out of order

**Solution:** TIMEUUID + Client-side sorting

```python
# Server assigns TIMEUUID
message_id = uuid.uuid1()  # Time-based UUID

# Client sorts by message_id (which encodes timestamp)
messages.sort(key=lambda m: m['message_id'])
```

**Handling offline users:**
```python
async def deliver_message(user_id: str, message: dict):
    is_online = await redis.exists(f"user:{user_id}:presence")

    if is_online:
        await send_via_websocket(user_id, message)
    else:
        # Queue for push notification
        await push_queue.send({
            "user_id": user_id,
            "message": message
        })
```

### 4. Presence System

```python
class PresenceService:
    HEARTBEAT_INTERVAL = 30  # seconds
    TIMEOUT = 60  # seconds

    async def heartbeat(self, user_id: str):
        # Update presence with TTL
        await redis.setex(f"user:{user_id}:presence", self.TIMEOUT, "online")
        await redis.set(f"user:{user_id}:last_seen", int(time.time()))

        # Notify friends
        friends = await get_friends(user_id)
        await self.broadcast_presence(user_id, "online", friends)

    async def get_status(self, user_id: str) -> dict:
        is_online = await redis.exists(f"user:{user_id}:presence")
        last_seen = await redis.get(f"user:{user_id}:last_seen")

        return {
            "user_id": user_id,
            "status": "online" if is_online else "offline",
            "last_seen": int(last_seen) if last_seen else None
        }
```

### 5. Group Chat Optimization

```
Small groups (<100): Fan-out on write
Large groups (>100): Fan-out on read

┌────────────────────────────────────┐
│           Group Message            │
└────────────────┬───────────────────┘
                 │
    ┌────────────┼────────────┐
    │ <100 members            │ >100 members
    ▼                         ▼
Fan-out write             Store once
(push to each)            (pull on read)
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Hot conversations | Partition by conversation_id |
| Connection limit per server | Horizontal scaling, sticky sessions |
| Cross-server messaging | Redis Pub/Sub |
| Large groups | Fan-out on read |
| Media storage | S3 + CDN |

---

## Trade-offs

### Poll vs Push
| Approach | Latency | Resources | Battery |
|----------|---------|-----------|---------|
| WebSocket | <100ms | High (connections) | Medium |
| Long Polling | 1-30s | Medium | Higher |
| Push Notification | Variable | Low | Low |

### Storage
| Database | Use case | Trade-off |
|----------|----------|-----------|
| Cassandra | Messages | High write throughput, eventual consistency |
| PostgreSQL | Conversations, users | ACID, complex queries |
| Redis | Presence, sessions | Speed, volatility |

---

## На интервью

### Ключевые моменты
1. **WebSocket management** — connection tracking, cross-server messaging
2. **Message ordering** — TIMEUUID, client-side sorting
3. **Presence** — heartbeat, TTL, fan-out
4. **Scalability** — partition by conversation, fan-out strategies

### Типичные follow-up
- Как синхронизировать историю на новом устройстве?
- Как реализовать end-to-end encryption?
- Как доставить сообщение offline пользователю?
- Как обработать группу с 10K участников?

---

[← Назад к списку тем](README.md)
