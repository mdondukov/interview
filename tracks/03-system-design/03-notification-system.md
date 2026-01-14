# 03. Notification System

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Push notifications (iOS, Android)
- SMS notifications
- Email notifications
- In-app notifications
- Scheduling (send later)
- Templates и personalization

### Нефункциональные
- Soft real-time (< 1 min delay acceptable)
- High availability (99.9%)
- Scale: 10M+ notifications/day
- At-least-once delivery
- Rate limiting per user

---

## Capacity Estimation

```
Daily notifications:
- 10M DAU × 5 notifications/user = 50M notifications/day
- Peak: 100M/day during campaigns

QPS:
- Average: 50M / 86400 ≈ 600 QPS
- Peak: 10,000 QPS (campaigns, breaking news)

Storage:
- Each notification: ~1KB (content + metadata)
- 30 day retention: 50M × 30 × 1KB = 1.5TB
```

---

## High-Level Design

```
┌──────────────────────────────────────────────────────────────────┐
│                        Notification Service                       │
└──────────────────────────────────────────────────────────────────┘

┌─────────┐    ┌─────────────┐    ┌───────────────┐
│ Service │───▶│ API Gateway │───▶│ Notification  │
│ A/B/C   │    └─────────────┘    │    Service    │
└─────────┘                       └───────┬───────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
              ┌─────▼─────┐         ┌─────▼─────┐         ┌─────▼─────┐
              │  Message  │         │  Message  │         │  Message  │
              │  Queue    │         │  Queue    │         │  Queue    │
              │  (Push)   │         │  (SMS)    │         │  (Email)  │
              └─────┬─────┘         └─────┬─────┘         └─────┬─────┘
                    │                     │                     │
              ┌─────▼─────┐         ┌─────▼─────┐         ┌─────▼─────┐
              │   Push    │         │    SMS    │         │   Email   │
              │  Workers  │         │  Workers  │         │  Workers  │
              └─────┬─────┘         └─────┬─────┘         └─────┬─────┘
                    │                     │                     │
              ┌─────▼─────┐         ┌─────▼─────┐         ┌─────▼─────┐
              │   APNs    │         │  Twilio   │         │ SendGrid  │
              │   FCM     │         │  Nexmo    │         │    SES    │
              └───────────┘         └───────────┘         └───────────┘
```

---

## API Design

### Send Notification
```json
POST /api/v1/notifications

Request:
{
    "user_ids": ["user1", "user2"],
    "channels": ["push", "email"],
    "template_id": "welcome_email",
    "data": {
        "user_name": "John",
        "promo_code": "SAVE20"
    },
    "schedule_time": "2024-01-15T10:00:00Z",  // optional
    "priority": "high"
}

Response:
{
    "notification_id": "notif_123",
    "status": "queued",
    "estimated_delivery": "2024-01-15T10:00:30Z"
}
```

### Get Notification Status
```json
GET /api/v1/notifications/{notification_id}

Response:
{
    "notification_id": "notif_123",
    "status": "delivered",
    "channel_status": {
        "push": "delivered",
        "email": "opened"
    },
    "created_at": "2024-01-15T10:00:00Z",
    "delivered_at": "2024-01-15T10:00:05Z"
}
```

---

## Data Model

### Notifications Table
```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    template_id     VARCHAR(100),
    channel         VARCHAR(20),  -- push, sms, email
    content         JSONB,
    status          VARCHAR(20),  -- pending, sent, delivered, failed
    priority        VARCHAR(10),
    scheduled_at    TIMESTAMP,
    sent_at         TIMESTAMP,
    delivered_at    TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_user_id (user_id),
    INDEX idx_status_scheduled (status, scheduled_at)
);
```

### User Preferences
```sql
CREATE TABLE user_notification_preferences (
    user_id         BIGINT PRIMARY KEY,
    push_enabled    BOOLEAN DEFAULT true,
    email_enabled   BOOLEAN DEFAULT true,
    sms_enabled     BOOLEAN DEFAULT false,
    quiet_hours     JSONB,  -- {"start": "22:00", "end": "08:00"}
    timezone        VARCHAR(50)
);
```

### Device Tokens
```sql
CREATE TABLE device_tokens (
    id              SERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    device_token    VARCHAR(500) NOT NULL,
    platform        VARCHAR(10),  -- ios, android
    app_version     VARCHAR(20),
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP,

    UNIQUE (user_id, device_token)
);
```

---

## Deep Dive

### 1. Push Notification Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Push Worker │────▶│ Token Store │────▶│   APNs/FCM  │
└──────┬──────┘     └─────────────┘     └──────┬──────┘
       │                                       │
       │         Feedback/Delivery Status      │
       └───────────────────────────────────────┘
