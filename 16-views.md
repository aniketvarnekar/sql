# Views

A view is a named, stored SQL query that behaves like a virtual table. When you query a view, MySQL executes the underlying `SELECT` statement and returns the result as if it were a table. Views do not store data themselves — they are simply saved query definitions. They simplify complex queries, provide a layer of abstraction over the underlying schema, and can be used for access control.

## Creating a View

```sql
CREATE VIEW active_customers AS
SELECT id, name, email
FROM customers
WHERE deleted_at IS NULL;
```

Once created, you query the view exactly like a table:

```sql
SELECT * FROM active_customers;
SELECT name, email FROM active_customers WHERE name LIKE 'A%';
```

Every query against the view is transparently rewritten to query the underlying table with the view's `SELECT` definition merged in. There is no separate copy of the data.

To replace an existing view with a new definition:

```sql
CREATE OR REPLACE VIEW active_customers AS
SELECT id, name, email, created_at
FROM customers
WHERE deleted_at IS NULL;
```

## ALTER VIEW and DROP VIEW

To modify an existing view:

```sql
ALTER VIEW active_customers AS
SELECT id, name, email, phone
FROM customers
WHERE deleted_at IS NULL AND is_verified = 1;
```

`ALTER VIEW` is equivalent to `CREATE OR REPLACE VIEW`. Both are common in practice.

To remove a view:

```sql
DROP VIEW active_customers;
DROP VIEW IF EXISTS active_customers;
```

Dropping a view only removes the view definition — the underlying table and its data are not affected.

## Views for Complex Query Simplification

Views excel at hiding multi-table join logic behind a simple, reusable name. Instead of requiring every developer to write a complex join correctly, you define it once in a view:

```sql
CREATE VIEW order_summary AS
SELECT
    o.id AS order_id,
    c.name AS customer_name,
    c.email AS customer_email,
    o.status,
    o.total,
    o.order_date,
    COUNT(oi.id) AS item_count
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, c.name, c.email, o.status, o.total, o.order_date;
```

Developers can now write:

```sql
SELECT customer_name, total FROM order_summary WHERE status = 'completed';
```

instead of rewriting the join every time.

## Updatable Views

Some views are updatable — you can run `INSERT`, `UPDATE`, and `DELETE` against them, and MySQL will translate the operation to the underlying table. A view is updatable when it meets all of the following conditions:

- It queries a single table (no joins)
- It does not use `DISTINCT`
- It does not use aggregate functions
- It does not use `GROUP BY` or `HAVING`
- It does not use `UNION`
- It does not contain a subquery in the `SELECT` list or `FROM` clause
- No column in the view is derived from an expression (computed columns are not updatable)

```sql
CREATE VIEW active_users AS
SELECT id, username, email FROM users WHERE is_active = 1;

-- This INSERT goes through to the users table:
INSERT INTO active_users (username, email) VALUES ('alice', 'alice@example.com');

-- This UPDATE modifies the underlying users table:
UPDATE active_users SET email = 'new@example.com' WHERE id = 42;
```

If a view is not updatable (for example, because it joins tables or uses aggregates), any attempt to `INSERT`, `UPDATE`, or `DELETE` through it raises an error.

## WITH CHECK OPTION

`WITH CHECK OPTION` ensures that any `INSERT` or `UPDATE` through an updatable view satisfies the view's `WHERE` condition. Without it, you could insert a row through a view that the view itself would then not show:

```sql
CREATE VIEW active_users AS
SELECT id, username, email, is_active
FROM users
WHERE is_active = 1
WITH CHECK OPTION;

-- This INSERT would fail because is_active = 0 violates the view's condition:
INSERT INTO active_users (username, email, is_active)
VALUES ('bob', 'bob@example.com', 0);
-- Error: CHECK OPTION failed for 'active_users'

-- This succeeds:
INSERT INTO active_users (username, email, is_active)
VALUES ('carol', 'carol@example.com', 1);
```

`WITH CHECK OPTION` is a safety mechanism that keeps views behaviorally consistent — rows you insert or update through the view remain visible in the view.

## Views for Access Control

Views provide a simple form of column-level and row-level access control. By granting a user access to a view rather than the underlying table, you can restrict which columns and rows they see:

```sql
-- Create a view that exposes only non-sensitive employee data:
CREATE VIEW employee_public_info AS
SELECT id, first_name, last_name, department, title
FROM employees;
-- Salary, SSN, and other sensitive columns are not included.

-- Grant access to the view only, not the underlying table:
GRANT SELECT ON company.employee_public_info TO 'hr_reader'@'%';
```

This pattern allows you to give read access to a subset of data without exposing the full table structure or sensitive columns.

## Performance Considerations

In MySQL, most views are not materialized — they are expanded inline at query time. This means querying a view is functionally identical to running the view's `SELECT` with your additional conditions merged in. The MySQL optimizer sees the combined query and can use indexes from the underlying tables.

