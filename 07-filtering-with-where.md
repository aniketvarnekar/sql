# Filtering with WHERE

The `WHERE` clause is how you restrict which rows a query retrieves, updates, or deletes. Without `WHERE`, a `SELECT` returns every row, an `UPDATE` modifies every row, and a `DELETE` removes every row. Mastering the operators and patterns available in `WHERE` is essential for writing queries that work on exactly the data you intend.

## Comparison Operators

MySQL supports the standard set of comparison operators for use in `WHERE` conditions:

- `=` — equal to
- `!=` or `<>` — not equal to
- `<` — less than
- `>` — greater than
- `<=` — less than or equal to
- `>=` — greater than or equal to

```sql
SELECT * FROM products WHERE price = 9.99;
SELECT * FROM products WHERE price != 9.99;
SELECT * FROM products WHERE price > 50.00;
SELECT * FROM products WHERE price <= 100.00;
```

These operators work on numeric, string, and date values. String comparisons use the column's collation (usually case-insensitive by default in MySQL 8). Date comparisons treat date strings correctly when they match the `YYYY-MM-DD` format:

```sql
SELECT * FROM orders WHERE order_date >= '2024-01-01';
SELECT * FROM employees WHERE hire_date < '2020-06-15';
```

## Logical Operators: AND, OR, NOT

Multiple conditions can be combined with `AND`, `OR`, and `NOT`. `AND` requires all conditions to be true. `OR` requires at least one condition to be true. `NOT` inverts a condition.

```sql
SELECT * FROM products WHERE price > 10 AND price < 100;
SELECT * FROM products WHERE category = 'Electronics' OR category = 'Computers';
SELECT * FROM products WHERE NOT discontinued;
```

When combining `AND` and `OR`, be explicit with parentheses. `AND` has higher precedence than `OR`, which can lead to unexpected behavior if you rely on implicit grouping:

```sql
-- Ambiguous: returns products with price > 100 OR (category = 'Books' AND in_stock = 1)
SELECT * FROM products WHERE price > 100 OR category = 'Books' AND in_stock = 1;

-- Clear: returns products that are either expensive or available books
SELECT * FROM products WHERE price > 100 OR (category = 'Books' AND in_stock = 1);
```

Always add parentheses when mixing `AND` and `OR` to make your intent unambiguous.

## BETWEEN

`BETWEEN low AND high` is inclusive on both ends — it matches values where `value >= low AND value <= high`:

```sql
SELECT * FROM products WHERE price BETWEEN 10.00 AND 50.00;
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';
```

`NOT BETWEEN` excludes the range:

```sql
SELECT * FROM products WHERE price NOT BETWEEN 10.00 AND 50.00;
```

## IN

`IN` matches a column against a list of values and is more readable than a series of `OR` conditions:

```sql
SELECT * FROM products WHERE category IN ('Books', 'Music', 'Movies');
-- Equivalent to:
SELECT * FROM products WHERE category = 'Books' OR category = 'Music' OR category = 'Movies';
```

`NOT IN` excludes rows where the column matches any value in the list:

```sql
SELECT * FROM orders WHERE status NOT IN ('cancelled', 'refunded');
```

A critical warning: `NOT IN` with a list that contains `NULL` always returns zero rows. If the `IN` list includes `NULL`, then `NOT IN` returns `NULL` (which is falsy) for every row, because `value != NULL` evaluates to `NULL`, not `TRUE`. Always ensure the `IN` list is free of `NULL` when using `NOT IN`. Use `NOT EXISTS` or `IS NOT NULL` checks as alternatives when `NULL` might be present.

## IS NULL and IS NOT NULL

`NULL` represents a missing or unknown value. It cannot be compared with `=` or `!=` — those comparisons always return `NULL`, which is falsy. Use `IS NULL` and `IS NOT NULL`:

```sql
-- Wrong (returns no rows even when NULL values exist):
SELECT * FROM employees WHERE manager_id = NULL;

-- Correct:
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE manager_id IS NOT NULL;
```

MySQL's `<=>` operator (the NULL-safe equality operator) returns `TRUE` when both sides are `NULL`, unlike `=`:

```sql
SELECT * FROM employees WHERE manager_id <=> NULL;
-- This correctly matches rows where manager_id IS NULL
```

## LIKE and Wildcard Patterns

`LIKE` matches a column against a pattern string using two wildcards:

- `%` matches any sequence of zero or more characters
- `_` matches exactly one character

```sql
-- Names starting with 'J':
SELECT * FROM customers WHERE name LIKE 'J%';

-- Names ending with 'son':
SELECT * FROM customers WHERE name LIKE '%son';

-- Names containing 'ar':
SELECT * FROM customers WHERE name LIKE '%ar%';

-- Names that are exactly 5 characters:
SELECT * FROM customers WHERE name LIKE '_____';

-- Names that start with any character followed by 'ob':
SELECT * FROM customers WHERE name LIKE '_ob';
```

`NOT LIKE` excludes rows that match the pattern:

```sql
SELECT * FROM customers WHERE email NOT LIKE '%@gmail.com';
```

`LIKE` is case-insensitive when using the default `utf8mb4_0900_ai_ci` collation. For case-sensitive matching, use `LIKE BINARY` or switch to a `_cs` (case-sensitive) collation.

`LIKE` with a leading `%` (e.g., `LIKE '%smith'`) cannot use a standard B-tree index and requires a full table scan. This is a significant performance problem on large tables. For full-text pattern matching on large datasets, MySQL's `FULLTEXT` indexes (covered in the Full-Text Search file) are more efficient.

To match a literal `%` or `_` character in a `LIKE` pattern, escape it with a backslash:

