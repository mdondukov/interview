# 01. Indexing

[← Назад к списку тем](README.md)

---

## Зачем нужны индексы

```
Без индекса (Full Table Scan):
┌────────────────────────────────────────────────┐
│  Читаем ВСЕ строки, проверяем условие          │
│  O(n) — медленно для больших таблиц            │
└────────────────────────────────────────────────┘

С индексом (Index Scan):
┌────────────────────────────────────────────────┐
│  Ищем в структуре данных (B-tree)              │
│  O(log n) — быстро даже для миллионов строк    │
└────────────────────────────────────────────────┘
```

---

## B-Tree Index (по умолчанию)

### Структура

```
                    ┌─────────────────┐
                    │  [30] [60] [90] │  Root
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌────────────┐    ┌────────────┐    ┌────────────┐
    │[10][20][30]│    │[40][50][60]│    │[70][80][90]│  Internal
    └─────┬──────┘    └─────┬──────┘    └─────┬──────┘
          │                 │                 │
    ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
    │ Data ptrs │     │ Data ptrs │     │ Data ptrs │  Leaf nodes
    └───────────┘     └───────────┘     └───────────┘
                            ↓
                      Указатели на строки
```

### Операции B-Tree

```sql
-- B-tree хорошо работает для:
=, <, >, <=, >=, BETWEEN, IN, IS NULL

-- Примеры
SELECT * FROM users WHERE id = 100;           -- O(log n)
SELECT * FROM users WHERE age > 30;           -- Index range scan
SELECT * FROM users WHERE name LIKE 'John%';  -- Prefix search работает
SELECT * FROM users WHERE name LIKE '%John';  -- НЕ работает (full scan)

-- Создание
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_age ON users(age);
```

### Когда B-Tree НЕ помогает

```sql
-- Функции на колонке
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
-- Решение: functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- NOT IN, !=
SELECT * FROM users WHERE status != 'active';  -- Обычно full scan

-- OR между разными колонками
SELECT * FROM users WHERE email = 'x' OR phone = 'y';
-- Может использовать Bitmap Index Scan

-- Низкая селективность
SELECT * FROM users WHERE gender = 'M';  -- 50% таблицы
-- Индекс не поможет — full scan быстрее
```

---

## Типы индексов

### Hash Index

```sql
-- Только для равенства (=)
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- Преимущества:
-- - O(1) для точного поиска
-- - Меньше места чем B-tree

-- Недостатки:
-- - Не поддерживает range queries (<, >, BETWEEN)
-- - Не поддерживает ORDER BY
-- - Crash recovery сложнее
```

### GIN (Generalized Inverted Index)

```sql
-- Для массивов, JSONB, полнотекстового поиска

-- Массивы
CREATE INDEX idx_posts_tags ON posts USING gin(tags);
SELECT * FROM posts WHERE tags @> ARRAY['go', 'database'];

-- JSONB
CREATE INDEX idx_data ON events USING gin(data);
SELECT * FROM events WHERE data @> '{"type": "click"}';

-- Full-text search
CREATE INDEX idx_articles_fts ON articles
    USING gin(to_tsvector('english', title || ' ' || body));

SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || body)
    @@ to_tsquery('english', 'database & index');
```

### GiST (Generalized Search Tree)

```sql
-- Для геометрии, ranges, full-text

-- Геолокация
CREATE INDEX idx_locations_point ON locations USING gist(point);
SELECT * FROM locations
WHERE point <@ circle '((0,0), 10)';  -- точки в радиусе

-- Range types
CREATE INDEX idx_events_period ON events USING gist(period);
SELECT * FROM events
WHERE period && '[2024-01-01, 2024-01-31]'::daterange;  -- пересечение
```

### BRIN (Block Range Index)

```sql
-- Для больших таблиц с естественной сортировкой
-- Хранит min/max для блоков страниц

CREATE INDEX idx_logs_created ON logs USING brin(created_at);

-- Идеально для:
-- - Таблиц с append-only данными (логи)
-- - Данных, которые естественно отсортированы (timestamps)
-- - Очень маленький размер индекса
```

---

## Composite Index (составной)

### Порядок колонок важен!

```sql
CREATE INDEX idx_users_country_city ON users(country, city);

-- ✅ Работает:
SELECT * FROM users WHERE country = 'US';
SELECT * FROM users WHERE country = 'US' AND city = 'NYC';

-- ❌ НЕ работает эффективно:
SELECT * FROM users WHERE city = 'NYC';  -- без country
-- Leftmost prefix rule!
```

