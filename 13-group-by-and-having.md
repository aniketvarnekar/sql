# GROUP BY and HAVING

`GROUP BY` divides query results into groups based on shared values in one or more columns, and aggregate functions then compute a single value per group. `HAVING` filters those groups, playing the same role for grouped results that `WHERE` plays for individual rows. Together they enable the category-level, time-period, and cohort analysis that makes SQL so powerful for reporting.

## How GROUP BY Works

When you add `GROUP BY` to a query, MySQL collapses all rows that share the same value(s) in the specified columns into a single output row. Each aggregate function in the `SELECT` list then computes across all the rows in that group:

```sql
SELECT category, COUNT(*) AS product_count, AVG(price) AS avg_price
FROM products
GROUP BY category;
```

If the `products` table has 50 rows across 5 categories, this query returns 5 rows — one per category — with the count and average price for each. Without `GROUP BY`, the aggregates would operate on all 50 rows and return a single row.

Every column in the `SELECT` list that is not inside an aggregate function must appear in the `GROUP BY` clause. This rule is enforced by MySQL's `ONLY_FULL_GROUP_BY` mode (enabled by default in MySQL 8). Violating it means selecting a column whose value is ambiguous — it differs across the rows being collapsed into one group:

```sql
-- This fails in ONLY_FULL_GROUP_BY mode:
SELECT category, name, COUNT(*) FROM products GROUP BY category;
-- 'name' is not in GROUP BY and not aggregated — which name should appear?

-- Correct: include name in GROUP BY, or aggregate it
SELECT category, COUNT(*), MIN(name) AS sample_product FROM products GROUP BY category;
```

## HAVING vs WHERE

`WHERE` filters individual rows before grouping. `HAVING` filters groups after the aggregate functions have computed their results. They cannot be swapped:

```sql
-- WHERE filters rows before grouping:
SELECT category, COUNT(*) AS product_count
FROM products
WHERE price > 10.00
GROUP BY category;
-- Counts only products priced above $10 in each category

-- HAVING filters groups after grouping:
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING COUNT(*) >= 5;
-- Returns only categories that have at least 5 products

-- Both together:
SELECT category, COUNT(*) AS product_count, AVG(price) AS avg_price
FROM products
WHERE price > 10.00
GROUP BY category
HAVING COUNT(*) >= 3 AND AVG(price) < 100.00;
```

The rule is straightforward: conditions that filter individual rows go in `WHERE`, conditions that filter groups go in `HAVING`. Attempting to use an aggregate function in `WHERE` raises an error:

```sql
-- WRONG: cannot use aggregate in WHERE
SELECT category FROM products WHERE COUNT(*) > 5 GROUP BY category;

-- CORRECT: use HAVING
SELECT category FROM products GROUP BY category HAVING COUNT(*) > 5;
```

## Grouping by Multiple Columns

You can group by more than one column. The result has one row per unique combination of all the grouped columns:

```sql
SELECT department, YEAR(hire_date) AS hire_year, COUNT(*) AS hires
FROM employees
GROUP BY department, YEAR(hire_date)
ORDER BY department, hire_year;
```

This returns one row per department per hire year — for example, Engineering 2020: 3 hires, Engineering 2021: 5 hires, Marketing 2020: 2 hires, and so on.

## Using Aliases in HAVING

In MySQL (but not in standard SQL), you can use a `SELECT` alias in the `HAVING` clause:

```sql
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING product_count >= 5;
```

This MySQL extension is convenient but not portable to all databases. Using the full expression (`HAVING COUNT(*) >= 5`) is more portable and equally readable.

## SQL Logical Execution Order

Understanding the logical order in which MySQL processes a query is essential for understanding which clauses can reference which other clauses:

1. `FROM` — identifies the tables and applies joins
2. `WHERE` — filters individual rows
3. `GROUP BY` — groups the remaining rows
4. `HAVING` — filters groups
5. `SELECT` — computes expressions and selects columns
6. `ORDER BY` — sorts the result
7. `LIMIT` / `OFFSET` — constrains the final row count

