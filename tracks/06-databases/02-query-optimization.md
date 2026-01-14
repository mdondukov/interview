# 02. Query Optimization

[← Назад к списку тем](README.md)

---

## EXPLAIN и Query Plans

### Базовый EXPLAIN

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Результат:
--                        QUERY PLAN
-- ---------------------------------------------------------
-- Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=256)
--   Index Cond: (email = 'test@example.com'::text)
```

### EXPLAIN ANALYZE (с выполнением)

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM users WHERE email = 'test@example.com';

-- Результат:
-- Index Scan using idx_users_email on users
--   (cost=0.42..8.44 rows=1 width=256)
--   (actual time=0.028..0.029 rows=1 loops=1)
--   Buffers: shared hit=4
-- Planning Time: 0.089 ms
-- Execution Time: 0.045 ms
```

### Метрики EXPLAIN

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EXPLAIN Метрики                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  cost=0.42..8.44                                                    │
│        ▲    ▲                                                       │
│        │    └── Total cost (startup + all rows)                     │
│        └─────── Startup cost (before first row)                     │
│                                                                     │
│  rows=1           — Estimated rows                                  │
│  width=256        — Average row size (bytes)                        │
│  actual time=...  — Real execution time (ms)                        │
│  loops=1          — How many times executed                         │
│  Buffers: shared hit=4  — Pages read from cache                     │
│           shared read=2 — Pages read from disk                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Типы Scan операций

### Sequential Scan

```sql
-- Читает ВСЕ строки таблицы
EXPLAIN SELECT * FROM users WHERE status = 'active';

-- Seq Scan on users  (cost=0.00..1234.00 rows=50000 width=256)
--   Filter: (status = 'active'::text)
--   Rows Removed by Filter: 50000

-- Когда Sequential Scan лучше:
-- - Маленькие таблицы
-- - Низкая селективность (много строк подходит)
-- - Нужно прочитать > ~10-20% таблицы
```

### Index Scan

```sql
-- Использует индекс + читает строки из таблицы
EXPLAIN SELECT * FROM users WHERE id = 100;

-- Index Scan using users_pkey on users
--   (cost=0.42..8.44 rows=1 width=256)
--   Index Cond: (id = 100)
```

### Index Only Scan

```sql
-- Читает только из индекса (covering index)
EXPLAIN SELECT email FROM users WHERE email = 'test@example.com';

-- Index Only Scan using idx_users_email on users
--   (cost=0.42..4.44 rows=1 width=32)
--   Index Cond: (email = 'test@example.com'::text)
--   Heap Fetches: 0  -- Не читали таблицу!
```

### Bitmap Scan

```sql
-- Двухэтапный: сначала bitmap из индекса, потом строки
EXPLAIN SELECT * FROM orders WHERE status IN ('pending', 'processing');

-- Bitmap Heap Scan on orders
--   Recheck Cond: (status = ANY ('{pending,processing}'::text[]))
--   ->  Bitmap Index Scan on idx_orders_status
--         Index Cond: (status = ANY ('{pending,processing}'::text[]))

-- Когда используется:
-- - Много строк подходит, но не все
-- - OR между индексированными колонками
-- - IN с несколькими значениями
```

---

## JOIN стратегии

### Nested Loop

```sql
-- Для каждой строки внешней таблицы ищем во внутренней
-- O(n * m), но быстро если внутренняя маленькая или есть индекс

EXPLAIN SELECT * FROM orders o JOIN users u ON o.user_id = u.id;

-- Nested Loop  (cost=...)
--   ->  Seq Scan on orders o
--   ->  Index Scan using users_pkey on users u
--         Index Cond: (id = o.user_id)

-- Хорошо когда:
-- - Внешняя таблица маленькая
-- - Есть индекс на join колонке внутренней таблицы
```

### Hash Join

```sql
-- Строит hash table из меньшей таблицы, сканирует большую

-- Hash Join  (cost=...)
--   Hash Cond: (o.user_id = u.id)
--   ->  Seq Scan on orders o
--   ->  Hash
--         ->  Seq Scan on users u

-- Хорошо когда:
-- - Нет подходящего индекса
-- - Обе таблицы большие
-- - Достаточно work_mem для hash table
```

### Merge Join

```sql
-- Сортирует обе таблицы, затем merge
-- Требует сортированного входа

-- Merge Join  (cost=...)
--   Merge Cond: (o.user_id = u.id)
--   ->  Sort
--         ->  Seq Scan on orders o
--   ->  Sort
--         ->  Seq Scan on users u

-- Хорошо когда:
-- - Данные уже отсортированы (индекс)
-- - Очень большие таблицы
-- - Эффективно для ORDER BY с JOIN
```

---

## N+1 Problem

### Проблема

```go
// ❌ Плохо: N+1 запросов
users := db.Query("SELECT * FROM users LIMIT 100")
for _, user := range users {
    orders := db.Query("SELECT * FROM orders WHERE user_id = ?", user.ID)
    // 1 + 100 = 101 запрос!
}
```

### Решение 1: JOIN

```go
// ✅ JOIN: 1 запрос
rows := db.Query(`
    SELECT u.*, o.*
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    LIMIT 100
`)
```

### Решение 2: Batch Loading (IN)

