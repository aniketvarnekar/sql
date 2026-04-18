# Selecting Data

The `SELECT` statement is the most frequently written SQL statement and the primary way of retrieving data from a MySQL database. Understanding its full range — from simple column retrieval to expressions, aliases, and queries without tables — is fundamental to everything that follows.

## SELECT * vs Selecting Specific Columns

`SELECT *` retrieves every column from the specified table:

```sql
SELECT * FROM books;
```

This is convenient for exploration and ad hoc queries, but it is a poor choice in application code. It transmits more data than may be necessary, prevents the query optimizer from using covering indexes, and causes application code to break silently when columns are added or reordered.

Selecting specific columns is almost always better:

```sql
SELECT id, title, price FROM books;
```

This documents exactly which data your query needs, allows the optimizer more flexibility, and reduces the amount of data transferred over the network.

## Column Aliases with AS

The `AS` keyword assigns an alias to a column in the result set. The alias appears as the column header in the output and is how application code references the value:

```sql
SELECT
    title AS book_title,
    price AS cost_usd,
    price * 1.1 AS price_with_tax
FROM books;
```

Aliases are useful when a column name is verbose, when you are returning a computed expression, or when you want to rename a column for clarity without modifying the table. The `AS` keyword is technically optional — `price cost_usd` works — but including it makes the intent explicit and readable.

Aliases defined in `SELECT` cannot be referenced in the `WHERE` clause of the same query, because `WHERE` is evaluated before `SELECT` in MySQL's logical execution order. However, aliases can be used in `ORDER BY` and `GROUP BY`.

```sql
-- This fails: alias not available in WHERE
SELECT price * 1.1 AS price_with_tax FROM books WHERE price_with_tax > 15;

-- This works: alias used in ORDER BY
SELECT price * 1.1 AS price_with_tax FROM books ORDER BY price_with_tax DESC;
```

## DISTINCT

`DISTINCT` removes duplicate rows from the result set. It operates on the entire selected row — two rows are considered duplicates only if every selected column has the same value:

```sql
SELECT DISTINCT author_id FROM books;
```

This returns each `author_id` value at most once, regardless of how many books that author has written.

```sql
SELECT DISTINCT genre, published_year FROM books;
```

This returns one row per unique combination of `genre` and `published_year`.

`DISTINCT` requires MySQL to sort or hash the result set to identify duplicates, so it adds overhead. Do not use it as a band-aid to hide accidental duplicates from a bad join — instead, fix the join so it doesn't produce duplicates in the first place.

## Computed Expressions in SELECT

The `SELECT` list can include any valid SQL expression, not just column names:

```sql
SELECT
    title,
    price,
    price * 0.9 AS discounted_price,
    CONCAT(first_name, ' ', last_name) AS full_name,
    YEAR(published_date) AS pub_year
FROM books
JOIN authors ON books.author_id = authors.id;
```

Arithmetic operators (`+`, `-`, `*`, `/`), string functions, date functions, and conditional expressions are all valid in the `SELECT` list.

## SELECT Without a Table

MySQL allows `SELECT` without a `FROM` clause. This is useful for evaluating expressions, testing functions, or quickly getting database information:

```sql
SELECT 1 + 1;
SELECT NOW();
SELECT VERSION();
SELECT UPPER('hello world');
SELECT CURDATE(), CURTIME();
SELECT UUID();
SELECT DATABASE();
SELECT USER();
```

`SELECT 1` is also commonly used as a lightweight "ping" query to check database connectivity, and in `EXISTS` subqueries where only the presence of rows matters, not their content.

MySQL also supports the standard SQL alias `DUAL` as a placeholder table name in queries that don't access real tables:

```sql
SELECT 42 AS the_answer FROM DUAL;
```

`FROM DUAL` is entirely optional in MySQL and produces the same result as omitting it.

## Column Ordering in SELECT

The columns in the `SELECT` list appear in the result in the order you specify them. You can reorder columns without affecting the underlying table:

```sql
SELECT price, title, id FROM books;
-- Returns: price | title | id
-- Even though the table defines them as: id | title | price
```

This is useful for presenting data in a specific order for reports or API responses.

## LIMIT Preview