### Выбор порядка

```sql
-- Правила:
-- 1. Колонки с = условиями первыми
-- 2. Колонки с range (>, <) последними
-- 3. Высокая селективность первой

-- Пример
SELECT * FROM orders
WHERE status = 'pending'      -- equality
  AND user_id = 123           -- equality
  AND created_at > '2024-01-01';  -- range

-- Лучший индекс:
CREATE INDEX idx_orders_status_user_created
    ON orders(status, user_id, created_at);
-- или
CREATE INDEX idx_orders_user_status_created
    ON orders(user_id, status, created_at);
-- Зависит от селективности status vs user_id
```

---

## Covering Index

```sql
-- Индекс содержит все нужные колонки
-- Index-only scan — не нужно читать таблицу

CREATE INDEX idx_users_email_name ON users(email, name);

-- Index-only scan:
SELECT email, name FROM users WHERE email = 'test@example.com';

-- Include syntax (PostgreSQL 11+):
CREATE INDEX idx_orders_status ON orders(status) INCLUDE (total, created_at);

-- Покрывающий индекс без влияния на сортировку/поиск
SELECT status, total, created_at FROM orders WHERE status = 'pending';
```

---

## Partial Index

```sql
-- Индекс только для части данных

-- Только активные пользователи
CREATE INDEX idx_users_active ON users(email)
WHERE status = 'active';

-- Только непрочитанные сообщения
CREATE INDEX idx_messages_unread ON messages(user_id, created_at)
WHERE read_at IS NULL;

-- Преимущества:
-- - Меньше размер индекса
-- - Быстрее обновление
-- - Целевая оптимизация
```

---

## Unique Index

```sql
-- Гарантирует уникальность
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Partial unique (уникальность с условием)
CREATE UNIQUE INDEX idx_users_email_active ON users(email)
WHERE deleted_at IS NULL;  -- Уникальный только среди не удалённых

-- Primary Key автоматически создаёт unique index
```

---

## Expression Index (Functional)

```sql
-- Индекс на выражении
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Теперь работает:
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- Другие примеры:
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM created_at));
CREATE INDEX idx_data_type ON events((data->>'type'));
```

---

## Index Maintenance

### Bloat и REINDEX

```sql
-- Проверка размера индекса
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- REINDEX для устранения bloat
REINDEX INDEX idx_users_email;
REINDEX TABLE users;

-- Concurrent reindex (без блокировки)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

### Мониторинг использования

```sql
-- Неиспользуемые индексы
SELECT
    schemaname || '.' || relname AS table,
    indexrelname AS index,
    idx_scan as scans,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Статистика по индексам
SELECT
    relname as table,
    seq_scan,
    idx_scan,
    ROUND(100.0 * idx_scan / NULLIF(seq_scan + idx_scan, 0), 2) as idx_percent
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

---

## Стратегии индексирования

### Когда создавать индекс

```
✅ Создавать:
- PRIMARY KEY, FOREIGN KEY
- WHERE условия с высокой селективностью
- JOIN колонки
- ORDER BY колонки (если часто)
- UNIQUE constraints

❌ Не создавать:
- Маленькие таблицы (< 1000 строк)
- Колонки с низкой селективностью (boolean, status с 2-3 значениями)
- Часто обновляемые колонки
- Редко используемые в запросах
```

### Trade-offs

```
                    ┌─────────────────────────┐
                    │      Больше индексов    │
                    ├─────────────────────────┤
                    │ ✅ Быстрее SELECT       │
                    │ ❌ Медленнее INSERT     │
                    │ ❌ Медленнее UPDATE     │
                    │ ❌ Больше места на диске │
                    │ ❌ Больше работы VACUUM │
                    └─────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **Как работает B-tree index?**
   - Сбалансированное дерево
   - O(log n) поиск
   - Поддержка range queries

2. **Composite index — порядок колонок?**
   - Leftmost prefix rule
   - Equality колонки первыми
   - Range колонки последними

3. **Covering index — зачем?**
   - Index-only scan
   - Не нужно читать таблицу
   - INCLUDE для дополнительных колонок

4. **Partial index — когда?**
   - Часто запрашиваемое подмножество
   - Меньше размер, быстрее update

5. **Как понять, используется ли индекс?**
   - EXPLAIN ANALYZE
   - pg_stat_user_indexes
   - idx_scan counter

---

[← Назад к списку тем](README.md)
