# 04 — Chat Messenger

Развёрнутые вопросы и ответы про систему чат-мессенджера: архитектура real-time доставки, управление WebSocket соединениями, presence система, упорядочивание сообщений, масштабирование для миллионов пользователей. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [03-notification-system](./03-notification-system.md) · Следующая тема: [05-news-feed](./05-news-feed.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**WebSocket** — протокол для полнодуплексной двусторонней связи над TCP соединением, который позволяет серверу и клиенту отправлять данные независимо друг от друга в реальном времени. WebSocket более эффективен, чем HTTP polling, так как использует одно постоянное соединение вместо множества повторных запросов. WebSocket является стандартом для real-time приложений, включая чат.

**Handshake** — процесс установления WebSocket соединения, при котором происходит HTTP upgrade запрос. Клиент отправляет специальный HTTP запрос с заголовком Upgrade: websocket, а сервер отвечает 101 Switching Protocols, преобразуя HTTP соединение в постоянное WebSocket. После handshake можно отправлять и получать сообщения в обе стороны.

**Persistent connection** — постоянное открытое TCP соединение между клиентом и сервером, которое остаётся живым между отправками сообщений. Persistent connection позволяет избежать overhead переподключения и обеспечивает low latency для real-time доставки. Приложение должно обнаруживать разрывы соединения и переподключаться при необходимости.

**Presence** — информация о том, онлайн ли пользователь (есть ли у него активное соединение) или офлайн. Presence система отслеживает какие пользователи находятся в чате в данный момент и рассылает обновления статуса всем членам группы. Обычно реализуется с Redis для быстрого доступа. Presence критична для UX, так как показывает доступность для общения.

**Heartbeat/Ping** — периодический сигнал (обычно каждые 30-60 секунд), который отправляется между клиентом и сервером для проверки живого соединения. Heartbeat позволяет обнаружить разорванное соединение (например, из-за выключенного Wi-Fi) без явных ошибок. При получении heartbeat обе стороны отвечают Pong, подтверждая живость соединения.

**Message ordering** — гарантия, что сообщения доставляются в порядке их отправления. Message ordering критична для когерентного диалога, когда контекст одного сообщения зависит от предыдущих. Требует присвоения sequence numbers сообщениям и хранения их в порядке в БД. Особенно сложно в распределённых системах.

**Read receipts** — подтверждение, что сообщение было прочитано адресатом (в отличие от "доставлено", но не прочитано). Read receipts улучшают UX, позволяя пользователям видеть, видел ли собеседник их сообщение. Требует отслеживания статуса для каждого сообщения в каждом чате. Может быть дорого в масштабируемых системах.

**Cassandra** — распределённая NoSQL база данных, оптимизированная для записи временных рядов (time-series) данных. Cassandra идеальна для хранения миллиардов сообщений чата благодаря высокой пропускной способности записей, горизонтальной масштабируемости и отказоустойчивости. Её партиционирование по ключам позволяет хранить все сообщения одного чата вместе.

**Sticky session** — техника, при которой load balancer привязывает клиента к одному и тому же серверу, чтобы весь трафик от клиента шёл на один сервер. Для WebSocket это упрощает управление состоянием соединения, так как весь контекст находится на одном сервере. Однако sticky sessions усложняют балансирование нагрузки и отказоустойчивость.

**Fan-out** — процесс копирования сообщения и отправки его всем адресатам (членам чата). Fan-out обеспечивает доставку сообщения каждому участнику группы чата. Это может быть сделано синхронно (медленно) или асинхронно через task queue (быстро). Fan-out — критичная операция для масштабируемости больших групп.

---

## Вопросы и разборы

### 1. Как спроектировать систему чат-мессенджера на 500M DAU?

**Зачем спрашивают.** Chat — это real-time система с высокими требованиями к пропускной способности и доступности. Интервьюер проверяет понимание масштабирования, управления соединениями и архитектуры.

**Короткий ответ.** Размерить capacity: 500M DAU × 40 сообщений/день = 230K QPS. Использовать WebSocket для real-time доставки, Cassandra для сообщений (write-heavy, time-series), Redis для presence, Kafka для асинхронной доставки. Партиционировать по conversation_id.

**Детальный разбор.**

**Фаза 1: Requirements gathering**
```
Функциональные требования:
├─ 1:1 messaging (direct conversations)
├─ Group chats (до 500 members)
├─ Online/offline status (presence)
├─ Read receipts (delivered, read)
├─ Media sharing (images, files up to 100MB)
└─ Message history & search

Нефункциональные требования:
├─ Real-time delivery (< 500ms latency)
├─ High availability (99.99% uptime)
├─ Message ordering guaranteed
├─ Message persistence (5 years)
└─ Scale: 500M DAU, 50M concurrent WebSocket connections
```

**Фаза 2: Capacity estimation**
```
DAU = 500M users
Active conversations per user = 8
Messages per user per day = 40
Concurrent users at peak = 10% of DAU = 50M

Message throughput:
  500M DAU × 40 messages/day / 86400 sec = 230,000 QPS

WebSocket connections:
  50M concurrent × 10KB per connection = 500GB RAM
  Distributed: 50K servers × 10GB = 500GB total

Message storage:
  20B messages/day × 200 bytes/message = 4TB/day
  5 years: 4TB × 365 × 5 = 7.3 PB
```

**Фаза 3: High-level architecture**
```
┌──────────────────────────────────────────────────────────────────┐
│                       Client Apps                                 │
│           (Web, iOS, Android, Desktop)                            │
└────────────┬─────────────────────────────────┬───────────────────┘
             │                                 │
    ┌────────▼────────┐            ┌──────────▼───────┐
    │   Load Balancer │            │  CDN (for media) │
    │  (WebSocket)    │            │                  │
    └────────┬────────┘            └──────────────────┘
             │
    ┌────────▼─────────────────────────────────────┐
    │        Chat Server Cluster                   │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
    │  │Server 1  │  │Server 2  │  │Server N  │    │
    │  │          │  │          │  │          │    │
    │  │WS Handler│  │WS Handler│  │WS Handler│    │
    │  └──────────┘  └──────────┘  └──────────┘    │
    └────┬───────────────────────────────────┬─────┘
         │                                   │
    ┌────▼────┐  ┌─────────┐  ┌───────────┐ │
    │  Kafka  │  │ Redis   │  │  Service  │ │
    │ (Queue) │  │ (Cache) │  │  Registry │ │
    └────┬────┘  └─────────┘  └───────────┘ │
         │                                   │
    ┌────▼──────────────────────────────────▼────┐
    │   Data Layer                               │
    │  ┌─────────────┐  ┌──────────────────┐    │
    │  │  Cassandra  │  │   PostgreSQL     │    │
    │  │ (Messages)  │  │ (Conversations,  │    │
    │  │ (Time-based)│  │  Users, Members) │    │
    │  └─────────────┘  └──────────────────┘    │
    │  ┌─────────────────────────────────────┐   │
    │  │ S3 (Media storage) + CDN            │   │
    │  └─────────────────────────────────────┘   │
    └──────────────────────────────────────────┘
```

**Компоненты:**
- **Load Balancer** — sticky sessions для привязки пользователя к серверу
- **Chat Servers** — управление WebSocket соединениями, fan-out сообщений
- **Presence Service** — online/offline status, last_seen
- **Message Service** — API для истории, поиска
- **Message Queue (Kafka)** — асинхронная доставка, offline-mode
- **Cache (Redis)** — presence, session, rate limiting
- **Cassandra** — write-optimized storage для сообщений
- **PostgreSQL** — conversations, members, metadata
- **S3 + CDN** — media storage и delivery

**Пример.**
```python
# Capacity Planning Spreadsheet
DAU = 500_000_000
Active_Percent = 0.1  # 10% concurrent
Messages_Per_User_Per_Day = 40
Message_Size = 200  # bytes

# Throughput
concurrent_users = DAU * Active_Percent  # 50M
messages_per_sec = (DAU * Messages_Per_User_Per_Day) / 86400  # 230K QPS

# Connection memory
connection_memory = concurrent_users * 10 * 1024  # 500GB
servers_needed = connection_memory / (10 * 1024**3)  # ~50K servers

# Storage
daily_storage = messages_per_sec * 86400 * Message_Size  # 4TB/day
yearly_storage = daily_storage * 365  # 1.46PB/year

print(f"""
Concurrent Users: {concurrent_users:,}
QPS: {messages_per_sec:,.0f}
Servers Needed: {servers_needed:,.0f}
Daily Storage: {daily_storage / 1024**4:.2f} TB
Yearly Storage: {yearly_storage / 1024**5:.2f} PB
""")
```

**Типичные ошибки.**
- Использовать MySQL вместо Cassandra — не выдержит write throughput.
- Отсутствие sticky sessions — переподключения, потеря состояния.
- Синхронное сохранение в DB перед отправкой — высокая latency.
- Не учитывать offline users — нужна очередь сообщений.

**На интервью.**
- Начни с требований и requirements.
- Размерь capacity на бумаге в реальном времени.
- Объясни, почему Cassandra а не PostgreSQL для сообщений.
- Follow-up: «Как обеспечить message ordering?» — TIMEUUID + client-side sorting.

---

### 2. Как работает WebSocket для real-time коммуникации?

**Зачем спрашивают.** WebSocket — основа real-time систем. Интервьюер проверяет понимание протокола, управления соединениями и отличие от HTTP.

**Короткий ответ.** WebSocket — это полнодуплексный протокол с постоянным соединением TCP. После handshake (upgrade from HTTP) оба конца могут отправлять данные асинхронно. Он более эффективен для real-time, чем polling или long polling, потому что не требует repeated handshakes.

**Детальный разбор.**

**WebSocket vs альтернативы:**
```
┌─────────────┬──────────────┬──────────────┬────────────┐
│ Mechanism   │ Latency      │ Resources    │ Use Case   │
├─────────────┼──────────────┼──────────────┼────────────┤
│ HTTP Polling│ 5-10 seconds │ Very High    │ Legacy     │
│ Long Polling│ 1-3 seconds  │ High         │ Fallback   │
│ SSE (Server │ 100-500ms    │ Medium       │ One-way    │
│ Sent Events)│              │              │ push       │
│ WebSocket   │ <100ms       │ Low          │ Real-time  │
│ gRPC        │ <50ms        │ Medium       │ Backend    │
└─────────────┴──────────────┴──────────────┴────────────┘

HTTP Polling:
Client                          Server
   │                               │
   │──── GET /messages ────────────│
   │                               │
   │◀─── 200 OK (empty) ───────────│
   │                               │
   │ (pause 5s)                    │
   │                               │
   │──── GET /messages ────────────│
   │                               │
   │◀─── 200 OK (new message) ─────│


WebSocket:
Client                          Server
   │                               │
   │──── WebSocket Handshake ─────│
   │                               │
   │◀─── 101 Switching Protocols ──│
   │                               │
   │  (persistent bidirectional connection)
   │                               │
   │◀─── Message 1 ────────────────│
   │                               │
   │──── Message 2 ────────────────│
   │                               │
   │  (latency < 100ms)            │
```

**WebSocket Handshake:**
```
Client requests upgrade:
GET /chat HTTP/1.1
Host: server.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Version: 13

Server responds:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
```

**Фреймы WebSocket:**
```
┌─────────────────────────────────────────────┐
│ FIN │ RSV │ Opcode │ MASK │ Payload Length │
├─────────────────────────────────────────────┤
│  1  │     │ 0x1    │  1   │      n bytes   │
│     │     │(Text)  │      │                │
├─────────────────────────────────────────────┤
│           Masking Key (4 bytes)             │
├─────────────────────────────────────────────┤
│           Payload Data (XOR masked)         │
└─────────────────────────────────────────────┘

Opcodes:
0x0 = Continuation frame
0x1 = Text frame
0x2 = Binary frame
0x8 = Close
0x9 = Ping (keep-alive)
0xA = Pong (keep-alive response)
```

**Жизненный цикл соединения:**
```
1. CONNECTING (handshake in progress)
   │
   ├─ Success ──► OPEN
   │              │
   │              ├─ Send message ──► data sent
   │              │
   │              ├─ Receive message ◄── data received
   │              │
   │              ├─ Network error ──┐
   │              │                  │
   │              └─ Close ─────────┐│
   │                                ││
   └─ Failure ──────────────────────┼┼──► CLOSED
                                    ││
                                    └┴──► CLOSED
                                         (reconnect from CONNECTING)
```

**Пример.**
```javascript
// Client-side WebSocket
class ChatClient {
  constructor(userId, serverUrl) {
    this.userId = userId;
    this.ws = null;
    this.serverUrl = serverUrl;
    this.messageQueue = [];
    this.isConnected = false;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    this.reconnectDelay = 1000; // ms
  }

  connect() {
    try {
      this.ws = new WebSocket(this.serverUrl);

      this.ws.onopen = () => {
        console.log('Connected to chat server');
        this.isConnected = true;
        this.reconnectAttempts = 0;

        // Flush queued messages
        while (this.messageQueue.length > 0) {
          const msg = this.messageQueue.shift();
          this.send(msg);
        }

        // Start heartbeat
        this.startHeartbeat();
      };

      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        this.handleMessage(data);
      };

      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        this.isConnected = false;
      };

      this.ws.onclose = () => {
        console.log('Disconnected from chat server');
        this.isConnected = false;
        this.stopHeartbeat();
        this.reconnect();
      };
    } catch (error) {
      console.error('Connection failed:', error);
      this.reconnect();
    }
  }

  send(message) {
    if (this.isConnected && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    } else {
      // Queue message for later
      this.messageQueue.push(message);
      if (!this.isConnected) {
        this.connect();
      }
    }
  }

  sendMessage(conversationId, content) {
    this.send({
      type: 'send_message',
      data: {
        conversation_id: conversationId,
        content: content,
        timestamp: Date.now()
      }
    });
  }

  handleMessage(data) {
    switch (data.type) {
      case 'new_message':
        this.onMessageReceived(data.data);
        break;
      case 'message_sent':
        this.onMessageSent(data.data);
        break;
      case 'presence_update':
        this.onPresenceUpdate(data.data);
        break;
      case 'typing_indicator':
        this.onTyping(data.data);
        break;
    }
  }

  startHeartbeat() {
    this.heartbeatInterval = setInterval(() => {
      if (this.isConnected) {
        this.send({ type: 'ping' });
      }
    }, 30000); // Every 30 seconds
  }

  stopHeartbeat() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
    }
  }

  reconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
      console.log(`Reconnecting in ${delay}ms...`);
      setTimeout(() => this.connect(), delay);
    } else {
      console.error('Max reconnection attempts reached');
      // Notify user, suggest page refresh
    }
  }

  disconnect() {
    this.stopHeartbeat();
    if (this.ws) {
      this.ws.close();
    }
  }
}

// Server-side (Python/FastAPI)
from fastapi import WebSocket, WebSocketDisconnect
import json
import asyncio

class ConnectionManager:
  def __init__(self):
    # user_id -> list of WebSocket connections
    self.active_connections: dict[str, list[WebSocket]] = {}

  async def connect(self, websocket: WebSocket, user_id: str):
    await websocket.accept()

    if user_id not in self.active_connections:
      self.active_connections[user_id] = []

    self.active_connections[user_id].append(websocket)
    print(f"User {user_id} connected. Total connections: {len(self.active_connections[user_id])}")

  def disconnect(self, user_id: str, websocket: WebSocket):
    if user_id in self.active_connections:
      self.active_connections[user_id].remove(websocket)
      if not self.active_connections[user_id]:
        del self.active_connections[user_id]
    print(f"User {user_id} disconnected")

  async def send_to_user(self, user_id: str, data: dict):
    """Send message to specific user's all connections"""
    if user_id in self.active_connections:
      disconnected = []
      for connection in self.active_connections[user_id]:
        try:
          await connection.send_json(data)
        except Exception as e:
          print(f"Error sending to {user_id}: {e}")
          disconnected.append(connection)

      # Clean up dead connections
      for conn in disconnected:
        self.active_connections[user_id].remove(conn)

  async def broadcast_to_conversation(self, conversation_id: str,
                                      sender_id: str, data: dict):
    """Send message to all users in conversation"""
    members = await get_conversation_members(conversation_id)
    for member_id in members:
      if member_id != sender_id:  # Don't send back to sender
        await self.send_to_user(member_id, data)

manager = ConnectionManager()

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
  await manager.connect(websocket, user_id)

  try:
    while True:
      # Receive message from client
      data = await websocket.receive_json()

      if data['type'] == 'send_message':
        # Process message
        await handle_send_message(user_id, data['data'])

      elif data['type'] == 'typing':
        # Broadcast typing indicator
        await manager.broadcast_to_conversation(
          data['data']['conversation_id'],
          user_id,
          {
            'type': 'typing_indicator',
            'data': {
              'user_id': user_id,
              'conversation_id': data['data']['conversation_id']
            }
          }
        )

  except WebSocketDisconnect:
    manager.disconnect(user_id, websocket)
```

**Типичные ошибки.**
- Отсутствие heartbeat/ping — соединение может разорваться без уведомления.
- Синхронная обработка сообщений — блокирует другие соединения.
- Отсутствие очереди сообщений — сообщения теряются при переподключении.
- Не закрывать соединения корректно — утечка памяти.

**На интервью.**
- Объясни процесс WebSocket handshake.
- Покажи разницу между WebSocket и long polling на диаграмме.
- Упомяни heartbeat для обнаружения мертвых соединений.
- Follow-up: «Как масштабировать WebSocket на несколько серверов?» — Redis Pub/Sub для cross-server messaging.

---

### 3. Как реализовать presence (online/offline статус)?

**Зачем спрашивают.** Presence — видимая часть системы, требует консистентности и высокой доступности. Интервьюер проверяет понимание trade-offs между accuracy и масштабируемостью.

**Короткий ответ.** Хранить presence в Redis с TTL. Клиент отправляет heartbeat каждые 30 секунд. Если heartbeat не поступает — ключ в Redis истекает и пользователь считается offline. Broadcast presence updates через WebSocket или Redis Pub/Sub.

**Детальный разбор.**

**Архитектура presence:**
```
┌─────────────────────────────────────────────────────┐
│                    Chat Server                       │
│  ┌──────────────────────────────────────────────┐   │
│  │         WebSocket Handler                    │   │
│  │  ┌────────────────────────────────────────┐  │   │
│  │  │ On Connection:                         │  │   │
│  │  │  - Update user:123:presence → "online" │  │   │
│  │  │  - Add to user:123:connections → srv_1 │  │   │
│  │  │  - Publish to friends' subscribers     │  │   │
│  │  └────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────┐  │   │
│  │  │ On Disconnect:                         │  │   │
│  │  │  - Remove from user:123:connections    │  │   │
│  │  │  - If no more connections:             │  │   │
│  │  │    - Delete user:123:presence          │  │   │
│  │  │    - Set user:123:last_seen → timestamp│  │   │
│  │  │    - Publish offline event to friends  │  │   │
│  │  └────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────┬───────────────────────────────────────┘
              │
    ┌─────────▼──────────┐
    │   Redis (Cluster)  │
    │                    │
    │ user:123:presence  │
    │  → "online"        │
    │  → TTL: 60s        │
    │                    │
    │ user:123:last_seen │
    │  → 1234567890      │
    │                    │
    │ user:123:connections
    │  → {srv_1, srv_2}  │
    └────────────────────┘
```

**Состояние presence:**
```
Online:
  - Есть активное WebSocket соединение
  - Redis ключ существует (TTL не истек)
  - Последний heartbeat < 60 секунд

Idle (optional):
  - Нет активности > 5 минут
  - WebSocket соединение живо
  - Redis ключ есть, но mark as idle

Offline:
  - Нет WebSocket соединений
  - Redis ключ истек (TTL истек)
  - Есть last_seen timestamp
```

**Пример.**
```python
import redis
import asyncio
import time
from datetime import datetime, timedelta
from typing import List, Set

class PresenceService:
  def __init__(self, redis_client: redis.Redis):
    self.redis = redis_client
    self.heartbeat_interval = 30  # seconds
    self.presence_ttl = 60  # seconds (2x heartbeat)
    self.server_id = "chat_server_1"

  async def on_user_connected(self, user_id: str):
    """Called when user establishes WebSocket connection"""

    # 1. Set presence in Redis with TTL
    presence_key = f"user:{user_id}:presence"
    self.redis.setex(presence_key, self.presence_ttl, "online")

    # 2. Add server to connection set (for tracking where user is connected)
    connections_key = f"user:{user_id}:connections"
    self.redis.sadd(connections_key, self.server_id)
    self.redis.expire(connections_key, self.presence_ttl)

    # 3. Update last_seen
    last_seen_key = f"user:{user_id}:last_seen"
    self.redis.set(last_seen_key, int(time.time()))

    # 4. Broadcast presence update to contacts
    await self.broadcast_presence(user_id, "online")

    print(f"User {user_id} connected (server: {self.server_id})")

  async def on_user_disconnected(self, user_id: str):
    """Called when user closes WebSocket connection"""

    # 1. Remove server from connections set
    connections_key = f"user:{user_id}:connections"
    self.redis.srem(connections_key, self.server_id)

    # 2. Check if user has any more connections
    remaining_connections = self.redis.scard(connections_key)

    if remaining_connections == 0:
      # User is fully offline
      presence_key = f"user:{user_id}:presence"
      self.redis.delete(presence_key)

      # Update last_seen
      last_seen_key = f"user:{user_id}:last_seen"
      self.redis.set(last_seen_key, int(time.time()))

      # Broadcast offline status
      await self.broadcast_presence(user_id, "offline")

    print(f"User {user_id} disconnected from {self.server_id}")

  async def heartbeat(self, user_id: str):
    """Called periodically to keep presence fresh"""

    presence_key = f"user:{user_id}:presence"
    connections_key = f"user:{user_id}:connections"

    # Refresh TTL
    self.redis.expire(presence_key, self.presence_ttl)
    self.redis.expire(connections_key, self.presence_ttl)

    # Update last_seen (but don't broadcast for every heartbeat)
    last_seen_key = f"user:{user_id}:last_seen"
    self.redis.set(last_seen_key, int(time.time()))

  def get_user_status(self, user_id: str) -> dict:
    """Get current presence status"""

    presence_key = f"user:{user_id}:presence"
    last_seen_key = f"user:{user_id}:last_seen"

    is_online = self.redis.exists(presence_key) > 0
    last_seen_timestamp = self.redis.get(last_seen_key)

    return {
      'user_id': user_id,
      'status': 'online' if is_online else 'offline',
      'last_seen': int(last_seen_timestamp) if last_seen_timestamp else None,
      'last_seen_at': datetime.fromtimestamp(
        int(last_seen_timestamp)
      ).isoformat() if last_seen_timestamp else None
    }

  async def get_contacts_status(self, user_id: str) -> List[dict]:
    """Get status of all user's contacts"""

    # Get user's contacts from DB
    contacts = await self.db.get_contacts(user_id)

    statuses = []
    for contact_id in contacts:
      status = self.get_user_status(contact_id)
      statuses.append(status)

    return statuses

  async def broadcast_presence(self, user_id: str, status: str):
    """Notify contacts about presence change"""

    # Get user's contacts
    contacts = await self.db.get_contacts(user_id)

    presence_event = {
      'type': 'presence_update',
      'data': {
        'user_id': user_id,
        'status': status,
        'timestamp': int(time.time())
      }
    }

    # Send to contacts via their connections
    for contact_id in contacts:
      await self.send_to_user(contact_id, presence_event)

  async def send_to_user(self, user_id: str, message: dict):
    """Send message to all user's connections (local and remote)"""

    # Local connections on this server
    await self.local_connection_manager.send_to_user(user_id, message)

    # Remote connections via Redis Pub/Sub
    connections_key = f"user:{user_id}:connections"
    servers = self.redis.smembers(connections_key)

    for server in servers:
      if server != self.server_id.encode():
        # Publish to other servers
        channel = f"server:{server.decode()}:messages"
        self.redis.publish(channel, json.dumps({
          'user_id': user_id,
          'message': message
        }))


# Client-side heartbeat
class ClientHeartbeat:
  def __init__(self, websocket_client: ChatClient):
    self.ws = websocket_client
    self.interval = 30  # seconds
    self.task = None

  async def start(self):
    """Start sending heartbeats"""
    while True:
      try:
        self.ws.send({'type': 'ping'})
        await asyncio.sleep(self.interval)
      except Exception as e:
        print(f"Heartbeat failed: {e}")
        break

  def stop(self):
    """Stop sending heartbeats"""
    if self.task:
      self.task.cancel()
```

**Trade-offs:**
```
Approach 1: Accuracy = High (every action updates presence)
├─ Плюсы: Real-time status updates
└─ Минусы: High Redis write load, latency variance

Approach 2: TTL + Heartbeat (текущий)
├─ Плюсы: Balanced load and accuracy
└─ Минусы: ~60s max delay for offline detection

Approach 3: Presence = Low (check on demand)
├─ Плюсы: Minimal Redis writes
└─ Минусы: Stale data, query latency
```

**Типичные ошибки.**
- Broadcast presence для каждого heartbeat — излишние сообщения.
- TTL слишком короткий (< heartbeat) — false offline.
- Не удалять presence при disconnect — пользователь видится online.
- Отсутствие last_seen — невозможно показать "последний онлайн".

**На интервью.**
- Объясни архитектуру: Redis TTL + heartbeat.
- Покажи, как обрабатывать multi-device presence.
- Упомяни использование Redis Pub/Sub для cross-server broadcast.
- Follow-up: «Как обрабатывать сетевые сбои?» — graceful degradation, retry с backoff.

---

### 4. Как обеспечить ordering сообщений?

**Зачем спрашивают.** Ordering — critical для UX. Пользователи ожидают видеть сообщения в правильном порядке. Интервьюер проверяет понимание distributed ordering и eventual consistency.

**Короткий ответ.** Использовать TIMEUUID (time-based UUID v1) как message_id. Cassandra хранит с этим как clustering key, что гарантирует порядок на хранении. На клиенте сортировать сообщения по timestamp из UUID, если они пришли out-of-order из-за сетевых задержек.

**Детальный разбор.**

**Проблема ordering в distributed системах:**
```
Сценарий 1: Messages arrive out of order
Client                    Server1           Server2          DB
  │                         │                 │               │
  │─ msg1 (ts=100) ────────────►(queue)────────────►msg1───────│
  │                         │                 │               │
  │─ msg2 (ts=101) ────────────►           (slower path)      │
  │                         │                 │               │
  │                         │        (network jitter)         │
  │                         │                 │               │
  │                         │             msg2───────►msg2───┐│
  │                         │                 │              ││
  │                         │                 │         ┌────▼▼
  │                         │                 │         │ msg2, msg1  ← Wrong order!
  │                         │                 │         │
  │                         │             msg1◄────────►

Сценарий 2: Multiple users writing to group chat
User A ─ message ts=100 ──┐
User B ─ message ts=99 ───┼─► Chat Server ─► Cassandra
User C ─ message ts=101 ──┘

Какой порядок?
- Arrival order?
- Timestamp order?
- Causal order (if message depends on previous)?
```

**TIMEUUID для ordering:**
```
TIMEUUID v1 = timestamp (60 bits) + clock sequence + node ID

Example: f7d0fbaa-b4f7-11ee-abf9-8c8590c3e66d
         └─────────────────┬────────────────┘
              Contains timestamp (millisecond precision)

Advantages:
- Sortable by time
- Universally unique
- Cassandra optimized for clustering on TIMEUUID
- No external time service needed
- Detects out-of-order messages

Disadvantages:
- Server must have accurate time
- Not suitable for high-frequency messages on same node
```

**Cassandra partitioning for ordering:**
```
CREATE TABLE messages (
    conversation_id UUID,
    message_id TIMEUUID,
    sender_id UUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

Partition key: conversation_id
  ├─ Groups messages by conversation
  └─ Ensures all messages for one conversation on same node(s)

Clustering key: message_id (TIMEUUID)
  ├─ Ensures messages sorted by timestamp within partition
  └─ Efficient range queries (get messages between T1 and T2)

Write path:
conversation_id (partition) → [Server handling this partition]
                             → Write message_id in timestamp order
                             → Commit log persistence

Read path:
SELECT * FROM messages
WHERE conversation_id = ?
ORDER BY message_id DESC
LIMIT 50;
→ Gets last 50 messages in correct order
```

**Multi-server ordering strategy:**
```
Server1        Server2        Server3
  │              │              │
  ├─ msg1 ──────►├─ msg2 ──────►├─ msg3
  │              │              │
  │         (may arrive        │
  │          out of order)     │
  │              │              │
  └──────────────┼──────────────►
           Cassandra (single source of truth)
                  │
         ┌────────┼────────┐
         │        │        │
      msg1      msg2      msg3  ← Guaranteed order by TIMEUUID
```

**Client-side handling out-of-order messages:**
```python
class MessageBuffer:
  def __init__(self):
    self.messages = {}  # message_id -> message
    self.sorted_ids = []  # sorted by timestamp

  def add_message(self, message):
    """Add message, maintaining sorted order"""

    msg_id = message['message_id']
    timestamp = extract_timestamp_from_timeuuid(msg_id)

    # Binary search for correct position
    insert_pos = bisect.bisect_left(self.sorted_ids, timestamp)

    self.messages[msg_id] = message
    self.sorted_ids.insert(insert_pos, timestamp)

  def get_ordered_messages(self):
    """Return messages in correct order"""
    return [
      self.messages[msg_id]
      for msg_id in self.sorted_ids
    ]

  def handle_out_of_order_message(self, late_message):
    """When message arrives after newer ones"""
    self.add_message(late_message)
    self.notify_ui_message_reordered()  # Re-render chat


# TIMEUUID extraction
import uuid
from datetime import datetime

def extract_timestamp_from_timeuuid(timeuuid_str: str) -> int:
  """Extract Unix timestamp from TIMEUUID"""
  timeuuid = uuid.UUID(timeuuid_str)

  # TIMEUUID v1 has timestamp in specific bits
  timestamp_100ns = timeuuid.int >> 64

  # Convert 100-nanosecond intervals since Oct 15, 1582 to Unix timestamp
  unix_epoch_offset = 0x01b21dd213814000  # Oct 15, 1582 in 100ns intervals
  timestamp_100ns -= unix_epoch_offset

  timestamp_ms = timestamp_100ns // 10000
  return timestamp_ms


# Server-side message handling
async def handle_message(sender_id: str, conversation_id: str, content: str):
  """Process incoming message"""

  # Generate TIMEUUID (server-assigned)
  message_id = uuid.uuid1()  # Time-based, sequential

  # Create message
  message = {
    'message_id': str(message_id),
    'conversation_id': conversation_id,
    'sender_id': sender_id,
    'content': content,
    'created_at': datetime.utcnow()
  }

  # Save to Cassandra (will be auto-sorted by TIMEUUID)
  await cassandra.insert_message(message)

  # Send ACK to sender with timestamp
  await send_to_user(sender_id, {
    'type': 'message_sent',
    'message_id': str(message_id),
    'timestamp': int(message_id.time / 10000 - 12219292800000)  # Unix ms
  })

  # Fan-out to recipients (they'll re-sort if needed)
  members = await get_conversation_members(conversation_id)
  for member_id in members:
    if member_id != sender_id:
      await send_to_user(member_id, {
        'type': 'new_message',
        'message_id': str(message_id),
        'sender_id': sender_id,
        'content': content,
        'timestamp': int(message_id.time / 10000 - 12219292800000)
      })
```

**Типичные ошибки.**
- Использовать UUID v4 вместо v1 — нельзя сортировать.
- Доверять network arrival order — сообщения могут прийти не в том порядке.
- Не обрабатывать clock skew — если сервера рассинхронизированы.
- Отсутствие re-sorting на клиенте — может показать неправильный порядок.

**На интервью.**
- Объясни, почему TIMEUUID а не sequence numbers.
- Покажи Cassandra schema с clustering order.
- Упомяни client-side re-sorting для edge cases.
- Follow-up: «Как обрабатывать edited messages?» — timestamp edit, но original order by message_id.

---

### 5. Как реализовать групповые чаты?

**Зачем спрашивают.** Group chats усложняют масштабирование: больше членов = больше fan-out. Интервьюер проверяет понимание trade-offs между fan-out strategies.

**Короткий ответ.** Для групп < 100 членов — fan-out на write (отправить каждому сразу). Для групп > 100 членов — fan-out на read (сохранить один раз, каждый читает). Между ними использовать hybrid подход.

**Детальный разбор.**

**Fan-out strategies:**
```
Small Groups (<100 members):

User A sends message
     │
     ├─ Save to DB
     │
     ├─ Send to User B ──────────┐
     ├─ Send to User C ──────────┼─► Receive immediately
     ├─ Send to User D ──────────┤   (push model)
     └─ Send to User E ──────────┘

Advantages: Real-time, simple
Disadvantages: Write load scales with group size


Large Groups (>100 members):

User A sends message
     │
     └─ Save to DB (once)
              │
         User B: SELECT * FROM messages WHERE group_id = X
         User C: SELECT * FROM messages WHERE group_id = X  (pull model)
         User D: SELECT * FROM messages WHERE group_id = X
         User E: SELECT * FROM messages WHERE group_id = X

Advantages: Write is constant, read is on-demand
Disadvantages: Read latency (may need caching)


Hybrid (threshold at 100):
<100  → Fan-out write
≥100  → Fan-out read
```

**Implementation:**
```python
class GroupChatService:
  FANOUT_THRESHOLD = 100  # members

  async def send_message(self,
                        conversation_id: str,
                        sender_id: str,
                        content: str):
    """Handle message sending for group"""

    # Get group size
    group_size = await self.get_group_member_count(conversation_id)

    # Generate message ID
    message_id = uuid.uuid1()

    # Save to database
    message = {
      'message_id': str(message_id),
      'conversation_id': conversation_id,
      'sender_id': sender_id,
      'content': content,
      'created_at': datetime.utcnow()
    }
    await self.db.insert_message(message)

    # Choose delivery strategy based on group size
    if group_size < self.FANOUT_THRESHOLD:
      # Fan-out on write
      await self._fanout_write(conversation_id, sender_id, message)
    else:
      # Fan-out on read (push only important updates)
      await self._fanout_read(conversation_id, sender_id, message)

  async def _fanout_write(self, conversation_id: str,
                          sender_id: str, message: dict):
    """Push to all members (small groups)"""

    members = await self.get_conversation_members(conversation_id)

    # Parallel sends for speed
    tasks = []
    for member_id in members:
      if member_id != sender_id:
        tasks.append(
          self.send_to_user(member_id, {
            'type': 'new_message',
            'data': message
          })
        )

    # Wait for all sends, but don't fail if some users offline
    await asyncio.gather(*tasks, return_exceptions=True)

  async def _fanout_read(self, conversation_id: str,
                         sender_id: str, message: dict):
    """Notify members to pull new message (large groups)"""

    members = await self.get_conversation_members(conversation_id)

    # Send notification about new message (lightweight)
    notification = {
      'type': 'new_message_available',
      'data': {
        'message_id': message['message_id'],
        'sender_id': sender_id,
        'conversation_id': conversation_id,
        'preview': message['content'][:100]  # preview only
      }
    }

    # Can use push notification service for offline members
    for member_id in members:
      if member_id != sender_id:
        # Send lightweight notification
        await self.send_to_user(member_id, notification)

        # Offline users get push notification
        if not await self.is_user_online(member_id):
          await self.send_push_notification(
            member_id,
            f"New message in {conversation_id}",
            notification
          )


# Database schema for groups
"""
CREATE TABLE groups (
    conversation_id UUID PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    created_by UUID,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    member_count INT,
    is_active BOOLEAN
);

CREATE TABLE group_members (
    conversation_id UUID,
    user_id UUID,
    role VARCHAR(20),  -- admin, member
    joined_at TIMESTAMP,
    last_read_message_id TIMEUUID,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX idx_group_members_user
ON group_members(user_id);

-- Messages table same as before
CREATE TABLE messages (
    conversation_id UUID,
    message_id TIMEUUID,
    sender_id UUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
"""
```

**Caching strategy for large groups:**
```python
class GroupMessageCache:
  def __init__(self, redis_client):
    self.redis = redis_client
    self.cache_ttl = 3600  # 1 hour

  async def get_recent_messages(self, group_id: str, limit: int = 50):
    """Get recent messages from cache or DB"""

    cache_key = f"group:{group_id}:messages"

    # Try cache first
    cached = self.redis.lrange(cache_key, 0, limit - 1)
    if cached:
      return [json.loads(msg) for msg in cached]

    # Cache miss: fetch from DB
    messages = await self.db.get_recent_messages(group_id, limit)

    # Store in cache
    pipe = self.redis.pipeline()
    for msg in messages:
      pipe.lpush(cache_key, json.dumps(msg))
    pipe.expire(cache_key, self.cache_ttl)
    pipe.execute()

    return messages

  async def add_message(self, group_id: str, message: dict):
    """Add message to cache and DB"""

    # Save to DB (persistent)
    await self.db.insert_message(message)

    # Add to cache (latest first)
    cache_key = f"group:{group_id}:messages"
    self.redis.lpush(cache_key, json.dumps(message))
    self.redis.expire(cache_key, self.cache_ttl)
```

**Handling large groups (>10K members):**
```python
class LargeGroupOptimization:
  """
  For very large groups, even fan-out on read is expensive.
  Solution: Sharding by interest/role
  """

  async def send_message_sharded(self,
                                 group_id: str,
                                 message: dict):
    """Send to sharded members"""

    # Example: Only notify "active" members in last 7 days
    active_members = await self.get_active_members(group_id)

    # Send only to active (reduces push load)
    for member_id in active_members:
      await self.send_to_user(member_id, {
        'type': 'new_message_available',
        'data': message
      })

    # Dormant members don't get push notification,
    # but message is available when they open app
    # (they'll fetch via fan-out on read)
```

**Типичные ошибки.**
- Fan-out write для всех групп — O(n) latency для больших групп.
- Отсутствие monitoring группы size — не переключается на fan-out read.
- Отправлять полное сообщение в notification — излишняя пропускная способность.
- Не обрабатывать offline members в fan-out write — сообщения теряются.

**На интервью.**
- Объясни trade-offs fan-out write vs fan-out read.
- Покажи, как выбирать threshold (обычно 100-500).
- Упомяни caching для улучшения read performance.
- Follow-up: «Как обрабатывать 10K member group?» — only push to active, others pull on demand.

---

### 6. Как работают read receipts?

**Зачем спрашивают.** Read receipts — интересный case изучения consistency и performance. Интервьюер проверяет понимание async processing и eventual consistency.

**Короткий ответ.** Клиент отправляет "mark_read" событие на сервер. Сервер асинхронно обновляет в DB и отправляет receipt обратно отправителю и членам группы. Использовать eventual consistency и batching для оптимизации.

**Детальный разбор.**

**Архитектура read receipts:**
```
Sender                    Receiver              Server
  │                         │                    │
  │──── message ────────────┤                    │
  │                         │                    │
  │                         │ (user sees)        │
  │                         │                    │
  │                         │─ mark_read ──────►│
  │                         │                    │
  │                         │                    │ (async process)
  │                         │                    │ - Save to DB
  │                         │                    │ - Update cache
  │                         │                    │ - Publish event
  │                         │                    │
  │◄───────────────────────────── read_receipt ─│
  │
  │ (sees "Read" indicator)
  │
```

**Database schema:**
```sql
-- Option 1: Store per message (simple, but sparse)
CREATE TABLE message_reads (
    conversation_id UUID,
    message_id TIMEUUID,
    reader_id UUID,
    read_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id, reader_id)
);

-- Option 2: Store last read position per user (efficient)
CREATE TABLE conversation_read_state (
    conversation_id UUID,
    user_id UUID,
    last_read_message_id TIMEUUID,
    last_read_at TIMESTAMP,
    PRIMARY KEY (conversation_id, user_id)
);

-- Option 3: Hybrid (most common)
CREATE TABLE message_reads_summary (
    message_id TIMEUUID,
    reader_count INT,
    delivered_count INT,
    PRIMARY KEY (message_id)
);

CREATE TABLE user_read_states (
    conversation_id UUID,
    user_id UUID,
    last_read_message_id TIMEUUID,
    last_read_at TIMESTAMP,
    PRIMARY KEY (conversation_id, user_id)
);
```

**Implementation:**
```python
class ReadReceiptService:
  def __init__(self, db, redis, message_bus):
    self.db = db
    self.redis = redis
    self.message_bus = message_bus  # Kafka/RabbitMQ
    self.batch_interval = 5  # seconds
    self.read_receipts_buffer = {}  # batch buffer

  async def mark_message_read(self,
                              conversation_id: str,
                              user_id: str,
                              message_id: str):
    """Handle read receipt (async)"""

    # 1. Quick cache update (optimistic)
    cache_key = f"conv:{conversation_id}:read_state:{user_id}"
    self.redis.set(cache_key, message_id)

    # 2. Send immediate ACK to client
    await self.send_to_user(user_id, {
      'type': 'read_receipt_ack',
      'data': {'message_id': message_id}
    })

    # 3. Buffer for batch write
    buffer_key = f"{conversation_id}:{user_id}"
    self.read_receipts_buffer[buffer_key] = {
      'conversation_id': conversation_id,
      'user_id': user_id,
      'message_id': message_id,
      'timestamp': time.time()
    }

    # 4. Async: persist and broadcast (not blocking)
    asyncio.create_task(
      self._process_read_receipt_async(
        conversation_id, user_id, message_id
      )
    )

  async def _process_read_receipt_async(self,
                                       conversation_id: str,
                                       user_id: str,
                                       message_id: str):
    """Async processing (fire and forget)"""

    try:
      # Save to database
      await self.db.update_read_state(
        conversation_id=conversation_id,
        user_id=user_id,
        last_read_message_id=message_id,
        read_at=datetime.utcnow()
      )

      # Publish event for broadcast
      await self.message_bus.publish('read_receipts', {
        'conversation_id': conversation_id,
        'user_id': user_id,
        'message_id': message_id,
        'timestamp': int(time.time())
      })

    except Exception as e:
      # Log error but don't fail request
      logger.error(f"Failed to process read receipt: {e}")
      # Could retry with exponential backoff

  async def broadcast_read_receipt(self,
                                   conversation_id: str,
                                   reader_id: str,
                                   message_id: str):
    """Send read receipt to sender and group"""

    # Get message details
    message = await self.db.get_message(message_id)
    sender_id = message['sender_id']

    # Send to message sender
    await self.send_to_user(sender_id, {
      'type': 'message_read',
      'data': {
        'message_id': message_id,
        'reader_id': reader_id,
        'read_at': int(time.time())
      }
    })

    # For groups, optionally send summary to all members
    if await self.is_group_chat(conversation_id):
      # Get read counts
      read_count = await self.get_read_count(message_id)
      member_count = await self.get_group_member_count(conversation_id)

      # Send summary every N reads
      if read_count % 5 == 0:  # Every 5th read
        await self.broadcast_read_summary(
          conversation_id, message_id,
          read_count, member_count
        )

  async def get_read_state(self,
                          conversation_id: str,
                          user_id: str) -> dict:
    """Get user's last read position"""

    # Try cache first
    cache_key = f"conv:{conversation_id}:read_state:{user_id}"
    cached = self.redis.get(cache_key)

    if cached:
      return {'last_read_message_id': cached}

    # Cache miss: fetch from DB
    state = await self.db.get_read_state(
      conversation_id, user_id
    )

    if state:
      # Update cache
      self.redis.setex(
        cache_key, 3600,  # 1 hour TTL
        state['last_read_message_id']
      )

    return state or {}

  async def get_unread_count(self,
                            conversation_id: str,
                            user_id: str) -> int:
    """Count unread messages"""

    # Get user's last read position
    read_state = await self.get_read_state(
      conversation_id, user_id
    )
    last_read_id = read_state.get('last_read_message_id')

    if not last_read_id:
      # User hasn't read anything, count all messages
      return await self.db.count_messages(conversation_id)

    # Count messages after last read
    return await self.db.count_messages_after(
      conversation_id, last_read_id
    )


# Client-side
class ClientReadReceipt:
  def __init__(self, websocket_client):
    self.ws = websocket_client
    self.read_timer = None
    self.visible_message_id = None

  def on_message_visible(self, message_id: str):
    """Called when message becomes visible to user"""

    self.visible_message_id = message_id

    # Debounce: wait a bit before marking read
    if self.read_timer:
      self.read_timer.cancel()

    self.read_timer = Timer(1.0, self.send_read_receipt)
    self.read_timer.start()

  def send_read_receipt(self):
    """Send mark_read event to server"""

    if self.visible_message_id:
      self.ws.send({
        'type': 'mark_read',
        'data': {
          'message_id': self.visible_message_id
        }
      })

  def on_message_hidden(self):
    """Called when message scrolls out of view"""

    if self.read_timer:
      self.read_timer.cancel()
      self.read_timer = None
```

**Типичные ошибки.**
- Отправлять read receipt синхронно — блокирует пользовательский ввод.
- Обновлять состояние при каждом прочтении — O(n) операций для n сообщений.
- Broadcast read receipt для всех членов группы — излишний трафик.
- Не debouncing на клиенте — множество redundant сообщений.

**На интервью.**
- Объясни архитектуру: cache + async processing.
- Покажи, как debouncing улучшает performance.
- Упомяни eventual consistency — read receipt не требует strong consistency.
- Follow-up: «Как показывать "Typing" indicator?» — аналогичный pattern с debouncing.

---

### 7. Как хранить историю сообщений?

**Зачем спрашивают.** Message storage — это большой объём данных с особыми требованиями (time-series, sequential read/write). Интервьюер проверяет выбор БД и оптимизацию.

**Короткий ответ.** Использовать Cassandra для горячих данных (recent messages). Это write-optimized, supports time-series queries, handles replication. Для холодных данных (>30 дней) архивировать в S3 в parquet формате. Индексировать через Elasticsearch для поиска.

**Детальный разбор.**

**Архитектура хранения:**
```
┌─────────────────────────────────────┐
│      Recent Messages (<30 days)     │
│                                     │
│  ┌────────────────────────────────┐│
│  │ Cassandra (SSD, fast)          ││
│  │ - Optimized for writes         ││
│  │ - Time-based partitioning      ││
│  │ - Replication for HA           ││
│  │ - TTL auto-purge               ││
│  └────────────────────────────────┘│
└──────┬──────────────────────────────┘
       │
       ├─ Read: Low latency (<50ms)
       ├─ Write: High throughput (1M/sec)
       └─ Retention: 30 days

       │
       ▼

┌─────────────────────────────────────┐
│      Archive & Search (>30 days)    │
│                                     │
│  ┌────────────────────────────────┐│
│  │ S3 (Parquet, compressed)       ││
│  │ - Long-term retention          ││
│  │ - Lifecycle policies (5 years) ││
│  │ - Cheap storage                ││
│  └────────────────────────────────┘│
│  ┌────────────────────────────────┐│
│  │ Elasticsearch (indexed)         ││
│  │ - Full-text search             ││
│  │ - Faceted search               ││
│  │ - Analytics                    ││
│  └────────────────────────────────┘│
└─────────────────────────────────────┘
```

**Cassandra schema optimization:**
```sql
-- Time-series partitioning (bucket by date)
CREATE TABLE messages_by_date (
    conversation_id UUID,
    date TEXT,  -- YYYY-MM-DD
    message_id TIMEUUID,
    sender_id UUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY ((conversation_id, date), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
    AND compaction = {'class': 'TimeWindowCompactionStrategy',
                      'compaction_window_size': '1',
                      'compaction_window_unit': 'DAYS'}
    AND gc_grace_seconds = 86400
    AND default_time_to_live = 2592000;  -- 30 days TTL

-- Why partition by date?
-- 1. Prevents single partition from growing too large
-- 2. Automatic cleanup after TTL
-- 3. Efficient range queries (get messages for May 15)
-- 4. Parallel compaction

-- Index for efficient search by sender
CREATE INDEX messages_sender ON messages_by_date(sender_id);

-- Materialized view for get by recipient
CREATE MATERIALIZED VIEW messages_by_recipient AS
    SELECT conversation_id, date, message_id,
           sender_id, content, created_at
    FROM messages_by_date
    WHERE conversation_id IS NOT NULL
        AND date IS NOT NULL
        AND message_id IS NOT NULL
    PRIMARY KEY ((conversation_id, sender_id), date, message_id);
```

**Query patterns:**
```python
class MessageStore:
  async def get_recent_messages(self,
                               conversation_id: str,
                               limit: int = 50):
    """Get last N messages (most common query)"""

    query = """
    SELECT * FROM messages_by_date
    WHERE conversation_id = ?
    ORDER BY message_id DESC
    LIMIT ?
    """

    # Note: This queries today's partition
    # For 30-day history, might need multiple queries across dates

    return await self.cassandra.execute(query, [conversation_id, limit])

  async def get_messages_between(self,
                                conversation_id: str,
                                start_time: datetime,
                                end_time: datetime):
    """Get messages in time range (pagination)"""

    # Convert to message_id (TIMEUUID)
    start_id = self._datetime_to_timeuuid(start_time)
    end_id = self._datetime_to_timeuuid(end_time)

    # Partition key includes date, so need multiple queries
    current_date = start_time.date()
    end_date = end_time.date()

    messages = []
    while current_date <= end_date:
      query = """
      SELECT * FROM messages_by_date
      WHERE conversation_id = ?
        AND date = ?
        AND message_id >= ?
        AND message_id <= ?
      """

      batch = await self.cassandra.execute(
        query,
        [conversation_id, str(current_date), start_id, end_id]
      )
      messages.extend(batch)
      current_date += timedelta(days=1)

    return sorted(messages, key=lambda m: m['message_id'])

  async def get_from_offset(self,
                           conversation_id: str,
                           message_id: str,
                           limit: int = 50):
    """Get messages starting from offset (pagination continuation)"""

    # Use paging_state for efficient pagination
    query = """
    SELECT * FROM messages_by_date
    WHERE conversation_id = ?
    ORDER BY message_id DESC
    LIMIT ?
    """

    return await self.cassandra.execute(
      query,
      [conversation_id, limit],
      paging_state=self._get_paging_state(message_id)
    )


# Archive old messages to S3
class MessageArchiver:
  async def archive_old_messages(self):
    """Move messages older than 30 days to S3"""

    cutoff_date = datetime.now() - timedelta(days=30)

    # Query all conversations with old messages
    conversations = await self.db.get_active_conversations()

    for conversation_id in conversations:
      # Get old messages
      old_messages = await self.cassandra.execute("""
        SELECT * FROM messages_by_date
        WHERE conversation_id = ?
        AND created_at < ?
      """, [conversation_id, cutoff_date])

      if not old_messages:
        continue

      # Write to S3 as Parquet (compressed)
      df = pl.from_dicts([dict(msg) for msg in old_messages])

      # Partition by conversation and month
      year_month = cutoff_date.strftime("%Y-%m")
      s3_path = f"s3://messages-archive/{conversation_id}/{year_month}.parquet"

      df.write_parquet(s3_path, compression='snappy')

      # Delete from Cassandra after confirming S3 write
      await self.cassandra.execute("""
        DELETE FROM messages_by_date
        WHERE conversation_id = ?
        AND created_at < ?
      """, [conversation_id, cutoff_date])

      logger.info(f"Archived {len(old_messages)} messages for {conversation_id}")


# Search in Elasticsearch
class MessageSearch:
  async def index_message(self, message: dict):
    """Index new message for search"""

    await self.es.index(
      index='messages',
      id=message['message_id'],
      body={
        'conversation_id': message['conversation_id'],
        'sender_id': message['sender_id'],
        'content': message['content'],
        'created_at': message['created_at'],
        'timestamp': int(message['created_at'].timestamp() * 1000)
      }
    )

  async def search_messages(self,
                           conversation_id: str,
                           query: str) -> List[dict]:
    """Full-text search in conversation"""

    results = await self.es.search(
      index='messages',
      body={
        'query': {
          'bool': {
            'must': [
              {'match': {'content': query}},
              {'term': {'conversation_id': conversation_id}}
            ]
          }
        },
        'sort': [{'created_at': {'order': 'desc'}}],
        'size': 50
      }
    )

    return [hit['_source'] for hit in results['hits']['hits']]
```

**Data retention & compliance:**
```python
class DataRetention:
  """Implement data retention policies"""

  # GDPR: User can request deletion
  async def delete_user_messages(self, user_id: str):
    """Delete all messages from a user"""

    # Find all conversations user participated in
    conversations = await self.db.get_user_conversations(user_id)

    for conversation_id in conversations:
      # Delete from Cassandra (recent)
      await self.cassandra.execute("""
        DELETE FROM messages_by_date
        WHERE conversation_id = ?
        AND sender_id = ?
      """, [conversation_id, user_id])

      # Delete from S3 (archived)
      # Use Athena to query and delete

      # Delete from Elasticsearch
      await self.es.delete_by_query(
        index='messages',
        body={
          'query': {
            'term': {'sender_id': user_id}
          }
        }
      )

  # Retention window (delete after N years)
  async def cleanup_expired_messages(self):
    """Delete messages older than retention window"""

    retention_years = 5
    cutoff_date = datetime.now() - timedelta(days=365 * retention_years)

    # Cassandra: use TTL (automatic)
    # Already configured in schema: default_time_to_live = 2592000

    # S3: use lifecycle policies
    # Configure in bucket settings to delete after N days

    # Elasticsearch: delete old indexes
    old_indexes = await self.es.cat.indices(
      format='json',
      h='index,creation.date'
    )

    for index in old_indexes:
      creation_date = datetime.fromtimestamp(int(index['creation.date']) / 1000)
      if creation_date < cutoff_date:
        await self.es.indices.delete(index=index['index'])
```

**Типичные ошибки.**
- Хранить все сообщения в одной таблице — partition becomes too large.
- Не использовать TTL — ручное управление retention сложное и ненадежное.
- Не архивировать в S3 — переплачиваешь за Cassandra storage.
- Индексировать всё в ES — дорого, индексируй только для search.

**На интервью.**
- Объясни почему Cassandra а не PostgreSQL: write-throughput, time-series optimization.
- Покажи date-based partitioning и TTL.
- Упомяни S3 архивирование для cost optimization.
- Follow-up: «Как импортировать архивированные сообщения?» — query S3 via Athena или загрузить в Cassandra.

---

### 8. Как реализовать поиск по сообщениям?

**Зачем спрашивают.** Full-text search в чате — это отдельный сложный компонент. Интервьюер проверяет понимание search индексации, relevance и масштабирования.

**Короткий ответ.** Использовать Elasticsearch для индексирования сообщений. При отправке сообщения асинхронно индексировать в ES. На поиск — query ES с фильтром по conversation_id. Для больших объёмов использовать Opensearch или самостоятельно управлять шардированием.

**Детальный разбор.**

**Search architecture:**
```
┌──────────────────────────────────────┐
│         Message Posted               │
└─────────────┬────────────────────────┘
              │
      ┌───────┴────────┐
      │                │
      ▼                ▼
┌──────────┐    ┌──────────────────┐
│Cassandra │    │ Message Queue    │
│(storage) │    │ (Async indexing) │
└──────────┘    └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ Elasticsearch    │
                │ (Search Index)   │
                └──────────────────┘
                         │
                         ▼
      ┌──────────────────────────────┐
      │  User Search Query           │
      │  /api/search?q=hello         │
      └──────────────┬───────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │ Query Elasticsearch  │
          │ (fast, <100ms)       │
          └──────────────────────┘
```

**Elasticsearch mapping:**
```json
{
  "mappings": {
    "properties": {
      "message_id": {
        "type": "keyword"
      },
      "conversation_id": {
        "type": "keyword"
      },
      "sender_id": {
        "type": "keyword"
      },
      "content": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "created_at": {
        "type": "date",
        "format": "epoch_millis"
      },
      "media_urls": {
        "type": "keyword"
      },
      "message_type": {
        "type": "keyword"
      }
    }
  }
}
```

**Implementation:**
```python
from elasticsearch import Elasticsearch
from kafka import KafkaConsumer
import json
import asyncio

class MessageSearchService:
  def __init__(self, es_client: Elasticsearch):
    self.es = es_client
    self.index_name = 'messages'

  async def index_message(self, message: dict):
    """Index new message for search"""

    try:
      await self.es.index(
        index=self.index_name,
        id=message['message_id'],
        body={
          'message_id': message['message_id'],
          'conversation_id': message['conversation_id'],
          'sender_id': message['sender_id'],
          'content': message['content'],
          'message_type': message.get('message_type', 'text'),
          'created_at': int(message['created_at'].timestamp() * 1000),
          'media_urls': message.get('media_urls', [])
        }
      )
    except Exception as e:
      logger.error(f"Failed to index message {message['message_id']}: {e}")
      # Don't fail the request, message is still in Cassandra

  async def search_in_conversation(self,
                                   conversation_id: str,
                                   query: str,
                                   limit: int = 50,
                                   offset: int = 0) -> dict:
    """Search messages in specific conversation"""

    # Build query
    es_query = {
      'bool': {
        'must': [
          {'term': {'conversation_id': conversation_id}},
          {
            'multi_match': {
              'query': query,
              'fields': ['content^2', 'media_urls'],  # boost content field
              'fuzziness': 'AUTO'  # handle typos
            }
          }
        ]
      }
    }

    # Execute search
    results = await self.es.search(
      index=self.index_name,
      body={
        'query': es_query,
        'sort': [{'created_at': {'order': 'desc'}}],
        'from': offset,
        'size': limit
      }
    )

    return {
      'total': results['hits']['total']['value'],
      'messages': [
        self._format_hit(hit) for hit in results['hits']['hits']
      ]
    }

  async def search_across_conversations(self,
                                       user_id: str,
                                       query: str) -> dict:
    """Search in all user's conversations"""

    # Get user's conversations
    conversations = await self.db.get_user_conversations(user_id)

    # Search across all conversations
    es_query = {
      'bool': {
        'must': [
          {'terms': {'conversation_id': conversations}},
          {'match': {'content': query}}
        ]
      }
    }

    results = await self.es.search(
      index=self.index_name,
      body={
        'query': es_query,
        'sort': [{'created_at': {'order': 'desc'}}],
        'size': 100
      }
    )

    # Group by conversation
    grouped = {}
    for hit in results['hits']['hits']:
      conv_id = hit['_source']['conversation_id']
      if conv_id not in grouped:
        grouped[conv_id] = []
      grouped[conv_id].append(self._format_hit(hit))

    return {'conversations': grouped}

  async def search_with_filters(self,
                               conversation_id: str,
                               query: str,
                               sender_id: str = None,
                               start_date: datetime = None,
                               end_date: datetime = None) -> dict:
    """Search with advanced filters"""

    # Build filter
    filters = [
      {'term': {'conversation_id': conversation_id}}
    ]

    if sender_id:
      filters.append({'term': {'sender_id': sender_id}})

    if start_date or end_date:
      date_filter = {'range': {'created_at': {}}}
      if start_date:
        date_filter['range']['created_at']['gte'] = int(start_date.timestamp() * 1000)
      if end_date:
        date_filter['range']['created_at']['lte'] = int(end_date.timestamp() * 1000)
      filters.append(date_filter)

    # Search
    es_query = {
      'bool': {
        'must': [
          {'match': {'content': query}}
        ],
        'filter': filters
      }
    }

    results = await self.es.search(
      index=self.index_name,
      body={
        'query': es_query,
        'sort': [{'created_at': {'order': 'desc'}}],
        'size': 50
      }
    )

    return {
      'total': results['hits']['total']['value'],
      'messages': [self._format_hit(hit) for hit in results['hits']['hits']]
    }

  def _format_hit(self, hit: dict) -> dict:
    """Convert ES hit to message format"""
    source = hit['_source']
    return {
      'message_id': source['message_id'],
      'sender_id': source['sender_id'],
      'content': source['content'],
      'created_at': datetime.fromtimestamp(source['created_at'] / 1000),
      'score': hit['_score']  # relevance score
    }


# Async indexing via Kafka
class MessageIndexer:
  def __init__(self, kafka_brokers: List[str], es_client: Elasticsearch):
    self.consumer = KafkaConsumer(
      'messages',
      bootstrap_servers=kafka_brokers,
      group_id='message-indexer',
      value_deserializer=lambda m: json.loads(m.decode('utf-8')),
      auto_offset_reset='earliest'
    )
    self.search_service = MessageSearchService(es_client)
    self.batch_size = 100
    self.buffer = []

  async def start(self):
    """Start consuming and indexing"""

    for message in self.consumer:
      self.buffer.append(message.value)

      if len(self.buffer) >= self.batch_size:
        await self.flush()

  async def flush(self):
    """Batch index messages"""

    if not self.buffer:
      return

    # Index all messages in batch
    bulk_ops = []
    for msg in self.buffer:
      bulk_ops.append({'index': {'_index': 'messages', '_id': msg['message_id']}})
      bulk_ops.append({
        'message_id': msg['message_id'],
        'conversation_id': msg['conversation_id'],
        'sender_id': msg['sender_id'],
        'content': msg['content'],
        'created_at': int(msg['created_at'].timestamp() * 1000),
        'message_type': msg.get('message_type', 'text')
      })

    await self.search_service.es.bulk(body=bulk_ops)
    self.buffer = []
    logger.info(f"Indexed {len(bulk_ops) // 2} messages")
```

**Advanced search features:**
```python
class AdvancedSearch:
  # Search for media files
  async def search_media(self,
                        conversation_id: str,
                        media_type: str = None) -> List[dict]:
    """Find all media messages"""

    filters = [
      {'term': {'conversation_id': conversation_id}},
      {'exists': {'field': 'media_urls'}}
    ]

    if media_type:  # 'image', 'video', 'document'
      filters.append({'term': {'message_type': media_type}})

    results = await self.es.search(
      index='messages',
      body={
        'query': {'bool': {'filter': filters}},
        'sort': [{'created_at': {'order': 'desc'}}]
      }
    )

    return [hit['_source'] for hit in results['hits']['hits']]

  # Autocomplete search
  async def autocomplete_search(self, prefix: str) -> List[str]:
    """Suggest search terms as user types"""

    # Use completion suggester
    results = await self.es.search(
      index='messages',
      body={
        'suggest': {
          'message-suggest': {
            'prefix': prefix,
            'completion': {'field': 'content.completion'}
          }
        }
      }
    )

    suggestions = results['suggest']['message-suggest'][0]['options']
    return [s['_source']['content'] for s in suggestions[:10]]

  # Trending topics
  async def get_trending_topics(self,
                               conversation_id: str,
                               time_window_hours: int = 24) -> List[dict]:
    """Find most mentioned keywords"""

    cutoff_time = int((time.time() - time_window_hours * 3600) * 1000)

    results = await self.es.search(
      index='messages',
      body={
        'query': {
          'bool': {
            'filter': [
              {'term': {'conversation_id': conversation_id}},
              {'range': {'created_at': {'gte': cutoff_time}}}
            ]
          }
        },
        'aggs': {
          'keywords': {
            'significant_terms': {
              'field': 'content',
              'size': 20
            }
          }
        },
        'size': 0
      }
    )

    return [
      {
        'keyword': bucket['key'],
        'count': bucket['doc_count'],
        'score': bucket['score']
      }
      for bucket in results['aggregations']['keywords']['buckets']
    ]
```

**Типичные ошибки.**
- Индексировать синхронно — блокирует message send.
- Не фильтровать по conversation_id — security issue, видит чужие сообщения.
- Не использовать TTL/rotation в ES — индекс растёт бесконечно.
- Забыть про pagination — search return все результаты, OOM.

**На интервью.**
- Объясни async индексирование через message queue.
- Покажи ES query с фильтрами.
- Упомяни important fields: content, created_at, conversation_id.
- Follow-up: «Как поискать без учёта регистра?» — lowercase analyzer в mapping.

---

### 9. Как обеспечить end-to-end encryption?

**Зачем спрашивают.** E2E encryption — важная privacy feature, но усложняет архитектуру. Интервьюер проверяет понимание криптографии и ее применения в system design.

**Короткий ответ.** Использовать асимметричное шифрование (RSA, ECDH) для распределения ключей, и AES-256 для шифрования сообщений. Публичные ключи хранить на сервере, приватные — только на клиентах. Сервер передает сообщения, не видя содержимого.

**Детальный разбор.**

**E2E Encryption flow:**
```
User A                              User B
├─ Generate pair:                   ├─ Generate pair:
│  - Public_A (stored on server)    │  - Public_B (stored on server)
│  - Private_A (kept secret)        │  - Private_B (kept secret)
│                                   │
│  Wants to send message            │
│                                   │
├─ Get Public_B from server         │
│                                   │
├─ Generate ephemeral:              │
│  - Session key (AES-256)          │
│  - HMAC key                       │
│                                   │
├─ Encrypt message with session key │
│  - ciphertext = AES_encrypt(      │
│      message,                     │
│      session_key                  │
│    )                              │
│                                   │
├─ Encrypt session key with Public_B│
│  - encrypted_key = RSA_encrypt(   │
│      session_key,                 │
│      Public_B                     │
│    )                              │
│                                   │
├─ Send to server:                  │
│  {                                │
│    "encrypted_key": ...,          │
│    "ciphertext": ...,             │
│    "hmac": ...                    │
│  }                                │
│                                   │
└─ Server forwards to User B        │
   (server cannot read content)     │
                                    │
                                    ├─ Receives ciphertext
                                    │
                                    ├─ Decrypt with Private_B:
                                    │  - session_key = RSA_decrypt(
                                    │      encrypted_key,
                                    │      Private_B
                                    │    )
                                    │
                                    ├─ Decrypt message:
                                    │  - message = AES_decrypt(
                                    │      ciphertext,
                                    │      session_key
                                    │    )
                                    │
                                    └─ Can now read message
```

**Implementation:**
```python
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import os

class E2EEncryption:
  # RSA configuration
  RSA_KEY_SIZE = 2048

  # AES configuration
  AES_KEY_SIZE = 32  # 256 bits
  AES_IV_SIZE = 16   # 128 bits

  def __init__(self):
    self.backend = default_backend()

  # Key Generation
  def generate_key_pair(self) -> tuple[str, str]:
    """Generate RSA key pair for user"""

    # Generate private key
    private_key = rsa.generate_private_key(
      public_exponent=65537,
      key_size=self.RSA_KEY_SIZE,
      backend=self.backend
    )

    # Serialize private key (PEM format, encrypted)
    private_pem = private_key.private_bytes(
      encoding=serialization.Encoding.PEM,
      format=serialization.PrivateFormat.PKCS8,
      encryption_algorithm=serialization.BestAvailableEncryption(b'user_password')
    )

    # Extract and serialize public key
    public_key = private_key.public_key()
    public_pem = public_key.public_bytes(
      encoding=serialization.Encoding.PEM,
      format=serialization.PublicFormat.SubjectPublicKeyInfo
    )

    return public_pem.decode(), private_pem.decode()

  def get_public_key(self, user_id: str) -> str:
    """Get user's public key from server"""

    # In real app: fetch from DB
    public_key_pem = self.db.get_public_key(user_id)
    return public_key_pem

  # Message Encryption
  def encrypt_message(self,
                     message: str,
                     recipient_public_key_pem: str) -> dict:
    """Encrypt message for recipient"""

    # 1. Generate ephemeral session key
    session_key = os.urandom(self.AES_KEY_SIZE)
    iv = os.urandom(self.AES_IV_SIZE)

    # 2. Encrypt message with AES-256-CBC
    cipher = Cipher(
      algorithms.AES(session_key),
      modes.CBC(iv),
      backend=self.backend
    )
    encryptor = cipher.encryptor()

    # Pad message to AES block size
    padded_message = self._pad_message(message.encode())
    ciphertext = encryptor.update(padded_message) + encryptor.finalize()

    # 3. Encrypt session key with recipient's public key
    public_key = serialization.load_pem_public_key(
      recipient_public_key_pem.encode(),
      backend=self.backend
    )

    encrypted_session_key = public_key.encrypt(
      session_key,
      padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
      )
    )

    # 4. Compute HMAC for integrity
    hmac_key = os.urandom(32)
    hmac_value = self._compute_hmac(ciphertext + iv, hmac_key)

    # 5. Encrypt HMAC key with recipient's public key
    encrypted_hmac_key = public_key.encrypt(
      hmac_key,
      padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
      )
    )

    return {
      'iv': self._bytes_to_base64(iv),
      'ciphertext': self._bytes_to_base64(ciphertext),
      'encrypted_session_key': self._bytes_to_base64(encrypted_session_key),
      'encrypted_hmac_key': self._bytes_to_base64(encrypted_hmac_key),
      'hmac': self._bytes_to_base64(hmac_value)
    }

  # Message Decryption
  def decrypt_message(self,
                     encrypted_data: dict,
                     private_key_pem: str,
                     private_key_password: str) -> str:
    """Decrypt message with private key"""

    # 1. Load private key
    private_key = serialization.load_pem_private_key(
      private_key_pem.encode(),
      password=private_key_password.encode(),
      backend=self.backend
    )

    # 2. Decrypt session key
    encrypted_session_key = self._base64_to_bytes(encrypted_data['encrypted_session_key'])
    try:
      session_key = private_key.decrypt(
        encrypted_session_key,
        padding.OAEP(
          mgf=padding.MGF1(algorithm=hashes.SHA256()),
          algorithm=hashes.SHA256(),
          label=None
        )
      )
    except Exception as e:
      raise ValueError(f"Failed to decrypt session key: {e}")

    # 3. Decrypt HMAC key
    encrypted_hmac_key = self._base64_to_bytes(encrypted_data['encrypted_hmac_key'])
    hmac_key = private_key.decrypt(
      encrypted_hmac_key,
      padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
      )
    )

    # 4. Verify HMAC
    ciphertext = self._base64_to_bytes(encrypted_data['ciphertext'])
    iv = self._base64_to_bytes(encrypted_data['iv'])
    expected_hmac = self._compute_hmac(ciphertext + iv, hmac_key)

    received_hmac = self._base64_to_bytes(encrypted_data['hmac'])
    if expected_hmac != received_hmac:
      raise ValueError("HMAC verification failed - message may be tampered")

    # 5. Decrypt message
    cipher = Cipher(
      algorithms.AES(session_key),
      modes.CBC(iv),
      backend=self.backend
    )
    decryptor = cipher.decryptor()
    padded_plaintext = decryptor.update(ciphertext) + decryptor.finalize()

    # 6. Remove padding
    plaintext = self._unpad_message(padded_plaintext)

    return plaintext.decode()

  # Helpers
  def _pad_message(self, message: bytes) -> bytes:
    """PKCS7 padding"""
    padding_length = 16 - (len(message) % 16)
    padding = bytes([padding_length] * padding_length)
    return message + padding

  def _unpad_message(self, padded: bytes) -> bytes:
    """Remove PKCS7 padding"""
    padding_length = padded[-1]
    return padded[:-padding_length]

  def _compute_hmac(self, data: bytes, key: bytes) -> bytes:
    """Compute HMAC-SHA256"""
    import hmac
    return hmac.new(key, data, 'sha256').digest()

  def _bytes_to_base64(self, data: bytes) -> str:
    import base64
    return base64.b64encode(data).decode()

  def _base64_to_bytes(self, data: str) -> bytes:
    import base64
    return base64.b64decode(data)


# Server-side (minimal changes)
class ChatServer:
  async def send_encrypted_message(self,
                                  sender_id: str,
                                  conversation_id: str,
                                  encrypted_data: dict):
    """Server just forwards encrypted data"""

    message = {
      'message_id': uuid.uuid1(),
      'conversation_id': conversation_id,
      'sender_id': sender_id,
      'encrypted_payload': encrypted_data,
      'created_at': datetime.utcnow(),
      'is_encrypted': True
    }

    # Save to DB (server doesn't decrypt)
    await self.db.save_message(message)

    # Fan-out to recipients
    # They will decrypt on their side
    members = await self.get_conversation_members(conversation_id)
    for member_id in members:
      if member_id != sender_id:
        await self.send_to_user(member_id, {
          'type': 'new_message',
          'data': message
        })


# Client-side integration
class EncryptedChatClient:
  def __init__(self, user_id: str, private_key_pem: str, private_key_password: str):
    self.user_id = user_id
    self.private_key_pem = private_key_pem
    self.private_key_password = private_key_password
    self.e2e = E2EEncryption()
    self.recipient_public_keys = {}  # Cache

  async def send_encrypted_message(self,
                                   recipient_id: str,
                                   content: str):
    """Send encrypted message"""

    # Get recipient's public key
    if recipient_id not in self.recipient_public_keys:
      public_key = await self.fetch_user_public_key(recipient_id)
      self.recipient_public_keys[recipient_id] = public_key

    recipient_public_key = self.recipient_public_keys[recipient_id]

    # Encrypt
    encrypted_data = self.e2e.encrypt_message(content, recipient_public_key)

    # Send to server
    await self.ws.send({
      'type': 'send_encrypted_message',
      'data': {
        'recipient_id': recipient_id,
        'encrypted_payload': encrypted_data
      }
    })

  async def receive_encrypted_message(self, message: dict):
    """Receive and decrypt message"""

    encrypted_data = message['encrypted_payload']

    try:
      plaintext = self.e2e.decrypt_message(
        encrypted_data,
        self.private_key_pem,
        self.private_key_password
      )

      # Display decrypted message
      self.display_message({
        'sender_id': message['sender_id'],
        'content': plaintext,
        'timestamp': message['created_at']
      })

    except ValueError as e:
      logger.error(f"Failed to decrypt message: {e}")
      self.display_message({
        'sender_id': message['sender_id'],
        'content': '[Decryption failed]',
        'timestamp': message['created_at']
      })
```

**Challenges & solutions:**
```
Challenge                          Solution
─────────────────────────────────────────────────────────
Deleted messages still encrypted   Implement message deletion with
                                   key destruction

Group chat encryption              Use group key + rotate periodically

Mobile key storage                 Use Keychain (iOS), KeyStore (Android)

Forward secrecy                    Use Double Ratchet Algorithm
                                   (Signal protocol)

User loses private key             Recovery key or passphrase backup
```

**Типичные ошибки.**
- Хранить приватный ключ в plaintext — любой может decrypt.
- Не использовать HMAC — не защищает от tampering.
- Очень длинные RSA ключи для message (>2048) — очень медленно.
- Не обновлять session keys — static key = extended attack window.

**На интервью.**
- Объясни asymmetric + symmetric combo и зачем обе нужны.
- Покажи проблемы и solutions (key management, etc).
- Упомяни Signal protocol как best practice.
- Follow-up: «Как добавить forward secrecy?» — use ECDH + ratcheting, не RSA.

---

### 10. Как масштабировать chat system на миллионы пользователей?

**Зачем спрашивают.** Scaling — это финальный тест. Интервьюер проверяет способность масштабировать архитектуру end-to-end.

**Короткий ответ.** Горизонтальное масштабирование на каждый слой: WebSocket servers (с sticky sessions за LB), Redis cluster (для presence/cache), Cassandra cluster (для messages), Kafka (для async), Elasticsearch (для search). Add observability (monitoring, logging, tracing).

**Детальный разбор.**

**Scaling by component:**

**1. WebSocket Server Scaling:**
```
┌──────────────────────────────────────────────────┐
│           Load Balancer (Layer 4 TCP)            │
│  sticky session = hash(user_id) % num_servers    │
└────┬──────────────┬──────────────┬───────────────┘
     │              │              │
┌────▼───┐    ┌────▼───┐    ┌────▼───┐
│ WS Srv │    │ WS Srv │    │ WS Srv │
│ Slot 1 │    │ Slot 2 │    │ Slot N │
│ (10K)  │    │ (10K)  │    │ (10K)  │
└────┬───┘    └────┬───┘    └────┬───┘
     │              │              │
     └──────────────┼──────────────┘
              Redis Pub/Sub
        (cross-server messaging)

Per server: ~10K concurrent connections × 10KB = 100MB
50M concurrent → 5000 servers
```

**2. Redis Scaling:**
```
┌──────────────────────────────────────────────────┐
│            Redis Cluster Mode                    │
│  16 slots (or 16384 in production)               │
│  Each slot → multiple replicas                   │
└────┬──────────────┬──────────────┬───────────────┘
     │              │              │
┌────▼───┐    ┌────▼───┐    ┌────▼───┐
│Redis 1 │    │Redis 2 │    │Redis 3 │
│Slots   │    │Slots   │    │Slots   │
│5461-   │    │10923-  │    │1-5460  │
│10922   │    │16383   │    │        │
└────────┘    └────────┘    └────────┘

Write throughput per node: ~100K ops/sec
50M active users → 500K ops/sec → 5+ nodes
```

**3. Cassandra Scaling:**
```
┌──────────────────────────────────────────────────┐
│      Cassandra Ring (Distributed)                │
│    Each node responsible for token range         │
└────┬──────────────┬──────────────┬───────────────┘
     │              │              │
┌────▼───┐    ┌────▼───┐    ┌────▼───┐
│ Node 1 │    │ Node 2 │    │ Node N │
│ Rf=3   │    │ Rf=3   │    │ Rf=3   │
│ 50MB/s │    │ 50MB/s │    │ 50MB/s │
└────────┘    └────────┘    └────────┘

Write throughput: ~230K QPS = ~46MB/s (assuming 200 bytes/msg)
Per node: ~50MB/s → 1 node handles 230K QPS comfortably
But for HA: 3 nodes (Rf=3) = 1:1:1 replication
```

**4. Kafka Scaling:**
```
┌──────────────────────────────────────────────────┐
│       Kafka Cluster                              │
│  Partitions = number of concurrent publishers    │
└────┬──────────────┬──────────────┬───────────────┘
     │              │              │
┌────▼──────┐ ┌────▼──────┐ ┌────▼──────┐
│Broker 1   │ │Broker 2   │ │Broker 3   │
│Partitions │ │Partitions │ │Partitions │
│0,3,6...   │ │1,4,7...   │ │2,5,8...   │
└───────────┘ └───────────┘ └───────────┘

Topics:
- messages (1000 partitions, Rf=3)
- read_receipts (500 partitions)
- presence (100 partitions)

Throughput: ~1M messages/sec on 10 brokers
```

**Implementation:**
```python
class ScalableArchitecture:

  # Multi-region routing
  async def get_best_server(self, user_id: str) -> str:
    """Route to best WebSocket server based on geography"""

    user_location = await self.geo_service.get_location(user_id)

    # Find nearest server in that region
    servers = await self.service_registry.discover(
      service='chat-server',
      region=user_location['region']
    )

    # Load balance with least connections
    return min(servers, key=lambda s: s['connection_count'])

  # Cluster-aware pub/sub
  async def send_across_servers(self,
                               user_id: str,
                               message: dict):
    """Send to user's connections across cluster"""

    # Get all servers with this user's connections
    connections = await self.redis.smembers(f"user:{user_id}:servers")

    # Local delivery
    if self.server_id in connections:
      await self.local_send(user_id, message)

    # Remote delivery via Redis Pub/Sub
    for server_id in connections:
      if server_id != self.server_id:
        channel = f"server:{server_id}:messages"
        await self.redis.publish(channel, json.dumps({
          'user_id': user_id,
          'message': message
        }))

  # Sharding strategy
  def get_cassandra_shard(self, conversation_id: str) -> str:
    """Determine which Cassandra shard for conversation"""

    shard_index = hash(conversation_id) % self.num_cassandra_shards
    return f"cassandra-shard-{shard_index}"

  def get_redis_shard(self, user_id: str) -> str:
    """Determine which Redis shard for user"""

    shard_index = hash(user_id) % self.num_redis_shards
    return f"redis-shard-{shard_index}"

  # Circuit breaker for degradation
  async def send_with_fallback(self,
                              user_id: str,
                              message: dict):
    """Send with graceful degradation"""

    try:
      # Try primary path
      await self.send_via_websocket(user_id, message)

    except ConnectionError:
      # WebSocket path failed, use fallback
      try:
        # Fallback 1: Push notification
        await self.send_push_notification(user_id, message)
      except Exception:
        # Fallback 2: Queue for later
        await self.queue_for_offline_delivery(user_id, message)


# Observability
class Observability:
  def __init__(self):
    self.metrics = {}

  def record_message_latency(self, latency_ms: float):
    """Track p50, p95, p99 latencies"""
    # Use prometheus client
    message_latency.observe(latency_ms)

  def record_server_connections(self, count: int):
    """Track connection count per server"""
    active_connections.set(count)

  def record_cassandra_write_latency(self, latency_ms: float):
    """Track DB write performance"""
    cassandra_write_latency.observe(latency_ms)

  def log_structured(self, event: str, **kwargs):
    """Structured logging for analysis"""
    logger.info(json.dumps({
      'event': event,
      'timestamp': time.time(),
      **kwargs
    }))


# Performance targets
class PerformanceTargets:
  """
  Target SLOs for production system

  Latency:
  - Message delivery: p50 <50ms, p99 <500ms
  - Presence update: p50 <100ms, p99 <1s
  - Search: p50 <200ms, p99 <2s

  Availability:
  - Chat service: 99.99% uptime
  - Message durability: 99.9999% (6 nines)

  Throughput:
  - 230K messages/sec
  - 50M concurrent connections
  - 1M operations/sec (presence, reads, etc)

  Scalability:
  - Linear scaling with servers added
  - No single point of failure
  """
  pass
```

**Monitoring dashboard:**
```python
class MonitoringDashboard:
  # Key metrics to monitor
  metrics = [
    "websocket_connections_active",
    "message_latency_p50_ms",
    "message_latency_p99_ms",
    "cassandra_write_latency_ms",
    "redis_operation_latency_ms",
    "kafka_consumer_lag_ms",
    "elasticsearch_query_latency_ms",
    "server_cpu_usage_percent",
    "server_memory_usage_gb",
    "disk_space_available_percent",
    "network_throughput_mbps",
    "error_rate_percent",
    "cache_hit_ratio_percent"
  ]

  # Alerts
  alerts = [
    "websocket_connections > 11K per server",
    "message_latency_p99 > 500ms",
    "cassandra_write_latency > 100ms",
    "error_rate > 0.1%",
    "cache_hit_ratio < 80%",
    "kafka_consumer_lag > 60s",
    "elasticsearch_index_size > 500GB"
  ]
```

**Типичные ошибки.**
- Не использовать sticky sessions — переподключения, потеря state.
- Single Cassandra node — no HA, bottleneck.
- Недостаточно Redis sharding — операции становятся медленными.
- Отсутствие monitoring — не видишь проблемы пока не упало.

**На интервью.**
- Нарисуй架构 для 500M DAU с явным распределением.
- Объясни scaling каждого компонента (WebSocket, Redis, Cassandra, Kafka).
- Упомяни observability и мониторинг как обязательные.
- Follow-up: «Что делать если один Cassandra node падает?» — replicas + failover.

---

## См. также

- [WebSocket & Real-time Systems](../00-go/08-networking-grpc.md) — глубокое погружение в протоколы
- [Distributed Systems & Consistency](../07-distributed-systems/02-consistency-models.md) — message ordering и consistency
- [Load Balancing & Service Discovery](../03-system-design/01-load-balancing.md) — routing и failover
- [Caching Strategies](../03-system-design/02-caching.md) — оптимизация performance
- [Message Queues (Kafka)](../03-system-design/03-message-queues.md) — асинхронная обработка

---

## Практика

1. **WebSocket management** — реализуй ConnectionManager с поддержкой multiple devices на одного пользователя. Добавь graceful disconnect.

2. **Presence heartbeat** — реализуй клиент-серверный heartbeat. На сервере используй Redis с TTL. Попробуй обнаружить мертвые соединения и перераспределить.

3. **TIMEUUID ordering** — создай тест, отправляющий сообщения из разных потоков. Убедись, что они упорядочиваются правильно несмотря на сетевые задержки.

4. **Fan-out vs Fan-in** — реализуй обе стратегии для группы. Измерь write latency для группы из 50 vs 500 членов.

5. **Read receipts** — реализуй асинхронную обработку с батчингом. Убедись, что не отправляешь каждый receipt отдельно.

6. **Search indexing** — реализуй асинхронное индексирование в Elasticsearch через Kafka. Добавь поиск с фильтрами (по дате, отправителю).

7. **Scaling simulation** — используй локальное кластеризирование (docker-compose) для simulate масштабирования. Добавь redis cluster, cassandra cluster, kafka.

---

## Дополнительные материалы

- [Designing Data-Intensive Applications](https://dataintensive.net/) — глава про messaging и stream processing
- [Signal Protocol Documentation](https://signal.org/docs/) — best practice для E2E encryption
- [Cassandra Best Practices](https://cassandra.apache.org/doc/latest/cassandra/architecture/index.html) — время-серийные данные
- [Elasticsearch Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html) — полнотекстовый поиск
- [WebSocket API Spec](https://tools.ietf.org/html/rfc6455) — официальная спецификация

---

← [03-notification-system](./03-notification-system.md) | [Трек System Design](./README.md) | [05-news-feed](./05-news-feed.md) →
