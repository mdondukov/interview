# Трек: Go Core

Фокус: глубокое понимание рантайма, параллелизма, инструментов и практических паттернов, необходимых для уверенной работы Go инженера.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по частоте вопросов на интервью

## Темы и прогресс

### Основы (чаще всего спрашивают)
- [ ] [00-language-overview](00-language-overview.md) — философия Go, система типов, generics, преимущества/ограничения
- [ ] [01-goroutines-channels](01-goroutines-channels.md) — буферизация, select, fan-in/out, pipelines, singleflight, errgroup
- [ ] [02-memory-slices-maps](02-memory-slices-maps.md) — структура slice/map, escape analysis, GC, оптимизации аллокаций
- [ ] [03-errors-context](03-errors-context.md) — error wrapping, sentinel vs typed, context cancellation/timeouts

### Продвинутые темы
- [ ] [04-go-runtime](04-go-runtime.md) — жизненный цикл горутин, work-stealing, G/M/P, GOMAXPROCS
- [ ] [05-concurrency-patterns](05-concurrency-patterns.md) — worker pools, rate limiting, Circuit Breaker, Bulkhead pattern
- [ ] [06-databases-persistence](06-databases-persistence.md) — database/sql, транзакции, миграции, ORM, тестирование с БД
- [ ] [07-tooling-testing](07-tooling-testing.md) — go modules, тестирование, fuzzing, golden files, contract tests

### Инфраструктура и сети
- [ ] [08-networking-grpc](08-networking-grpc.md) — net/http, middleware, gRPC, WebSocket, GraphQL vs REST
- [ ] [09-performance-profiling](09-performance-profiling.md) — pprof, tracing, GC tuning, GOMEMLIMIT, continuous profiling

### Архитектура (senior/architect level)
- [ ] [10-distributed-systems](10-distributed-systems.md) — консенсус, Saga, idempotency, distributed tracing, graceful degradation
- [ ] [11-message-queues-events](11-message-queues-events.md) — Kafka, NATS, RabbitMQ, exactly-once, event sourcing
- [ ] [12-security-crypto](12-security-crypto.md) — хранение паролей, JWT, TLS/mTLS, rate limiting, OWASP
- [ ] [13-reflection-codegen](13-reflection-codegen.md) — reflect, struct tags, go generate, wire, AST

## Структура каждого вопроса

Каждый вопрос содержит 6 секций:
1. **Зачем спрашивают** — контекст вопроса
2. **Короткий ответ** — 2-3 предложения для быстрого ответа
3. **Детальный разбор** — глубокое объяснение с диаграммами
4. **Пример** — рабочий код
5. **Типичные ошибки** — что проверяет интервьюер
6. **На интервью** — как отвечать + follow-up вопросы

## Статистика

- **Модулей:** 14
- **Вопросов:** ~130
- **Уровень:** Middle → Senior → Architect
