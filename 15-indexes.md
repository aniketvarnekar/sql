# Indexes

An index is a data structure that MySQL maintains alongside a table to make lookups faster. Without an index, MySQL must scan every row in a table to find matching records — a full table scan that grows linearly with table size. With an appropriate index, MySQL can jump directly to the relevant rows in logarithmic time. Understanding how indexes work, when to add them, and when they hurt more than they help is one of the most impactful skills in database performance tuning.

## How Indexes Work Internally

MySQL's default storage engine, InnoDB, uses a B-tree (balanced tree) data structure for most indexes. A B-tree keeps values sorted and allows MySQL to locate a specific value in O(log n) time — for a table with 1 million rows, that means roughly 20 comparisons instead of 1,000,000.

When you create an index on a column, InnoDB creates a separate B-tree structure that stores the indexed column values in sorted order, alongside pointers to the corresponding rows in the table. When a query filters on that column, MySQL traverses the B-tree to find the matching key and then follows the pointer to the full row.

The primary key in InnoDB is special: it is the clustered index. The full row data is stored in the leaf pages of the primary key B-tree, not in a separate structure. All secondary indexes store the primary key value as their row pointer, so a secondary index lookup requires two B-tree lookups: one to find the key in the secondary index, then one to fetch the row from the primary key index.

## Types of Indexes

### PRIMARY KEY

Every InnoDB table should have a primary key. The primary key is automatically a unique, non-null, clustered index:

```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255)
);
```

If you don't define a primary key, InnoDB uses the first `UNIQUE NOT NULL` column as the clustered index, and if none exists, creates a hidden 6-byte row id.

### UNIQUE Index

A `UNIQUE` index enforces uniqueness but also speeds up lookups on that column. It allows one NULL value (multiple NULLs are allowed because NULL is not considered equal to any value):

```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE
);

-- Or as a separate statement:
CREATE UNIQUE INDEX idx_users_email ON users (email);
```

### Regular INDEX

A non-unique index speeds up reads without enforcing uniqueness:

```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_status ON orders (status);
```

This is appropriate for foreign key columns, columns used frequently in `WHERE` clauses, and columns used in `ORDER BY` or `GROUP BY`.

### FULLTEXT Index

A `FULLTEXT` index is designed for natural language text search. It indexes the individual words in a text column, enabling fast searching for documents containing specific words:

```sql
CREATE FULLTEXT INDEX idx_articles_body ON articles (title, body);
```

Covered in detail in the Full-Text Search file.

## Composite Indexes

A composite index is an index on two or more columns. It is more useful than separate single-column indexes for queries that filter on multiple columns:

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

This composite index can serve:
- Queries filtering on `customer_id` only (leftmost prefix)
- Queries filtering on `customer_id` AND `status`

But it cannot serve queries filtering on `status` alone, because the index is sorted first by `customer_id` and then by `status` within each `customer_id` value. The leftmost prefix rule states that MySQL can only use the index starting from the leftmost column.

Column order in a composite index matters enormously. Put the column with the highest selectivity (most distinct values) first, and put equality conditions before range conditions. A composite index on `(status, customer_id)` would efficiently serve queries filtering on `status` alone or on `status AND customer_id`, but not `customer_id` alone.

## Viewing Existing Indexes

To see all indexes on a table:

```sql
SHOW INDEXES FROM orders;
SHOW INDEXES FROM orders\G  -- vertical format, easier to read
```

The output shows the key name, column name, cardinality (estimated distinct values), and whether values can be NULL.

## Dropping an Index

```sql
DROP INDEX idx_orders_status ON orders;
ALTER TABLE orders DROP INDEX idx_orders_status;
```

`ALTER TABLE ... DROP INDEX` and `DROP INDEX` are equivalent. Dropping a primary key requires:

```sql
ALTER TABLE orders DROP PRIMARY KEY;
```

## When Indexes Help

Indexes are most beneficial when:

- Filtering with `WHERE` on the indexed column(s)
- Joining tables on the indexed column (foreign key columns should almost always be indexed)
- Sorting with `ORDER BY` on the indexed column
- Finding min/max values on the indexed column
- Using `DISTINCT` or `GROUP BY` on the indexed column

## When Indexes Don't Help (or Hurt)

Indexes are not used and may even be bypassed when:

- The query filters on only a small percentage of the data (for low-cardinality columns like `status` with 3-4 values, MySQL may prefer a full table scan)
- A function is applied to the indexed column in the `WHERE` clause: `WHERE YEAR(created_at) = 2024` cannot use an index on `created_at`; rewrite as `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'`
- The query uses `LIKE` with a leading wildcard: `WHERE name LIKE '%smith'` cannot use a B-tree index on `name`
- The column has many NULL values and `IS NULL` is the condition (NULLs are indexed in InnoDB, so this is a nuance: single-column `IS NULL` can use an index)

