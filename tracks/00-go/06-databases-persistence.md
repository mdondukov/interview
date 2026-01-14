# 06 — Databases & Persistence

Разбор вопросов о работе с базами данных в Go: database/sql, пулы соединений, транзакции, ORM, миграции и тестирование.

**Навигация:** [← 05-concurrency-patterns](./05-concurrency-patterns.md) | [Трек Go](./README.md) | [07-tooling-testing →](./07-tooling-testing.md)

---

## Вопросы и разборы

### 1. Как работает database/sql?

**Зачем спрашивают.**
`database/sql` — стандартный интерфейс для работы с SQL базами. Интервьюер проверяет понимание архитектуры: connection pool, prepared statements, абстракция над драйверами.

**Короткий ответ.**
`database/sql` предоставляет generic интерфейс для SQL баз. `sql.DB` — это connection pool (не одно соединение). Драйверы регистрируются через `sql.Register`. Prepared statements кешируются и переиспользуются.

**Детальный разбор.**

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────┐
│                      Application                             │
│                          │                                   │
│                    ┌─────┴─────┐                             │
│                    │  sql.DB   │  ← Connection Pool          │
│                    └─────┬─────┘                             │
│                          │                                   │
│              ┌───────────┼───────────┐                       │
│              ▼           ▼           ▼                       │
│         ┌────────┐  ┌────────┐  ┌────────┐                   │
│         │ conn 1 │  │ conn 2 │  │ conn 3 │  ← sql.Conn       │
│         └────────┘  └────────┘  └────────┘                   │
├─────────────────────────────────────────────────────────────┤
│                     Driver Layer                             │
│         ┌─────────────────────────────────────┐             │
│         │  driver.Driver (lib/pq, pgx, etc.)  │             │
│         └─────────────────────────────────────┘             │
├─────────────────────────────────────────────────────────────┤
│                      Database                                │
│         ┌─────────────────────────────────────┐             │
│         │  PostgreSQL / MySQL / SQLite / etc. │             │
│         └─────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

**Connection pool параметры:**
```
┌────────────────────────┬────────────────────────────────────┐
│ Метод                  │ Назначение                         │
├────────────────────────┼────────────────────────────────────┤
│ SetMaxOpenConns(n)     │ Макс. открытых соединений          │
│                        │ Default: unlimited (опасно!)       │
├────────────────────────┼────────────────────────────────────┤
│ SetMaxIdleConns(n)     │ Макс. idle соединений в пуле       │
│                        │ Default: 2                         │
├────────────────────────┼────────────────────────────────────┤
│ SetConnMaxLifetime(d)  │ Макс. время жизни соединения       │
│                        │ Default: 0 (без лимита)            │
├────────────────────────┼────────────────────────────────────┤
│ SetConnMaxIdleTime(d)  │ Макс. время idle до закрытия       │
│                        │ Default: 0 (без лимита)            │
└────────────────────────┴────────────────────────────────────┘
```

**Пример.**
```go
import (
    "database/sql"
    "time"

    _ "github.com/lib/pq"  // драйвер PostgreSQL
)

func NewDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }

    // Настройка connection pool
    db.SetMaxOpenConns(25)              // для большинства приложений 25-50
    db.SetMaxIdleConns(5)               // 10-20% от MaxOpen
    db.SetConnMaxLifetime(5 * time.Minute)  // для load balancer / failover
    db.SetConnMaxIdleTime(1 * time.Minute)

    // Проверка соединения
    if err := db.Ping(); err != nil {
        return nil, err
    }

    return db, nil
}

// Использование
func GetUser(ctx context.Context, db *sql.DB, id int64) (*User, error) {
    var user User

    err := db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&user.ID, &user.Name, &user.Email)

    if err == sql.ErrNoRows {
        return nil, nil  // не найден
    }
    if err != nil {
        return nil, err
    }

    return &user, nil
}

// Query для множества строк
func ListUsers(ctx context.Context, db *sql.DB) ([]User, error) {
    rows, err := db.QueryContext(ctx, "SELECT id, name, email FROM users")
    if err != nil {
        return nil, err
    }
    defer rows.Close()  // ВАЖНО: всегда закрывать!

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, err
        }
        users = append(users, u)
    }

    // Проверка ошибок после итерации
    if err := rows.Err(); err != nil {
        return nil, err
    }

    return users, nil
}
```

**Prepared statements:**
```go
// Explicit prepared statement
stmt, err := db.PrepareContext(ctx, "SELECT * FROM users WHERE id = $1")
if err != nil {
    return err
}
defer stmt.Close()

// Многократное использование
for _, id := range ids {
    rows, err := stmt.QueryContext(ctx, id)
    // ...
}

// Implicit prepared statements (database/sql делает автоматически)
// Каждый вызов Query/Exec с плейсхолдерами:
// 1. Prepare на одном соединении
// 2. Execute
// 3. Close statement
// Менее эффективно при частых однотипных запросах
```

**Типичные ошибки.**
1. **Не закрыть rows** — утечка соединений, pool исчерпывается
2. **Не ограничить MaxOpenConns** — exhaustion соединений БД
3. **Нет context** — запросы не отменяются, зависают
4. **Игнорировать rows.Err()** — ошибки при итерации теряются

