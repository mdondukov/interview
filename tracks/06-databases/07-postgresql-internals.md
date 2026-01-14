# 07. PostgreSQL Internals

[← Назад к списку тем](README.md)

---

## MVCC и VACUUM

### Как работает MVCC в PostgreSQL

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL MVCC                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Каждая строка имеет:                                               │
│  - xmin: transaction ID создавший строку                            │
│  - xmax: transaction ID удаливший/обновивший                        │
│  - ctid: physical location (page, offset)                           │
│                                                                     │
│  UPDATE = INSERT новой версии + пометка старой (xmax)               │
│  DELETE = пометка xmax                                              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Page 0                                                       │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │ Row v1: xmin=100, xmax=105, data="old"  ← dead tuple   │  │  │
│  │  │ Row v2: xmin=105, xmax=∞,   data="new"  ← live tuple   │  │  │
│  │  │ Row v3: xmin=110, xmax=110  ← aborted, dead            │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Table Bloat

```sql
-- Dead tuples накапливаются
-- Таблица растёт, производительность падает

-- Проверить bloat
SELECT
    schemaname || '.' || relname as table,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    n_live_tup as live_tuples,
    n_dead_tup as dead_tuples,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_percent,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### VACUUM

```sql
-- VACUUM: освобождает место для повторного использования
-- НЕ возвращает место в OS!
VACUUM users;

-- VACUUM FULL: полностью перестраивает таблицу
-- Возвращает место в OS, но БЛОКИРУЕТ таблицу!
VACUUM FULL users;

-- VACUUM ANALYZE: vacuum + обновление статистики
VACUUM ANALYZE users;

-- Настройки autovacuum (postgresql.conf):
-- autovacuum = on
-- autovacuum_vacuum_threshold = 50
-- autovacuum_vacuum_scale_factor = 0.2  -- 20% dead tuples
-- autovacuum_analyze_threshold = 50
-- autovacuum_analyze_scale_factor = 0.1

-- Для конкретной таблицы
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- Чаще vacuum
    autovacuum_vacuum_threshold = 100
);
```

### Мониторинг VACUUM

```sql
-- Текущие процессы autovacuum
SELECT
    pid,
    datname,
    relid::regclass as table,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    ROUND(100.0 * heap_blks_scanned / NULLIF(heap_blks_total, 0), 2) as percent_done
FROM pg_stat_progress_vacuum;

-- Последний vacuum
SELECT
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY last_vacuum DESC NULLS LAST;
```

---

## WAL (Write-Ahead Logging)

### Как работает

```
┌─────────────────────────────────────────────────────────────────────┐
│                         WAL Process                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Transaction writes                                              │
│     ┌───────────────┐                                               │
│     │  Client       │                                               │
│     │  INSERT INTO..│                                               │
│     └───────┬───────┘                                               │
│             │                                                       │
│             ▼                                                       │
│  2. Write to WAL buffer (memory)                                    │
│     ┌───────────────────────────────────────┐                       │
│     │  WAL Buffer                            │                       │
│     │  [record1][record2][record3]...        │                       │
│     └───────────────────────────────────────┘                       │
│             │                                                       │
│             ▼ fsync on COMMIT                                       │
│  3. WAL written to disk                                             │
│     ┌───────────────────────────────────────┐                       │
│     │  pg_wal/                               │                       │
│     │  000000010000000000000001              │                       │
│     │  000000010000000000000002              │                       │
│     └───────────────────────────────────────┘                       │
│             │                                                       │
│             ▼ checkpoint (background)                               │
│  4. Data pages written to data files                                │
│     ┌───────────────────────────────────────┐                       │
│     │  base/                                 │                       │
│     │  Table data files                      │                       │
│     └───────────────────────────────────────┘                       │
│                                                                     │
│  Recovery: replay WAL from last checkpoint                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Настройки WAL

