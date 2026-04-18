# Joins

A join combines rows from two or more tables based on a related column between them. Joins are the defining capability of relational databases — they allow you to model data across normalized tables and then reassemble it at query time with great flexibility. Understanding each type of join and when to use it is one of the most important SQL skills.

## Why Joins Exist

Data in a relational database is stored in separate tables to avoid duplication. An `orders` table stores order information but references customers by `customer_id`, not by the customer's name and address. A join lets you combine these tables so that a single query can return the order details alongside the customer's name — without needing to denormalize the data.

## INNER JOIN

`INNER JOIN` returns only the rows where a match exists in both tables. Rows that have no match in the other table are excluded:

```sql
SELECT o.id AS order_id, o.total, c.name AS customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;
```

The `ON` clause specifies the join condition — the column or expression that links the rows. `INNER JOIN` is the default join type and can be written as just `JOIN`:

```sql
SELECT o.id, c.name FROM orders o JOIN customers c ON o.customer_id = c.id;
```

Table aliases (`o` for orders, `c` for customers) are essential in join queries to keep the SQL readable and to disambiguate column names that appear in both tables.

## LEFT JOIN

`LEFT JOIN` (also written as `LEFT OUTER JOIN`) returns all rows from the left table and the matching rows from the right table. When there is no match in the right table, the right table's columns appear as `NULL`:

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

This returns every customer, including those who have never placed an order. For customers without orders, `order_id` and `total` will be `NULL`.

`LEFT JOIN` is the correct choice when you want "all records from A, with matching records from B if any exist."

## RIGHT JOIN

`RIGHT JOIN` (also written as `RIGHT OUTER JOIN`) is the mirror image of `LEFT JOIN` — all rows from the right table are returned, with matching rows from the left table or `NULL` where there is no match:

```sql
SELECT c.name, o.id AS order_id
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.id;
```

This produces the same result as the `LEFT JOIN` example above with the tables swapped. In practice, `RIGHT JOIN` is rarely used because any `RIGHT JOIN` can be rewritten as a `LEFT JOIN` by swapping the table positions. Use `LEFT JOIN` consistently and swap the table order when needed.

## FULL OUTER JOIN Workaround

MySQL does not support `FULL OUTER JOIN` directly. A full outer join returns all rows from both tables, with `NULL` on either side where there is no match. The standard workaround in MySQL uses `UNION` to combine a `LEFT JOIN` and a `RIGHT JOIN`:

```sql
SELECT c.name AS customer, o.id AS order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id

UNION

SELECT c.name AS customer, o.id AS order_id
FROM customers c
RIGHT JOIN orders o ON o.customer_id = c.id;
```

`UNION` removes duplicates; use `UNION ALL` if you know there are no overlapping rows (for performance).

## CROSS JOIN

`CROSS JOIN` produces the Cartesian product of two tables — every row from the first table is combined with every row from the second table. If table A has 5 rows and table B has 4 rows, the result has 20 rows:

```sql
SELECT colors.name AS color, sizes.name AS size
FROM colors
CROSS JOIN sizes;
```

`CROSS JOIN` is useful for generating all combinations of two sets — for example, all color-size combinations for a product. It is rarely used with large tables because the result set grows multiplicatively.

## Self JOIN

A self join joins a table to itself. This is useful for hierarchical data where rows reference other rows in the same table, such as an employee-manager relationship:

```sql
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

This query lists every employee and their manager's name. Employees at the top of the hierarchy (with `manager_id IS NULL`) return `NULL` for manager — hence the `LEFT JOIN` rather than `INNER JOIN`, which would exclude them.

## Joining More Than Two Tables

Joins chain naturally. Each additional `JOIN` clause extends the query to include another table:

```sql
SELECT
    o.id AS order_id,
    c.name AS customer,
    p.name AS product,
    oi.quantity,
    oi.unit_price
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'delivered';
```

This query joins four tables: `orders`, `customers`, `order_items`, and `products`. The join conditions follow the foreign key relationships in the schema. MySQL processes the joins in sequence, building up the intermediate result set with each join.

## ON vs USING

When the join condition compares columns with the same name in both tables, `USING(column_name)` is a shorthand alternative to `ON`:

```sql
-- Using ON:
SELECT o.id, c.name FROM orders o JOIN customers c ON o.customer_id = c.customer_id;

