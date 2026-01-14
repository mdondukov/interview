# 00. System Design Fundamentals

[← Назад к списку тем](README.md)

---

## Framework для System Design Interview

### Структура интервью (45-60 минут)

| Этап | Время | Что делать |
|------|-------|------------|
| 1. Requirements | 5 мин | Уточнить scope, функциональные и нефункциональные требования |
| 2. Capacity Estimation | 5 мин | DAU, RPS, storage, bandwidth |
| 3. High-Level Design | 10-15 мин | Основные компоненты, data flow |
| 4. Deep Dive | 15-20 мин | Детали 2-3 ключевых компонентов |
| 5. Bottlenecks & Trade-offs | 5-10 мин | Проблемы, альтернативы |

---

## 1. Requirements Gathering

### Функциональные требования (FR)
**Что система должна делать:**
- Какие основные use cases?
- Кто пользователи? (B2C, B2B, internal)
- Какие операции? (read-heavy, write-heavy, mixed)

### Нефункциональные требования (NFR)
**Как система должна работать:**

| Требование | Вопросы |
|------------|---------|
| Scale | Сколько пользователей? DAU? Peak load? |
| Latency | Допустимое время ответа? p50, p99? |
| Availability | Какой uptime нужен? 99.9%? 99.99%? |
| Consistency | Strong или eventual? |
| Durability | Можно ли терять данные? |

### Пример вопросов

```
- Сколько пользователей ожидается?
- Какое соотношение read/write?
- Нужна ли real-time доставка?
- Какая допустимая latency?
- Какой регион/география?
- Есть ли требования к хранению данных (GDPR)?
```

---

## 2. Capacity Estimation

### Формулы

```
QPS (Queries Per Second) = DAU × queries_per_user / 86400

Peak QPS = QPS × peak_factor (обычно 2-3x)

Storage = users × data_per_user × retention_period

Bandwidth = QPS × request_size
```

### Полезные числа

```
1 день = 86,400 секунд ≈ 100,000 секунд
1 месяц ≈ 2.5 миллиона секунд
1 год ≈ 30 миллионов секунд

1 KB = 1,000 bytes
1 MB = 1,000 KB
1 GB = 1,000 MB
1 TB = 1,000 GB

1 миллион = 10^6
1 миллиард = 10^9
```

### Пример расчёта

```
URL Shortener:
- 100M DAU
- 1 URL creation per user per day = 100M writes/day
- Write QPS = 100M / 100K = 1000 QPS
- Read:Write = 100:1 → Read QPS = 100,000 QPS
- Peak = 3x → 300,000 read QPS

Storage (5 years):
- 100M × 365 × 5 = 180B URLs
- Each URL: 100 bytes → 18 TB
```

---

## 3. High-Level Design

### Типичные компоненты

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Clients   │────▶│ Load Balancer│────▶│  API Gateway│
└─────────────┘     └──────────────┘     └─────────────┘
                                                │
                    ┌───────────────────────────┼───────────────────────────┐
                    │                           │                           │
              ┌─────▼─────┐              ┌──────▼──────┐             ┌──────▼──────┐
              │  Service A │              │  Service B  │             │  Service C  │
              └─────┬─────┘              └──────┬──────┘             └──────┬──────┘
                    │                           │                           │
              ┌─────▼─────┐              ┌──────▼──────┐             ┌──────▼──────┐
              │  Database │              │    Cache    │             │  Message Q  │
              └───────────┘              └─────────────┘             └─────────────┘
```

### Ключевые компоненты

| Компонент | Назначение | Примеры |
|-----------|------------|---------|
| Load Balancer | Распределение нагрузки | Nginx, HAProxy, AWS ALB |
| API Gateway | Routing, auth, rate limiting | Kong, AWS API Gateway |
| Application Server | Бизнес-логика | Custom services |
| Database | Persistent storage | PostgreSQL, MySQL, MongoDB |
| Cache | Fast access | Redis, Memcached |
| Message Queue | Async processing | Kafka, RabbitMQ, SQS |
| CDN | Static content delivery | CloudFront, Cloudflare |
| Object Storage | Large files | S3, GCS |

---

## 4. Database Design

### SQL vs NoSQL

| Критерий | SQL | NoSQL |
|----------|-----|-------|
| Schema | Fixed | Flexible |
| Scaling | Vertical (sharding сложнее) | Horizontal |
| Transactions | ACID | Usually eventual |
| Joins | Supported | Limited/None |
| Use case | Complex queries, relations | High scale, simple access |

### Типы NoSQL

| Тип | Примеры | Use case |
|-----|---------|----------|
| Key-Value | Redis, DynamoDB | Caching, sessions |
| Document | MongoDB, Couchbase | Flexible schema |
| Wide-Column | Cassandra, HBase | Time-series, analytics |
| Graph | Neo4j | Social networks, recommendations |

### Data Partitioning

```
Horizontal Partitioning (Sharding):
- По user_id: user_id % num_shards
- По geography: region-based
- По time: year/month partitions