**На интервью.**
Объясни, что `sql.DB` — это pool, а не соединение. Покажи правильную настройку pool. Расскажи, зачем defer rows.Close() и rows.Err().

*Частые follow-up вопросы:*
- Как выбрать MaxOpenConns?
- Что происходит, когда pool исчерпан?
- Зачем ConnMaxLifetime?

---

### 2. Что такое sql.Null* типы и как с ними работать?

**Зачем спрашивают.**
NULL в SQL — частый источник багов. Интервьюер проверяет понимание работы с nullable колонками и альтернативных подходов.

**Короткий ответ.**
`sql.NullString`, `sql.NullInt64` и др. — обёртки для nullable значений. Содержат поле `Valid bool` для проверки NULL. Альтернативы: указатели (`*string`), `COALESCE` в SQL.

**Детальный разбор.**

**Стандартные Null типы:**
```
┌─────────────────┬──────────────────────────────────────────┐
│ Тип             │ Структура                                │
├─────────────────┼──────────────────────────────────────────┤
│ sql.NullString  │ { String string; Valid bool }            │
│ sql.NullInt64   │ { Int64 int64; Valid bool }              │
│ sql.NullInt32   │ { Int32 int32; Valid bool }              │
│ sql.NullFloat64 │ { Float64 float64; Valid bool }          │
│ sql.NullBool    │ { Bool bool; Valid bool }                │
│ sql.NullTime    │ { Time time.Time; Valid bool }           │
│ sql.NullByte    │ { Byte byte; Valid bool } (Go 1.22+)     │
└─────────────────┴──────────────────────────────────────────┘
```

**Пример.**
```go
type User struct {
    ID        int64
    Name      string
    Email     sql.NullString  // может быть NULL
    Age       sql.NullInt64
    CreatedAt time.Time
}

func GetUser(ctx context.Context, db *sql.DB, id int64) (*User, error) {
    var user User

    err := db.QueryRowContext(ctx, `
        SELECT id, name, email, age, created_at
        FROM users WHERE id = $1
    `, id).Scan(
        &user.ID,
        &user.Name,
        &user.Email,  // sql.NullString
        &user.Age,    // sql.NullInt64
        &user.CreatedAt,
    )

    return &user, err
}

// Использование
func PrintUser(u *User) {
    fmt.Println("Name:", u.Name)

    if u.Email.Valid {
        fmt.Println("Email:", u.Email.String)
    } else {
        fmt.Println("Email: not set")
    }

    if u.Age.Valid {
        fmt.Printf("Age: %d\n", u.Age.Int64)
    }
}

// Вставка с NULL
func InsertUser(ctx context.Context, db *sql.DB, name string, email *string) error {
    var nullEmail sql.NullString
    if email != nil {
        nullEmail = sql.NullString{String: *email, Valid: true}
    }

    _, err := db.ExecContext(ctx,
        "INSERT INTO users (name, email) VALUES ($1, $2)",
        name, nullEmail,
    )
    return err
}
```

**Альтернатива: указатели:**
```go
type User struct {
    ID    int64
    Name  string
    Email *string  // nil = NULL
    Age   *int64
}

func GetUser(ctx context.Context, db *sql.DB, id int64) (*User, error) {
    var user User

    err := db.QueryRowContext(ctx, `
        SELECT id, name, email, age FROM users WHERE id = $1
    `, id).Scan(&user.ID, &user.Name, &user.Email, &user.Age)

    return &user, err
}

// Проще в использовании
if user.Email != nil {
    fmt.Println(*user.Email)
}
```

**Альтернатива: COALESCE в SQL:**
```go
// Если NULL не нужен в приложении
err := db.QueryRowContext(ctx, `
    SELECT id, name, COALESCE(email, '') as email
    FROM users WHERE id = $1
`, id).Scan(&user.ID, &user.Name, &user.Email)

// user.Email будет пустой строкой вместо NULL
```

**Generic Null тип (Go 1.22+):**
```go
// sql.Null[T] — generic версия
type User struct {
    ID    int64
    Email sql.Null[string]
    Age   sql.Null[int]
}

if user.Email.Valid {
    fmt.Println(user.Email.V)  // V — значение
}
```

**Типичные ошибки.**
1. **Scan в обычный тип** — ошибка при NULL значении
2. **Не проверять Valid** — использование zero value вместо NULL
3. **NullString в JSON** — сериализуется как `{String: "", Valid: false}`
4. **Забыть про COALESCE** — проще обработать в SQL

**На интервью.**
Покажи разницу между `sql.NullString` и `*string`. Объясни, когда использовать COALESCE. Расскажи про проблемы с JSON сериализацией.

*Частые follow-up вопросы:*
- Как сериализовать NullString в JSON правильно?
- Что выбрать: указатель или sql.Null*?
- Как создать свой nullable тип?

---

### 3. Как правильно обрабатывать транзакции в Go?

**Зачем спрашивают.**
Транзакции — критичная часть работы с БД. Интервьюер проверяет понимание ACID, правильного управления Tx и обработки ошибок.

