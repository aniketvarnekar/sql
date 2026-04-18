# Updating and Deleting

The `UPDATE` and `DELETE` statements modify and remove existing data. They are the most dangerous statements in everyday SQL use: unlike a bad `SELECT`, a bad `UPDATE` or `DELETE` can silently destroy data across an entire table. Writing them carefully and safely is one of the most important habits to develop.

## UPDATE

`UPDATE` modifies the values of one or more columns in one or more rows:

```sql
UPDATE products SET price = 14.99 WHERE id = 42;
```

The `SET` clause specifies the column-value pairs to change. The `WHERE` clause restricts which rows are affected. Multiple columns can be updated in a single statement:

```sql
UPDATE products
SET price = 14.99, updated_at = NOW()
WHERE id = 42;
```

You can update values relative to their current value:

```sql
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE id = 42;
UPDATE accounts SET balance = balance + 500.00 WHERE account_id = 101;
```

### Updating Multiple Rows

When the `WHERE` clause matches multiple rows, all matching rows are updated:

```sql
UPDATE products SET discount_percent = 10 WHERE category = 'Books';
```

This is powerful and intentional — you might want to apply a discount to all books at once. But it means that an incorrect `WHERE` clause can silently update many more rows than intended.

### The Danger of Omitting WHERE

An `UPDATE` without a `WHERE` clause modifies every row in the table:

```sql
-- DANGEROUS: Sets price to 0.00 for every product in the table
UPDATE products SET price = 0.00;
```

This is rarely intentional. Before running any `UPDATE`, ask yourself: does the `WHERE` clause correctly identify only the rows I want to change? One safe practice is to run the equivalent `SELECT` first to verify which rows the condition matches:

```sql
-- First, verify with SELECT:
SELECT id, name, price FROM products WHERE category = 'Books';

-- Then, update:
UPDATE products SET discount_percent = 10 WHERE category = 'Books';
```

## DELETE

`DELETE` removes rows from a table:

```sql
DELETE FROM orders WHERE status = 'cancelled' AND order_date < '2022-01-01';
```

Unlike `TRUNCATE`, `DELETE` removes rows one at a time and fires any `DELETE` triggers defined on the table. `DELETE` can be rolled back inside a transaction.

### The Danger of Omitting WHERE in DELETE

A `DELETE` without a `WHERE` clause removes every row in the table:

```sql
-- DANGEROUS: Removes every row from orders
DELETE FROM orders;
```

This is equivalent to `TRUNCATE TABLE orders` in terms of effect (all rows gone), but slower and transaction-safe. The same verification habit applies: run a `SELECT` with the same `WHERE` clause before executing the `DELETE`.

## Safe Update Mode

MySQL has a built-in safety feature called safe update mode, controlled by the `sql_safe_updates` system variable. When enabled, MySQL rejects `UPDATE` and `DELETE` statements that don't include a `WHERE` clause using a key column (primary key or indexed column):

```sql
-- Enable safe update mode for the current session:
SET SESSION sql_safe_updates = 1;

-- This will now fail because 'category' may not be an indexed key:
UPDATE products SET discount_percent = 10 WHERE category = 'Books';
-- Error: You are using safe update mode and you tried to update a table
-- without a WHERE that uses a KEY column

-- This succeeds because id is the primary key:
UPDATE products SET discount_percent = 10 WHERE id = 5;
```

MySQL Workbench enables safe update mode by default. The CLI does not. You can add `SET sql_safe_updates = 1;` to the top of session scripts that run `UPDATE` or `DELETE` for an additional safety layer.

To disable for the current session when you genuinely need an unfiltered update or delete:

```sql
SET SESSION sql_safe_updates = 0;
```

## UPDATE with JOIN

MySQL supports joining tables in an `UPDATE` statement, which allows you to update rows in one table based on data in another:

```sql
UPDATE orders o
JOIN customers c ON o.customer_id = c.id
SET o.discount_percent = 15
WHERE c.tier = 'VIP';
```

This updates the `discount_percent` in `orders` for all orders belonging to VIP customers, without needing a subquery.

## DELETE with JOIN

Similarly, `DELETE` supports joins in MySQL:

```sql
DELETE o
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.is_deleted = TRUE;
```

Note the syntax: the table to delete from (`o`) is specified between `DELETE` and `FROM`. Only rows from `orders` are deleted, not from `customers`.

## Soft Deletes

Physically deleting rows from a database can cause problems: lost audit history, broken foreign key references, and unrecoverable mistakes. Many applications use "soft deletes" instead — marking a row as deleted without physically removing it:

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP NULL;

-- Soft delete: mark as deleted
UPDATE users SET deleted_at = NOW() WHERE id = 42;

-- Query active users only:
SELECT * FROM users WHERE deleted_at IS NULL;

