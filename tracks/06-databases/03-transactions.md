# 03. Transactions

[← Назад к списку тем](README.md)

---

## ACID

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ACID Properties                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Atomicity (Атомарность)                                            │
│  ─────────────────────                                              │
│  Все операции выполняются или ни одна.                              │
│  "All or nothing"                                                   │
│                                                                     │
│  Consistency (Согласованность)                                      │
│  ────────────────────────────                                       │
│  Транзакция переводит БД из одного валидного состояния в другое.    │
│  Все constraints соблюдаются.                                       │
│                                                                     │
│  Isolation (Изолированность)                                        │
│  ──────────────────────────                                         │
│  Параллельные транзакции не влияют друг на друга.                   │
│  Как будто выполняются последовательно.                             │
│                                                                     │
│  Durability (Стойкость)                                             │
│  ─────────────────────                                              │
│  После COMMIT данные сохранены даже при сбое.                       │
│  Записаны в WAL на диск.                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Isolation Levels

### Аномалии чтения

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Read Anomalies                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Dirty Read                                                         │
│  ──────────                                                         │
│  T1: UPDATE accounts SET balance = 100 WHERE id = 1                 │
│  T2: SELECT balance FROM accounts WHERE id = 1  → 100               │
│  T1: ROLLBACK                                                       │
│  T2 прочитал данные, которых нет!                                   │
│                                                                     │
│  Non-Repeatable Read                                                │
│  ────────────────────                                               │
│  T1: SELECT balance ... → 100                                       │
│  T2: UPDATE accounts SET balance = 200 WHERE id = 1; COMMIT         │
│  T1: SELECT balance ... → 200 (другое значение!)                    │
│                                                                     │
│  Phantom Read                                                       │
│  ────────────                                                       │
│  T1: SELECT COUNT(*) FROM orders WHERE status = 'new' → 5           │
│  T2: INSERT INTO orders (status) VALUES ('new'); COMMIT             │
│  T1: SELECT COUNT(*) ... → 6 (появилась новая строка!)              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Уровни изоляции

```
┌──────────────────┬────────────┬──────────────────┬─────────────┐
│ Isolation Level  │ Dirty Read │ Non-Repeatable   │ Phantom     │
├──────────────────┼────────────┼──────────────────┼─────────────┤
│ READ UNCOMMITTED │ Возможно   │ Возможно         │ Возможно    │
│ READ COMMITTED   │ Нет        │ Возможно         │ Возможно    │
│ REPEATABLE READ  │ Нет        │ Нет              │ Возможно*   │
│ SERIALIZABLE     │ Нет        │ Нет              │ Нет         │
└──────────────────┴────────────┴──────────────────┴─────────────┘

* В PostgreSQL REPEATABLE READ также предотвращает phantom reads
```

### Примеры

```sql
-- Установить уровень для транзакции
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- или
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Проверить текущий уровень
SHOW transaction_isolation;

-- Default в PostgreSQL: READ COMMITTED
```

---

## MVCC (Multi-Version Concurrency Control)

### Как работает

```
┌─────────────────────────────────────────────────────────────────────┐
│                           MVCC                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Каждая строка имеет:                                               │
│  - xmin: ID транзакции, создавшей строку                            │
│  - xmax: ID транзакции, удалившей/обновившей строку                 │
│                                                                     │
│  ┌─────────────────────────────────────────────────────┐            │
│  │ Row version 1: xmin=100, xmax=105, data="old"       │            │
│  │ Row version 2: xmin=105, xmax=∞,   data="new"       │            │
│  └─────────────────────────────────────────────────────┘            │
│                                                                     │
│  Транзакция видит строку если:                                      │
│  - xmin committed И xmin < current_txid                             │
│  - xmax не установлен ИЛИ xmax > current_txid ИЛИ xmax aborted      │
│                                                                     │
│  Преимущества:                                                      │
│  - Readers не блокируют writers                                     │
│  - Writers не блокируют readers                                     │
│  - Консистентные снимки (snapshot)                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Snapshot в PostgreSQL

```sql
-- Каждая транзакция получает snapshot:
-- - Список активных транзакций на момент начала
-- - Видит только committed данные до этого момента

BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Snapshot создан здесь

-- Все SELECT в этой транзакции видят данные на момент BEGIN
SELECT * FROM accounts;  -- Snapshot consistent

-- Другая транзакция может менять данные
-- Но эта транзакция их не увидит
COMMIT;
```

---

## Deadlocks

### Как возникает

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Deadlock                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  T1: BEGIN; UPDATE accounts SET ... WHERE id = 1;                   │
│  T2: BEGIN; UPDATE accounts SET ... WHERE id = 2;                   │
│                                                                     │
│  T1: UPDATE accounts SET ... WHERE id = 2;  -- ЖДЁТ T2              │
│  T2: UPDATE accounts SET ... WHERE id = 1;  -- ЖДЁТ T1              │
│                                                                     │
│       T1 ──────── ждёт ────────▶ T2                                 │
│        ◀──────── ждёт ────────                                      │
│                                                                     │
│  PostgreSQL обнаруживает и убивает одну транзакцию:                 │
│  ERROR: deadlock detected                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Предотвращение

```sql
-- 1. Всегда блокировать в одном порядке
-- ❌ Плохо
T1: lock(A), lock(B)
T2: lock(B), lock(A)

-- ✅ Хорошо
T1: lock(A), lock(B)
T2: lock(A), lock(B)

