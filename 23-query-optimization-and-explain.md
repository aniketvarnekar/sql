# Query Optimization and EXPLAIN

Understanding why a query is slow and how to fix it is one of the most valuable skills in database work. MySQL provides `EXPLAIN` and `EXPLAIN ANALYZE` to show exactly how the query optimizer plans to execute a query, what indexes it uses, and how many rows it expects to examine. This file covers how to read that output and the most impactful optimization techniques.

## EXPLAIN and EXPLAIN ANALYZE

`EXPLAIN` shows the query execution plan without actually running the query:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';
```

`EXPLAIN ANALYZE` actually executes the query and reports both the estimated plan and the real measured performance (MySQL 8.0.18+):

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

`EXPLAIN FORMAT=JSON` produces a detailed JSON representation of the plan, which contains more information than the default tabular format:

```sql
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE customer_id = 42;
```

## Reading EXPLAIN Output

The default tabular `EXPLAIN` output has several key columns:

### id
The query block identifier. Subqueries and derived tables get their own id. Rows with the same id are executed as a unit; lower ids execute first.

### select_type
The type of SELECT operation:
- `SIMPLE` — a simple SELECT without subqueries or unions
- `PRIMARY` — the outermost SELECT in a query with subqueries
- `SUBQUERY` — a subquery in the SELECT or WHERE clause
- `DERIVED` — a derived table (subquery in FROM clause)
- `UNION` — the second or later SELECT in a UNION

### type (Access Method)

The `type` column is the most important column in `EXPLAIN` output. It shows how MySQL accesses the table rows, from fastest to slowest:

- `system` — the table has only one row (special case of `const`)
- `const` — MySQL reads exactly one row using a primary key or unique key equality condition. Extremely fast.
- `eq_ref` — one row is read from the table for each row in the previous table, using a primary or unique key. Occurs in joins on primary keys.
- `ref` — rows with matching key values are read for each row in the previous table. Uses a non-unique index. Good.
- `range` — only rows within a given range are retrieved using an index. Used for BETWEEN, IN, and comparison operators on indexed columns.
- `index` — the full index is scanned (all index leaves). Better than `ALL` only if the index is much smaller than the table, or if you're getting a covering index scan.
- `ALL` — a full table scan. Every row is read. This is the worst case for large tables and almost always indicates a missing index.

### key
The index actually chosen by the optimizer. `NULL` means no index is used.

### key_len
The length of the index key used. Shorter is better; this also tells you how many columns of a composite index are being used.

### rows
The optimizer's estimate of how many rows it needs to examine. Multiply this across all rows in a join to get a sense of total work.

### Extra
Additional information about the execution:
- `Using index` — the query is served entirely from the index (covering index). Excellent.
- `Using where` — a WHERE filter is applied after reading rows from storage.
- `Using filesort` — MySQL must sort the result set with an external sort (no usable index for ORDER BY). Can be slow for large result sets.
- `Using temporary` — MySQL must create a temporary table to process the result (common with GROUP BY or UNION without an appropriate index).
- `Using index condition` — Index Condition Pushdown (ICP) is used, where part of the WHERE is evaluated at the storage engine level.

## Query Types in Practice

```sql
-- const: primary key lookup
EXPLAIN SELECT * FROM users WHERE id = 42;
-- type: const, rows: 1

-- ref: non-unique index lookup
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- type: ref (if customer_id is indexed), rows: estimated matches

-- range: range scan on indexed column
EXPLAIN SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31';
-- type: range

-- ALL: full table scan (no index on the filtered column)
EXPLAIN SELECT * FROM orders WHERE notes LIKE '%refund%';
-- type: ALL, rows: entire table
```

## Slow Query Log

MySQL's slow query log records queries that exceed a specified execution time. It is invaluable for finding queries that need optimization in production:

```sql
-- Check if the slow query log is enabled:
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';

-- Enable it for the current session:
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1; -- Log queries taking more than 1 second