Вертикальное разделение:
- Разные таблицы в разных БД
- Часто используемые колонки отдельно
```

---

## 5. Caching

### Caching Strategies

| Стратегия | Описание | Когда использовать |
|-----------|----------|-------------------|
| Cache-Aside | App читает/пишет кэш | Read-heavy, tolerance to stale |
| Write-Through | Write to cache → DB | Consistency важнее latency |
| Write-Behind | Write to cache, async to DB | Write-heavy |
| Read-Through | Cache reads from DB | Simplicity |

### Cache-Aside Pattern

```python
def get_user(user_id):
    # 1. Check cache
    user = cache.get(f"user:{user_id}")
    if user:
        return user

    # 2. Cache miss - read from DB
    user = db.get_user(user_id)
    if user:
        # 3. Populate cache
        cache.set(f"user:{user_id}", user, ttl=3600)

    return user
```

### Cache Invalidation

```
1. TTL (Time To Live) - простейший способ
2. Event-based - при изменении данных
3. Write-through - автоматически при записи
```

---

## 6. Consistency & Availability

### CAP Theorem

```
       Consistency
           /\
          /  \
         /    \
        /      \
       /   CA   \
      /          \
     /____________\
Availability    Partition
                Tolerance
```

- **CP:** Консистентность + Partition Tolerance (MongoDB, HBase)
- **AP:** Availability + Partition Tolerance (Cassandra, DynamoDB)
- **CA:** Не существует в распределённых системах

### PACELC

```
If Partition:
  Choose Availability or Consistency
Else:
  Choose Latency or Consistency
```

### Consistency Models

| Модель | Описание |
|--------|----------|
| Strong | Все читают последнее записанное |
| Eventual | Eventually все увидят одно и то же |
| Causal | Причинно связанные операции упорядочены |
| Read-your-writes | Клиент видит свои записи |

---

## 7. Communication Patterns

### Synchronous

```
REST API:
+ Простота, стандартизация
- Coupling, latency

gRPC:
+ Производительность, типизация
- Сложнее дебажить
```

### Asynchronous

```
Message Queue:
+ Decoupling, reliability, buffering
- Complexity, eventual consistency

Event-Driven:
+ Loose coupling, scalability
- Debugging сложнее
```

### Long Polling vs WebSocket vs SSE

| Метод | Направление | Use case |
|-------|-------------|----------|
| Long Polling | Server → Client | Простая реализация |
| WebSocket | Bidirectional | Chat, gaming |
| SSE | Server → Client | Notifications, feeds |

---

## 8. Scalability Patterns

### Horizontal vs Vertical Scaling

```
Vertical (Scale Up):
- Больше CPU/RAM/Disk
- Простота
- Есть предел

Horizontal (Scale Out):
- Больше машин
- Требует distributed design
- Почти нет предела
```

### Load Balancing Algorithms

| Алгоритм | Описание |
|----------|----------|
| Round Robin | По очереди |
| Weighted Round Robin | С учётом мощности |
| Least Connections | К наименее загруженному |
| IP Hash | Sticky sessions |

### Database Scaling

```
1. Read Replicas - для read-heavy
2. Sharding - для write-heavy
3. Caching - для горячих данных
```

---

## 9. Reliability Patterns

### Replication

```
Single Leader:
- Один writer, много readers
- Простота, но bottleneck на write

Multi-Leader:
- Несколько writers
- Conflict resolution нужен

Leaderless:
- Quorum reads/writes
- W + R > N для consistency
```

### Failure Handling

| Паттерн | Описание |
|---------|----------|
| Timeout | Ограничение времени ожидания |
| Retry | Повторные попытки (с backoff) |
| Circuit Breaker | Прекращение запросов при failures |
| Bulkhead | Изоляция failures |
| Fallback | Запасной вариант |

---

## 10. На интервью

### Чек-лист

1. **Уточни требования** — не делай предположений
2. **Начни с high-level** — не погружайся сразу в детали
3. **Думай вслух** — показывай ход мысли
4. **Обсуждай trade-offs** — нет идеального решения
5. **Используй числа** — capacity estimation
6. **Будь готов к deep dive** — знай детали компонентов

### Типичные ошибки

1. Пропуск requirements gathering
2. Слишком детальный дизайн сразу
3. Один "правильный" ответ вместо обсуждения trade-offs
4. Игнорирование non-functional requirements
5. Отсутствие capacity estimation

### Полезные фразы

```
"Let me clarify the requirements..."
"The key trade-off here is..."
"One approach would be... but we could also..."
"Let me do a quick capacity estimation..."
"The bottleneck here is... we can solve it by..."
```

---

[← Назад к списку тем](README.md)
