# Aggregate Functions

Aggregate functions compute a single result from a set of rows. They are the foundation of analytical queries — answering questions like "how many orders were placed today", "what is the average product price", or "who are our top-spending customers". Understanding how these functions handle NULL values and interact with `GROUP BY` is critical for writing correct analytics queries.

## COUNT

`COUNT(*)` counts the total number of rows in the result set, including rows with NULL values in any column:

```sql
SELECT COUNT(*) AS total_orders FROM orders;
```

`COUNT(column_name)` counts only the rows where that column is not NULL:

```sql
SELECT COUNT(shipped_at) AS shipped_orders FROM orders;
-- Returns only the count of orders where shipped_at is not NULL
```

`COUNT(DISTINCT column_name)` counts unique non-NULL values:

```sql
SELECT COUNT(DISTINCT customer_id) AS unique_customers FROM orders;
```

The distinction between `COUNT(*)` and `COUNT(column)` is one of the most common sources of bugs in analytics queries. Use `COUNT(*)` when you want the total number of rows. Use `COUNT(column)` when you want the count of non-NULL values in a specific column.

## SUM

`SUM(column)` returns the sum of all non-NULL values in the column. If all values are NULL, `SUM` returns NULL (not 0):

```sql
SELECT SUM(total) AS revenue FROM orders WHERE status = 'completed';
```

Use `COALESCE(SUM(column), 0)` if you need 0 instead of NULL when the result set is empty:

```sql
SELECT COALESCE(SUM(total), 0) AS revenue FROM orders WHERE status = 'completed';
```

## AVG

`AVG(column)` returns the arithmetic mean of all non-NULL values:

```sql
SELECT AVG(price) AS average_price FROM products WHERE category = 'Electronics';
```

Like `SUM`, `AVG` ignores NULL values entirely — it does not count NULL as zero. This is usually the correct behavior: if a salary is not recorded (NULL), it should not drag down the average. But if a missing value should be treated as zero for your use case, use `AVG(COALESCE(column, 0))`.

## MIN and MAX

`MIN(column)` returns the smallest value. `MAX(column)` returns the largest value. Both work on numeric, string, and date columns, and both ignore NULL values:

```sql
SELECT MIN(price) AS cheapest, MAX(price) AS most_expensive FROM products;
SELECT MIN(order_date) AS first_order, MAX(order_date) AS latest_order FROM orders;
SELECT MIN(name) AS first_alphabetically FROM customers;
```

For strings, `MIN` and `MAX` use the column's collation for comparison (alphabetical ordering by default).

## NULL Behavior in Aggregates

All aggregate functions except `COUNT(*)` ignore NULL values. This is an important and often misunderstood behavior:

```sql
-- Sample data: salaries are (50000, 60000, NULL, 70000)
SELECT
    COUNT(*) AS total_rows,       -- 4 (counts all rows including NULL)
    COUNT(salary) AS non_null,    -- 3 (ignores the NULL row)
    SUM(salary) AS total,         -- 180000 (50000+60000+70000; ignores NULL)
    AVG(salary) AS average,       -- 60000 (180000/3, not 180000/4)
    MIN(salary) AS minimum,       -- 50000
    MAX(salary) AS maximum        -- 70000
FROM employees;
```

The `AVG` divides by 3, not 4, because the NULL value is not counted in the denominator. Whether this is the correct behavior depends on your domain: for salaries, excluding unknown values is typically correct; for sensor readings where a missing value means zero, you should substitute 0 with `COALESCE`.

## DISTINCT Inside Aggregates

The `DISTINCT` modifier can be used inside aggregate functions to operate on unique values only:

```sql
SELECT COUNT(DISTINCT product_id) FROM order_items;
-- Counts unique products that have ever been ordered

SELECT SUM(DISTINCT price) FROM products;
-- Sums each unique price exactly once (rarely useful, but valid)

SELECT AVG(DISTINCT salary) FROM employees;
-- Averages unique salary values (employees with the same salary are counted once)
```

## Using Multiple Aggregates Together

A single `SELECT` can include multiple aggregate functions:

```sql
SELECT
    COUNT(*) AS total_products,
    COUNT(DISTINCT category) AS unique_categories,
    ROUND(AVG(price), 2) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    SUM(stock_quantity) AS total_stock
FROM products
WHERE discontinued = 0;
```

## Aggregates Without GROUP BY

When aggregate functions are used without `GROUP BY`, the entire result set is treated as one group and a single row is returned:

```sql
SELECT COUNT(*), SUM(total), AVG(total) FROM orders;
-- Returns a single row with the aggregates for all orders
```

If you mix aggregate functions with non-aggregate column references without a `GROUP BY`, MySQL's behavior depends on whether `ONLY_FULL_GROUP_BY` mode is enabled (it is, by default in MySQL 8). In this mode, MySQL raises an error if a non-aggregate column in the `SELECT` list is not in the `GROUP BY` clause.

## Practice Problems

### Problem 1
Write a query that returns the total number of orders, the number of completed orders, the total revenue from completed orders, the average order value (for all orders), and the highest single order total.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    status ENUM('pending', 'completed', 'cancelled') NOT NULL,
    total DECIMAL(10, 2) NOT NULL
);

INSERT INTO orders (customer_id, status, total) VALUES
    (1, 'completed', 250.00),
    (2, 'cancelled', 100.00),
    (1, 'completed', 750.00),
    (3, 'pending', 175.00),
    (2, 'completed', 430.00),
    (4, 'cancelled', 90.00);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed_orders,
    COALESCE(SUM(CASE WHEN status = 'completed' THEN total END), 0) AS completed_revenue,
    ROUND(AVG(total), 2) AS avg_order_value,
    MAX(total) AS highest_order
FROM orders;
```

**Explanation:**
`COUNT(*)` counts all 6 orders. `COUNT(CASE WHEN status = 'completed' THEN 1 END)` uses a conditional expression inside `COUNT` — the `CASE` returns 1 for completed orders and NULL for all others, and since `COUNT` ignores NULLs, only completed rows are counted. `SUM(CASE WHEN ...)` applies the same pattern to sum only completed order totals. `COALESCE(..., 0)` guards against the case where no completed orders exist. `AVG(total)` includes all orders. `MAX(total)` finds the single highest total.

### Problem 2
Write a query that returns the number of distinct customers who have placed at least one order, the total number of order line items across all orders, and the average quantity per line item.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL
);

CREATE TABLE order_items (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id INT UNSIGNED NOT NULL,
    product_id INT UNSIGNED NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(8, 2) NOT NULL
);

INSERT INTO orders (customer_id) VALUES (1), (2), (1), (3);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 10, 2, 9.99),
    (1, 11, 1, 24.99),
    (2, 10, 3, 9.99),
    (3, 12, 1, 49.99),
    (4, 11, 2, 24.99),
    (4, 13, 4, 4.99);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    COUNT(oi.id) AS total_line_items,
    ROUND(AVG(oi.quantity), 2) AS avg_quantity_per_line
FROM orders o
JOIN order_items oi ON oi.order_id = o.id;
```

**Explanation:**
The join brings `order_items` and `orders` together. `COUNT(DISTINCT o.customer_id)` counts unique customers who have at least one line item — because of the join, customers without orders don't appear. `COUNT(oi.id)` counts the total number of line items (6 in this data). `AVG(oi.quantity)` computes the average quantity across all line items. All three aggregates work on the same joined result set, so a single query computes all three efficiently.

### Problem 3
Write a query that finds the product with the highest total quantity sold (the sum of `quantity` from `order_items`). Return only the top product with its name and total sold quantity.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE order_items (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id INT UNSIGNED NOT NULL,
    product_id INT UNSIGNED NOT NULL,
    quantity INT NOT NULL
);

INSERT INTO products (id, name) VALUES (1, 'Widget'), (2, 'Gadget'), (3, 'Doohickey');
INSERT INTO order_items (order_id, product_id, quantity) VALUES
    (1, 1, 3), (1, 2, 1), (2, 1, 5), (3, 3, 2), (4, 2, 4), (5, 1, 2);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT p.name, SUM(oi.quantity) AS total_sold
FROM products p
JOIN order_items oi ON oi.product_id = p.id
GROUP BY p.id, p.name
ORDER BY total_sold DESC
LIMIT 1;
```

**Explanation:**
The join connects each line item to its product. `GROUP BY p.id, p.name` groups all line items by product, and `SUM(oi.quantity)` sums the quantities within each group. `ORDER BY total_sold DESC` puts the highest seller first, and `LIMIT 1` returns only that single row. Widget (product id 1) has quantities 3+5+2 = 10 and is the top seller. Including `p.id` in `GROUP BY` along with `p.name` is the correct practice — it guarantees uniqueness even if two products somehow shared a name, and satisfies MySQL's `ONLY_FULL_GROUP_BY` requirement.