```

**APNs (Apple Push Notification service):**
- HTTP/2 connection
- JWT authentication
- Device token validation

**FCM (Firebase Cloud Messaging):**
- HTTP v1 API
- Topic messaging
- Batch sending

```python
async def send_push(user_id: str, message: dict):
    tokens = await get_device_tokens(user_id)

    for token in tokens:
        if token.platform == 'ios':
            await apns_client.send(token.device_token, message)
        elif token.platform == 'android':
            await fcm_client.send(token.device_token, message)
```

### 2. Fanout Strategies

**Direct Send:**
```
For each user:
    For each channel:
        Send notification
```
- Simple, good for small scale
- Slow for large audiences

**Message Queue Fanout:**
```
┌──────────────┐     ┌───────────────┐
│  Broadcast   │────▶│  User Queue   │────▶ Worker
│   Message    │     │  (per user)   │
└──────────────┘     └───────────────┘
```

**Batch Processing:**
```python
async def send_batch_push(user_ids: list, message: dict):
    # Group by platform
    ios_tokens = []
    android_tokens = []

    for user_id in user_ids:
        tokens = await get_device_tokens(user_id)
        for token in tokens:
            if token.platform == 'ios':
                ios_tokens.append(token)
            else:
                android_tokens.append(token)

    # Batch send (FCM supports up to 500 per request)
    for batch in chunk(android_tokens, 500):
        await fcm_client.send_multicast(batch, message)
```

### 3. Rate Limiting & Throttling

```python
class NotificationRateLimiter:
    def __init__(self):
        self.user_limits = {
            'push': 100,    # per hour
            'sms': 10,      # per hour
            'email': 50     # per hour
        }

    async def can_send(self, user_id: str, channel: str) -> bool:
        key = f"rate:{user_id}:{channel}"
        count = await redis.incr(key)

        if count == 1:
            await redis.expire(key, 3600)

        return count <= self.user_limits[channel]
```

### 4. Retry & Dead Letter Queue

```
┌──────────┐     ┌─────────┐     ┌──────────┐
│  Queue   │────▶│ Worker  │────▶│ Provider │
└──────────┘     └────┬────┘     └────┬─────┘
                      │               │
                      │  Failure      │
                      ▼               │
                ┌──────────┐          │
                │  Retry   │◄─────────┘
                │  Queue   │
                └────┬─────┘
                     │ Max retries
                     ▼
                ┌──────────┐
                │   DLQ    │
                └──────────┘
```

```python
async def process_notification(notification: dict):
    retry_count = notification.get('retry_count', 0)
    max_retries = 3

    try:
        await send_notification(notification)
        await mark_as_sent(notification['id'])
    except ProviderError as e:
        if retry_count < max_retries:
            delay = exponential_backoff(retry_count)  # 1s, 2s, 4s
            await retry_queue.send(notification, delay=delay)
        else:
            await dlq.send(notification)
            await mark_as_failed(notification['id'])
```

### 5. Template Engine

```python
# Template storage
templates = {
    "welcome_email": {
        "subject": "Welcome to {{app_name}}, {{user_name}}!",
        "body": "Hi {{user_name}}, use code {{promo_code}} for 20% off!"
    }
}

def render_template(template_id: str, data: dict) -> dict:
    template = templates[template_id]
    return {
        "subject": template["subject"].format(**data),
        "body": template["body"].format(**data)
    }
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Provider rate limits | Queue с backpressure, multiple providers |
| Burst traffic | Pre-queue, scheduling |
| Invalid tokens | Feedback processing, token cleanup |
| User spam | Per-user rate limiting |
| Delivery tracking | Webhooks от providers |

---

## Trade-offs

### Push vs Pull
| Approach | Latency | Reliability | Battery |
|----------|---------|-------------|---------|
| Push | Low | Medium | Higher drain |
| Pull (polling) | High | High | Lower drain |

### At-least-once vs Exactly-once
- At-least-once: Проще, возможны дубликаты
- Exactly-once: Сложнее, требует idempotency

---

## На интервью

### Ключевые моменты
1. **Channel abstraction** — единый интерфейс для разных каналов
2. **Reliability** — retries, DLQ, feedback processing
3. **Scale** — queue per channel, batch processing
4. **User preferences** — opt-out, quiet hours, timezone

### Типичные follow-up
- Как обработать 1B пользователей для breaking news?
- Как приоритизировать urgent vs marketing?
- Как отслеживать delivery rate по каналам?
- Как реализовать A/B тестирование шаблонов?

---

## См. также

- [Очереди сообщений и события](../00-go/11-message-queues-events.md) — работа с Kafka, RabbitMQ и паттерны асинхронной обработки
- [Event-Driven архитектура](../08-architecture/05-event-driven.md) — проектирование событийных систем

---

[← Назад к списку тем](README.md)
