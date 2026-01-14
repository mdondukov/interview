# 06. Sharding

[← Назад к списку тем](README.md)

---

## Зачем шардирование

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Scaling Strategies                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Vertical Scaling (Scale Up)                                        │
│  ───────────────────────────                                        │
│  ┌──────────┐      ┌──────────────┐                                 │
│  │ 4 CPU    │  →   │  64 CPU      │                                 │
│  │ 16GB RAM │      │  512GB RAM   │                                 │
│  │ 1TB SSD  │      │  10TB NVMe   │                                 │
│  └──────────┘      └──────────────┘                                 │
│  Ограничено железом, дорого                                         │
│                                                                     │
│  Horizontal Scaling (Scale Out) - Sharding                          │
│  ─────────────────────────────────────────                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │  │ Shard N  │             │
│  │  25%     │  │  25%     │  │  25%     │  │  25%     │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
│  Линейное масштабирование, сложнее управлять                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Shard Key Selection

### Критерии выбора

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Good Shard Key Properties                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. High Cardinality                                                │
│     - Много уникальных значений                                     │
│     - ❌ status (3 значения) — плохо                                │
│     - ✅ user_id (миллионы) — хорошо                                │
│                                                                     │
│  2. Even Distribution                                               │
│     - Равномерное распределение данных                              │
│     - ❌ created_date — hotspots на свежих данных                   │
│     - ✅ hash(user_id) — равномерно                                 │
│                                                                     │
│  3. Query Locality                                                  │
│     - Запросы попадают на 1 shard                                   │
│     - ✅ user_id если запросы "данные пользователя"                 │
│     - ❌ product_id если часто JOIN users и products                │
│                                                                     │
│  4. Immutable                                                       │
│     - Значение не меняется                                          │
│     - Иначе нужен reshard                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Примеры shard keys

```sql
-- E-commerce
-- Shard by user_id: все данные пользователя на одном шарде
-- orders, cart, wishlists, reviews — всё вместе

-- Multi-tenant SaaS
-- Shard by tenant_id: изоляция между клиентами
-- Compliance, data locality

-- Social Network
-- Shard by user_id для user data
-- Отдельно: shard by post_id для глобального feed

-- Analytics
-- Shard by date: time-series data
-- Старые шарды можно архивировать
```

---

## Sharding Strategies

### Range-based Sharding

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Range-based Sharding                             │
│                                                                     │
│  Key Range          Shard                                           │
│  ─────────          ─────                                           │
│  A - F              Shard 1                                         │
│  G - M              Shard 2                                         │
│  N - S              Shard 3                                         │
│  T - Z              Shard 4                                         │
│                                                                     │
│  ✅ Простые range queries (все записи A-C)                          │
│  ✅ Легко добавить шард (split range)                               │
│  ❌ Hotspots (популярные ranges)                                    │
│  ❌ Неравномерное распределение                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Hash-based Sharding

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Hash-based Sharding                             │
│                                                                     │
│  shard = hash(key) % num_shards                                     │
│                                                                     │
│  user_id = 12345                                                    │
│  hash(12345) = 7823456                                              │
│  shard = 7823456 % 4 = 0  → Shard 0                                 │
│                                                                     │
│  ✅ Равномерное распределение                                        │
│  ✅ Нет hotspots                                                     │
│  ❌ Range queries требуют scatter-gather                            │
│  ❌ Resharding сложен (все данные перемещаются)                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Consistent Hashing

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Consistent Hashing                               │
│                                                                     │
│              ●──── Shard A                                          │
│           ╱     ╲                                                   │
│         ╱         ╲                                                 │
│        ●           ●──── Shard B                                    │
│        │   Ring    │                                                │
│        ●           ●──── Shard C                                    │
│         ╲         ╱                                                 │
│           ╲     ╱                                                   │
│              ●──── Shard D                                          │
│                                                                     │
│  key → hash(key) → найти следующий node по часовой                  │
│                                                                     │
│  При добавлении/удалении node:                                      │
│  - Перемещается только 1/N данных                                   │
│  - Вместо всех данных как в hash % N                                │
│                                                                     │
│  Virtual nodes:                                                     │
│  - Каждый физический node = много виртуальных                       │
│  - Более равномерное распределение                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Directory-based Sharding

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Directory-based Sharding                           │
│                                                                     │
│         ┌─────────────────────┐                                     │
│         │  Lookup Service     │                                     │
│         │  ─────────────────  │                                     │
│         │  key → shard        │                                     │
│         │                     │                                     │
│         │  user_1  → Shard A  │                                     │
│         │  user_2  → Shard B  │                                     │
│         │  user_3  → Shard A  │                                     │
│         │  ...                │                                     │
│         └──────────┬──────────┘                                     │
│                    │                                                │
│      ┌─────────────┼─────────────┐                                  │
│      ▼             ▼             ▼                                  │
│  ┌───────┐    ┌───────┐    ┌───────┐                                │
│  │Shard A│    │Shard B│    │Shard C│                                │
│  └───────┘    └───────┘    └───────┘                                │
│                                                                     │
│  ✅ Гибкость: любой маппинг                                          │
│  ✅ Легко перемещать данные                                          │
│  ❌ Single point of failure                                          │
│  ❌ Дополнительный hop                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Cross-Shard Operations

### Scatter-Gather

