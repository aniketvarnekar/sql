# Inserting Data

The `INSERT` statement adds new rows to a table. Mastering its various forms — single-row inserts, bulk inserts, conflict handling, and inserting from queries — is essential for loading and maintaining data efficiently.

## Basic INSERT Syntax

The standard form of `INSERT` specifies the table name, the list of columns to populate, and a matching list of values:

```sql
INSERT INTO books (title, author_id, price, published_date)
VALUES ('The Great Gatsby', 1, 12.99, '1925-04-10');
```

Always specify the column list explicitly. Omitting it forces you to supply values for every column in the exact order they were defined in the table, which makes the statement fragile — it breaks silently if someone later adds or reorders columns. With an explicit column list, columns not mentioned will receive their default value (or NULL if no default is defined and the column allows NULL).

For columns with `AUTO_INCREMENT`, omit the column from the list entirely or pass `NULL` or `DEFAULT` — MySQL will assign the next available value:

```sql
INSERT INTO books (title, price) VALUES ('Dune', 14.99);
-- id is AUTO_INCREMENT; it will be assigned automatically
```

To find the `AUTO_INCREMENT` value assigned by the most recent `INSERT` in the current session:

```sql
SELECT LAST_INSERT_ID();
```

## Inserting Multiple Rows

MySQL allows inserting multiple rows in a single statement by providing multiple value lists separated by commas:

```sql
INSERT INTO books (title, author_id, price) VALUES
    ('1984', 2, 10.99),
    ('Animal Farm', 2, 8.99),
    ('Brave New World', 3, 11.50);
```

Multi-row inserts are significantly faster than issuing one `INSERT` per row, particularly when inserting thousands of rows. MySQL can optimize the write as a batch operation, reducing overhead from transaction commits, index updates, and network round trips. For large data loads, bulk inserting in batches of 500–1000 rows per statement is a common and effective pattern.

## INSERT IGNORE

By default, `INSERT` raises an error if a unique constraint or primary key is violated. `INSERT IGNORE` suppresses the error and silently skips any rows that would cause a violation:

```sql
INSERT IGNORE INTO books (id, title, price)
VALUES (1, 'Duplicate Book', 9.99);
-- If id=1 already exists, this row is silently skipped with a warning.
```

Use `INSERT IGNORE` when you are importing data and want to skip duplicates without failing. Be aware that it suppresses all ignorable errors, not just duplicate key violations — it can hide other problems if you are not careful.

## ON DUPLICATE KEY UPDATE

`ON DUPLICATE KEY UPDATE` is a more precise alternative to `INSERT IGNORE`. Instead of skipping a duplicate row, it updates the existing row with new values:

```sql
INSERT INTO books (id, title, price)
VALUES (1, 'The Great Gatsby', 15.99)
ON DUPLICATE KEY UPDATE price = VALUES(price);
```

When the `INSERT` would violate a unique key constraint, MySQL instead performs the `UPDATE` specified in the clause. This is commonly called an "upsert" (insert or update). The `VALUES(column_name)` function references the value that was being inserted, which is the standard idiom for this pattern.

In MySQL 8.0.20+, the recommended syntax uses an alias for the new row instead of the deprecated `VALUES()` function:

```sql
INSERT INTO books (id, title, price)
VALUES (1, 'The Great Gatsby', 15.99) AS new_row
ON DUPLICATE KEY UPDATE price = new_row.price;
```

A common use case is maintaining a running count or updating a record to the latest version:

```sql
INSERT INTO page_views (page_id, view_count)
VALUES (42, 1)
ON DUPLICATE KEY UPDATE view_count = view_count + 1;
```

## INSERT ... SELECT

You can insert rows derived from a `SELECT` query. This is useful for copying data between tables, transforming data, or populating a summary table:

```sql
INSERT INTO archived_books (id, title, price, archived_at)
SELECT id, title, price, NOW()
FROM books
WHERE published_date < '2000-01-01';
```

The column order and types from the `SELECT` must match the column list in the `INSERT`. No `VALUES` clause is used when inserting from a `SELECT`.

A common pattern is using `INSERT ... SELECT` to copy an entire table:

```sql
CREATE TABLE books_backup LIKE books;
INSERT INTO books_backup SELECT * FROM books;
```

`CREATE TABLE ... LIKE` creates a new empty table with the same column definitions and indexes as the source table.

## Inserting with Expressions and Functions

Values in an `INSERT` can be expressions or function calls, not just literals:

```sql
INSERT INTO events (name, created_at, expires_at)
VALUES ('Summer Sale', NOW(), DATE_ADD(NOW(), INTERVAL 30 DAY));
```

