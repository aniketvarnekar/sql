# Subqueries

A subquery is a `SELECT` statement nested inside another SQL statement. Subqueries allow you to express complex data retrieval logic by using the result of one query as input to another. They appear in `WHERE`, `FROM`, `SELECT`, and other clauses, each with different behavior and performance characteristics.

## Subquery in WHERE

The most common use of a subquery is in the `WHERE` clause to filter rows based on values computed from another table or query:

```sql
-- Find customers who have placed at least one order:
SELECT id, name FROM customers
WHERE id IN (SELECT DISTINCT customer_id FROM orders);

-- Find products more expensive than the average price:
SELECT name, price FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Find the employee with the highest salary:
SELECT name, salary FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);
```

A subquery in `WHERE` that returns a single value (one row, one column) is called a scalar subquery. A subquery that returns a list of values (one column, many rows) is used with `IN`, `NOT IN`, `ANY`, or `ALL`.

The `ANY` and `ALL` operators compare a value against a set returned by a subquery:

```sql
-- Price greater than any product in the 'Books' category:
SELECT name, price FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'Books');

-- Price greater than all products in the 'Books' category (i.e., greater than the max):
SELECT name, price FROM products
WHERE price > ALL (SELECT price FROM products WHERE category = 'Books');
```

## Subquery in FROM (Derived Table)

A subquery in the `FROM` clause creates a temporary, inline result set called a derived table. The derived table must be given an alias:

```sql
SELECT dept_summary.department, dept_summary.avg_salary
FROM (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) AS dept_summary
WHERE dept_summary.avg_salary > 70000;
```

Derived tables are useful when you need to filter on the result of an aggregate (which you cannot do in a single-level `WHERE`), or when you want to break a complex query into logical steps. In MySQL 8.x, CTEs (covered in the CTEs and Recursive Queries file) are usually cleaner than deeply nested derived tables.

## Subquery in SELECT

A scalar subquery can appear in the `SELECT` list to compute a value for each row:

```sql
SELECT
    p.name,
    p.price,
    (SELECT AVG(price) FROM products) AS catalog_avg,
    p.price - (SELECT AVG(price) FROM products) AS diff_from_avg
FROM products p;
```

Subqueries in `SELECT` must return exactly one row and one column (scalar). The subquery in the example above runs once for the entire query because it doesn't reference the outer query's rows — MySQL's optimizer can recognize this and cache the result.

## Correlated Subqueries

A correlated subquery references a column from the outer query. It is re-evaluated for every row in the outer query, making it functionally similar to a loop:

```sql
-- Find employees who earn more than the average salary in their department:
SELECT name, department, salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department = e.department
);
```

The inner query (`SELECT AVG(salary) FROM employees WHERE department = e.department`) references `e.department` from the outer query. MySQL evaluates this subquery once per row in the outer `employees` table.

Correlated subqueries are expressive but can be slow on large tables because they execute once per outer row. They can often be rewritten as a join or CTE for better performance:

```sql
-- Equivalent using a JOIN to a derived table:
SELECT e.name, e.department, e.salary
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_avg ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_salary;
```

The join version computes department averages once and then looks up the result per row, which is substantially faster.

## EXISTS and NOT EXISTS

`EXISTS(subquery)` returns `TRUE` if the subquery produces at least one row, and `FALSE` if it produces no rows. It short-circuits — once the first matching row is found, MySQL stops evaluating the subquery:

```sql
-- Find customers who have placed at least one order:
SELECT name FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.id
);
```

The `SELECT 1` inside `EXISTS` is conventional — you don't care about the column values, only whether any rows exist. `SELECT *` or `SELECT any_column` would work equally well, but `SELECT 1` makes the intent clear.

`NOT EXISTS` is the negation: returns `TRUE` if the subquery produces no rows:

```sql
-- Find customers who have never placed an order:
SELECT name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.id
);
```

`NOT EXISTS` handles `NULL` correctly, unlike `NOT IN`. If the subquery might return `NULL` values, prefer `NOT EXISTS` over `NOT IN`.

## Subqueries vs Joins vs CTEs

Subqueries, joins, and CTEs can often express the same logic. The choice affects readability and sometimes performance:

- Use a **JOIN** when combining rows from two tables where the relationship is straightforward and you need columns from both tables in the output.
- Use a **subquery in WHERE** when filtering based on values from another table but you don't need columns from that table in the output.
- Use a **derived table** or **CTE** when you need to compute an intermediate result that is then queried further.
- Use a **correlated subquery** sparingly — it is expressive but can be slow; prefer a join to a derived table or CTE when performance matters.

