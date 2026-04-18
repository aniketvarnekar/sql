# Partitioning

Partitioning divides a single large table into multiple physical segments (partitions) that are stored separately but appear as one logical table to queries. At massive scale — hundreds of millions or billions of rows — partitioning can dramatically speed up queries that access only a specific range of data, and it simplifies data lifecycle management by allowing entire partitions (rather than individual rows) to be dropped or archived instantly.

## What Partitioning Is and Why It Helps

Without partitioning, a table's data is stored in a single set of data files. Queries must scan through all the data (or use an index) to find relevant rows. With partitioning, the data is physically split into multiple files based on a partitioning key. When a query includes a condition that maps to one or a few partitions, MySQL only reads those partitions — this is called partition pruning.

Partitioning is most beneficial when:
- A table has billions of rows and queries almost always filter on the partition key
- You need to delete old data in bulk (drop a partition instead of deleting rows)
- You are working with time-series data and partition by time period

Partitioning does not help when:
- Queries don't filter on the partition key (MySQL must scan all partitions)
- The table is small enough that indexes are sufficient
- You need foreign keys (partitioned tables in MySQL cannot have foreign key constraints)

## RANGE Partitioning

`RANGE` partitioning divides rows into partitions based on column value ranges. Each partition holds rows where the partitioning expression falls within a specified range:

```sql
CREATE TABLE orders (
    id INT UNSIGNED NOT NULL,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_year INT NOT NULL
)
PARTITION BY RANGE (order_year) (
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

`MAXVALUE` acts as a catch-all for values that don't fit in any earlier partition. It is good practice to include a `MAXVALUE` partition so that inserts with unexpected values don't fail.

`RANGE COLUMNS` allows partitioning on `DATE` or `DATETIME` columns directly, which is more natural for time-series data:

```sql
CREATE TABLE events (
    id INT UNSIGNED NOT NULL,
    event_name VARCHAR(200) NOT NULL,
    event_date DATE NOT NULL
)
PARTITION BY RANGE COLUMNS (event_date) (
    PARTITION p2022 VALUES LESS THAN ('2023-01-01'),
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION pfuture VALUES LESS THAN MAXVALUE
);
```

## LIST Partitioning

`LIST` partitioning divides rows based on matching a column value against an explicit list:

```sql
CREATE TABLE employees (
    id INT UNSIGNED NOT NULL,
    name VARCHAR(200) NOT NULL,
    region_code CHAR(2) NOT NULL
)
PARTITION BY LIST COLUMNS (region_code) (
    PARTITION p_north VALUES IN ('MN', 'WI', 'MI', 'OH'),
    PARTITION p_south VALUES IN ('TX', 'FL', 'GA', 'AL'),
    PARTITION p_west  VALUES IN ('CA', 'OR', 'WA', 'NV'),
    PARTITION p_east  VALUES IN ('NY', 'PA', 'NJ', 'MA')
);
```

An `INSERT` with a region code not listed in any partition will fail with an error, unless you include a catch-all partition. Unlike `RANGE`, `LIST` does not have a `MAXVALUE` equivalent — you must list every valid value.

## HASH Partitioning

`HASH` partitioning distributes rows evenly across a fixed number of partitions by applying a hash function to the partitioning expression. It is used when you want to evenly distribute data without a natural range or list:

```sql
CREATE TABLE sessions (
    id INT UNSIGNED NOT NULL,
    user_id INT UNSIGNED NOT NULL,
    data TEXT
)
PARTITION BY HASH (user_id)
PARTITIONS 8;
```

MySQL computes `MOD(user_id, 8)` internally to determine which of the 8 partitions receives each row. `HASH` partitioning eliminates hot spots when data would otherwise concentrate in a few range partitions. The main limitation is that partition pruning is less effective — a query for a specific `user_id` can be pruned to exactly one partition, but range queries on `user_id` may still scan many partitions.

## KEY Partitioning

`KEY` partitioning is similar to `HASH` but uses MySQL's internal hashing function and can use the primary key or any indexed column by default. When no column is specified, MySQL uses the primary key:

```sql
CREATE TABLE products (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    PRIMARY KEY (id)
)
PARTITION BY KEY ()
PARTITIONS 4;
```

`KEY` partitioning works on columns of any data type, including strings and dates, whereas `HASH` requires an integer expression.

## Partition Pruning

Partition pruning is MySQL's ability to skip partitions that cannot contain rows matching the query's `WHERE` conditions. It is what makes partitioning valuable for read performance.

For `RANGE` partitioning, pruning works when the `WHERE` clause uses the partition key:

```sql
-- Only reads the p2023 partition (pruning skips p2021, p2022, p2024, pmax):
SELECT * FROM orders WHERE order_year = 2023;

-- Only reads p2022 and p2023:
SELECT * FROM orders WHERE order_year BETWEEN 2022 AND 2023;
```

Use `EXPLAIN PARTITIONS` to see which partitions a query accesses:

```sql
EXPLAIN SELECT * FROM orders WHERE order_year = 2023\G
-- partitions: p2023
```

## Adding, Dropping, and Reorganizing Partitions

To add a new partition (typically when a new time period needs to be added):

```sql
-- First remove the MAXVALUE catch-all, then add new partitions:
ALTER TABLE orders REORGANIZE PARTITION pmax INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

To drop an entire partition and all its data (fast — does not delete row by row):

```sql
ALTER TABLE orders DROP PARTITION p2021;
```

Dropping a partition is effectively instantaneous, regardless of how many rows it contained. This is the primary operational advantage of partitioning for data lifecycle management — rolling off old data is a metadata operation rather than a slow DELETE.