-- Restore a soft-deleted user:
UPDATE users SET deleted_at = NULL WHERE id = 42;
```

Soft deletes add complexity to every query (you must always filter on `deleted_at IS NULL`) but preserve data history and allow recovery from accidental deletions.

## LIMIT in UPDATE and DELETE

MySQL allows adding `LIMIT` to `UPDATE` and `DELETE` to cap the number of rows affected in a single statement. This is useful for deleting large amounts of data in chunks to avoid long-running transactions and table locks:

```sql
-- Delete 1000 old orders at a time
DELETE FROM orders WHERE status = 'cancelled' LIMIT 1000;

-- Run repeatedly in a loop until 0 rows are affected
```

Chunked deletes are a standard practice in production databases where deleting millions of rows in a single statement would hold locks for an unacceptable duration.

## Practice Problems

### Problem 1
A product's price needs to increase by 5% and its `updated_at` field needs to be set to the current time. Write the `UPDATE` statement, and also write the `SELECT` you would run first to verify the correct row is targeted.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

INSERT INTO products (name, price) VALUES
    ('Widget Pro', 99.99),
    ('Widget Lite', 49.99),
    ('Gadget Max', 149.99);
```

**Your query:**
```sql
-- Write the verification SELECT and then the UPDATE.
```

**Solution:**
```sql
-- Step 1: Verify which row will be affected
SELECT id, name, price FROM products WHERE name = 'Widget Pro';

-- Step 2: Perform the update
UPDATE products
SET price = ROUND(price * 1.05, 2)
WHERE name = 'Widget Pro';

-- Step 3: Confirm the change
SELECT id, name, price, updated_at FROM products WHERE name = 'Widget Pro';
```

**Explanation:**
Running the `SELECT` first confirms you are targeting the right row before making any change. `ROUND(price * 1.05, 2)` increases the price by 5% and rounds to 2 decimal places to avoid floating-point artifacts in a `DECIMAL` calculation. The `updated_at` column uses `ON UPDATE CURRENT_TIMESTAMP`, so MySQL updates it automatically whenever the row changes — you don't need to include it explicitly in `SET`.

### Problem 2
Write a query that deletes all orders older than two years that have a status of 'cancelled'. First show the `SELECT` you would use to preview the affected rows.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    status ENUM('pending', 'completed', 'cancelled') NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO orders (customer_id, status, total, order_date) VALUES
    (1, 'completed', 250.00, '2021-03-10'),
    (2, 'cancelled', 100.00, '2020-11-15'),
    (1, 'cancelled', 75.00, '2022-01-20'),
    (3, 'completed', 400.00, '2023-06-05'),
    (2, 'cancelled', 50.00, '2019-08-01');
```

**Your query:**
```sql
-- Write the preview SELECT and the DELETE.
```

**Solution:**
```sql
-- Preview:
SELECT id, customer_id, status, order_date
FROM orders
WHERE status = 'cancelled'
  AND order_date < DATE_SUB(CURDATE(), INTERVAL 2 YEAR);

-- Delete:
DELETE FROM orders
WHERE status = 'cancelled'
  AND order_date < DATE_SUB(CURDATE(), INTERVAL 2 YEAR);
```

**Explanation:**
`DATE_SUB(CURDATE(), INTERVAL 2 YEAR)` calculates the date exactly two years before today, making the condition dynamic — it will always target orders older than two years at the time the query runs. Running the `SELECT` with the same `WHERE` clause first shows exactly which rows will be deleted, allowing you to confirm the count and sample the data before committing to the deletion.

### Problem 3
Implement a soft delete for a `users` table. Write the `UPDATE` to soft-delete user id 5, then write the query that retrieves all active (non-deleted) users, and finally write the query that restores the soft-deleted user.

**Schema used:**
```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    deleted_at TIMESTAMP NULL DEFAULT NULL
);

INSERT INTO users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com'),
    ('carol', 'carol@example.com'),
    ('dave', 'dave@example.com'),
    ('eve', 'eve@example.com');
```

**Your query:**
```sql
-- Write three statements: soft delete, list active, restore.
```

**Solution:**
```sql
-- 1. Soft delete user id 5:
UPDATE users SET deleted_at = NOW() WHERE id = 5;

-- 2. List all active users:
SELECT id, username, email FROM users WHERE deleted_at IS NULL;

-- 3. Restore the soft-deleted user:
UPDATE users SET deleted_at = NULL WHERE id = 5;
```

**Explanation:**
A soft delete sets `deleted_at` to the current timestamp rather than removing the row. This preserves the user's data and history, making the deletion reversible. The `deleted_at IS NULL` filter in the active users query is the standard pattern — it must be added to every query that should only see active records. This is the main operational cost of soft deletes: every query that accesses the table needs to be aware of the soft-delete column. Many applications add a database view that pre-filters deleted rows, so that typical queries don't need to remember the filter.