-- Using USING (when column name is identical in both tables):
SELECT o.id, c.name FROM orders o JOIN customers c USING (customer_id);
```

`USING` also deduplicates the named column in `SELECT *` — with `ON`, `SELECT *` would include `customer_id` twice (once from each table); with `USING`, it appears only once. In practice, `ON` is more common and more explicit about the join condition.

## Common Join Mistakes

Writing a join without a proper `ON` condition produces a Cartesian product, which is usually unintentional on large tables and returns rows in the millions:

```sql
-- Accidental Cartesian product (missing ON clause):
SELECT * FROM orders, customers;
-- Returns orders_count * customers_count rows
```

Using the old comma-separated table syntax from SQL-89 (`FROM table1, table2 WHERE table1.col = table2.col`) is functionally equivalent to an `INNER JOIN` but mixes the join condition into the `WHERE` clause, making queries harder to read and maintain. Always use explicit `JOIN` syntax.

## Practice Problems

### Problem 1
Write a query that returns the name of each customer and the number of orders they have placed. Include customers who have never placed an order (show 0 for their order count).

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL
);

INSERT INTO customers (name) VALUES ('Alice'), ('Bob'), ('Carol'), ('Dave');
INSERT INTO orders (customer_id, total) VALUES
    (1, 150.00), (1, 320.00), (2, 75.00), (3, 480.00), (3, 90.00);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name
ORDER BY order_count DESC;
```

**Explanation:**
`LEFT JOIN` ensures that Dave (who has no orders) appears in the result. When Dave is included, `o.id` is `NULL`, and `COUNT(o.id)` counts non-NULL values — so Dave correctly gets a count of 0. Using `COUNT(*)` instead would count the NULL row and return 1 for Dave, which would be wrong. Grouping by `c.id, c.name` groups by customer, and `COUNT(o.id)` counts the matching orders per customer.

### Problem 2
Using a self join, write a query that lists each employee's name alongside their direct manager's name. Employees without a manager (top-level executives) should still appear in the result, with NULL shown for their manager.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    manager_id INT UNSIGNED
);

INSERT INTO employees (id, name, manager_id) VALUES
    (1, 'CEO', NULL),
    (2, 'VP Engineering', 1),
    (3, 'VP Marketing', 1),
    (4, 'Senior Engineer', 2),
    (5, 'Engineer', 2),
    (6, 'Marketing Lead', 3);
```

**Your query:**
```sql
-- Write the self join SELECT here.
```

**Solution:**
```sql
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY m.name, e.name;
```

**Explanation:**
The self join uses the `employees` table twice with different aliases: `e` for the employee being listed and `m` for the manager referenced by `e.manager_id`. The `LEFT JOIN` ensures the CEO (whose `manager_id` is NULL) appears in the result, with `manager` as NULL. An `INNER JOIN` would exclude the CEO because there is no row in `employees` where `id = NULL`. The `ORDER BY` sorts by manager name then employee name, grouping each manager's direct reports together.

### Problem 3
Write a query that finds all customers who have never placed an order. Use a `LEFT JOIN` approach and also show the equivalent approach using `NOT EXISTS`.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) NOT NULL
);

CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL
);

INSERT INTO customers (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Carol', 'carol@example.com');
INSERT INTO orders (customer_id, total) VALUES (1, 200.00), (3, 400.00);
```

**Your query:**
```sql
-- Write both approaches.
```

**Solution:**
```sql
-- Approach 1: LEFT JOIN with NULL check
SELECT c.id, c.name, c.email
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;

-- Approach 2: NOT EXISTS
SELECT c.id, c.name, c.email
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

**Explanation:**
The `LEFT JOIN` approach works by joining all customers to their orders, then filtering to only the rows where the join produced no match (i.e., `o.id IS NULL`). This "anti-join" pattern is a common idiom. The `NOT EXISTS` approach is logically equivalent and sometimes more readable, especially when the exclusion logic is complex. Both return Bob, who has no orders. Performance-wise, both approaches tend to be optimized similarly by MySQL's query optimizer, but `NOT EXISTS` can occasionally be more efficient on very large datasets because it can short-circuit as soon as one matching row is found.