To add a partition to a table without a `MAXVALUE` catch-all:

```sql
ALTER TABLE orders ADD PARTITION (
    PARTITION p2026 VALUES LESS THAN (2027)
);
```

## Limitations and Gotchas in MySQL Partitioning

Partitioning has several important restrictions in MySQL:

- **No foreign keys:** A partitioned table cannot be the child or parent in a foreign key relationship. This is a major limitation for normalized schemas.
- **Unique keys must include the partition key:** Every `UNIQUE` index (including the primary key) must include the partitioning expression's column(s). This often forces the partitioning column into the primary key, changing the schema design.
- **Partition key in every query for pruning:** If the partitioning column is not in the `WHERE` clause, MySQL scans all partitions, which can be slower than an equivalent unpartitioned table with a good index.
- **Limited to 8192 partitions per table** (MySQL 8.0).
- **Some DDL operations are slower** on partitioned tables because the operation must be applied to each partition.

Before adding partitioning to an existing system, benchmark with real data and query patterns. In many cases, adding appropriate indexes on an unpartitioned table is faster to implement and performs better than partitioning.

## Practice Problems

### Problem 1
Create a `RANGE`-partitioned `orders` table partitioned by year, then verify partition pruning is working with `EXPLAIN`.

**Schema used:**
```sql
-- No pre-existing schema needed.
```

**Your query:**
```sql
-- Create the partitioned table and verify pruning.
```

**Solution:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    order_year YEAR NOT NULL,
    PRIMARY KEY (id, order_year)  -- partition key must be in primary key
)
PARTITION BY RANGE (order_year) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Insert test data:
INSERT INTO orders (customer_id, total, order_year) VALUES
    (1, 100.00, 2022),
    (2, 200.00, 2023),
    (3, 300.00, 2024);

-- Verify partition pruning:
EXPLAIN SELECT * FROM orders WHERE order_year = 2023;
-- 'partitions' column should show: p2023 only
```

**Explanation:**
The primary key includes `order_year` because MySQL requires the partitioning column to be part of every unique key on the table. Without this, MySQL raises an error: "A PRIMARY KEY must include all columns in the table's partitioning function." The `EXPLAIN` output shows `partitions: p2023`, confirming that MySQL is reading only the 2023 partition rather than all partitions. Without partition pruning, the same query would show all partition names, meaning MySQL would scan all data.

### Problem 2
Demonstrate the "drop old partition" pattern for data archiving — show how to add a new year partition (splitting the MAXVALUE partition) and drop the oldest partition to archive old data.

**Schema used:**
```sql
-- Use the partitioned orders table from Problem 1.
```

**Your query:**
```sql
-- Add 2025 partition (split from pmax) and drop the 2022 partition.
```

**Solution:**
```sql
-- Step 1: Reorganize pmax to carve out 2025 and keep the catch-all:
ALTER TABLE orders REORGANIZE PARTITION pmax INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Verify the new partition exists:
SELECT partition_name, partition_description
FROM information_schema.partitions
WHERE table_name = 'orders' AND table_schema = DATABASE()
ORDER BY partition_ordinal_position;

-- Step 2: Archive and drop 2022 data:
-- First, optionally move old data to an archive table:
INSERT INTO orders_archive SELECT * FROM orders PARTITION (p2022);

-- Then drop the partition (instantly removes all 2022 data):
ALTER TABLE orders DROP PARTITION p2022;
```

**Explanation:**
`REORGANIZE PARTITION pmax INTO (...)` splits the catch-all partition into a specific 2025 partition and a new catch-all. This is the standard pattern for maintaining a rolling window of partitioned data — add a new partition for the upcoming period while dropping the oldest. `DROP PARTITION p2022` deletes all data in that partition as a single metadata operation, which is nearly instantaneous regardless of row count. In contrast, `DELETE FROM orders WHERE order_year = 2022` would need to delete each row individually, which could take minutes or hours for large datasets.

### Problem 3
Explain why a query without the partition key in the WHERE clause bypasses partition pruning, and show how to verify this with EXPLAIN.

**Schema used:**
```sql
-- Use the partitioned orders table from Problem 1.
```

**Your query:**
```sql
-- Show EXPLAIN for queries with and without the partition key.
```

**Solution:**
```sql
-- WITH partition key: pruned to one partition (fast)
EXPLAIN SELECT * FROM orders WHERE order_year = 2023;
-- partitions: p2023
-- type: ref or ALL within that one partition

-- WITHOUT partition key: scans ALL partitions (potentially slow)
EXPLAIN SELECT * FROM orders WHERE customer_id = 2;
-- partitions: p2022,p2023,p2024,pmax (all partitions scanned)
-- type: ALL (if no index on customer_id)

-- Solution: add an index on customer_id to help queries that don't use partition key:
CREATE INDEX idx_orders_customer ON orders (customer_id);
EXPLAIN SELECT * FROM orders WHERE customer_id = 2;
-- Still scans all partitions, but uses the index within each (type: ref)
-- The index helps, but you still pay the partition overhead vs. a non-partitioned table
```

**Explanation:**
Partition pruning only works when the `WHERE` clause includes a condition on the partitioning column that MySQL can evaluate without reading data. A query on `customer_id` cannot be pruned because MySQL doesn't know which partitions contain rows with `customer_id = 2` without reading all of them. Adding an index on `customer_id` helps — MySQL can use the index within each partition — but it still touches all partitions, which is overhead compared to a non-partitioned table with the same index. This is why it's critical to partition on the column that appears in the vast majority of WHERE clauses; otherwise, partitioning adds overhead rather than reducing it.