-- Find the log file location:
SHOW VARIABLES LIKE 'slow_query_log_file';
```

The `mysqldumpslow` command-line tool summarizes slow query log entries by query pattern, showing which queries appear most often and take the longest.

## Common Optimization Techniques

### Use Covering Indexes

A covering index includes all columns that a query needs, allowing MySQL to answer it entirely from the index without reading the full table rows. This is the single most impactful index optimization for read-heavy queries:

```sql
-- Query:
SELECT customer_id, status, total FROM orders WHERE customer_id = 42;

-- Covering index that satisfies the WHERE and returns all selected columns:
CREATE INDEX idx_orders_cust_covering ON orders (customer_id, status, total);
-- EXPLAIN shows: Using index (no table lookup needed)
```

### Avoid SELECT *

`SELECT *` forces MySQL to fetch every column, even those not needed. It also prevents covering indexes from being used. Select only the columns your query actually needs:

```sql
-- Slow: forces full row fetch even if an index covers part of it
SELECT * FROM orders WHERE customer_id = 42;

-- Fast: can use a covering index
SELECT id, status, total FROM orders WHERE customer_id = 42;
```

### Avoid Functions on Indexed Columns in WHERE

Wrapping an indexed column in a function prevents MySQL from using the index:

```sql
-- Cannot use index on created_at:
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';

-- Can use index on created_at:
SELECT * FROM orders WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16';
```

The same applies to `YEAR()`, `MONTH()`, `UPPER()`, `LOWER()`, arithmetic operations, and any other function applied directly to the indexed column.

### Use LIMIT to Reduce Work

When you only need a few rows, add `LIMIT` so MySQL can stop scanning early:

```sql
-- With a suitable index + ORDER BY + LIMIT, MySQL uses an index scan and stops early:
SELECT id, name FROM products ORDER BY price DESC LIMIT 10;
```

### Optimize JOIN Order and Conditions

MySQL's optimizer generally determines the best join order, but you can help it by ensuring all join columns are indexed:

```sql
-- Index the foreign key columns used in joins:
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_order_items_order ON order_items (order_id);
```

### Query Cache (Removed in MySQL 8)

MySQL 5.x had a query cache that stored the complete result of SELECT queries and returned cached results for identical queries. It was removed in MySQL 8.0 because it provided minimal benefit in practice and introduced serious performance problems under concurrent write workloads: any write to a table invalidated all cached queries against that table, causing cache thrashing. MySQL 8 instead relies on the InnoDB buffer pool (which caches data pages) and external caching layers (Memcached, Redis) for application-level query result caching.

## Practice Problems

### Problem 1
Run `EXPLAIN` on a query against a table without an index on the filtered column. Then add an appropriate index and run `EXPLAIN` again to show the improvement.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    status VARCHAR(20) NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

-- Insert some test data:
INSERT INTO orders (customer_id, status, total, order_date)
SELECT
    FLOOR(RAND() * 1000) + 1,
    ELT(FLOOR(RAND() * 3) + 1, 'pending', 'completed', 'cancelled'),
    ROUND(RAND() * 500 + 10, 2),
    DATE_ADD('2023-01-01', INTERVAL FLOOR(RAND() * 365) DAY)
FROM information_schema.columns LIMIT 200;
```

**Your query:**
```sql
-- Run EXPLAIN before and after adding an index.
```

**Solution:**
```sql
-- Before index: likely shows type=ALL (full table scan)
EXPLAIN SELECT id, total FROM orders WHERE customer_id = 42;

-- Add index:
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

-- After index: shows type=ref, key=idx_orders_customer_id
EXPLAIN SELECT id, total FROM orders WHERE customer_id = 42;

-- Even better with a covering index:
DROP INDEX idx_orders_customer_id ON orders;
CREATE INDEX idx_orders_cust_covering ON orders (customer_id, total);

-- Now EXPLAIN shows: type=ref, Extra=Using index (covering index)
EXPLAIN SELECT id, total FROM orders WHERE customer_id = 42;
```