**Короткий ответ.**
`db.BeginTx()` начинает транзакцию, возвращает `*sql.Tx`. Используй defer для Rollback, Commit явно при успехе. Context отменяет транзакцию при таймауте.

**Детальный разбор.**

**Жизненный цикл транзакции:**
```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   BeginTx ───────► Tx ───────► Commit                       │
│      │              │             │                          │
│      │              │             └──► Success               │
│      │              │                                        │
│      │              └──► Rollback                            │
│      │                      │                                │
│      │                      └──► Error / Cancel              │
│      │                                                        │
│      └──► Context Cancel ──► Auto Rollback                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Базовый паттерн:**
```go
func TransferMoney(ctx context.Context, db *sql.DB, from, to int64, amount float64) error {
    // Начинаем транзакцию
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    // defer Rollback безопасен даже после Commit
    defer tx.Rollback()

    // Списание
    result, err := tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
        amount, from,
    )
    if err != nil {
        return err
    }

    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        return errors.New("insufficient funds or account not found")
    }

    // Зачисление
    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, to,
    )
    if err != nil {
        return err
    }

    // Фиксация
    return tx.Commit()
}
```

**Хелпер для транзакций:**
```go
// WithTx — универсальный хелпер
func WithTx(ctx context.Context, db *sql.DB, fn func(tx *sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    if err := fn(tx); err != nil {
        return err
    }

    return tx.Commit()
}

// Использование
err := WithTx(ctx, db, func(tx *sql.Tx) error {
    if _, err := tx.ExecContext(ctx, "UPDATE ..."); err != nil {
        return err
    }
    if _, err := tx.ExecContext(ctx, "INSERT ..."); err != nil {
        return err
    }
    return nil
})
```

**Уровни изоляции:**
```go
// Опции транзакции
opts := &sql.TxOptions{
    Isolation: sql.LevelSerializable,  // или LevelReadCommitted и др.
    ReadOnly:  true,                    // read-only транзакция
}

tx, err := db.BeginTx(ctx, opts)
```

```
┌─────────────────────┬──────────────────────────────────────┐
│ Уровень             │ Что защищает                         │
├─────────────────────┼──────────────────────────────────────┤
│ LevelDefault        │ Зависит от БД (обычно ReadCommitted) │
│ LevelReadUncommitted│ Ничего (dirty reads возможны)        │
│ LevelReadCommitted  │ От dirty reads                       │
│ LevelRepeatableRead │ + non-repeatable reads               │
│ LevelSerializable   │ + phantom reads (полная изоляция)    │
└─────────────────────┴──────────────────────────────────────┘
```

**Savepoints (PostgreSQL):**
```go
func ComplexOperation(ctx context.Context, tx *sql.Tx) error {
    // Основная операция
    _, err := tx.ExecContext(ctx, "INSERT INTO orders ...")
    if err != nil {
        return err
    }

    // Savepoint для опциональной операции
    _, err = tx.ExecContext(ctx, "SAVEPOINT sp1")
    if err != nil {
        return err
    }

    _, err = tx.ExecContext(ctx, "INSERT INTO notifications ...")
    if err != nil {
        // Откатываем только до savepoint
        tx.ExecContext(ctx, "ROLLBACK TO SAVEPOINT sp1")
        // Основная операция всё ещё валидна
    }

    return nil
}
```

**Типичные ошибки.**
1. **Забыть defer Rollback** — соединение зависает
2. **Rollback после Commit** — ошибка (но defer безопасен)
3. **Использовать db вместо tx** — операции вне транзакции
4. **Нет таймаута** — транзакция может висеть вечно

**На интервью.**
Покажи паттерн с defer Rollback. Объясни, почему это безопасно после Commit. Расскажи про уровни изоляции и когда нужен Serializable.

*Частые follow-up вопросы:*
- Как обрабатывать deadlock?
- Что такое optimistic locking?
- Как тестировать код с транзакциями?

---

### 4. Что такое sqlx и чем он лучше стандартного database/sql?

**Зачем спрашивают.**
sqlx — популярное расширение database/sql. Интервьюер проверяет знание экосистемы и понимание trade-offs между библиотеками.

**Короткий ответ.**
sqlx добавляет struct scanning (`Get`, `Select`), named parameters (`:name`), и удобные методы поверх database/sql. Не ORM, а качественное расширение стандартной библиотеки.

**Детальный разбор.**

**Сравнение database/sql и sqlx:**
```
┌──────────────────────┬────────────────────────────────────────┐
│ database/sql         │ sqlx                                   │
├──────────────────────┼────────────────────────────────────────┤
│ rows.Scan(&a, &b, &c)│ db.Get(&user, query, args)            │
│ Ручной маппинг       │ Автоматический маппинг в struct       │
├──────────────────────┼────────────────────────────────────────┤
│ Позиционные $1, $2   │ Named :name, :email                   │
├──────────────────────┼────────────────────────────────────────┤
│ Нет слайсов          │ db.Select(&users, query)              │
├──────────────────────┼────────────────────────────────────────┤
│ Нет IN (...)         │ sqlx.In(query, slice)                 │
└──────────────────────┴────────────────────────────────────────┘
```

**Пример.**

**Struct scanning:**
```go
import "github.com/jmoiron/sqlx"

type User struct {
    ID        int64     `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    CreatedAt time.Time `db:"created_at"`
}

func GetUser(ctx context.Context, db *sqlx.DB, id int64) (*User, error) {
    var user User

    // Get — для одной строки
    err := db.GetContext(ctx, &user,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", id)

    if err == sql.ErrNoRows {
        return nil, nil
    }
    return &user, err
}

func ListUsers(ctx context.Context, db *sqlx.DB) ([]User, error) {
    var users []User

    // Select — для множества строк
    err := db.SelectContext(ctx, &users,
        "SELECT id, name, email, created_at FROM users ORDER BY id")

    return users, err
}
```

**Named parameters:**
```go
// Именованные параметры из struct
user := User{Name: "Alice", Email: "alice@example.com"}

result, err := db.NamedExecContext(ctx,
    "INSERT INTO users (name, email) VALUES (:name, :email)",
    user,
)

// Или из map
params := map[string]interface{}{
    "name":  "Bob",
    "email": "bob@example.com",
}
result, err := db.NamedExecContext(ctx,
    "INSERT INTO users (name, email) VALUES (:name, :email)",
    params,
)
```

**IN clause с sqlx.In:**
```go
// Проблема: SELECT * FROM users WHERE id IN (?, ?, ?)
// Количество плейсхолдеров заранее неизвестно

ids := []int64{1, 2, 3, 4, 5}

// sqlx.In раскрывает слайс
query, args, err := sqlx.In(
    "SELECT * FROM users WHERE id IN (?)",
    ids,
)
if err != nil {
    return err
}

// Rebind для конкретной БД (PostgreSQL: $1, $2...)
query = db.Rebind(query)

var users []User
err = db.SelectContext(ctx, &users, query, args...)
```

**Транзакции:**
```go
func TransferWithSqlx(ctx context.Context, db *sqlx.DB) error {
    tx, err := db.BeginTxx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    var balance float64
    err = tx.GetContext(ctx, &balance,
        "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE", 1)
    if err != nil {
        return err
    }

    // ... остальные операции

    return tx.Commit()
}
```

**Embedded structs:**
```go
type Address struct {
    City    string `db:"city"`
    Country string `db:"country"`
}

type UserWithAddress struct {
    ID   int64  `db:"id"`
    Name string `db:"name"`
    Address      // embedded
}

// SELECT id, name, city, country FROM users_with_address
var user UserWithAddress
err := db.GetContext(ctx, &user, query)
// user.City и user.Country заполнены
```

**Типичные ошибки.**
1. **Неправильные db tags** — поля не маппятся
2. **SELECT * с embedded** — конфликты имён колонок
3. **Забыть Rebind** — неправильные плейсхолдеры
4. **Использовать sqlx как ORM** — это не ORM

**На интервью.**
Покажи разницу между Scan и Get/Select. Объясни, как работает sqlx.In. Расскажи, почему sqlx не ORM и когда это плюс.

*Частые follow-up вопросы:*
- Как обрабатывать NULL в sqlx?
- Можно ли использовать sqlx с pgx?
- Когда лучше database/sql без sqlx?

---

### 5. Как работать с миграциями в Go?

**Зачем спрашивают.**
Миграции — часть CI/CD пайплайна. Интервьюер проверяет знание инструментов и практик версионирования схемы БД.

**Короткий ответ.**
Миграции — версионированные изменения схемы БД. Инструменты: golang-migrate, goose, atlas. Каждая миграция — up/down SQL файлы. Применяются при деплое, откатываются при проблемах.

**Детальный разбор.**

**Концепция миграций:**
```
┌─────────────────────────────────────────────────────────────┐
│                        Версии схемы                          │
│                                                              │
│   v1 ────► v2 ────► v3 ────► v4 (текущая)                   │
│        up1     up2     up3                                   │
│   ◄────    ◄────    ◄────                                   │
│        down1   down2   down3                                 │
│                                                              │
│   schema_migrations table:                                   │
│   ┌─────────────────────────────────────────────────┐       │
│   │ version │ dirty │ applied_at                     │       │
│   │ 4       │ false │ 2024-01-15 10:30:00           │       │
│   └─────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**Структура файлов:**
```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_add_email_index.up.sql
├── 000002_add_email_index.down.sql
├── 000003_create_orders.up.sql
└── 000003_create_orders.down.sql
```

**Пример.**

**golang-migrate:**
```bash
# Установка
$ go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Создание миграции
$ migrate create -ext sql -dir migrations -seq create_users

# Применить все миграции
$ migrate -path migrations -database "postgres://localhost/mydb?sslmode=disable" up

# Откатить последнюю
$ migrate -path migrations -database "..." down 1

# Проверить версию
$ migrate -path migrations -database "..." version
```

**Миграция up:**
```sql
-- 000001_create_users.up.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**Миграция down:**
```sql
-- 000001_create_users.down.sql
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;
```

**Программное применение:**
```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(dbURL string) error {
    m, err := migrate.New(
        "file://migrations",
        dbURL,
    )
    if err != nil {
        return err
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }

    return nil
}

// Embed миграции в бинарник
//go:embed migrations/*.sql
var migrationsFS embed.FS

func RunEmbeddedMigrations(dbURL string) error {
    d, err := iofs.New(migrationsFS, "migrations")
    if err != nil {
        return err
    }

    m, err := migrate.NewWithSourceInstance("iofs", d, dbURL)
    if err != nil {
        return err
    }
    defer m.Close()

    return m.Up()
}
```

**goose (альтернатива):**
```go
import "github.com/pressly/goose/v3"

func RunMigrations(db *sql.DB) error {
    goose.SetBaseFS(migrationsFS)

    if err := goose.SetDialect("postgres"); err != nil {
        return err
    }

    return goose.Up(db, "migrations")
}
```

**Best practices:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. Миграции идемпотентны                                    │
│    CREATE TABLE IF NOT EXISTS                               │
│    DROP TABLE IF EXISTS                                     │
├─────────────────────────────────────────────────────────────┤
│ 2. Отдельная транзакция на миграцию                         │
│    При ошибке откатывается только текущая                   │
├─────────────────────────────────────────────────────────────┤
│ 3. Без DDL в транзакциях (PostgreSQL)                       │
│    CREATE INDEX CONCURRENTLY — вне транзакции               │
├─────────────────────────────────────────────────────────────┤
│ 4. Zero-downtime миграции                                   │
│    Сначала добавить колонку, потом удалить старую           │
│    Expand → Migrate data → Contract                         │
└─────────────────────────────────────────────────────────────┘
```

**Типичные ошибки.**
1. **Редактировать применённые миграции** — расхождение схемы
2. **Нет down миграции** — невозможно откатить
3. **Большие транзакции** — блокировка таблиц надолго
4. **Миграции в runtime** — лучше при старте или отдельно

**На интервью.**
Покажи структуру миграционных файлов. Объясни, как работает таблица версий. Расскажи про zero-downtime миграции.

*Частые follow-up вопросы:*
- Как откатить "плохую" миграцию в production?
- Что делать с dirty state?
- Как мигрировать данные между схемами?

---

### 6. ORM vs Query Builder vs Raw SQL — когда что использовать?

**Зачем спрашивают.**
Выбор подхода к работе с БД влияет на архитектуру. Интервьюер проверяет понимание trade-offs и способность выбрать правильный инструмент.

**Короткий ответ.**
Raw SQL — максимальный контроль, минимум магии. Query Builder (squirrel, goqu) — типобезопасное построение запросов. ORM (GORM, ent) — объектный маппинг, меньше кода, больше магии.

**Детальный разбор.**

**Сравнение подходов:**
```
┌─────────────────┬──────────────┬───────────────┬──────────────┐
│ Критерий        │ Raw SQL      │ Query Builder │ ORM          │
├─────────────────┼──────────────┼───────────────┼──────────────┤
│ Контроль        │ Полный       │ Высокий       │ Средний      │
│ Магия           │ Нет          │ Минимум       │ Много        │
│ Производительн. │ Максимальная │ Высокая       │ Зависит      │
│ SQL injection   │ Осторожно    │ Защищён       │ Защищён      │
│ Рефакторинг     │ Ручной       │ IDE помогает  │ Автоматич.   │
│ Сложные запросы │ Легко        │ Возможно      │ Сложно       │
│ Кривая обучения │ SQL          │ Низкая        │ Высокая      │
└─────────────────┴──────────────┴───────────────┴──────────────┘
```

**Пример.**

**Raw SQL (database/sql, sqlx):**
```go
// Полный контроль, но строки
func GetUserOrders(ctx context.Context, db *sqlx.DB, userID int64) ([]Order, error) {
    var orders []Order
    err := db.SelectContext(ctx, &orders, `
        SELECT o.id, o.total, o.status, o.created_at
        FROM orders o
        JOIN users u ON u.id = o.user_id
        WHERE u.id = $1 AND o.status != 'cancelled'
        ORDER BY o.created_at DESC
        LIMIT 100
    `, userID)
    return orders, err
}
```

**Query Builder (squirrel):**
```go
import sq "github.com/Masterminds/squirrel"

func GetUserOrders(ctx context.Context, db *sqlx.DB, userID int64, status string) ([]Order, error) {
    psql := sq.StatementBuilder.PlaceholderFormat(sq.Dollar)

    query := psql.Select("o.id", "o.total", "o.status", "o.created_at").
        From("orders o").
        Join("users u ON u.id = o.user_id").
        Where(sq.Eq{"u.id": userID}).
        Where(sq.NotEq{"o.status": "cancelled"}).
        OrderBy("o.created_at DESC").
        Limit(100)

    // Динамические условия
    if status != "" {
        query = query.Where(sq.Eq{"o.status": status})
    }

    sql, args, err := query.ToSql()
    if err != nil {
        return nil, err
    }

    var orders []Order
    err = db.SelectContext(ctx, &orders, sql, args...)
    return orders, err
}
```

**ORM (GORM):**
```go
import "gorm.io/gorm"

type User struct {
    gorm.Model
    Name   string
    Email  string
    Orders []Order
}

type Order struct {
    gorm.Model
    UserID    uint
    Total     float64
    Status    string
    CreatedAt time.Time
}

func GetUserOrders(db *gorm.DB, userID uint) ([]Order, error) {
    var orders []Order

    err := db.Where("user_id = ?", userID).
        Where("status != ?", "cancelled").
        Order("created_at DESC").
        Limit(100).
        Find(&orders).Error

    return orders, err
}

// N+1 проблема решается Preload
func GetUsersWithOrders(db *gorm.DB) ([]User, error) {
    var users []User
    err := db.Preload("Orders").Find(&users).Error
    return users, err
}
```

**ORM (ent — type-safe):**
```go
// Схема генерирует Go код
// ent/schema/user.go
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.String("email").Unique(),
    }
}