## Practice Problems

### Problem 1
Write a query that returns the names of all products whose price is above the average price of all products in the same category.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(100) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);

INSERT INTO products (name, category, price) VALUES
    ('Widget Basic', 'Widgets', 5.99),
    ('Widget Pro', 'Widgets', 12.99),
    ('Widget Ultra', 'Widgets', 8.49),
    ('Gadget Lite', 'Gadgets', 19.99),
    ('Gadget Pro', 'Gadgets', 49.99),
    ('Gadget Max', 'Gadgets', 34.99);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
-- Using correlated subquery:
SELECT name, category, price
FROM products p
WHERE price > (
    SELECT AVG(price) FROM products WHERE category = p.category
);

-- More efficient using JOIN:
SELECT p.name, p.category, p.price
FROM products p
JOIN (
    SELECT category, AVG(price) AS avg_price
    FROM products
    GROUP BY category
) cat_avg ON p.category = cat_avg.category
WHERE p.price > cat_avg.avg_price;
```

**Explanation:**
The correlated subquery approach evaluates the average for each product's category once per product row — if there are 1000 products in 10 categories, the inner query runs 1000 times. The join approach computes the average for each category once (10 executions) and then joins it back to the products, which is more efficient for large datasets. Both approaches return the same result: Widget Pro (12.99 vs Widgets average 9.16) and Gadget Pro (49.99 vs Gadgets average 34.99). Widget Ultra (8.49) is below the Widgets average and is excluded. Gadget Lite (19.99) and Gadget Max (34.99) are both at or below the Gadgets average of 34.99 — because the query uses strict `>`, Gadget Max at exactly the average is also excluded.

### Problem 2
Write a query using `EXISTS` that finds all categories that have at least one product with a price below $10. Then write the equivalent query using `IN` with a subquery.

**Schema used:**
```sql
CREATE TABLE categories (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    category_id INT UNSIGNED NOT NULL,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);

INSERT INTO categories (id, name) VALUES (1, 'Widgets'), (2, 'Gadgets'), (3, 'Tools');
INSERT INTO products (category_id, name, price) VALUES
    (1, 'Widget Basic', 5.99),
    (1, 'Widget Pro', 12.99),
    (2, 'Gadget Pro', 49.99),
    (3, 'Hammer', 8.99);
```

**Your query:**
```sql
-- Write both versions.
```

**Solution:**
```sql
-- Using EXISTS:
SELECT c.id, c.name
FROM categories c
WHERE EXISTS (
    SELECT 1 FROM products p
    WHERE p.category_id = c.id AND p.price < 10.00
);

-- Using IN:
SELECT id, name
FROM categories
WHERE id IN (
    SELECT DISTINCT category_id FROM products WHERE price < 10.00
);
```

**Explanation:**
Both queries return Widgets (Widget Basic at $5.99) and Tools (Hammer at $8.99). Gadgets is excluded because its only product costs $49.99. The `EXISTS` version short-circuits after finding the first matching product for each category, which can be faster when categories have many matching products. The `IN` version computes the full list of qualifying category IDs first, then checks each category against that list. The `DISTINCT` in the `IN` subquery is optional (since `IN` handles duplicates), but it reduces the list size that MySQL must scan.

### Problem 3
Write a query that returns, for each order, the order id, total, and a column showing how many other orders the same customer has placed (not counting the current order itself).

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO orders (customer_id, total, order_date) VALUES
    (1, 200.00, '2023-01-10'),
    (1, 350.00, '2023-03-15'),
    (2, 100.00, '2023-02-20'),
    (1, 175.00, '2023-06-01'),
    (3, 450.00, '2023-04-10'),
    (2, 280.00, '2023-05-25');
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    o.id AS order_id,
    o.customer_id,
    o.total,
    (
        SELECT COUNT(*)
        FROM orders other
        WHERE other.customer_id = o.customer_id
          AND other.id != o.id
    ) AS other_orders_by_customer
FROM orders o
ORDER BY o.customer_id, o.order_date;
```

**Explanation:**
The correlated subquery in the `SELECT` list runs once per row in the outer query. For each order `o`, it counts all other orders placed by the same customer (`other.customer_id = o.customer_id`) excluding the current order itself (`other.id != o.id`). Customer 1 has 3 orders, so each of their order rows shows `other_orders_by_customer = 2`. Customer 2 has 2 orders, so each shows 1. Customer 3 has 1 order, so theirs shows 0. This pattern is practical and expressive, though for very large tables a window function (`COUNT(*) OVER (PARTITION BY customer_id) - 1`) would be more efficient.