**Explanation:**
The first `EXPLAIN` (before any index) shows `type: ALL` and `rows` equal to the full table count — MySQL must scan every row to find `customer_id = 42`. After adding the basic index, `type` changes to `ref` and `rows` drops to a small estimate representing only the rows for customer 42. The covering index goes further: because `customer_id` and `total` are both in the index, MySQL never needs to read the actual table rows — `EXPLAIN` shows `Using index` in the `Extra` column, indicating the query is answered entirely from the index structure.

### Problem 2
Show the difference in `EXPLAIN` output between a query that cannot use an index (function on the column) and the equivalent rewritten query that can.

**Schema used:**
```sql
CREATE TABLE events (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    event_date DATE NOT NULL
);

CREATE INDEX idx_events_date ON events (event_date);
```

**Your query:**
```sql
-- Show both EXPLAIN outputs.
```

**Solution:**
```sql
-- Cannot use the index: YEAR() applied to indexed column
EXPLAIN SELECT id, name FROM events WHERE YEAR(event_date) = 2024;
-- type: ALL (or index), Extra: Using where
-- MySQL cannot use the range-scan capability of the index on event_date

-- Can use the index: range condition on the column directly
EXPLAIN SELECT id, name FROM events WHERE event_date >= '2024-01-01' AND event_date < '2025-01-01';
-- type: range, key: idx_events_date, Extra: Using index condition
-- MySQL performs a range scan on the index, examining only 2024 dates
```

**Explanation:**
When `YEAR(event_date)` is used in the `WHERE` clause, MySQL must compute `YEAR()` for every row and compare — it cannot seek into the B-tree index using a function result because the index stores actual date values, not year numbers. The rewritten form uses a range condition (`>=` and `<`) directly on the indexed column, allowing MySQL to use the index to jump to 2024-01-01 and scan forward to 2025-01-01, examining only the relevant portion of the index. This transformation — from a function on a column to an equivalent range on the raw column — is one of the most common and impactful query rewrites.

### Problem 3
Use `EXPLAIN ANALYZE` to compare the actual vs estimated rows for a query, and demonstrate what `Using filesort` means and how to eliminate it.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(100) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);

CREATE INDEX idx_products_category ON products (category);
```

**Your query:**
```sql
-- Show EXPLAIN for a query with filesort, then fix it.
```

**Solution:**
```sql
-- Query that causes filesort (index on category, but ORDER BY is on price):
EXPLAIN SELECT id, name, price
FROM products
WHERE category = 'Electronics'
ORDER BY price DESC;
-- Extra: Using index condition; Using filesort
-- MySQL uses the index for the WHERE, then must sort the result by price

-- Fix: composite index that supports both WHERE and ORDER BY
CREATE INDEX idx_products_cat_price ON products (category, price);

EXPLAIN SELECT id, name, price
FROM products
WHERE category = 'Electronics'
ORDER BY price DESC;
-- Extra: Using index condition (or Using where)
-- The composite index is traversed in category + price order, eliminating the sort

-- EXPLAIN ANALYZE shows actual vs estimated rows:
EXPLAIN ANALYZE SELECT id, name, price
FROM products
WHERE category = 'Electronics'
ORDER BY price DESC;
-- Output includes: "actual rows=X, loops=1" vs "rows=Y" (estimate)
```

**Explanation:**
`Using filesort` appears when MySQL cannot use an index to supply rows in the required ORDER BY order. For a query that filters on `category` and sorts by `price`, a single-column index on `category` can filter the rows but cannot provide them sorted by `price`. MySQL must therefore read all matching rows and sort them afterward — this sort is the "filesort." Creating a composite index on `(category, price)` lets MySQL traverse the index in `category` + `price` order, so the result is already sorted when it exits the index scan and no separate sort step is needed. For large result sets, eliminating a filesort can dramatically reduce query time.
