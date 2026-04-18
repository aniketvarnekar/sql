# Sorting and Limiting

Retrieving data in a meaningful order and constraining result sets to a manageable size are fundamental to building useful applications. `ORDER BY` controls the sort order of query results, while `LIMIT` and `OFFSET` control how many rows are returned and from what position. Together they enable everything from alphabetical lists to paginated APIs.

## ORDER BY

`ORDER BY` sorts query results by one or more columns or expressions. Without `ORDER BY`, MySQL returns rows in an unspecified order that may vary between queries and MySQL versions. Never rely on the implicit ordering of query results — if order matters to your application, always specify `ORDER BY` explicitly.

```sql
SELECT id, name, price FROM products ORDER BY price;
```

The default sort direction is ascending (`ASC`). To sort in descending order:

```sql
SELECT id, name, price FROM products ORDER BY price DESC;
```

You can sort by a `SELECT` alias or by a column expression:

```sql
SELECT name, price * 1.1 AS price_with_tax
FROM products
ORDER BY price_with_tax DESC;

SELECT name, YEAR(hire_date) AS hire_year
FROM employees
ORDER BY hire_year;
```

## Sorting by Multiple Columns

`ORDER BY` can take a comma-separated list of columns. MySQL sorts by the first column, then uses subsequent columns to break ties:

```sql
SELECT last_name, first_name, department
FROM employees
ORDER BY department ASC, last_name ASC, first_name ASC;
```

This sorts employees alphabetically by department, then by last name within each department, then by first name when last names also match. Each column can have its own direction:

```sql
SELECT product_name, category, price
FROM products
ORDER BY category ASC, price DESC;
```

This lists categories alphabetically, and within each category shows the most expensive products first.

## Sorting and NULL Values

`NULL` values in MySQL sort before non-NULL values when sorting in ascending order (NULLs first), and after non-NULL values when sorting in descending order (NULLs last). This is the opposite of PostgreSQL's behavior.

To explicitly control where NULLs appear, use a conditional expression:

```sql
-- Sort NULLs last in ascending order:
SELECT name, salary FROM employees ORDER BY (salary IS NULL) ASC, salary ASC;

-- Sort NULLs first in descending order:
SELECT name, salary FROM employees ORDER BY (salary IS NULL) DESC, salary DESC;
```

`(salary IS NULL)` evaluates to 1 when NULL and 0 when not NULL. Sorting this expression ascending puts 0s (non-NULLs) first.

## LIMIT

`LIMIT n` restricts the result to at most `n` rows:

```sql
SELECT * FROM products ORDER BY price DESC LIMIT 10;
```

This returns the 10 most expensive products. `LIMIT` is evaluated after `ORDER BY`, so you get the correct top-N rows.

`LIMIT` is also essential during development to avoid accidentally fetching millions of rows:

```sql
SELECT * FROM large_table LIMIT 5;
```

## OFFSET

`OFFSET n` skips the first `n` rows before returning results. It is used together with `LIMIT` to implement pagination:

```sql
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40;
-- Skips the first 40 rows, returns rows 41-60
```

MySQL supports an alternative two-argument syntax for `LIMIT` that combines limit and offset:

```sql
SELECT * FROM products ORDER BY id LIMIT 40, 20;
-- Equivalent to LIMIT 20 OFFSET 40
-- First argument is the offset, second is the count
```

The two-argument form is concise but easy to confuse because the argument order is reversed from the verbose `LIMIT ... OFFSET ...` form. Using the explicit `OFFSET` keyword is more readable.

## Pagination Pattern

A classic pagination query for page P with page_size rows per page:

```sql
-- Page 1 (rows 1-20):
SELECT id, name, price FROM products ORDER BY id LIMIT 20 OFFSET 0;

-- Page 2 (rows 21-40):
SELECT id, name, price FROM products ORDER BY id LIMIT 20 OFFSET 20;

-- Page N:
-- OFFSET = (page_number - 1) * page_size
-- LIMIT = page_size
```

In application code, this typically looks like:

```sql
SELECT id, name, price
FROM products
ORDER BY id
LIMIT :page_size OFFSET :offset;
-- where :offset = (:page - 1) * :page_size
```

### Performance Warning: Deep Pagination

`LIMIT ... OFFSET` with large offsets is inefficient. A query like `LIMIT 20 OFFSET 100000` causes MySQL to read and discard 100,000 rows before returning the 20 you want. This gets progressively slower as the offset grows.

For large datasets, the keyset pagination (also called cursor-based pagination) pattern is far more efficient. Instead of skipping rows by count, it uses the last seen value of the sort key as a starting point:

