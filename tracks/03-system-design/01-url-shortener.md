# 01. URL Shortener

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Создание короткой ссылки из длинной
- Редирект по короткой ссылке
- Опционально: кастомные alias, expiration, analytics

### Нефункциональные
- High availability
- Low latency redirect (< 100ms)
- Короткие URL должны быть непредсказуемыми
- Scale: 100M DAU

---

## Capacity Estimation

```
Write:
- 100M DAU × 1 URL/user/day = 100M URLs/day
- Write QPS = 100M / 86400 ≈ 1,200 QPS
- Peak: 3,600 QPS

Read:
- Read:Write = 100:1
- Read QPS = 120,000 QPS
- Peak: 360,000 QPS

Storage (5 years):
- 100M × 365 × 5 = 182B URLs
- Each record: ~500 bytes (URL + metadata)
- Total: 182B × 500B = 91 TB

Bandwidth:
- Read: 360K × 500B = 180 MB/s
```

---

## High-Level Design

```
                                    ┌─────────────┐
                                    │    CDN      │
                                    └──────┬──────┘
                                           │
┌──────────┐    ┌───────────────┐    ┌─────▼─────┐    ┌─────────────┐
│  Client  │───▶│ Load Balancer │───▶│ API Server│───▶│    Cache    │
└──────────┘    └───────────────┘    └─────┬─────┘    └──────┬──────┘
                                           │                 │
                                           │          ┌──────▼──────┐
                                           └─────────▶│  Database   │
                                                      └─────────────┘
```

### Компоненты
1. **Load Balancer** — распределение нагрузки
2. **API Servers** — stateless, горизонтально масштабируются
3. **Cache** — Redis для популярных URL
4. **Database** — хранение маппинга short → long URL

---

## API Design

### Create Short URL
```
POST /api/v1/shorten
Request:
{
    "long_url": "https://example.com/very/long/path",
    "custom_alias": "my-link",    // optional
    "expiration": "2025-12-31"    // optional
}

Response:
{
    "short_url": "https://short.ly/abc123",
    "long_url": "https://example.com/very/long/path",
    "expiration": "2025-12-31"
}
```

### Redirect
```
GET /{short_code}

Response: 301/302 Redirect to long_url
```

### Get Analytics (optional)
```
GET /api/v1/analytics/{short_code}

Response:
{
    "clicks": 1234,
    "created_at": "2024-01-01",
    "last_accessed": "2024-06-15"
}
```

---

## Data Model

### URL Table
```sql
CREATE TABLE urls (
    id              BIGINT PRIMARY KEY,
    short_code      VARCHAR(10) UNIQUE NOT NULL,
    long_url        TEXT NOT NULL,
    user_id         BIGINT,
    created_at      TIMESTAMP DEFAULT NOW(),
    expires_at      TIMESTAMP,
    click_count     BIGINT DEFAULT 0,

    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id)
);
```

### Analytics Table (optional)
```sql
CREATE TABLE clicks (
    id          BIGINT PRIMARY KEY,
    url_id      BIGINT REFERENCES urls(id),
    timestamp   TIMESTAMP,
    ip_address  VARCHAR(45),
    user_agent  TEXT,
    referer     TEXT,
    country     VARCHAR(2)
);
```

---

## Deep Dive

### 1. Short Code Generation

**Option A: Hash-based**
```python
import hashlib
import base64

def generate_short_code(long_url: str) -> str:
    # MD5 hash → first 6 bytes → base62
    hash_bytes = hashlib.md5(long_url.encode()).digest()[:6]
    return base62_encode(int.from_bytes(hash_bytes, 'big'))

def base62_encode(num: int) -> str:
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    result = []
    while num > 0:
        result.append(chars[num % 62])
        num //= 62
    return ''.join(reversed(result)) or '0'
```

**Проблема:** Коллизии
**Решение:** Append counter при коллизии

**Option B: Counter-based (Distributed ID)**
```python
# Используем Snowflake-like ID generator
short_code = base62_encode(snowflake_id())
```

