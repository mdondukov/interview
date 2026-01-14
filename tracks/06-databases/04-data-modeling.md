# 04. Data Modeling

[← Назад к списку тем](README.md)

---

## Нормализация

### Нормальные формы

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Normal Forms                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1NF (First Normal Form)                                            │
│  ───────────────────────                                            │
│  - Атомарные значения (нет массивов в ячейках)                      │
│  - Уникальные строки (есть первичный ключ)                          │
│                                                                     │
│  ❌ Нарушение: tags = "go,python,java"                              │
│  ✅ Правильно: отдельная таблица tags                               │
│                                                                     │
│  2NF (Second Normal Form)                                           │
│  ────────────────────────                                           │
│  - Соответствует 1NF                                                │
│  - Нет частичных зависимостей от составного ключа                   │
│                                                                     │
│  ❌ Нарушение:                                                      │
│  order_items(order_id, product_id, product_name, qty)               │
│  product_name зависит только от product_id, не от (order_id, prod_id)│
│                                                                     │
│  ✅ Правильно:                                                      │
│  order_items(order_id, product_id, qty)                             │
│  products(product_id, product_name)                                 │
│                                                                     │
│  3NF (Third Normal Form)                                            │
│  ───────────────────────                                            │
│  - Соответствует 2NF                                                │
│  - Нет транзитивных зависимостей                                    │
│                                                                     │
│  ❌ Нарушение:                                                      │
│  employees(id, department_id, department_name)                      │
│  department_name зависит от department_id, не от id                 │
│                                                                     │
│  ✅ Правильно:                                                      │
│  employees(id, department_id)                                       │
│  departments(department_id, department_name)                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример нормализации

```sql
-- ❌ Ненормализованная таблица
CREATE TABLE orders (
    id SERIAL,
    customer_name TEXT,
    customer_email TEXT,
    customer_address TEXT,
    product_names TEXT[],      -- Нарушение 1NF
    product_prices NUMERIC[],
    total NUMERIC
);

-- ✅ Нормализованная схема (3NF)
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE addresses (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    street TEXT,
    city TEXT,
    zip TEXT
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC NOT NULL
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    shipping_address_id INTEGER REFERENCES addresses(id),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL,
    price_at_time NUMERIC NOT NULL  -- Снимок цены
);
```

---

## Денормализация

### Когда денормализовать

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Нормализация vs Денормализация                    │
├──────────────────────────────┬──────────────────────────────────────┤
│        Нормализация          │          Денормализация              │
├──────────────────────────────┼──────────────────────────────────────┤
│ Меньше дублирования          │ Быстрее чтение                       │
│ Проще обновление             │ Меньше JOINs                         │
│ Целостность данных           │ Сложнее обновление                   │
│ Больше JOINs                 │ Возможна рассинхронизация            │
│ Хорошо для OLTP              │ Хорошо для OLAP, отчётов             │
└──────────────────────────────┴──────────────────────────────────────┘
```

### Техники денормализации

```sql
-- 1. Копирование полей (снимок)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    customer_name TEXT,       -- Копия из customers
    customer_email TEXT,      -- Копия из customers
    created_at TIMESTAMP
);

-- 2. Агрегированные значения
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    price NUMERIC,
    review_count INTEGER DEFAULT 0,   -- Денормализовано
    avg_rating NUMERIC DEFAULT 0      -- Денормализовано
);

-- Обновляем триггером или в приложении
CREATE OR REPLACE FUNCTION update_product_stats()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products
    SET review_count = (SELECT COUNT(*) FROM reviews WHERE product_id = NEW.product_id),
        avg_rating = (SELECT AVG(rating) FROM reviews WHERE product_id = NEW.product_id)
    WHERE id = NEW.product_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 3. Материализованные представления
CREATE MATERIALIZED VIEW product_stats AS
SELECT
    p.id,
    p.name,
    COUNT(r.id) as review_count,
    AVG(r.rating) as avg_rating
FROM products p
LEFT JOIN reviews r ON r.product_id = p.id
GROUP BY p.id;

-- Обновлять периодически
REFRESH MATERIALIZED VIEW CONCURRENTLY product_stats;
```

---

## Связи

### One-to-One

```sql
-- Разделение таблицы (для производительности или безопасности)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name TEXT
);

CREATE TABLE user_profiles (
    user_id INTEGER PRIMARY KEY REFERENCES users(id),
    bio TEXT,
    avatar_url TEXT,
    settings JSONB
);
```

### One-to-Many

```sql
CREATE TABLE authors (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    author_id INTEGER REFERENCES authors(id),
    title TEXT NOT NULL
);

-- Индекс на foreign key
CREATE INDEX idx_books_author ON books(author_id);
```

### Many-to-Many

```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL
);

-- Junction table
CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(id),
    course_id INTEGER REFERENCES courses(id),
    enrolled_at TIMESTAMP DEFAULT NOW(),
    grade TEXT,
    PRIMARY KEY (student_id, course_id)
);
```

### Self-referencing

```sql
-- Иерархия (дерево)
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id INTEGER REFERENCES categories(id)
);

