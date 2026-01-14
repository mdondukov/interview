# Databases

Теория баз данных и практические паттерны для Senior-level интервью.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по частоте на интервью

## Темы и прогресс

### SQL и оптимизация
- [ ] [00-sql-fundamentals](00-sql-fundamentals.md) — SELECT, JOINs, subqueries, window functions
- [ ] [01-indexing](01-indexing.md) — B-tree, hash, composite, covering, partial indexes
- [ ] [02-query-optimization](02-query-optimization.md) — EXPLAIN, query plans, N+1, batch loading

### Транзакции и моделирование
- [ ] [03-transactions](03-transactions.md) — ACID, isolation levels, MVCC, deadlocks
- [ ] [04-data-modeling](04-data-modeling.md) — normalization, denormalization, trade-offs

### Масштабирование
- [ ] [05-replication](05-replication.md) — sync/async, leader-follower, multi-leader
- [ ] [06-sharding](06-sharding.md) — horizontal/vertical, shard key, resharding

### PostgreSQL
- [ ] [07-postgresql-internals](07-postgresql-internals.md) — VACUUM, WAL, connection pooling

### NoSQL
- [ ] [08-nosql-patterns](08-nosql-patterns.md) — document, key-value, column, graph
- [ ] [09-redis-patterns](09-redis-patterns.md) — data structures, pub/sub, Lua scripts

## Формат вопроса

```markdown
## Тема: [Название]

### Теория
[Объяснение концепции]

### Пример
[SQL/код с объяснением]

### На интервью
- Типичные вопросы
- Ключевые моменты
```

## Статистика

- **Модулей:** 10
- **Вопросов:** ~50
- **Уровень:** Middle → Senior