Every index consumes disk space and adds overhead to `INSERT`, `UPDATE`, and `DELETE` operations because the index must be maintained alongside the table. A table with 10 indexes on a heavily written table has significantly more write overhead than the same table with 2 indexes. Add indexes thoughtfully — only when you can demonstrate a query performance problem that the index solves.

## Covering Indexes

A covering index is an index that contains all the columns a query needs, allowing MySQL to answer the query entirely from the index without reading the actual table rows. This is the most efficient form of index usage:

```sql
-- Index on (customer_id, total):
CREATE INDEX idx_orders_cust_total ON orders (customer_id, total);

-- This query is answered entirely from the index (no table row lookup needed):
SELECT customer_id, SUM(total) FROM orders GROUP BY customer_id;
```

When `EXPLAIN` shows `Using index` in the `Extra` column, the query is using a covering index.

## Practice Problems

### Problem 1
Given an `orders` table with columns `id`, `customer_id`, `status`, `total`, and `order_date`, create the most appropriate indexes to support the following query pattern: "Find all orders for a specific customer where the status is 'pending', ordered by order date descending."

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    status VARCHAR(20) NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);
```

**Your query:**
```sql
-- Write the CREATE INDEX statement(s) and the query they support.
```

**Solution:**
```sql
-- The composite index that best serves this query:
CREATE INDEX idx_orders_customer_status_date ON orders (customer_id, status, order_date);

-- The query it serves:
SELECT id, total, order_date
FROM orders
WHERE customer_id = 42 AND status = 'pending'
ORDER BY order_date DESC;
```

**Explanation:**
The index columns are ordered: `customer_id` first (equality filter, highest selectivity), then `status` (equality filter), then `order_date` (used for sorting). This order means MySQL can use the index to satisfy the `WHERE` clause and also to sort the results without a separate sort step. Putting `order_date` after the two equality filters means the index values within each `(customer_id, status)` group are already sorted by `order_date`, so MySQL can use the index for `ORDER BY` as well. A separate index on `order_date` alone would not help because MySQL would still need to filter first.

### Problem 2
Demonstrate the function-on-column anti-pattern: show a query that cannot use an index on `created_at`, and rewrite it to be index-friendly.

**Schema used:**
```sql
CREATE TABLE events (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    created_at DATETIME NOT NULL
);

CREATE INDEX idx_events_created_at ON events (created_at);
```

**Your query:**
```sql
-- Write the non-index-friendly version and the optimized version.
```

**Solution:**
```sql
-- Cannot use the index on created_at (function applied to the column):
SELECT id, name FROM events
WHERE YEAR(created_at) = 2024 AND MONTH(created_at) = 3;

-- Index-friendly rewrite using a range condition:
SELECT id, name FROM events
WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
```

**Explanation:**
`YEAR(created_at) = 2024` wraps the indexed column in a function call. MySQL cannot use the B-tree index on `created_at` to satisfy this condition because the index stores actual `DATETIME` values sorted chronologically, not the result of `YEAR()`. To find matching rows, MySQL would have to apply the function to every row and compare — effectively a full table scan. The rewrite `WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01'` uses a range condition directly on the column value, which the B-tree index can satisfy efficiently with a range scan.

### Problem 3
Show how to display the indexes on a table and how to identify whether a query is using an index, and drop an index that is no longer needed.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    city VARCHAR(100)
);

CREATE INDEX idx_customers_city ON customers (city);
CREATE INDEX idx_customers_name ON customers (name);
```

**Your query:**
```sql
-- Show indexes, use EXPLAIN, then drop the unused index.
```

**Solution:**
```sql
-- View all indexes on the table:
SHOW INDEXES FROM customers;

-- Check if a query uses an index:
EXPLAIN SELECT id, name FROM customers WHERE email = 'alice@example.com';
-- type: 'const' or 'ref' indicates index use; 'ALL' means full table scan

EXPLAIN SELECT id, name FROM customers WHERE city = 'New York';
-- type: 'ref' if the city index is used

-- If the city index is never used in practice, drop it:
DROP INDEX idx_customers_city ON customers;

-- Verify the index is gone:
SHOW INDEXES FROM customers;
```

**Explanation:**
`SHOW INDEXES FROM customers` lists all indexes including the primary key and any secondary indexes. `EXPLAIN` analyzes how MySQL would execute the query without actually running it — the `type` column tells you the access method. `const` means MySQL found exactly one row via a unique/primary key lookup (very fast). `ref` means MySQL used a non-unique index to find matching rows. `ALL` means a full table scan. The `key` column shows which index is actually used. After inspecting query plans with `EXPLAIN`, you can make informed decisions about which indexes are providing value and which are just consuming space and write overhead.
