# 00. SQL Fundamentals

[← Назад к списку тем](README.md)

---

## SELECT и базовые операции

### Порядок выполнения SQL

```
Написание:          Выполнение:
SELECT              1. FROM
FROM                2. JOIN
JOIN                3. WHERE
WHERE               4. GROUP BY
GROUP BY            5. HAVING
HAVING              6. SELECT
ORDER BY            7. DISTINCT
LIMIT               8. ORDER BY
                    9. LIMIT/OFFSET
```

### Базовые запросы

```sql
-- Выборка с условиями
SELECT
    u.id,
    u.name,
    u.email,
    COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at >= '2024-01-01'
  AND u.status = 'active'
GROUP BY u.id, u.name, u.email
HAVING COUNT(o.id) > 5
ORDER BY order_count DESC
LIMIT 10 OFFSET 20;
```

---

## JOINs

### Типы JOIN

```
┌─────────────────────────────────────────────────────────────────────┐
│                          JOIN Types                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  INNER JOIN          LEFT JOIN           RIGHT JOIN                 │
│  ┌───┬───┐          ┌───┬───┐          ┌───┬───┐                   │
│  │ A │ B │          │ A │ B │          │ A │ B │                   │
│  │   │░░░│          │░░░│░░░│          │   │░░░│                   │
│  │   │░░░│          │░░░│░░░│          │   │░░░│                   │
│  └───┴───┘          └───┴───┘          └───┴───┘                   │
│  Только совпадения  A + совпадения     B + совпадения              │
│                                                                     │
│  FULL OUTER JOIN     CROSS JOIN                                     │
│  ┌───┬───┐          A × B = все комбинации                         │
│  │░░░│░░░│                                                         │
│  │░░░│░░░│          SELECT * FROM a CROSS JOIN b                   │
│  └───┴───┘          -- или просто FROM a, b                        │
│  Все записи обеих                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Примеры JOIN

```sql
-- INNER JOIN: только пользователи с заказами
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id;

-- LEFT JOIN: все пользователи, даже без заказов
SELECT u.name, COALESCE(SUM(o.total), 0) as total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;

-- Self JOIN: иерархия сотрудников
SELECT
    e.name as employee,
    m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Multiple JOINs
SELECT
    o.id as order_id,
    u.name as customer,
    p.name as product,
    oi.quantity
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'completed';
```

---

## Subqueries

### Типы подзапросов

```sql
-- Scalar subquery (возвращает одно значение)
SELECT
    name,
    salary,
    (SELECT AVG(salary) FROM employees) as avg_salary
FROM employees;

-- Table subquery (в FROM)
SELECT dept, avg_salary
FROM (
    SELECT department as dept, AVG(salary) as avg_salary
    FROM employees
    GROUP BY department
) dept_stats
WHERE avg_salary > 50000;

-- Correlated subquery (ссылается на внешний запрос)
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department = e.department
);

-- EXISTS (проверка существования)
SELECT u.name
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
    AND o.total > 1000
);

-- IN с подзапросом
SELECT name
FROM products
WHERE category_id IN (
    SELECT id FROM categories WHERE active = true
);
```

### Subquery vs JOIN

```sql
-- Subquery (менее эффективно для больших таблиц)
SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE status = 'vip');

-- JOIN (обычно быстрее)
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'vip';

-- Когда subquery лучше:
-- 1. EXISTS/NOT EXISTS
-- 2. Scalar subqueries
-- 3. Сложные агрегации
```

---

## Window Functions

### Синтаксис

```sql
function_name() OVER (
    [PARTITION BY columns]
    [ORDER BY columns]
    [frame_clause]
)
```

### Ranking Functions

```sql
-- ROW_NUMBER: уникальный номер
-- RANK: с пропусками при дублях
-- DENSE_RANK: без пропусков

SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num,
    RANK() OVER (ORDER BY salary DESC) as rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank
FROM employees;

-- Результат для зарплат 100, 100, 90:
-- row_num: 1, 2, 3
-- rank:    1, 1, 3  (пропуск 2)
-- dense_rank: 1, 1, 2 (без пропуска)

-- Top-N в каждой группе
SELECT * FROM (
    SELECT
        name,
        department,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) as rn
    FROM employees
) ranked
WHERE rn <= 3;
```

### Aggregate Window Functions

```sql
-- Агрегаты как window functions
SELECT
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_total,
    AVG(amount) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg_3,
    SUM(amount) OVER (PARTITION BY EXTRACT(MONTH FROM date)) as monthly_total
FROM transactions;
```

### LAG / LEAD

```sql
-- Доступ к предыдущей/следующей строке
SELECT
    date,
    price,
    LAG(price, 1) OVER (ORDER BY date) as prev_price,
    LEAD(price, 1) OVER (ORDER BY date) as next_price,
    price - LAG(price, 1) OVER (ORDER BY date) as price_change
FROM stock_prices;

-- Поиск gaps в последовательности
SELECT
    id,
    LAG(id) OVER (ORDER BY id) as prev_id,
    id - LAG(id) OVER (ORDER BY id) as gap
FROM records
WHERE id - LAG(id) OVER (ORDER BY id) > 1;
```

### FIRST_VALUE / LAST_VALUE

```sql
SELECT
    employee_id,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) as highest_in_dept,
    salary * 100.0 / FIRST_VALUE(salary) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) as percent_of_highest
FROM employees;
```

### Frame Clause

```sql
-- ROWS: физические строки
-- RANGE: логический диапазон значений