This order explains why:
- `WHERE` cannot reference `SELECT` aliases (aliases don't exist yet at step 2)
- `HAVING` can reference `SELECT` aliases in MySQL (MySQL resolves aliases early as an extension)
- `ORDER BY` can reference `SELECT` aliases (aliases are resolved by step 6)
- Aggregate functions cannot appear in `WHERE` (aggregation happens at step 3, after `WHERE`)

Keeping this order in mind prevents confusion when debugging queries that reference columns or aliases in the wrong clause.

## GROUP BY with Expressions

You can group by an expression rather than a raw column:

```sql
SELECT YEAR(order_date) AS order_year, COUNT(*) AS orders_placed
FROM orders
GROUP BY YEAR(order_date)
ORDER BY order_year;

SELECT LEFT(last_name, 1) AS initial, COUNT(*) AS count
FROM customers
GROUP BY LEFT(last_name, 1)
ORDER BY initial;
```

## Common Patterns

Counting with a condition using `SUM(condition)`:

```sql
SELECT
    category,
    COUNT(*) AS total,
    SUM(in_stock = 1) AS in_stock_count,
    SUM(in_stock = 0) AS out_of_stock_count
FROM products
GROUP BY category;
```

`in_stock = 1` evaluates to 1 (true) or 0 (false) in MySQL, so `SUM` effectively counts the true rows.

Finding groups with the maximum aggregate value:

```sql
SELECT department, MAX(salary) AS top_salary
FROM employees
GROUP BY department
ORDER BY top_salary DESC;
```

## Practice Problems

### Problem 1
Write a query that shows the total revenue, average order value, and number of orders for each order status. Only include statuses that have at least two orders.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    status ENUM('pending', 'completed', 'cancelled', 'refunded') NOT NULL,
    total DECIMAL(10, 2) NOT NULL
);

INSERT INTO orders (status, total) VALUES
    ('completed', 200.00),
    ('completed', 450.00),
    ('completed', 125.00),
    ('cancelled', 75.00),
    ('pending', 300.00),
    ('pending', 180.00),
    ('refunded', 95.00);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    status,
    COUNT(*) AS order_count,
    ROUND(SUM(total), 2) AS total_revenue,
    ROUND(AVG(total), 2) AS avg_order_value
FROM orders
GROUP BY status
HAVING COUNT(*) >= 2
ORDER BY total_revenue DESC;
```

**Explanation:**
`GROUP BY status` creates one group per status value. The aggregate functions compute per-group. `HAVING COUNT(*) >= 2` filters out statuses with only one order — 'refunded' (1 order) is excluded. The result includes 'completed' (3 orders) and 'pending' (2 orders) and 'cancelled' (1 order, excluded). `ORDER BY total_revenue DESC` sorts the remaining groups by revenue descending.

### Problem 2
Write a query that finds all departments where the average salary exceeds $70,000 and where there are at least three employees. Return the department name, employee count, and average salary.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    department VARCHAR(100) NOT NULL,
    salary DECIMAL(10, 2) NOT NULL
);

INSERT INTO employees (name, department, salary) VALUES
    ('Alice', 'Engineering', 95000),
    ('Bob', 'Engineering', 85000),
    ('Carol', 'Engineering', 105000),
    ('Dave', 'Marketing', 65000),
    ('Eve', 'Marketing', 72000),
    ('Frank', 'Sales', 55000),
    ('Grace', 'Sales', 60000),
    ('Henry', 'Sales', 58000),
    ('Iris', 'Engineering', 90000);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    department,
    COUNT(*) AS employee_count,
    ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department
HAVING COUNT(*) >= 3 AND AVG(salary) > 70000
ORDER BY avg_salary DESC;
```

**Explanation:**
`GROUP BY department` produces one row per department. `HAVING COUNT(*) >= 3` requires at least 3 employees, which eliminates Marketing (2 employees). `HAVING AVG(salary) > 70000` requires average salary above $70k, which eliminates Sales (average: ~57,667). Only Engineering (4 employees, average: ~93,750) meets both conditions. Note that both conditions are in `HAVING` because they depend on aggregate values — they cannot go in `WHERE`.

### Problem 3
Write a query that shows monthly revenue for the year 2023, including the month name, total revenue, and the number of orders. Sort by month chronologically.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO orders (total, order_date) VALUES
    (320.00, '2023-01-15'),
    (180.00, '2023-01-28'),
    (450.00, '2023-02-10'),
    (210.00, '2023-03-05'),
    (390.00, '2023-03-22'),
    (500.00, '2023-03-30'),
    (125.00, '2023-04-11');
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    MONTH(order_date) AS month_number,
    MONTHNAME(order_date) AS month_name,
    COUNT(*) AS order_count,
    ROUND(SUM(total), 2) AS monthly_revenue
FROM orders
WHERE YEAR(order_date) = 2023
GROUP BY MONTH(order_date), MONTHNAME(order_date)
ORDER BY month_number;
```

**Explanation:**
`WHERE YEAR(order_date) = 2023` filters to the correct year before grouping. `GROUP BY MONTH(order_date), MONTHNAME(order_date)` groups by month — both expressions are included in `GROUP BY` because `MONTHNAME(order_date)` appears in the `SELECT` but is not aggregated. Including both ensures correctness and satisfies `ONLY_FULL_GROUP_BY`. `ORDER BY month_number` sorts chronologically using the numeric month (1-12), which is correct. Sorting by `month_name` alphabetically would produce April, February, January, March — incorrect order.