func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("orders", Order.Type),
    }
}

// Использование — полностью типобезопасно
users, err := client.User.
    Query().
    Where(user.NameContains("Alice")).
    WithOrders(func(q *ent.OrderQuery) {
        q.Where(order.StatusNEQ("cancelled"))
    }).
    All(ctx)
```

**Когда что использовать:**
```
┌─────────────────────────────────────────────────────────────┐
│ Raw SQL / sqlx:                                              │
│ • Сложные аналитические запросы                             │
│ • Максимальная производительность критична                  │
│ • Команда хорошо знает SQL                                  │
│ • Нужен полный контроль                                     │
├─────────────────────────────────────────────────────────────┤
│ Query Builder (squirrel, goqu):                              │
│ • Динамические WHERE условия                                │
│ • Защита от SQL injection важна                             │
│ • Не нужен объектный маппинг                                │
├─────────────────────────────────────────────────────────────┤
│ ORM (GORM, ent):                                             │
│ • CRUD-тяжёлое приложение                                   │
│ • Нужны relations и preloading                              │
│ • Быстрая разработка важнее производительности              │
│ • ent — когда нужна типобезопасность                        │
└─────────────────────────────────────────────────────────────┘
```

**Типичные ошибки.**
1. **ORM для аналитики** — неэффективные запросы
2. **Raw SQL с конкатенацией** — SQL injection
3. **N+1 без Preload** — тысячи запросов
4. **Игнорировать EXPLAIN** — неоптимальные планы

**На интервью.**
Приведи примеры, когда ORM не подходит. Покажи N+1 проблему и решение. Объясни, почему squirrel — не ORM.

*Частые follow-up вопросы:*
- Как отлаживать запросы GORM?
- Что такое ent и чем он лучше GORM?
- Как оптимизировать ORM для высоких нагрузок?

---

### 7. Как реализовать паттерн Repository в Go?

**Зачем спрашивают.**
Repository — стандартный паттерн для абстракции доступа к данным. Интервьюер проверяет понимание clean architecture и тестируемости.

**Короткий ответ.**
Repository — интерфейс, скрывающий детали хранения. Позволяет заменить реализацию (PostgreSQL → MongoDB) и упрощает тестирование через моки.

**Детальный разбор.**

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────┐
│                       Handlers/Controllers                   │
│                              │                               │
│                              ▼                               │
│                    ┌─────────────────┐                       │
│                    │     Service     │  Business logic       │
│                    └────────┬────────┘                       │
│                             │                                │
│                             ▼                                │
│                    ┌─────────────────┐                       │
│                    │   Repository    │  Interface            │
│                    │   (interface)   │                       │
│                    └────────┬────────┘                       │
│                             │                                │
│          ┌──────────────────┼──────────────────┐             │
│          ▼                  ▼                  ▼             │
│    ┌───────────┐     ┌───────────┐      ┌───────────┐       │
│    │ PostgresRepo│    │ MongoRepo │      │ MockRepo  │       │
│    └───────────┘     └───────────┘      └───────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Определение интерфейса:**
```go
// domain/user.go
type User struct {
    ID        int64
    Name      string
    Email     string
    CreatedAt time.Time
}