```sql
INSERT INTO users (username, password_hash)
VALUES ('alice', SHA2('my_password', 256));
```

## Common Mistakes

Inserting a value into a `NOT NULL` column without a default will raise an error. In MySQL's strict SQL mode (enabled by default in MySQL 8.x via `ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,...`), implicit truncation, out-of-range values, and missing values for `NOT NULL` columns all produce errors rather than being silently adjusted. This is correct behavior — strict mode surfaces data quality problems rather than hiding them.

If you insert a string that is longer than the column's `VARCHAR` or `CHAR` length in strict mode, MySQL raises an error. In non-strict mode, it would silently truncate the value, which is a data loss bug disguised as success.

## Practice Problems

### Problem 1
Insert three new authors into an `authors` table and then insert two books for the first author using a single multi-row `INSERT` statement.

**Schema used:**
```sql
CREATE TABLE authors (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    birth_year YEAR
);

CREATE TABLE books (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    author_id INT UNSIGNED NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);
```

**Your query:**
```sql
-- Write the INSERT statements here.
```

**Solution:**
```sql
INSERT INTO authors (name, birth_year) VALUES
    ('George Orwell', 1903),
    ('Aldous Huxley', 1894),
    ('F. Scott Fitzgerald', 1896);

-- Assuming George Orwell was assigned id=1:
INSERT INTO books (title, author_id, price) VALUES
    ('1984', 1, 10.99),
    ('Animal Farm', 1, 8.49);
```

**Explanation:**
The multi-row `INSERT` for authors sends a single statement to MySQL, which is more efficient than three separate `INSERT` statements. The same applies to the books insert. If you need to know the `id` assigned to George Orwell before inserting his books, you would run `SELECT LAST_INSERT_ID()` after the author insert — but note that `LAST_INSERT_ID()` returns the id of the first row inserted in a multi-row insert, not the last. For the book insert, the `author_id` references the author's id, establishing the relationship.

### Problem 2
A `products` table tracks products with a unique `sku` column. Write an upsert that inserts a new product if its SKU does not exist, or updates the `price` and `stock_quantity` if it does.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sku VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0
);

INSERT INTO products (sku, name, price, stock_quantity)
VALUES ('ABC-123', 'Widget', 9.99, 100);
```

**Your query:**
```sql
-- Write an upsert for SKU 'ABC-123' that updates price to 11.99
-- and stock_quantity to 150 if it already exists.
```

**Solution:**
```sql
INSERT INTO products (sku, name, price, stock_quantity)
VALUES ('ABC-123', 'Widget', 11.99, 150) AS new_row
ON DUPLICATE KEY UPDATE
    price = new_row.price,
    stock_quantity = new_row.stock_quantity;

-- Verify:
SELECT * FROM products WHERE sku = 'ABC-123';
```

**Explanation:**
When MySQL tries to insert a row with `sku = 'ABC-123'`, it detects the unique constraint violation and executes the `ON DUPLICATE KEY UPDATE` clause instead of the insert. Only `price` and `stock_quantity` are updated — `name` and `id` remain unchanged. The row alias `new_row` (MySQL 8.0.20+ syntax) makes the code more readable than the older `VALUES()` function approach. If you run this statement when the SKU does not exist, it behaves as a normal insert.

### Problem 3
You have a `customers` table and a `vip_customers` table with the same structure. Write a query that copies all customers whose `total_spent` exceeds $1000 into `vip_customers`, skipping any that are already there.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    total_spent DECIMAL(10, 2) NOT NULL DEFAULT 0.00
);

CREATE TABLE vip_customers (
    id INT UNSIGNED PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    total_spent DECIMAL(10, 2) NOT NULL,
    promoted_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO customers (name, email, total_spent) VALUES
    ('Alice', 'alice@example.com', 1500.00),
    ('Bob', 'bob@example.com', 200.00),
    ('Carol', 'carol@example.com', 2300.00);
```

**Your query:**
```sql
-- Write the INSERT ... SELECT statement.
```

**Solution:**
```sql
INSERT IGNORE INTO vip_customers (id, name, email, total_spent)
SELECT id, name, email, total_spent
FROM customers
WHERE total_spent > 1000.00;
```

**Explanation:**
`INSERT ... SELECT` pulls matching rows from `customers` and inserts them into `vip_customers` in a single operation. The `WHERE` clause filters to only high-spending customers. `INSERT IGNORE` prevents an error if some of these customers are already in `vip_customers` — it simply skips any row that would violate the primary key or unique email constraint. The `promoted_at` column is not in the insert list, so it defaults to `CURRENT_TIMESTAMP` automatically, recording when each customer was promoted to VIP status.
