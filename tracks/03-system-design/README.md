# Трек: System Design

Фокус: проектирование масштабируемых распределённых систем. Каждый модуль — классическая задача с полным разбором: требования, API, data model, deep dive, trade-offs.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по частоте на интервью

## Темы и прогресс

### Фундамент
- [ ] [00-design-fundamentals](00-design-fundamentals.md) — framework, capacity estimation, CAP

### Классические задачи (часто спрашивают)
- [ ] [01-url-shortener](01-url-shortener.md) — hashing, base62, analytics
- [ ] [02-rate-limiter](02-rate-limiter.md) — token bucket, sliding window, distributed
- [ ] [03-notification-system](03-notification-system.md) — push, email, SMS, fanout
- [ ] [04-chat-messenger](04-chat-messenger.md) — WebSocket, presence, ordering
- [ ] [05-news-feed](05-news-feed.md) — fanout, ranking, caching

### Поиск и хранение
- [ ] [06-search-autocomplete](06-search-autocomplete.md) — trie, ranking, type-ahead
- [ ] [07-distributed-cache](07-distributed-cache.md) — consistent hashing, invalidation
- [ ] [08-file-storage](08-file-storage.md) — blob storage, chunking, CDN

### Сложные системы
- [ ] [09-video-streaming](09-video-streaming.md) — transcoding, adaptive bitrate, live
- [ ] [10-payment-system](10-payment-system.md) — idempotency, ledger, reconciliation
- [ ] [11-booking-system](11-booking-system.md) — inventory, distributed locks
- [ ] [12-ride-sharing](12-ride-sharing.md) — geospatial, matching, ETA

### Инфраструктура
- [ ] [13-distributed-id](13-distributed-id.md) — Snowflake, UUID, ULID
- [ ] [14-monitoring-alerting](14-monitoring-alerting.md) — metrics, time-series, anomaly

### AI Systems
- [ ] [15-ai-systems](15-ai-systems.md) — chatbot platform, RAG knowledge base, AI agents

## Формат задачи

```markdown
## Система: [Название]

### Требования
**Функциональные:** что система делает
**Нефункциональные:** latency, availability, scale

### Capacity Estimation
- DAU, RPS, storage

### High-Level Design
[Диаграмма]

### API Design
[REST/gRPC endpoints]

### Data Model
[Схема БД]

### Deep Dive
[Ключевые компоненты]

### Bottlenecks & Solutions
[Проблемы и решения]

### Trade-offs
[Альтернативные подходы]

### На интервью
[Советы]
```

## Статистика

- **Модулей:** 16
- **Уровень:** Senior → Staff