// repository/user.go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id int64) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id int64) error
    List(ctx context.Context, offset, limit int) ([]User, error)
}
```

**Реализация для PostgreSQL:**
```go
// repository/postgres/user.go
type userRepository struct {
    db *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(ctx context.Context, user *User) error {
    query := `
        INSERT INTO users (name, email, created_at)
        VALUES ($1, $2, $3)
        RETURNING id
    `
    return r.db.QueryRowContext(ctx, query,
        user.Name, user.Email, time.Now(),
    ).Scan(&user.ID)
}

func (r *userRepository) GetByID(ctx context.Context, id int64) (*User, error) {
    var user User
    err := r.db.GetContext(ctx, &user,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", id)

    if err == sql.ErrNoRows {
        return nil, ErrNotFound
    }
    return &user, err
}

func (r *userRepository) Update(ctx context.Context, user *User) error {
    result, err := r.db.ExecContext(ctx, `
        UPDATE users SET name = $1, email = $2 WHERE id = $3
    `, user.Name, user.Email, user.ID)
    if err != nil {
        return err
    }

    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        return ErrNotFound
    }
    return nil
}

func (r *userRepository) Delete(ctx context.Context, id int64) error {
    result, err := r.db.ExecContext(ctx,
        "DELETE FROM users WHERE id = $1", id)
    if err != nil {
        return err
    }

    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        return ErrNotFound
    }
    return nil
}