```go
// Query goes to all shards, results aggregated
func searchProducts(ctx context.Context, query string) ([]*Product, error) {
    results := make(chan []*Product, len(shards))
    errors := make(chan error, len(shards))

    // Scatter: query all shards
    for _, shard := range shards {
        go func(s *Shard) {
            products, err := s.Search(ctx, query)
            if err != nil {
                errors <- err
                return
            }
            results <- products
        }(shard)
    }

    // Gather: collect results
    var allProducts []*Product
    for i := 0; i < len(shards); i++ {
        select {
        case products := <-results:
            allProducts = append(allProducts, products...)
        case err := <-errors:
            return nil, err
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }

    // Sort and limit
    sort.Slice(allProducts, func(i, j int) bool {
        return allProducts[i].Score > allProducts[j].Score
    })

    return allProducts[:min(100, len(allProducts))], nil
}
```

### Cross-Shard Joins

```sql
-- ❌ Плохо: join между шардами
SELECT o.*, p.*
FROM orders o        -- sharded by user_id
JOIN products p      -- sharded by product_id (другой шард!)
ON o.product_id = p.id;

-- ✅ Решение 1: Денормализация
-- Храним product_name, product_price в orders

-- ✅ Решение 2: Application-level join
-- 1. Получить orders с текущего шарда
-- 2. Собрать product_ids
-- 3. Batch-запрос к products шарду
-- 4. Объединить в памяти

-- ✅ Решение 3: Global tables
-- Небольшие справочники реплицируются на все шарды
-- products, categories, countries
```

### Distributed Transactions

```go
// Two-Phase Commit (2PC)
func transfer(ctx context.Context, fromUser, toUser int64, amount int64) error {
    fromShard := getShard(fromUser)
    toShard := getShard(toUser)

    // Phase 1: Prepare
    tx1, err := fromShard.Begin(ctx)
    if err != nil {
        return err
    }
    tx2, err := toShard.Begin(ctx)
    if err != nil {
        tx1.Rollback()
        return err
    }

    // Debit
    if err := tx1.Exec("UPDATE accounts SET balance = balance - $1 WHERE user_id = $2", amount, fromUser); err != nil {
        tx1.Rollback()
        tx2.Rollback()
        return err
    }

    // Credit
    if err := tx2.Exec("UPDATE accounts SET balance = balance + $1 WHERE user_id = $2", amount, toUser); err != nil {
        tx1.Rollback()
        tx2.Rollback()
        return err
    }

    // Phase 2: Commit
    if err := tx1.Commit(); err != nil {
        tx2.Rollback()
        return err
    }
    if err := tx2.Commit(); err != nil {
        // Проблема: tx1 уже committed!
        // Нужен компенсационный механизм
        return err
    }

    return nil
}

// Лучше: Saga pattern (см. architecture/06-integration-patterns.md)
```

---

## Resharding

### Когда нужен resharding

```
- Shard переполнен (storage/CPU)
- Hotspot на одном шарде
- Изменение количества шардов
- Изменение shard key
```

### Стратегии

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Resharding Strategies                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Double-write migration                                          │
│     - Пишем в старый И новый шард                                   │
│     - Копируем старые данные                                        │
│     - Переключаем reads                                             │
│     - Выключаем writes в старый                                     │
│                                                                     │
│  2. Backfill + cutover                                              │
│     - Копируем snapshot                                             │
│     - Ловим новые изменения (CDC)                                   │
│     - Атомарный cutover                                             │
│                                                                     │
│  3. Virtual shards                                                  │
│     - Много логических шардов (1000+)                               │
│     - Маппинг: logical shard → physical node                        │
│     - Перемещаем logical shards между nodes                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Инструменты

### Vitess (для MySQL)

```yaml
# Vitess keyspace sharding
keyspaces:
  - name: commerce
    sharded: true
    vindexes:
      user_hash:
        type: hash
    tables:
      users:
        column_vindexes:
          - column: id
            name: user_hash
      orders:
        column_vindexes:
          - column: user_id
            name: user_hash
```

### Citus (для PostgreSQL)

```sql
-- Создание distributed table
SELECT create_distributed_table('orders', 'user_id');

-- Создание reference table (реплицируется на все ноды)
SELECT create_reference_table('products');

-- Query routing автоматический
SELECT * FROM orders WHERE user_id = 123;
-- Идёт только на нужный шард!

SELECT * FROM orders WHERE created_at > '2024-01-01';
-- Scatter-gather на все шарды
```

---

## Trade-offs

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Sharding Trade-offs                              │
├────────────────────────────────┬────────────────────────────────────┤
│            Pros                │            Cons                    │
├────────────────────────────────┼────────────────────────────────────┤
│ Horizontal scalability         │ Operational complexity             │
│ Data locality                  │ Cross-shard queries slow           │
│ Isolation (multi-tenant)       │ Joins become hard                  │
│ Geographic distribution        │ Transactions across shards         │
│                                │ Resharding is painful              │
│                                │ Backup/restore complexity          │
└────────────────────────────────┴────────────────────────────────────┘

Когда НЕ шардировать:
- Таблица < 100GB
- Можно vertical scaling
- Много cross-shard queries
- Сложные JOINs между таблицами
```

---

## См. также

- [Replication](./05-replication.md) — репликация для высокой доступности
- [Partitioning](../07-distributed-systems/06-partitioning.md) — партиционирование в распределённых системах

---

## На интервью

### Типичные вопросы

1. **Как выбрать shard key?**
   - High cardinality
   - Even distribution
   - Query locality
   - Immutable

2. **Hash vs Range sharding?**
   - Hash: равномерность, но нет range queries
   - Range: range queries, но hotspots

3. **Что такое consistent hashing?**
   - Ring-based distribution
   - Минимальное перемещение при resize

4. **Как делать JOIN между шардами?**
   - Денормализация
   - Application-level join
   - Global/reference tables

5. **Как происходит resharding?**
   - Double-write
   - Backfill + cutover
   - Virtual shards

---

[← Назад к списку тем](README.md)