When exploring a large table interactively, `LIMIT` prevents you from accidentally fetching millions of rows:

```sql
SELECT * FROM books LIMIT 10;
```

`LIMIT` is covered fully in the Sorting and Limiting file, but it is worth knowing early because it is essential for safe interactive exploration.

## Viewing Table Structure in a Query

Sometimes you want to see column names and types as part of a query result rather than using `DESCRIBE`. The `information_schema` database, which MySQL maintains automatically, contains metadata about all objects:

```sql
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'bookstore'
  AND table_name = 'books'
ORDER BY ordinal_position;
```

This is useful for writing queries that need to inspect the schema dynamically.

## Practice Problems

### Problem 1
Write a query against a `products` table that returns the product name, a 15% discounted price aliased as `sale_price`, and a column aliased as `savings` showing how much the customer saves at the discounted price. Round both computed columns to two decimal places.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);

INSERT INTO products (name, price) VALUES
    ('Laptop', 999.99),
    ('Mouse', 29.99),
    ('Keyboard', 79.99),
    ('Monitor', 349.99);
```

**Your query:**
```sql
-- Write the SELECT statement here.
```

**Solution:**
```sql
SELECT
    name,
    ROUND(price * 0.85, 2) AS sale_price,
    ROUND(price * 0.15, 2) AS savings
FROM products;
```

**Explanation:**
`ROUND(expression, 2)` rounds the computed value to 2 decimal places, which is appropriate for monetary amounts. The discount is 15%, so the sale price is `price * 0.85` (100% - 15% = 85% of original price) and the savings is `price * 0.15`. Although we could use the `sale_price` alias in the `savings` expression in some databases, MySQL does not allow referencing a `SELECT` alias in the same `SELECT` list, so we must repeat the base expression.

### Problem 2
The `employees` table has `first_name`, `last_name`, and `department` columns. Write a query that returns a list of unique departments, and separately write a query that returns the unique combinations of department and whether the employee is active (`is_active`).

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    department VARCHAR(100),
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

INSERT INTO employees (first_name, last_name, department, is_active) VALUES
    ('Alice', 'Smith', 'Engineering', TRUE),
    ('Bob', 'Jones', 'Engineering', TRUE),
    ('Carol', 'Lee', 'Marketing', TRUE),
    ('Dave', 'Kim', 'Marketing', FALSE),
    ('Eve', 'Park', 'Sales', TRUE);
```

**Your query:**
```sql
-- Write two SELECT statements.
```

**Solution:**
```sql
-- Unique departments only:
SELECT DISTINCT department FROM employees ORDER BY department;

-- Unique department + is_active combinations:
SELECT DISTINCT department, is_active FROM employees ORDER BY department, is_active;
```

**Explanation:**
The first query returns three rows: Engineering, Marketing, Sales. The second query returns four rows: (Engineering, 1), (Marketing, 0), (Marketing, 1), (Sales, 1). `DISTINCT` considers both columns together when determining uniqueness — two rows are only duplicates if both `department` and `is_active` match. Note that `is_active` displays as 1 (true) or 0 (false) because MySQL stores `BOOLEAN` as `TINYINT(1)`.

### Problem 3
Without referencing any table, write a single `SELECT` statement that returns the current date and time, the current database name, your MySQL version, and the result of the expression `(100 * 3.14159) / 2` aliased as `half_circumference`.

**Schema used:**
```sql
-- No table needed.
```

**Your query:**
```sql
-- Write the SELECT without a FROM clause.
```

**Solution:**
```sql
SELECT
    NOW() AS current_datetime,
    CURDATE() AS current_date,
    DATABASE() AS active_database,
    VERSION() AS mysql_version,
    ROUND((100 * 3.14159) / 2, 4) AS half_circumference;
```

**Explanation:**
MySQL allows `SELECT` without a `FROM` clause for expressions and function calls that don't require table data. `NOW()` returns the current date and time as a `DATETIME` value. `CURDATE()` returns just the current date. `DATABASE()` returns the name of the currently selected database (NULL if none is selected). `VERSION()` returns the MySQL server version string. This form of `SELECT` is extremely useful for quickly testing functions or expressions in the MySQL CLI without needing to create or use a table.