func (r *userRepository) List(ctx context.Context, offset, limit int) ([]User, error) {
    var users []User
    err := r.db.SelectContext(ctx, &users, `
        SELECT id, name, email, created_at FROM users
        ORDER BY id LIMIT $1 OFFSET $2
    `, limit, offset)
    return users, err
}
```

**Service использует интерфейс:**
```go
// service/user.go
type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) Register(ctx context.Context, name, email string) (*User, error) {
    // Проверка уникальности email
    existing, err := s.repo.GetByEmail(ctx, email)
    if err != nil && err != ErrNotFound {
        return nil, err
    }
    if existing != nil {
        return nil, ErrEmailExists
    }

    user := &User{Name: name, Email: email}
    if err := s.repo.Create(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}
```

**Mock для тестов:**
```go
// repository/mock/user.go
type MockUserRepository struct {
    users map[int64]*User
    mu    sync.RWMutex
    nextID int64
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        users: make(map[int64]*User),
    }
}

func (m *MockUserRepository) Create(ctx context.Context, user *User) error {
    m.mu.Lock()
    defer m.mu.Unlock()

    m.nextID++
    user.ID = m.nextID
    user.CreatedAt = time.Now()
    m.users[user.ID] = user
    return nil
}

func (m *MockUserRepository) GetByID(ctx context.Context, id int64) (*User, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()

    user, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return user, nil
}