However, complex views — especially those with aggregations, subqueries, or derived tables — may not be optimizable. In some cases, MySQL must fully execute the view's query before applying your outer filters, which is called an unmerged view or a derived table view. This can lead to performance problems. Use `EXPLAIN` to check whether MySQL is merging the view into the outer query or treating it as a separate derived table.

MySQL 8.x handles many views better than MySQL 5.x due to improved query optimization. The `information_schema.views` table shows all views and their definitions:

```sql
SELECT table_name, view_definition FROM information_schema.views
WHERE table_schema = 'your_database_name';
```

## Practice Problems

### Problem 1
Create a view called `high_value_orders` that shows orders with a total above $500. Include the order id, customer name (from a joined customers table), total, status, and order date. Then query it to find all pending high-value orders.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    status VARCHAR(20) NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO customers (name) VALUES ('Alice'), ('Bob'), ('Carol');
INSERT INTO orders (customer_id, status, total, order_date) VALUES
    (1, 'pending', 750.00, '2024-01-10'),
    (2, 'completed', 200.00, '2024-01-15'),
    (1, 'completed', 1200.00, '2024-02-01'),
    (3, 'pending', 350.00, '2024-02-10'),
    (2, 'pending', 620.00, '2024-02-20');
```

**Your query:**
```sql
-- Write the CREATE VIEW and the query against it.
```

**Solution:**
```sql
CREATE VIEW high_value_orders AS
SELECT
    o.id AS order_id,
    c.name AS customer_name,
    o.total,
    o.status,
    o.order_date
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.total > 500.00;

-- Query the view:
SELECT order_id, customer_name, total, order_date
FROM high_value_orders
WHERE status = 'pending'
ORDER BY total DESC;
```

**Explanation:**
The view encapsulates the join between `orders` and `customers` and the `WHERE total > 500` filter. The query against the view adds a further filter for `status = 'pending'`. MySQL merges these conditions — the actual executed query is effectively a join with `WHERE o.total > 500 AND o.status = 'pending'`. The view returns orders 1 ($750, Alice, pending) and 5 ($620, Bob, pending). The completed $1200 order does not appear because it is not pending.

### Problem 2
Create an updatable view called `active_products` that shows only products where `is_active = 1`. Add `WITH CHECK OPTION`, then demonstrate what happens when you try to insert a product with `is_active = 0` through the view.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL,
    is_active TINYINT(1) NOT NULL DEFAULT 1
);

INSERT INTO products (name, price, is_active) VALUES
    ('Widget', 9.99, 1),
    ('Old Widget', 4.99, 0);
```

**Your query:**
```sql
-- Create the view with WITH CHECK OPTION, then try both inserts.
```

**Solution:**
```sql
CREATE VIEW active_products AS
SELECT id, name, price, is_active
FROM products
WHERE is_active = 1
WITH CHECK OPTION;

-- This succeeds (is_active = 1 satisfies the view's WHERE clause):
INSERT INTO active_products (name, price, is_active) VALUES ('New Widget', 14.99, 1);

-- This fails (is_active = 0 violates WITH CHECK OPTION):
INSERT INTO active_products (name, price, is_active) VALUES ('Inactive Widget', 7.99, 0);
-- Error: CHECK OPTION failed for 'active_products'

-- Query to verify only active products appear:
SELECT * FROM active_products;
```

**Explanation:**
`WITH CHECK OPTION` ensures that any data modification through the view satisfies the view's filter condition. The first insert succeeds because `is_active = 1` matches `WHERE is_active = 1`. The second insert fails because `is_active = 0` would violate the view's `WHERE` clause — the newly inserted row would not be visible through the view, which `WITH CHECK OPTION` prevents. This enforces that the view remains a consistent, trustworthy window into the active products.

### Problem 3
Demonstrate how to use a view for access control by creating a view that exposes only non-sensitive columns from an `employees` table, and explain how you would grant access to it.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    department VARCHAR(100) NOT NULL,
    salary DECIMAL(10, 2) NOT NULL,
    ssn CHAR(11) NOT NULL,
    email VARCHAR(255) NOT NULL
);
```

**Your query:**
```sql
-- Create the restricted view and show the GRANT statement.
```

**Solution:**
```sql
-- Create a view that omits salary and SSN:
CREATE VIEW employee_directory AS
SELECT id, first_name, last_name, department, email
FROM employees;

-- Grant read access to the view for a specific user:
-- (Run as a privileged MySQL user)
GRANT SELECT ON company.employee_directory TO 'directory_reader'@'%';

-- The user can query the view:
SELECT * FROM employee_directory WHERE department = 'Engineering';

-- But they cannot access the underlying table directly:
-- SELECT salary FROM employees; -- Error: access denied
```

**Explanation:**
By granting `SELECT` on the view rather than on the `employees` table, the `directory_reader` user can query names, departments, and emails but cannot see salaries or Social Security numbers. MySQL enforces this at the privilege level: the user simply has no access to the `employees` table and can only see what the view exposes. This is a simple but effective form of column-level access control that requires no changes to application code — the restriction is enforced by the database.