```go
// ✅ Batch: 2 запроса
users := db.Query("SELECT * FROM users LIMIT 100")
userIDs := extractIDs(users)

orders := db.Query(`
    SELECT * FROM orders
    WHERE user_id = ANY($1)
`, userIDs)
// Группируем orders по user_id в памяти
```

### Решение 3: DataLoader Pattern

```go
// Batching + Caching
type OrderLoader struct {
    cache map[string][]*Order
    batch []string
}

func (l *OrderLoader) Load(userID string) []*Order {
    if cached, ok := l.cache[userID]; ok {
        return cached
    }
    l.batch = append(l.batch, userID)
    // Batch execution at end of request
    return nil
}

func (l *OrderLoader) Execute() {
    orders := db.Query("SELECT * FROM orders WHERE user_id = ANY($1)", l.batch)
    // Populate cache
}
```

---

## Оптимизация запросов

### Избегайте функций на индексированных колонках

```sql
-- ❌ Плохо: функция убивает индекс
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ Хорошо: range query
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- Или создать functional index:
CREATE INDEX idx_users_year ON users(EXTRACT(YEAR FROM created_at));
```

### LIMIT для пагинации

```sql
-- ❌ Плохо для больших offset
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- Читает 10020 строк, отбрасывает 10000

-- ✅ Хорошо: cursor-based pagination
SELECT * FROM orders
WHERE created_at < '2024-01-15 10:30:00'  -- last item from previous page
ORDER BY created_at DESC
LIMIT 20;

-- Или keyset pagination
SELECT * FROM orders
WHERE (created_at, id) < ('2024-01-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

### Избегайте SELECT *

```sql
-- ❌ Плохо: читает все колонки
SELECT * FROM users WHERE id = 100;

-- ✅ Хорошо: только нужные колонки
SELECT id, name, email FROM users WHERE id = 100;
-- Может использовать covering index
```

### Подзапросы vs JOINs

```sql
-- ❌ Correlated subquery (выполняется для каждой строки)
SELECT *
FROM orders o
WHERE o.total > (
    SELECT AVG(total) FROM orders WHERE user_id = o.user_id
);

-- ✅ JOIN с CTE
WITH user_avg AS (
    SELECT user_id, AVG(total) as avg_total
    FROM orders
    GROUP BY user_id
)
SELECT o.*
FROM orders o
JOIN user_avg ua ON o.user_id = ua.user_id
WHERE o.total > ua.avg_total;
```

### EXISTS vs IN

```sql
-- Для больших подзапросов EXISTS обычно лучше
-- EXISTS останавливается на первом совпадении

-- IN (может быть медленным)
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- EXISTS (обычно быстрее)
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 1000
);
```

---

## Статистика и Planner

### ANALYZE

```sql
-- Обновить статистику таблицы
ANALYZE users;
ANALYZE orders;

-- Автоматически: autovacuum делает это
-- Но после большого INSERT/DELETE нужно вручную
```

### Настройки планировщика

```sql
-- Посмотреть текущие настройки
SHOW random_page_cost;    -- 4.0 (для HDD), 1.1 для SSD
SHOW seq_page_cost;       -- 1.0
SHOW effective_cache_size; -- оценка RAM для кеша

-- Настроить для SSD
SET random_page_cost = 1.1;

-- Отключить определённые стратегии (для отладки)
SET enable_seqscan = off;
SET enable_hashjoin = off;
```

### pg_stat_statements

```sql
-- Топ медленных запросов
SELECT
    calls,
    ROUND(total_exec_time::numeric, 2) as total_ms,
    ROUND(mean_exec_time::numeric, 2) as avg_ms,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER())::numeric, 2) as percent,
    SUBSTRING(query, 1, 100) as query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## Common Optimizations Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Optimization Checklist                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  □ Проверить EXPLAIN ANALYZE                                        │
│  □ Добавить недостающие индексы                                     │
│  □ Проверить использование индексов (pg_stat_user_indexes)          │
│  □ Избежать SELECT *                                                │
│  □ Избежать функций на индексированных колонках                     │
│  □ Проверить N+1 проблему                                           │
│  □ Использовать cursor pagination вместо OFFSET                     │
│  □ Проверить статистику (ANALYZE)                                   │
│  □ Проверить work_mem для сложных запросов                          │
│  □ Рассмотреть materialized views для отчётов                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **Как оптимизировать медленный запрос?**
   - EXPLAIN ANALYZE
   - Проверить индексы
   - Проверить статистику
   - Переписать запрос

2. **Что такое N+1 problem?**
   - 1 запрос + N дополнительных
   - Решение: JOIN или batch loading

3. **Nested Loop vs Hash Join vs Merge Join?**
   - Nested Loop: маленькие таблицы или хороший индекс
   - Hash Join: большие таблицы без индекса
   - Merge Join: отсортированные данные

4. **Почему индекс не используется?**
   - Функция на колонке
   - Низкая селективность
   - Устаревшая статистика
   - Неправильный порядок колонок в composite index

5. **Как пагинация влияет на производительность?**
   - OFFSET сканирует и отбрасывает
   - Cursor-based пагинация эффективнее

---

[← Назад к списку тем](README.md)