SELECT
    date,
    amount,
    -- Последние 7 дней (включая текущий)
    SUM(amount) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
    ) as week_sum,
    -- Последние 3 строки
    AVG(amount) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg
FROM daily_sales;

-- Границы:
-- UNBOUNDED PRECEDING - от начала
-- n PRECEDING - n строк/значений назад
-- CURRENT ROW - текущая строка
-- n FOLLOWING - n строк/значений вперёд
-- UNBOUNDED FOLLOWING - до конца
```

---

## Common Table Expressions (CTE)

### Базовый CTE

```sql
WITH active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active'
      AND last_login > CURRENT_DATE - INTERVAL '30 days'
)
SELECT au.name, COUNT(o.id) as order_count
FROM active_users au
LEFT JOIN orders o ON o.user_id = au.id
GROUP BY au.id, au.name;
```

### Multiple CTEs

```sql
WITH
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) as month,
        SUM(total) as revenue
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY DATE_TRUNC('month', order_date)
),
growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) as prev_revenue,
        (revenue - LAG(revenue) OVER (ORDER BY month)) /
            LAG(revenue) OVER (ORDER BY month) * 100 as growth_percent
    FROM monthly_sales
)
SELECT * FROM growth
WHERE growth_percent IS NOT NULL
ORDER BY month;
```

### Recursive CTE

```sql
-- Иерархия категорий
WITH RECURSIVE category_tree AS (
    -- Базовый случай: корневые категории
    SELECT id, name, parent_id, 1 as level,
           name::text as path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Рекурсивный случай
    SELECT c.id, c.name, c.parent_id, ct.level + 1,
           ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree
ORDER BY path;

-- Генерация последовательности дат
WITH RECURSIVE dates AS (
    SELECT DATE '2024-01-01' as date
    UNION ALL
    SELECT date + 1
    FROM dates
    WHERE date < '2024-12-31'
)
SELECT date FROM dates;
```

---

## Агрегатные функции

### Базовые агрегаты

```sql
SELECT
    COUNT(*) as total_rows,
    COUNT(DISTINCT category) as unique_categories,
    COUNT(email) as non_null_emails,  -- NULL не считаются
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MIN(created_at) as first_created,
    MAX(created_at) as last_created
FROM orders;
```

### GROUP BY и HAVING

```sql
-- HAVING фильтрует группы после агрегации
SELECT
    category,
    COUNT(*) as product_count,
    AVG(price) as avg_price
FROM products
GROUP BY category
HAVING COUNT(*) >= 10 AND AVG(price) > 100;

-- GROUPING SETS
SELECT
    COALESCE(category, 'ALL') as category,
    COALESCE(brand, 'ALL') as brand,
    SUM(sales) as total_sales
FROM products
GROUP BY GROUPING SETS (
    (category, brand),
    (category),
    (brand),
    ()  -- grand total
);

-- ROLLUP (иерархическая агрегация)
SELECT
    year,
    quarter,
    month,
    SUM(sales)
FROM sales
GROUP BY ROLLUP (year, quarter, month);
-- Даёт: (year, quarter, month), (year, quarter), (year), ()
```

### Array агрегаты (PostgreSQL)

```sql
-- Собрать значения в массив
SELECT
    user_id,
    ARRAY_AGG(product_name ORDER BY order_date) as purchased_products,
    STRING_AGG(product_name, ', ' ORDER BY order_date) as products_list
FROM orders o
JOIN products p ON o.product_id = p.id
GROUP BY user_id;

-- JSON агрегат
SELECT
    user_id,
    JSON_AGG(
        JSON_BUILD_OBJECT(
            'product', product_name,
            'date', order_date,
            'amount', amount
        )
    ) as orders
FROM orders
GROUP BY user_id;
```

---

## CASE и условная логика

```sql
-- Simple CASE
SELECT
    name,
    CASE status
        WHEN 'A' THEN 'Active'
        WHEN 'I' THEN 'Inactive'
        WHEN 'P' THEN 'Pending'
        ELSE 'Unknown'
    END as status_name
FROM users;

-- Searched CASE
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 100000 THEN 'Senior'
        WHEN salary >= 70000 THEN 'Middle'
        WHEN salary >= 40000 THEN 'Junior'
        ELSE 'Intern'
    END as level
FROM employees;

-- CASE в агрегации (pivot)
SELECT
    product_id,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN amount ELSE 0 END) as jan,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 2 THEN amount ELSE 0 END) as feb,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 3 THEN amount ELSE 0 END) as mar
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2024
GROUP BY product_id;

-- COALESCE и NULLIF
SELECT
    COALESCE(nickname, name, 'Anonymous') as display_name,
    NULLIF(discount, 0) as effective_discount  -- NULL если 0
FROM users;
```

---

## На интервью

### Типичные вопросы

1. **Разница между WHERE и HAVING?**
   - WHERE: фильтрует строки до агрегации
   - HAVING: фильтрует группы после агрегации

2. **INNER JOIN vs LEFT JOIN?**
   - INNER: только совпадающие записи
   - LEFT: все из левой + совпадения из правой

3. **Когда использовать Window Functions?**
   - Ranking, running totals, moving averages
   - Сравнение с предыдущими/следующими строками
   - Агрегаты без GROUP BY

4. **CTE vs Subquery?**
   - CTE: читаемость, reusability, рекурсия
   - Subquery: inline, иногда быстрее

5. **Порядок выполнения SQL?**
   - FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

---

[← Назад к списку тем](README.md)