```sql
-- First page:
SELECT id, name, price FROM products ORDER BY id LIMIT 20;
-- Returns rows with ids 1-20

-- Next page (using last seen id = 20 as cursor):
SELECT id, name, price FROM products WHERE id > 20 ORDER BY id LIMIT 20;
-- Returns rows with ids 21-40
```

Because `WHERE id > 20` uses the index on `id`, MySQL jumps directly to the right position without scanning and discarding rows. This performs equally well regardless of how deep into the dataset you are.

## LIMIT in Non-SELECT Statements

`LIMIT` can also be used in `UPDATE` and `DELETE` statements to cap the number of rows affected:

```sql
DELETE FROM log_entries WHERE created_at < '2022-01-01' LIMIT 1000;
UPDATE products SET featured = 0 WHERE featured = 1 LIMIT 100;
```

This is useful for breaking large data modifications into smaller chunks to avoid holding locks for too long.

## Practice Problems

### Problem 1
Write a query that returns the top 5 most expensive products in the 'Electronics' category, showing the product name, category, and price, sorted from most expensive to least expensive.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(100) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);

INSERT INTO products (name, category, price) VALUES
    ('4K Monitor', 'Electronics', 499.99),
    ('Laptop Stand', 'Accessories', 39.99),
    ('Gaming Keyboard', 'Electronics', 129.99),
    ('USB-C Hub', 'Electronics', 59.99),
    ('Noise Cancelling Headphones', 'Electronics', 349.99),
    ('Mechanical Keyboard', 'Electronics', 179.99),
    ('Webcam HD', 'Electronics', 89.99),
    ('Desk Lamp', 'Accessories', 24.99);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT name, category, price
FROM products
WHERE category = 'Electronics'
ORDER BY price DESC
LIMIT 5;
```

**Explanation:**
The `WHERE` clause filters to Electronics only before sorting, which reduces the number of rows MySQL needs to sort. `ORDER BY price DESC` sorts from most expensive to least, and `LIMIT 5` takes only the top five. If there are fewer than five Electronics products, `LIMIT 5` simply returns all of them without error.

### Problem 2
Implement keyset-based pagination for a query that lists products ordered by `price` ascending and then by `id` ascending (as a tiebreaker). Show the query for the first page (20 items), then show how to get the next page given that the last row of the first page had `price = 59.99` and `id = 8`.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);
-- Assume many rows are in the table.
```

**Your query:**
```sql
-- Write the first page query and the next page query.
```

**Solution:**
```sql
-- First page:
SELECT id, name, price
FROM products
ORDER BY price ASC, id ASC
LIMIT 20;

-- Next page (cursor: price=59.99, id=8):
SELECT id, name, price
FROM products
WHERE (price > 59.99) OR (price = 59.99 AND id > 8)
ORDER BY price ASC, id ASC
LIMIT 20;
```

**Explanation:**
The keyset pagination condition `(price > 59.99) OR (price = 59.99 AND id > 8)` correctly handles ties in the sort key: it continues from items with a higher price, or items with the same price but a higher id. This compound condition ensures no rows are skipped or duplicated at page boundaries when prices are not unique. Because the query uses a `WHERE` clause with index-friendly conditions rather than `OFFSET`, it performs the same regardless of how far into the dataset you are.

### Problem 3
Write a query that returns employees sorted by department (alphabetically), then by salary (highest first within each department), then by last name (alphabetically) for salary ties. Show only columns `last_name`, `first_name`, `department`, and `salary`.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    department VARCHAR(100) NOT NULL,
    salary DECIMAL(10, 2) NOT NULL
);

INSERT INTO employees (first_name, last_name, department, salary) VALUES
    ('Alice', 'Smith', 'Engineering', 95000.00),
    ('Bob', 'Jones', 'Engineering', 85000.00),
    ('Carol', 'Adams', 'Marketing', 75000.00),
    ('Dave', 'Brown', 'Marketing', 75000.00),
    ('Eve', 'Clark', 'Engineering', 105000.00),
    ('Frank', 'Davis', 'Sales', 65000.00);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT last_name, first_name, department, salary
FROM employees
ORDER BY department ASC, salary DESC, last_name ASC;
```

**Explanation:**
The three-column `ORDER BY` defines a strict ordering. First, all Engineering employees come before Marketing, which comes before Sales. Within Engineering, the highest salaries come first (Eve at $105k, then Alice at $95k, then Bob at $85k). Within Marketing, both Carol and Dave earn $75,000, so the third sort key (last name) breaks the tie: Adams before Brown. This pattern — primary sort, secondary sort, tertiary tiebreaker — is the standard approach for deterministic, well-defined ordering.
