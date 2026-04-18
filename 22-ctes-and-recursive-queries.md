# CTEs and Recursive Queries

A Common Table Expression (CTE) is a named temporary result set defined within a query using the `WITH` clause. CTEs make complex queries more readable by breaking them into named, composable parts, and they enable recursive queries — queries that repeatedly reference themselves to traverse hierarchical or graph-structured data. CTEs were introduced in MySQL 8.0.

## WITH Clause (Common Table Expressions)

A CTE is defined before the main `SELECT` and can be referenced by name as if it were a table:

```sql
WITH high_value_orders AS (
    SELECT id, customer_id, total
    FROM orders
    WHERE total > 500
)
SELECT c.name, hvo.total
FROM high_value_orders hvo
JOIN customers c ON hvo.customer_id = c.id
ORDER BY hvo.total DESC;
```

The CTE `high_value_orders` is computed first, then the outer query references it by name. The CTE exists only for the duration of the statement — it is not stored anywhere.

## Multiple CTEs in One Query

Multiple CTEs can be chained in a single `WITH` clause, separated by commas. Later CTEs can reference earlier ones:

```sql
WITH
order_totals AS (
    SELECT customer_id, SUM(total) AS lifetime_value
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
top_customers AS (
    SELECT customer_id, lifetime_value
    FROM order_totals
    WHERE lifetime_value > 1000
)
SELECT c.name, tc.lifetime_value
FROM top_customers tc
JOIN customers c ON tc.customer_id = c.id
ORDER BY tc.lifetime_value DESC;
```

`top_customers` references `order_totals`, which was defined before it. This chaining allows you to build up a complex transformation in clear, named steps.

## CTE vs Subquery vs Temp Table

All three mechanisms can express similar logic. The choice depends on readability, reuse, and performance.

A **subquery** (derived table in `FROM`) is anonymous and cannot be referenced more than once in a query. It is appropriate for simple, one-off intermediate results:

```sql
SELECT * FROM (SELECT id, total FROM orders WHERE total > 500) AS big_orders;
```

A **CTE** is named and can be referenced multiple times in the same query. It improves readability for complex logic and is the preferred choice in MySQL 8.x over deeply nested subqueries. However, MySQL does not materialize CTEs by default — they are expanded inline each time they are referenced, so referencing the same CTE twice may execute it twice:

```sql
WITH expensive AS (SELECT id FROM products WHERE price > 100)
SELECT * FROM expensive e1 JOIN expensive e2 ON e1.id != e2.id;
-- 'expensive' subquery may run twice
```

A **temporary table** is materialized — the data is actually stored (in memory or on disk) and the result is computed only once. Use a temporary table when a CTE is referenced many times and its underlying query is expensive:

```sql
CREATE TEMPORARY TABLE expensive AS SELECT id FROM products WHERE price > 100;
SELECT * FROM expensive e1 JOIN expensive e2 ON e1.id != e2.id;
DROP TEMPORARY TABLE expensive;
```

Temporary tables persist for the duration of the session and are automatically dropped when the connection closes.

## Recursive CTEs

A recursive CTE references itself. It has two parts separated by `UNION ALL`:

1. The **anchor member** (base case) — a non-recursive query that produces the initial rows
2. The **recursive member** — a query that references the CTE by name and is executed repeatedly until it produces no more rows

```sql
WITH RECURSIVE counting AS (
    -- Anchor: start at 1
    SELECT 1 AS n
    UNION ALL
    -- Recursive: add 1 each iteration until n >= 10
    SELECT n + 1 FROM counting WHERE n < 10
)
SELECT n FROM counting;
-- Returns: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

The keyword `RECURSIVE` is required in the `WITH` clause when the CTE references itself. MySQL enforces a maximum recursion depth controlled by `cte_max_recursion_depth` (default: 1000). Queries that would recurse more deeply raise an error.

## Example: Hierarchy Traversal (Employee-Manager Tree)

Recursive CTEs are the natural tool for traversing hierarchical data:

```sql
WITH RECURSIVE org_tree AS (
    -- Anchor: start with the top-level employee (no manager)
    SELECT id, name, manager_id, 0 AS depth, CAST(name AS CHAR(1000)) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: find direct reports of the current level
    SELECT e.id, e.name, e.manager_id, ot.depth + 1,
           CONCAT(ot.path, ' > ', e.name)
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT depth, name, path
FROM org_tree
ORDER BY path;
```

Each recursive step finds employees whose `manager_id` matches a row already in `org_tree`. The `depth` column tracks how many levels down from the root each employee is. The `path` column builds a readable string showing the reporting chain.

## Example: Generating a Number Series

Recursive CTEs can generate sequences of numbers without needing a numbers table:

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 100
)
SELECT n FROM numbers;
```

This can be combined with a date to generate a series of dates:

```sql
WITH RECURSIVE date_series AS (
    SELECT '2024-01-01' AS dt
    UNION ALL
    SELECT DATE_ADD(dt, INTERVAL 1 DAY)
    FROM date_series
    WHERE dt < '2024-01-31'
)
SELECT dt FROM date_series;
```

Date series are useful for reporting queries that need to show all days in a range, even days with no data:

```sql
WITH RECURSIVE date_series AS (
    SELECT '2024-01-01' AS dt
    UNION ALL
    SELECT DATE_ADD(dt, INTERVAL 1 DAY) FROM date_series WHERE dt < '2024-01-31'
)
SELECT ds.dt, COALESCE(SUM(o.total), 0) AS daily_revenue
FROM date_series ds
LEFT JOIN orders o ON DATE(o.created_at) = ds.dt
GROUP BY ds.dt
ORDER BY ds.dt;
```

Without the CTE-generated series, days with zero orders would simply not appear in the result.

## Practice Problems

### Problem 1
Use a CTE to find the top-spending customer in each country. The CTE should compute lifetime spending per customer, and the outer query should find the maximum spender per country.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    country VARCHAR(100) NOT NULL
);

CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL
);

INSERT INTO customers (name, country) VALUES
    ('Alice', 'USA'), ('Bob', 'USA'), ('Carol', 'Canada'), ('Dave', 'Canada');
INSERT INTO orders (customer_id, total) VALUES
    (1, 500), (1, 300), (2, 900), (3, 400), (3, 200), (4, 150);
```

**Your query:**
```sql
-- Write the CTE-based query.
```

**Solution:**
```sql
WITH customer_spending AS (
    SELECT c.id, c.name, c.country, SUM(o.total) AS total_spent
    FROM customers c
    JOIN orders o ON o.customer_id = c.id
    GROUP BY c.id, c.name, c.country
),
max_per_country AS (
    SELECT country, MAX(total_spent) AS max_spent
    FROM customer_spending
    GROUP BY country
)
SELECT cs.name, cs.country, cs.total_spent
FROM customer_spending cs
JOIN max_per_country mpc ON cs.country = mpc.country AND cs.total_spent = mpc.max_spent
ORDER BY cs.country, cs.name;
```

**Explanation:**
The first CTE `customer_spending` computes the lifetime spending per customer. The second CTE `max_per_country` finds the maximum spending value per country. The final join connects back to `customer_spending` to retrieve the names of the top spenders. This pattern handles ties: if two customers in the same country share the maximum spending, both appear. Chaining CTEs keeps each step readable and independently understandable.

### Problem 2
Use a recursive CTE to traverse the employee hierarchy and return each employee, their manager's name, their depth in the org chart, and the full reporting path from the top.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    manager_id INT UNSIGNED
);

INSERT INTO employees (id, name, manager_id) VALUES
    (1, 'CEO', NULL),
    (2, 'CTO', 1),
    (3, 'CFO', 1),
    (4, 'Lead Engineer', 2),
    (5, 'Engineer A', 4),
    (6, 'Engineer B', 4),
    (7, 'Finance Manager', 3);
```

**Your query:**
```sql
-- Write the recursive CTE.
```

**Solution:**
```sql
WITH RECURSIVE org_tree AS (
    SELECT
        e.id,
        e.name,
        e.manager_id,
        0 AS depth,
        CAST(e.name AS CHAR(500)) AS path
    FROM employees e
    WHERE e.manager_id IS NULL

    UNION ALL

    SELECT
        e.id,
        e.name,
        e.manager_id,
        ot.depth + 1,
        CONCAT(ot.path, ' > ', e.name)
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT
    ot.name AS employee,
    m.name AS manager,
    ot.depth,
    ot.path
FROM org_tree ot
LEFT JOIN employees m ON ot.manager_id = m.id
ORDER BY ot.path;
```

**Explanation:**
The anchor selects the CEO (where `manager_id IS NULL`) with depth 0. Each recursive step joins `employees` to the current result, finding direct reports and incrementing depth by 1. `CAST(e.name AS CHAR(500))` is required because MySQL needs to know the column type for the recursive result set — `CHAR(500)` provides enough space for the concatenated path. The `LEFT JOIN` to `employees m` in the final query retrieves the manager's name for display, with NULL for the CEO who has no manager.

### Problem 3
Generate a series of all dates in January 2024 and join it with an orders table to produce a daily revenue report. Days with no orders should show 0 revenue.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO orders (total, order_date) VALUES
    (150.00, '2024-01-05'),
    (250.00, '2024-01-05'),
    (100.00, '2024-01-12'),
    (400.00, '2024-01-20'),
    (75.00, '2024-01-20');
```

**Your query:**
```sql
-- Write the recursive date series + LEFT JOIN query.
```

**Solution:**
```sql
WITH RECURSIVE january AS (
    SELECT DATE('2024-01-01') AS day
    UNION ALL
    SELECT DATE_ADD(day, INTERVAL 1 DAY)
    FROM january
    WHERE day < '2024-01-31'
)
SELECT
    j.day,
    COALESCE(SUM(o.total), 0) AS daily_revenue,
    COUNT(o.id) AS order_count
FROM january j
LEFT JOIN orders o ON o.order_date = j.day
GROUP BY j.day
ORDER BY j.day;
```

**Explanation:**
The recursive CTE generates all 31 days of January 2024. The `LEFT JOIN` brings in order totals where they exist. For days with no orders, `o.total` is NULL, and `COALESCE(SUM(o.total), 0)` converts that NULL sum to 0. Without the date series, a simple `GROUP BY order_date` would only return the 3 days that have orders, making it impossible to see that most days had zero revenue. This pattern is one of the most practical applications of recursive CTEs in reporting queries.