```sql
-- postgresql.conf settings:
-- wal_level = replica
--   (minimal: минимум, нельзя replication)
--   (replica: для streaming replication, default)
--   (logical: для logical replication)
--
-- synchronous_commit = on
--   (on: ждать fsync, durability гарантирована)
--   (off: не ждать, быстрее, но риск потери)
--
-- checkpoint_timeout = 5min
-- checkpoint_completion_target = 0.9
-- max_wal_size = 1GB
```

### Мониторинг WAL

```sql
-- Текущая позиция WAL
SELECT pg_current_wal_lsn();

-- Размер WAL
SELECT pg_size_pretty(pg_wal_lsn_diff(
    pg_current_wal_lsn(),
    '0/0'::pg_lsn
)) as wal_size;

-- WAL activity
SELECT
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) as pending_bytes,
    pg_wal_lsn_diff(sent_lsn, flush_lsn) as write_lag_bytes,
    pg_wal_lsn_diff(flush_lsn, replay_lsn) as replay_lag_bytes
FROM pg_stat_replication;
```

---

## Connection Pooling

### Проблема

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Connection Problem                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PostgreSQL: 1 connection = 1 process (~10MB RAM)                   │
│                                                                     │
│  100 app servers × 10 connections = 1000 connections                │
│  = 10GB RAM только на connections!                                  │
│                                                                     │
│  max_connections = 100 (default)                                    │
│  Увеличение → больше RAM, context switching                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### PgBouncer

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PgBouncer                                     │
│                                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                              │
│  │  App 1  │  │  App 2  │  │  App 3  │  100+ connections each       │
│  └────┬────┘  └────┬────┘  └────┬────┘                              │
│       │            │            │                                   │
│       └────────────┼────────────┘                                   │
│                    │                                                │
│                    ▼                                                │
│            ┌──────────────┐                                         │
│            │   PgBouncer  │  Connection pooler                      │
│            │  1000 client │                                         │
│            │  connections │                                         │
│            └──────┬───────┘                                         │
│                   │                                                 │
│                   ▼                                                 │
│            ┌──────────────┐                                         │
│            │  PostgreSQL  │  20 actual connections                  │
│            └──────────────┘                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Режимы PgBouncer

```ini
# pgbouncer.ini

[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool modes:

# session: connection привязана к клиенту до disconnect
# Безопасно, но меньше sharing
pool_mode = session

# transaction: connection возвращается в pool после транзакции
# Хорошо для большинства случаев
pool_mode = transaction

# statement: connection возвращается после каждого statement
# Максимальный sharing, но нельзя использовать:
# - prepared statements
# - SET commands
# - transactions
pool_mode = statement

# Размеры пулов
default_pool_size = 20
max_client_conn = 1000
```

### Ограничения transaction mode

```sql
-- ❌ Не работает в transaction mode:

-- Prepared statements
PREPARE my_stmt AS SELECT * FROM users WHERE id = $1;
EXECUTE my_stmt(1);

-- SET commands (session-level)
SET search_path TO myschema;

-- LISTEN/NOTIFY
LISTEN my_channel;

-- Cursors с HOLD
DECLARE my_cursor CURSOR WITH HOLD FOR SELECT * FROM big_table;
```

---

## pg_stat_* Views

### Основные представления

```sql
-- Статистика по таблицам
SELECT
    schemaname,
    relname,
    seq_scan,           -- Sequential scans (плохо если много)
    idx_scan,           -- Index scans (хорошо)
    n_tup_ins,          -- Inserts
    n_tup_upd,          -- Updates
    n_tup_del,          -- Deletes
    n_live_tup,         -- Live rows
    n_dead_tup          -- Dead rows (нужен vacuum)
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Статистика по индексам
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,           -- Сколько раз использован
    idx_tup_read,       -- Строк прочитано из индекса
    idx_tup_fetch       -- Строк получено из таблицы
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Неиспользуемые индексы
SELECT
    schemaname || '.' || relname as table,
    indexrelname as index,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    idx_scan as scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### pg_stat_statements

```sql
-- Включить расширение
CREATE EXTENSION pg_stat_statements;

-- postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- Топ запросов по времени
SELECT
    calls,
    ROUND(total_exec_time::numeric, 2) as total_ms,
    ROUND(mean_exec_time::numeric, 2) as avg_ms,
    ROUND((100 * total_exec_time / SUM(total_exec_time) OVER())::numeric, 2) as percent,
    rows,
    SUBSTRING(query, 1, 80) as query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Запросы с большим количеством rows
SELECT
    calls,
    rows,
    ROUND(rows::numeric / calls, 2) as rows_per_call,
    SUBSTRING(query, 1, 80) as query
FROM pg_stat_statements
WHERE calls > 100
ORDER BY rows DESC
LIMIT 20;

-- Сбросить статистику
SELECT pg_stat_statements_reset();
```

### pg_stat_activity

```sql
-- Текущие запросы
SELECT
    pid,
    usename,
    datname,
    state,
    wait_event_type,
    wait_event,
    NOW() - query_start as duration,
    SUBSTRING(query, 1, 80) as query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid != pg_backend_pid()
ORDER BY query_start;

-- Заблокированные запросы
SELECT
    blocked.pid as blocked_pid,
    blocked.query as blocked_query,
    blocking.pid as blocking_pid,
    blocking.query as blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks l ON l.locktype = bl.locktype
    AND l.database = bl.database
    AND l.relation = bl.relation
    AND l.pid != bl.pid
JOIN pg_stat_activity blocking ON l.pid = blocking.pid
WHERE NOT bl.granted;

-- Убить запрос
SELECT pg_cancel_backend(pid);  -- Мягко (SIGINT)
SELECT pg_terminate_backend(pid);  -- Жёстко (SIGTERM)
```

---

## Memory Configuration

```sql
-- Основные настройки памяти

-- shared_buffers: кеш страниц
-- Рекомендация: 25% RAM
shared_buffers = 4GB

-- work_mem: память на операцию (sort, hash)
-- Осторожно: умножается на количество операций!
work_mem = 64MB

-- maintenance_work_mem: для VACUUM, CREATE INDEX
maintenance_work_mem = 512MB

-- effective_cache_size: оценка для планировщика
-- Рекомендация: 50-75% RAM
effective_cache_size = 12GB

-- Мониторинг cache hit ratio
SELECT
    SUM(heap_blks_read) as heap_read,
    SUM(heap_blks_hit) as heap_hit,
    ROUND(100.0 * SUM(heap_blks_hit) /
          NULLIF(SUM(heap_blks_hit) + SUM(heap_blks_read), 0), 2) as ratio
FROM pg_statio_user_tables;
-- Цель: > 99%
```

---

## См. также

- [Query Optimization](./02-query-optimization.md) — оптимизация запросов и EXPLAIN ANALYZE
- [Indexing](./01-indexing.md) — индексы в PostgreSQL

---

## На интервью

### Типичные вопросы

1. **Что такое VACUUM и зачем нужен?**
   - Очистка dead tuples
   - Освобождение места для повторного использования
   - VACUUM FULL для возврата места в OS

2. **Как работает WAL?**
   - Write-ahead logging
   - Durability через fsync на commit
   - Recovery через replay

3. **Зачем нужен connection pooler?**
   - PostgreSQL: 1 connection = 1 process
   - Экономия памяти
   - PgBouncer modes: session, transaction, statement

4. **Как найти медленные запросы?**
   - pg_stat_statements
   - pg_stat_activity
   - EXPLAIN ANALYZE

5. **Ключевые настройки памяти?**
   - shared_buffers: 25% RAM
   - work_mem: осторожно, per operation
   - effective_cache_size: 50-75% RAM

---

[← Назад к списку тем](README.md)