-- Запрос дерева
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 as depth
    FROM categories WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree;
```

---

## Типы данных

### Числовые

```sql
-- Целые
SMALLINT       -- 2 bytes, -32768 to 32767
INTEGER        -- 4 bytes, -2B to 2B
BIGINT         -- 8 bytes, -9 quintillion to 9 quintillion
SERIAL         -- auto-increment INTEGER
BIGSERIAL      -- auto-increment BIGINT

-- Дробные
NUMERIC(10,2)  -- Точное (для денег!)
DECIMAL(10,2)  -- Синоним NUMERIC
REAL           -- 6 decimal digits
DOUBLE PRECISION -- 15 decimal digits

-- Для денег ВСЕГДА NUMERIC!
price NUMERIC(10,2) -- до 99999999.99
```

### Строковые

```sql
TEXT           -- Любая длина (предпочтительно)
VARCHAR(n)     -- До n символов
CHAR(n)        -- Фиксированная длина (редко используется)

-- Поиск
LIKE '%pattern%'  -- Медленно
ILIKE            -- Case-insensitive
~ 'regex'        -- Регулярки
```

### Дата и время

```sql
DATE           -- Только дата
TIME           -- Только время
TIMESTAMP      -- Дата + время (без timezone)
TIMESTAMPTZ    -- Дата + время с timezone (рекомендуется!)

-- Всегда используй TIMESTAMPTZ
created_at TIMESTAMPTZ DEFAULT NOW()

-- Полезные функции
NOW()
CURRENT_DATE
CURRENT_TIMESTAMP
DATE_TRUNC('month', created_at)
EXTRACT(YEAR FROM created_at)
```

### JSON

```sql
JSON           -- Хранит как текст
JSONB          -- Binary, быстрее запросы, индексы

-- JSONB операторы
data -> 'key'          -- JSON значение
data ->> 'key'         -- Text значение
data #> '{a,b}'        -- Вложенный путь
data @> '{"type":"x"}' -- Содержит

-- Индекс
CREATE INDEX idx_data ON events USING gin(data);
```

### UUID

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT UNIQUE
);

-- Или в PostgreSQL 13+
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);
```

### Arrays

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[]
);

-- Запросы
SELECT * FROM posts WHERE 'go' = ANY(tags);
SELECT * FROM posts WHERE tags @> ARRAY['go', 'postgres'];

-- Индекс
CREATE INDEX idx_posts_tags ON posts USING gin(tags);
```

---

## Constraints

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    external_order_id TEXT,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'completed', 'cancelled')),
    total NUMERIC NOT NULL CHECK (total >= 0),
    created_at TIMESTAMPTZ DEFAULT NOW(),

    -- Составной уникальный
    UNIQUE (user_id, external_order_id)
);

-- Именованные constraints
ALTER TABLE orders
ADD CONSTRAINT orders_status_check
CHECK (status IN ('pending', 'processing', 'completed', 'cancelled'));

-- Partial unique
CREATE UNIQUE INDEX idx_users_email_active
ON users(email) WHERE deleted_at IS NULL;
```

---

## Паттерны проектирования

### Soft Delete

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE,
    deleted_at TIMESTAMPTZ
);

-- Запросы всегда фильтруют
SELECT * FROM users WHERE deleted_at IS NULL;

-- Partial index
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    record_id INTEGER NOT NULL,
    action TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    old_data JSONB,
    new_data JSONB,
    changed_by INTEGER,
    changed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Триггер для аудита
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_by)
    VALUES (
        TG_TABLE_NAME,
        COALESCE(NEW.id, OLD.id),
        TG_OP,
        CASE WHEN TG_OP = 'DELETE' THEN to_jsonb(OLD) ELSE NULL END,
        CASE WHEN TG_OP != 'DELETE' THEN to_jsonb(NEW) ELSE NULL END,
        current_setting('app.current_user_id', true)::integer
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### Versioning

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE document_versions (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES documents(id),
    content TEXT NOT NULL,
    version INTEGER NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## См. также

- [DDD Tactical Patterns](../08-architecture/02-ddd-tactical.md) — тактические паттерны Domain-Driven Design
- [NoSQL Patterns](./08-nosql-patterns.md) — моделирование данных в NoSQL

---

## На интервью

### Типичные вопросы

1. **Когда нормализовать, когда денормализовать?**
   - Нормализация: OLTP, частые updates
   - Денормализация: OLAP, отчёты, read-heavy

2. **3NF — объясните**
   - Атомарные значения
   - Нет частичных зависимостей
   - Нет транзитивных зависимостей

3. **UUID vs SERIAL для primary key?**
   - UUID: распределённые системы, no collision
   - SERIAL: проще, меньше, быстрее index

4. **Soft delete — плюсы/минусы?**
   - Плюсы: восстановление, аудит
   - Минусы: все запросы фильтруют, рост таблицы

5. **Как хранить деньги?**
   - NUMERIC (точное представление)
   - Или INTEGER (центы)
   - Никогда FLOAT!

---

[← Назад к списку тем](README.md)