**Option C: Pre-generated Keys**
```
1. Отдельный Key Generation Service
2. Заранее генерирует уникальные ключи
3. API Server запрашивает batch ключей
4. Помечает использованные ключи в БД
```

### 2. Encoding

**Base62 vs Base64**
- Base62: [0-9a-zA-Z] — URL-safe
- Base64: включает +/= — нужно encoding

**Длина кода:**
```
62^6 = 56.8 billion combinations
62^7 = 3.5 trillion combinations
```

### 3. Database Choice

**SQL (PostgreSQL):**
- ACID транзакции
- Простые JOIN для analytics
- Sharding по short_code hash

**NoSQL (DynamoDB, Cassandra):**
- Лучше scale
- Key-value access pattern подходит
- Partition key = short_code

### 4. Caching Strategy

```python
def redirect(short_code: str):
    # 1. Check cache
    long_url = cache.get(short_code)
    if long_url:
        return redirect_to(long_url)

    # 2. Check DB
    url_record = db.get_by_short_code(short_code)
    if not url_record:
        return 404

    # 3. Update cache (LRU)
    cache.set(short_code, url_record.long_url, ttl=3600)

    # 4. Async update click count
    analytics_queue.send({"short_code": short_code, ...})

    return redirect_to(url_record.long_url)
```

**Cache sizing:**
```
Top 20% URLs = 80% traffic (Pareto)
20% of 182B = 36B URLs
36B × 500B = 18 TB cache

With LRU and hot data only:
~1-5 TB Redis cluster
```

### 5. 301 vs 302 Redirect

| Code | Type | Caching | Use case |
|------|------|---------|----------|
| 301 | Permanent | Browser caches | Better SEO, less server load |
| 302 | Temporary | No caching | Analytics tracking |

**Рекомендация:** 302 для analytics, 301 для максимальной производительности

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Write hotspot | Pre-generated keys, distributed ID |
| Read hotspot | Cache popular URLs, CDN |
| Database bottleneck | Sharding by short_code hash |
| Single point of failure | Replication, multi-AZ |
| Collision handling | Retry with suffix, unique constraint |

---

## Trade-offs

### Hash vs Counter
| Approach | Pros | Cons |
|----------|------|------|
| Hash | Stateless, deterministic | Collisions possible |
| Counter | No collisions | Requires coordination |
| Pre-generated | Fast, no collisions | Storage overhead |

### SQL vs NoSQL
| Approach | Pros | Cons |
|----------|------|------|
| SQL | ACID, analytics joins | Harder to scale |
| NoSQL | Easy scaling | Limited queries |

---

## Масштабирование

### Read Scaling
1. Read replicas
2. CDN для популярных URL
3. Redis cluster

### Write Scaling
1. Sharding по hash(short_code) % N
2. Key Generation Service
3. Async analytics (Kafka → clickstream)

### Geographic Distribution
```
┌─────────┐         ┌─────────┐
│  US-DC  │◄───────►│  EU-DC  │
└────┬────┘         └────┬────┘
     │                   │
     ▼                   ▼
┌─────────┐         ┌─────────┐
│ US Users│         │ EU Users│
└─────────┘         └─────────┘
```

---

## На интервью

### Ключевые моменты
1. **Short code generation** — главный design decision
2. **Read-heavy** — кэширование критично
3. **Idempotency** — один long_url = один short_code?
4. **Analytics** — async, отдельный pipeline

### Типичные follow-up
- Как обработать 10x traffic spike?
- Как реализовать custom aliases?
- Как удалять expired URLs?
- Как предотвратить abuse (spam URLs)?

---

## См. также

- [Генерация распределённых ID](./13-distributed-id.md) — алгоритмы Snowflake, ULID и другие подходы к генерации уникальных идентификаторов
- [Шардирование баз данных](../06-databases/06-sharding.md) — стратегии горизонтального масштабирования хранилища

---

[← Назад к списку тем](README.md)