// ... остальные методы
```

**Тест сервиса:**
```go
func TestUserService_Register(t *testing.T) {
    repo := NewMockUserRepository()
    service := NewUserService(repo)

    user, err := service.Register(context.Background(), "Alice", "alice@example.com")
    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
    assert.NotZero(t, user.ID)

    // Повторная регистрация — ошибка
    _, err = service.Register(context.Background(), "Alice2", "alice@example.com")
    assert.ErrorIs(t, err, ErrEmailExists)
}
```

**Типичные ошибки.**
1. **Repository с бизнес-логикой** — это Data Access, не бизнес
2. **Слишком много методов** — interface bloat
3. **Возвращать sql.ErrNoRows** — абстракция протекает
4. **Нет транзакций** — нужен Unit of Work или TransactionManager

**На интервью.**
Покажи интерфейс и реализацию. Объясни, как тестировать через mock. Расскажи про Unit of Work для транзакций.

*Частые follow-up вопросы:*
- Как обрабатывать транзакции через несколько репозиториев?
- Когда repository — оверинжиниринг?
- Как генерировать моки автоматически?

---

### 8. Как тестировать код с базой данных?

**Зачем спрашивают.**
Тестирование с БД — сложная задача. Интервьюер проверяет знание подходов: моки, testcontainers, fixtures.

**Короткий ответ.**
Unit тесты — моки/sqlmock. Integration тесты — testcontainers (реальная БД в Docker). E2E — staging БД. Каждый подход для своего уровня.

**Детальный разбор.**

**Пирамида тестов для БД:**
```
┌─────────────────────────────────────────────────────────────┐
│                          E2E                                 │
│                  (staging database)                          │
│              ┌───────────────────────┐                       │
│              │      Slow, Real       │                       │
├──────────────┴───────────────────────┴──────────────────────┤
│                     Integration                              │
│              (testcontainers, test DB)                       │
│         ┌─────────────────────────────────┐                  │
│         │    Real DB, Isolated, Docker    │                  │
├─────────┴─────────────────────────────────┴─────────────────┤
│                        Unit                                  │
│                (mocks, sqlmock)                              │
│    ┌─────────────────────────────────────────────┐          │
│    │     Fast, No DB, Mock Repository            │          │
└────┴─────────────────────────────────────────────┴──────────┘
```

**Пример.**

**sqlmock — mock для database/sql:**
```go
import "github.com/DATA-DOG/go-sqlmock"

func TestGetUser_Success(t *testing.T) {
    // Создаём mock
    db, mock, err := sqlmock.New()
    require.NoError(t, err)
    defer db.Close()

    // Ожидаемый запрос и результат
    rows := sqlmock.NewRows([]string{"id", "name", "email"}).
        AddRow(1, "Alice", "alice@example.com")

    mock.ExpectQuery("SELECT id, name, email FROM users WHERE id = ?").
        WithArgs(1).
        WillReturnRows(rows)

    // Тестируем
    repo := NewUserRepository(db)
    user, err := repo.GetByID(context.Background(), 1)

    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
    assert.NoError(t, mock.ExpectationsWereMet())
}