-- 2. Использовать SELECT FOR UPDATE
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2)
ORDER BY id
FOR UPDATE;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- 3. Короткие транзакции
-- Чем быстрее транзакция, тем меньше шанс deadlock
```

### Обработка в коде

```go
func transferMoney(db *sql.DB, from, to int64, amount int64) error {
    for retries := 0; retries < 3; retries++ {
        err := tryTransfer(db, from, to, amount)
        if err == nil {
            return nil
        }

        // Проверяем, это deadlock?
        var pgErr *pq.Error
        if errors.As(err, &pgErr) && pgErr.Code == "40P01" {
            // Deadlock, retry
            time.Sleep(time.Duration(retries*10) * time.Millisecond)
            continue
        }

        return err // Другая ошибка
    }
    return errors.New("max retries exceeded")
}
```

---

## Locks

### Row-level Locks

```sql
-- Exclusive lock (FOR UPDATE)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Другие транзакции не могут UPDATE/DELETE эту строку

-- Share lock (FOR SHARE)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
-- Другие могут SELECT FOR SHARE, но не UPDATE

-- SKIP LOCKED (пропустить заблокированные)
SELECT * FROM tasks WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Отлично для job queues!

-- NOWAIT (не ждать, сразу ошибка)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
```

### Table-level Locks

```sql
-- Явная блокировка таблицы
LOCK TABLE accounts IN EXCLUSIVE MODE;

-- Уровни:
-- ACCESS SHARE: SELECT
-- ROW SHARE: SELECT FOR UPDATE
-- ROW EXCLUSIVE: UPDATE, DELETE, INSERT
-- SHARE: CREATE INDEX CONCURRENTLY
-- EXCLUSIVE: blocks reads
-- ACCESS EXCLUSIVE: ALTER TABLE, DROP TABLE
```

### Advisory Locks

```sql
-- Произвольные блокировки на уровне приложения

-- Session-level lock
SELECT pg_advisory_lock(123);       -- Заблокировать
SELECT pg_advisory_unlock(123);     -- Разблокировать

-- Transaction-level lock
SELECT pg_advisory_xact_lock(123);  -- Освободится при COMMIT/ROLLBACK

-- Try lock (не ждать)
SELECT pg_try_advisory_lock(123);   -- Вернёт true/false

-- Пример: предотвращение дублей
BEGIN;
SELECT pg_advisory_xact_lock(hashtext('user:' || $1));
-- Проверить существует ли
-- Если нет, создать
COMMIT;
```

---

## Optimistic Locking

```sql
-- Через version column
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status TEXT,
    version INTEGER DEFAULT 1
);

-- При обновлении проверяем version
UPDATE orders
SET status = 'shipped', version = version + 1
WHERE id = 123 AND version = 5;

-- Если rows affected = 0, значит кто-то изменил раньше
```

```go
// В коде
func updateOrder(db *sql.DB, order *Order) error {
    result, err := db.Exec(`
        UPDATE orders
        SET status = $1, version = version + 1
        WHERE id = $2 AND version = $3
    `, order.Status, order.ID, order.Version)

    if err != nil {
        return err
    }

    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return ErrConcurrentModification
    }

    order.Version++
    return nil
}
```

---

## Patterns

### Read-Modify-Write

```sql
-- ❌ Плохо: race condition
SELECT balance FROM accounts WHERE id = 1;  -- 100
-- Другая транзакция: balance = 100 + 50 = 150
UPDATE accounts SET balance = 150 WHERE id = 1;  -- Перезаписали!

-- ✅ Хорошо: атомарный UPDATE
UPDATE accounts SET balance = balance + 50 WHERE id = 1;

-- Или SELECT FOR UPDATE
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Расчёты в коде
UPDATE accounts SET balance = 150 WHERE id = 1;
COMMIT;
```

### Upsert (INSERT ... ON CONFLICT)

```sql
-- Атомарный insert или update
INSERT INTO user_stats (user_id, page_views)
VALUES (123, 1)
ON CONFLICT (user_id)
DO UPDATE SET page_views = user_stats.page_views + 1;

-- С returning
INSERT INTO users (email, name)
VALUES ('test@example.com', 'Test')
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name
RETURNING id, email, name;
```

### Savepoints

```sql
BEGIN;
INSERT INTO orders (user_id) VALUES (1);

SAVEPOINT sp1;
INSERT INTO order_items (order_id, product_id) VALUES (1, 100);
-- Ошибка?
ROLLBACK TO SAVEPOINT sp1;
-- Продолжаем без этого item

INSERT INTO order_items (order_id, product_id) VALUES (1, 200);
COMMIT;
```

---

## См. также

- [Distributed Transactions](../07-distributed-systems/07-distributed-transactions.md) — транзакции в распределённых системах
- [Replication](./05-replication.md) — репликация и согласованность данных

---

## На интервью

### Типичные вопросы

1. **ACID — что это?**
   - Atomicity, Consistency, Isolation, Durability
   - Гарантии транзакций

2. **Isolation levels — какой когда?**
   - READ COMMITTED: default, достаточно для большинства
   - REPEATABLE READ: нужен consistent snapshot
   - SERIALIZABLE: полная изоляция, но дорого

3. **Что такое MVCC?**
   - Multi-Version Concurrency Control
   - Читатели не блокируют писателей
   - Каждая транзакция видит snapshot

4. **Как предотвратить deadlock?**
   - Блокировать ресурсы в одном порядке
   - Короткие транзакции
   - Retry при обнаружении

5. **Optimistic vs Pessimistic locking?**
   - Optimistic: version column, проверка при UPDATE
   - Pessimistic: SELECT FOR UPDATE
   - Optimistic лучше при низком contention

---

[← Назад к списку тем](README.md)
