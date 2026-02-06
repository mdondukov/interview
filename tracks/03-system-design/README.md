# Трек: System Design

Фокус: глубокое понимание проектирования масштабируемых систем, практические паттерны и trade-offs, необходимые для уверенного прохождения system design интервью.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по частоте вопросов на интервью

## Темы и прогресс

### Основы (с этого начинать)
- [ ] [00-design-fundamentals](00-design-fundamentals.md) — CAP теорема, consistency models, SQL vs NoSQL, паттерны масштабирования, consistent hashing

### Классические задачи (чаще всего спрашивают)
- [ ] [01-url-shortener](01-url-shortener.md) — hashing, base62, collision handling, analytics, caching strategy
- [ ] [02-rate-limiter](02-rate-limiter.md) — token bucket, sliding window, distributed rate limiting, Redis lua scripts
- [ ] [03-notification-system](03-notification-system.md) — push/email/SMS, priority queues, fanout, delivery guarantees
- [ ] [04-chat-messenger](04-chat-messenger.md) — WebSocket, presence, message ordering, group chats, read receipts
- [ ] [05-news-feed](05-news-feed.md) — fanout on write/read, ranking algorithms, caching, pagination

### Поиск и хранение
- [ ] [06-search-autocomplete](06-search-autocomplete.md) — trie structures, ranking, type-ahead, персонализация
- [ ] [07-distributed-cache](07-distributed-cache.md) — consistent hashing, cache invalidation, write policies, Redis cluster
- [ ] [08-file-storage](08-file-storage.md) — blob storage, chunking, deduplication, CDN integration

### Сложные системы (senior/staff level)
- [ ] [09-video-streaming](09-video-streaming.md) — transcoding pipeline, adaptive bitrate, live streaming, CDN
- [ ] [10-payment-system](10-payment-system.md) — idempotency, ledger design, reconciliation, PCI compliance
- [ ] [11-booking-system](11-booking-system.md) — inventory management, distributed locks, overbooking prevention
- [ ] [12-ride-sharing](12-ride-sharing.md) — geospatial indexing, matching algorithms, ETA prediction, surge pricing

### Инфраструктура
- [ ] [13-distributed-id](13-distributed-id.md) — Snowflake, UUID, ULID, clock synchronization
- [ ] [14-monitoring-alerting](14-monitoring-alerting.md) — metrics collection, time-series DB, anomaly detection, alerting

### AI Systems
- [ ] [15-ai-systems](15-ai-systems.md) — chatbot platform, RAG architecture, AI agents, LLM serving

### Теория распределённых систем (критично для Staff)
- [ ] [16-consensus-raft](16-consensus-raft.md) — Raft алгоритм, leader election, log replication, split-brain prevention
- [ ] [17-distributed-transactions](17-distributed-transactions.md) — Saga pattern, 2PC, компенсирующие транзакции, Temporal
- [ ] [18-consistency-models](18-consistency-models.md) — linearizability, causal consistency, vector clocks, quorum
- [ ] [19-failure-modes](19-failure-modes.md) — split-brain, Byzantine failures, circuit breaker, graceful degradation

## Структура каждого вопроса

Каждый вопрос содержит 6 секций:
1. **Зачем спрашивают** — контекст вопроса
2. **Короткий ответ** — 2-3 предложения для быстрого ответа
3. **Детальный разбор** — глубокое объяснение с диаграммами
4. **Пример** — рабочий код или конфигурация
5. **Типичные ошибки** — что проверяет интервьюер
6. **На интервью** — как отвечать + follow-up вопросы

## Статистика

- **Модулей:** 19
- **Вопросов:** ~180
- **Уровень:** Middle → Senior → Staff

## Рекомендуемый порядок изучения

### Для Middle → Senior (2-5 YOE)
1. Основы (00) → Классические задачи (01-05) → Кэш и поиск (06-07) → Сложные системы (09-12)

### Для Senior → Staff (5+ YOE)
1. Быстрый обзор классики (01-05) → Теория распределённых систем (16-19) → Сложные системы (10-12) → Mock interviews