```sql
SELECT * FROM coupons WHERE code LIKE '10\%OFF';
-- Matches rows where code is literally '10%OFF'
```

## Combining Multiple Conditions

Here is an example combining several operators in a realistic query:

```sql
SELECT id, name, price, category
FROM products
WHERE
    category IN ('Electronics', 'Computers')
    AND price BETWEEN 100.00 AND 1000.00
    AND name LIKE '%Pro%'
    AND discontinued IS NULL
ORDER BY price ASC;
```

This query finds non-discontinued electronics or computers with "Pro" in the name, priced between $100 and $1000, sorted by price.

## Practice Problems

### Problem 1
Write a query to find all orders placed in the year 2023 that are neither 'cancelled' nor 'refunded', and have a total greater than $500.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded') NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO orders (customer_id, status, total, order_date) VALUES
    (1, 'delivered', 750.00, '2023-03-15'),
    (2, 'cancelled', 200.00, '2023-05-20'),
    (1, 'delivered', 1200.00, '2023-11-01'),
    (3, 'refunded', 600.00, '2023-07-10'),
    (2, 'shipped', 450.00, '2023-09-05'),
    (4, 'delivered', 85.00, '2024-01-20');
```

**Your query:**
```sql
-- Write the SELECT statement here.
```

**Solution:**
```sql
SELECT id, customer_id, status, total, order_date
FROM orders
WHERE order_date BETWEEN '2023-01-01' AND '2023-12-31'
  AND status NOT IN ('cancelled', 'refunded')
  AND total > 500.00
ORDER BY order_date;
```

**Explanation:**
`BETWEEN '2023-01-01' AND '2023-12-31'` captures all dates in 2023, inclusive. An alternative is `YEAR(order_date) = 2023`, but that form cannot use an index on `order_date`, while the `BETWEEN` form can. `NOT IN ('cancelled', 'refunded')` is cleaner than writing `status != 'cancelled' AND status != 'refunded'`. The total > 500 condition filters by amount. The result returns only the two matching delivered orders for customer_id 1.

### Problem 2
Write a query to find all employees whose name contains 'an' (case-insensitive), whose manager is unknown (NULL), and who were hired before 2020.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    manager_id INT UNSIGNED,
    hire_date DATE NOT NULL,
    salary DECIMAL(10, 2)
);

INSERT INTO employees (name, manager_id, hire_date, salary) VALUES
    ('Brandon Lee', NULL, '2017-03-01', 75000.00),
    ('Diana Prince', 1, '2019-08-15', 85000.00),
    ('Marcus Webb', NULL, '2016-06-20', 90000.00),
    ('Anna Chen', NULL, '2021-02-10', 70000.00),
    ('Stanley Kim', 1, '2015-11-30', 95000.00);
```

**Your query:**
```sql
-- Write the SELECT statement here.
```

**Solution:**
```sql
SELECT id, name, manager_id, hire_date
FROM employees
WHERE name LIKE '%an%'
  AND manager_id IS NULL
  AND hire_date < '2020-01-01';
```

**Explanation:**
All three conditions apply simultaneously via `AND`. `LIKE '%an%'` matches names containing the substring 'an': Brandon (Br-**an**-don), Diana (Di-**an**-a), Anna (**An**na), Stanley (St-**an**-ley) all match; Marcus (M-a-r-c-u-s) does not. Of those four, `manager_id IS NULL` eliminates Diana (manager_id = 1) and Stanley (manager_id = 1), leaving Brandon, and Anna. `hire_date < '2020-01-01'` then eliminates Anna (hired 2021-02-10). The correct result is Brandon Lee only. Notice how combining conditions narrows the result significantly.

### Problem 3
Demonstrate the `NOT IN` null trap: write two queries against a table with a `manager_id` column that includes NULL values. The first uses `NOT IN` with a subquery that returns NULLs, and the second uses the correct alternative. Explain why the first returns no rows.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED PRIMARY KEY,
    name VARCHAR(100),
    manager_id INT UNSIGNED
);

INSERT INTO employees VALUES
    (1, 'CEO', NULL),
    (2, 'VP', 1),
    (3, 'Manager', 1),
    (4, 'Staff', 2);
```

**Your query:**
```sql
-- Query 1: NOT IN with a subquery that can return NULL
-- Query 2: Correct alternative
```

**Solution:**
```sql
-- Query 1: Broken -- returns 0 rows because manager_id includes NULL
SELECT name FROM employees
WHERE id NOT IN (SELECT manager_id FROM employees);
-- Returns: (empty) -- because the subquery returns {NULL, 1}, and
-- NOT IN a list containing NULL is always NULL (falsy).

-- Query 2: Correct using NOT EXISTS
SELECT name FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM employees m WHERE m.id = e.id AND m.manager_id IS NOT NULL
      -- Alternative: filter NULLs from the subquery
);

-- Cleaner alternative: filter NULLs from the subquery directly
SELECT name FROM employees
WHERE id NOT IN (
    SELECT manager_id FROM employees WHERE manager_id IS NOT NULL
);
-- Returns: CEO, Staff -- employees whose id is not any manager's id
```

**Explanation:**
When a `NOT IN` subquery returns any `NULL` value, the entire `NOT IN` condition evaluates to `NULL` (not `TRUE`) for every row in the outer query. This is because `x NOT IN (NULL, 1)` is equivalent to `x != NULL AND x != 1`, and `x != NULL` is always `NULL`. The fix is to add `WHERE manager_id IS NOT NULL` to the subquery so that no NULLs are returned. Alternatively, use `NOT EXISTS` with a correlated subquery, which handles NULLs correctly by nature. This is one of the most common and hard-to-notice bugs in SQL, so always inspect `NOT IN` subqueries for potential NULLs.