func TestGetUser_NotFound(t *testing.T) {
    db, mock, err := sqlmock.New()
    require.NoError(t, err)
    defer db.Close()

    mock.ExpectQuery("SELECT id, name, email FROM users WHERE id = ?").
        WithArgs(999).
        WillReturnError(sql.ErrNoRows)

    repo := NewUserRepository(db)
    user, err := repo.GetByID(context.Background(), 999)

    assert.Nil(t, user)
    assert.ErrorIs(t, err, ErrNotFound)
}
```

**testcontainers — реальная БД в Docker:**
```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func TestIntegration_UserRepository(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()

    // Запускаем PostgreSQL в контейнере
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)

    // Получаем connection string
    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    // Подключаемся
    db, err := sqlx.Connect("postgres", connStr)
    require.NoError(t, err)
    defer db.Close()

    // Применяем миграции
    require.NoError(t, runMigrations(db))

    // Тестируем
    repo := NewUserRepository(db)

    t.Run("Create and Get", func(t *testing.T) {
        user := &User{Name: "Alice", Email: "alice@test.com"}
        err := repo.Create(ctx, user)
        require.NoError(t, err)
        assert.NotZero(t, user.ID)

        got, err := repo.GetByID(ctx, user.ID)
        require.NoError(t, err)
        assert.Equal(t, "Alice", got.Name)
    })

    t.Run("Update", func(t *testing.T) {
        // ...
    })
}
```

**Test fixtures:**
```go
// testdata/fixtures/users.sql
INSERT INTO users (id, name, email) VALUES
(1, 'Alice', 'alice@test.com'),
(2, 'Bob', 'bob@test.com'),
(3, 'Charlie', 'charlie@test.com');

// В тесте
func setupFixtures(t *testing.T, db *sql.DB) {
    t.Helper()

    data, err := os.ReadFile("testdata/fixtures/users.sql")
    require.NoError(t, err)

    _, err = db.Exec(string(data))
    require.NoError(t, err)
}

func cleanupTable(t *testing.T, db *sql.DB, table string) {
    t.Helper()
    _, err := db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", table))
    require.NoError(t, err)
}
```

**Параллельные тесты с изоляцией:**
```go
func TestParallel_Repository(t *testing.T) {
    db := setupTestDB(t)

    t.Run("group", func(t *testing.T) {
        t.Run("test1", func(t *testing.T) {
            t.Parallel()
            // Использовать уникальные данные для изоляции
            email := fmt.Sprintf("user-%s@test.com", t.Name())
            // ...
        })

        t.Run("test2", func(t *testing.T) {
            t.Parallel()
            email := fmt.Sprintf("user-%s@test.com", t.Name())
            // ...
        })
    })
}
```

**Транзакционные тесты (откат после каждого теста):**
```go
func TestWithTxRollback(t *testing.T) {
    db := getTestDB()

    tx, err := db.BeginTx(context.Background(), nil)
    require.NoError(t, err)
    defer tx.Rollback()  // Всегда откатываем!

    // Тест использует tx вместо db
    repo := NewUserRepository(tx)

    user := &User{Name: "Test", Email: "test@test.com"}
    err = repo.Create(context.Background(), user)
    require.NoError(t, err)

    // После теста транзакция откатится — БД чистая
}
```

**Типичные ошибки.**
1. **Моки не покрывают реальное поведение** — нужны integration тесты
2. **Тесты зависят друг от друга** — нет изоляции
3. **Медленные тесты без -short** — блокируют CI
4. **Hardcoded connection strings** — не работает в CI

**На интервью.**
Покажи разницу между sqlmock и testcontainers. Объясни, когда какой подход использовать. Расскажи про изоляцию параллельных тестов.

*Частые follow-up вопросы:*
- Как ускорить integration тесты?
- Как тестировать миграции?
- Как организовать fixtures?

---

## См. также

- [SQL Fundamentals](../06-databases/00-sql-fundamentals.md) — основы SQL
- [Transactions](../06-databases/03-transactions.md) — теория транзакций и изоляции
- [Indexing](../06-databases/01-indexing.md) — индексы и оптимизация запросов
- [Python Databases & ORM](../02-python/09-databases-orm.md) — работа с БД в Python
- [Distributed Transactions](../07-distributed-systems/07-distributed-transactions.md) — распределённые транзакции

---

## Практика

1. **database/sql:** Напиши connection pool с правильными настройками. Добавь graceful shutdown.

2. **Транзакции:** Реализуй перевод денег между счетами с pessimistic locking (FOR UPDATE).

3. **sqlx:** Перепиши CRUD с database/sql на sqlx. Сравни количество кода.

4. **Миграции:** Настрой golang-migrate с embed для CI. Добавь rollback в Makefile.

5. **Repository:** Реализуй Repository + Service слой. Напиши unit тесты с моками и integration с testcontainers.

6. **Query Builder:** Реализуй поиск с динамическими фильтрами через squirrel.

---

## Дополнительные материалы

- [database/sql tutorial](https://go.dev/doc/database/sql)
- [sqlx documentation](https://jmoiron.github.io/sqlx/)
- [golang-migrate](https://github.com/golang-migrate/migrate)
- [testcontainers-go](https://golang.testcontainers.org/)
- [sqlmock](https://github.com/DATA-DOG/go-sqlmock)
- [GORM](https://gorm.io/docs/)
- [ent](https://entgo.io/docs/getting-started)
- [squirrel](https://github.com/Masterminds/squirrel)
