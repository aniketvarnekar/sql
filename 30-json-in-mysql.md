# JSON in MySQL

MySQL 8 includes a native `JSON` data type that validates and stores JSON documents efficiently. It provides a rich set of functions for querying, modifying, and indexing JSON data, bridging the gap between relational and document-oriented storage within a single database.

## JSON Data Type in MySQL 8

The `JSON` data type stores JSON documents in an optimized binary format that allows fast access to individual keys without parsing the entire document on every read. Inserting invalid JSON raises an error immediately:

```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    attributes JSON
);
```

A `JSON` column can store any valid JSON value: objects, arrays, strings, numbers, booleans, and `null`. The maximum size is limited by `max_allowed_packet` (default 64 MB in MySQL 8).

## Inserting and Selecting JSON Data

JSON data can be inserted as a JSON string literal:

```sql
INSERT INTO products (name, attributes) VALUES
    ('Laptop', '{"color": "silver", "ram_gb": 16, "storage_gb": 512, "tags": ["portable", "fast"]}'),
    ('Monitor', '{"color": "black", "size_inches": 27, "resolution": "4K", "tags": ["display"]}'),
    ('Keyboard', '{"color": "white", "type": "mechanical", "tags": ["input", "wireless"]}');
```

To select the full JSON document, query the column as usual. MySQL returns the JSON as a formatted string:

```sql
SELECT id, name, attributes FROM products;
```

## JSON Functions: JSON_OBJECT, JSON_ARRAY, JSON_EXTRACT

`JSON_OBJECT(key, value, ...)` creates a JSON object from key-value pairs:

```sql
SELECT JSON_OBJECT('name', 'Alice', 'age', 30, 'active', TRUE);
-- {"name": "Alice", "age": 30, "active": true}
```

`JSON_ARRAY(value, ...)` creates a JSON array:

```sql
SELECT JSON_ARRAY('red', 'green', 'blue');
-- ["red", "green", "blue"]
```

`JSON_EXTRACT(json_doc, path)` extracts a value from a JSON document using a JSONPath expression. Paths start with `$` (the root of the document):

```sql
SELECT JSON_EXTRACT(attributes, '$.color') AS color FROM products;
-- "silver"
-- "black"
-- "white"
```

Path syntax:
- `$.key` — access an object key
- `$.key.nested` — access a nested key
- `$.array[0]` — access the first element of an array
- `$.array[*]` — access all elements of an array (returns a JSON array of values)

## The -> and ->> Operators

MySQL provides two shorthand operators as aliases for `JSON_EXTRACT`:

The `->` operator extracts a value but keeps it JSON-encoded (strings include quotes):

```sql
SELECT attributes->'$.color' AS color FROM products;
-- Returns: "silver" (note the quotes — it is a JSON string)
```

The `->>` operator (double arrow) extracts a value and unquotes it, returning a plain string:

```sql
SELECT attributes->>'$.color' AS color FROM products;
-- Returns: silver (no quotes — it is a plain MySQL string)
```

Use `->` when you need to compare against a JSON value. Use `->>` when you need the value as a plain string for comparisons, `LIKE`, or display:

```sql
-- Filter by JSON value using ->>:
SELECT name FROM products WHERE attributes->>'$.color' = 'silver';

-- Filter by JSON value using -> (must include quotes in the comparison):
SELECT name FROM products WHERE attributes->'$.color' = '"silver"';
```

## JSON_SET, JSON_INSERT, JSON_REPLACE, JSON_REMOVE

These functions modify JSON documents without replacing the entire document.

`JSON_SET(json_doc, path, value, ...)` inserts or replaces a value at the specified path:

```sql
UPDATE products
SET attributes = JSON_SET(attributes, '$.ram_gb', 32, '$.storage_gb', 1024)
WHERE name = 'Laptop';
```

`JSON_INSERT(json_doc, path, value)` inserts a value only if the path does not already exist:

```sql
UPDATE products
SET attributes = JSON_INSERT(attributes, '$.warranty_years', 2)
WHERE name = 'Laptop';
-- Only adds warranty_years if it doesn't exist; otherwise, no change
```

`JSON_REPLACE(json_doc, path, value)` replaces a value only if the path already exists:

```sql
UPDATE products
SET attributes = JSON_REPLACE(attributes, '$.color', 'space gray')
WHERE name = 'Laptop';
-- Only updates color if the 'color' key exists; otherwise, no change
```

`JSON_REMOVE(json_doc, path, ...)` removes the value at the specified path:

```sql
UPDATE products
SET attributes = JSON_REMOVE(attributes, '$.tags')
WHERE name = 'Monitor';
```

## Additional Useful JSON Functions

`JSON_CONTAINS(json_doc, val, path)` checks whether a JSON document contains a specific value:

```sql
-- Find products whose tags array contains 'wireless':
SELECT name FROM products
WHERE JSON_CONTAINS(attributes->'$.tags', '"wireless"');
```

`JSON_ARRAYAGG(expr)` aggregates values into a JSON array (MySQL 5.7.22+):

```sql
SELECT customer_id, JSON_ARRAYAGG(total) AS order_totals
FROM orders
GROUP BY customer_id;
```

`JSON_OBJECTAGG(key, value)` aggregates key-value pairs into a JSON object:

```sql
SELECT JSON_OBJECTAGG(name, price) AS price_map FROM products;
-- {"Laptop": 999.99, "Monitor": 449.99, "Keyboard": 89.99}
```

`JSON_KEYS(json_doc)` returns a JSON array of the keys in a JSON object:

```sql
SELECT name, JSON_KEYS(attributes) AS keys FROM products;
```

`JSON_TYPE(json_doc)` returns the type of a JSON value:

```sql
SELECT JSON_TYPE(attributes) FROM products; -- OBJECT
SELECT JSON_TYPE(attributes->'$.tags');      -- ARRAY
SELECT JSON_TYPE(attributes->'$.ram_gb');    -- INTEGER
```

## Indexing JSON Fields with Generated Columns

JSON columns themselves cannot be directly indexed. However, you can create a generated column that extracts a specific JSON value and then index that generated column:

```sql
ALTER TABLE products
    ADD COLUMN color VARCHAR(50)
        GENERATED ALWAYS AS (attributes->>'$.color') VIRTUAL,
    ADD INDEX idx_color (color);
```

A `VIRTUAL` generated column computes its value when the row is read without storing it on disk. A `STORED` generated column computes and stores the value when the row is inserted or updated, using more disk space but allowing it to be indexed.

After adding this index, queries that filter on `color` use the index:

```sql
SELECT name FROM products WHERE color = 'silver';
-- Uses idx_color (type: ref in EXPLAIN)

-- The original JSON path query also benefits via the generated column:
SELECT name FROM products WHERE attributes->>'$.color' = 'silver';
-- MySQL may still use idx_color depending on optimizer
```

Multi-valued indexes on JSON arrays (MySQL 8.0.17+) allow indexing all elements of a JSON array:

```sql
ALTER TABLE products
    ADD INDEX idx_tags ((CAST(attributes->'$.tags' AS CHAR(50) ARRAY)));

-- Find products with a specific tag using member-of operator:
SELECT name FROM products WHERE 'wireless' MEMBER OF (attributes->'$.tags');
```

## When to Use JSON in MySQL and When Not To

Use JSON in MySQL when:
- You have truly variable or unpredictable attributes that differ significantly per row (product attributes, user preferences, API payloads)
- You are storing data received from external systems as-is and rarely need to query individual fields
- You need to prototype quickly without defining a rigid schema upfront

Do not use JSON in MySQL when:
- You regularly query, filter, sort, or aggregate individual fields — put those in proper columns with indexes
- You need referential integrity between JSON values and other tables — foreign keys don't work on JSON content
- Your JSON structure is actually consistent and predictable — that's a signal to model it as proper columns
- You need to run complex analytics on the data — JSON queries are harder to optimize than column-based queries

The hybrid approach — storing structured, frequently queried data in proper columns and truly variable metadata in a `JSON` column — is the most practical pattern for most applications.

## Practice Problems

### Problem 1
Insert three product records with JSON attributes. Then write queries to extract specific fields, filter by a JSON value, and update a JSON field.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL,
    specs JSON
);
```

**Your query:**
```sql
-- Write the INSERT statements and three query variations.
```

**Solution:**
```sql
-- Insert:
INSERT INTO products (name, price, specs) VALUES
    ('Laptop Pro', 1299.99, '{"cpu": "Intel i7", "ram_gb": 16, "storage_gb": 512, "color": "silver"}'),
    ('Wireless Mouse', 49.99, '{"color": "black", "dpi": 1600, "wireless": true}'),
    ('USB Hub', 29.99, '{"ports": 7, "usb_version": "3.0", "color": "gray"}');

-- Extract a specific field (unquoted string):
SELECT name, specs->>'$.color' AS color FROM products;

-- Filter by a JSON value:
SELECT name, price FROM products WHERE specs->>'$.color' = 'silver';

-- Update a specific JSON field without replacing the whole document:
UPDATE products
SET specs = JSON_SET(specs, '$.ram_gb', 32)
WHERE name = 'Laptop Pro';

-- Verify the update:
SELECT name, specs->>'$.ram_gb' AS ram FROM products WHERE name = 'Laptop Pro';
-- Returns: 32
```

**Explanation:**
`specs->>'$.color'` uses the double-arrow operator to extract the `color` key from the `specs` JSON object as a plain string (without surrounding quotes). This is necessary for the `= 'silver'` comparison — using `->` (single arrow) would require `= '"silver"'` with embedded quotes. `JSON_SET(specs, '$.ram_gb', 32)` modifies only the `ram_gb` field, leaving all other fields in the document unchanged. This is critical: replacing the entire `specs` column with a new JSON document would overwrite fields you didn't intend to change.

### Problem 2
Add a generated column and index for the `color` field from a JSON `specs` column, then verify the index is used with EXPLAIN.

**Schema used:**
```sql
-- Use the products table from Problem 1 with data already inserted.
```

**Your query:**
```sql
-- Add the generated column and index, then verify with EXPLAIN.
```

**Solution:**
```sql
-- Add a generated column that extracts 'color' from the JSON:
ALTER TABLE products
    ADD COLUMN color VARCHAR(50)
        GENERATED ALWAYS AS (specs->>'$.color') VIRTUAL,
    ADD INDEX idx_products_color (color);

-- Query using the generated column directly:
SELECT name, price, color FROM products WHERE color = 'silver';

-- The original JSON path expression may also use the index:
EXPLAIN SELECT name, price FROM products WHERE specs->>'$.color' = 'silver';
-- Check: 'key' column should show idx_products_color

-- View the generated column in the table definition:
SHOW CREATE TABLE products\G
```

**Explanation:**
A `VIRTUAL` generated column computes its value from `specs->>'$.color'` when the row is read. Adding an index on this generated column lets MySQL find rows by color without scanning and parsing JSON in every row. The generated column appears as a regular column in queries — `WHERE color = 'silver'` works just like any other column filter. `EXPLAIN` should show `key: idx_products_color` and `type: ref`, confirming the index is used. This is the standard pattern for making specific JSON fields queryable at index speed without converting the schema to relational columns.

### Problem 3
Demonstrate `JSON_ARRAYAGG` and `JSON_OBJECTAGG` to produce JSON-formatted query results, and show `JSON_CONTAINS` for searching within JSON arrays.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    tags JSON
);

INSERT INTO orders (customer_id, total, tags) VALUES
    (1, 150.00, '["priority", "express"]'),
    (1, 300.00, '["standard"]'),
    (2, 75.00, '["express", "gift"]'),
    (2, 450.00, '["priority", "gift", "express"]');
```

**Your query:**
```sql
-- Write queries using JSON_ARRAYAGG, JSON_OBJECTAGG, and JSON_CONTAINS.
```

**Solution:**
```sql
-- JSON_ARRAYAGG: aggregate order totals into a JSON array per customer:
SELECT
    customer_id,
    JSON_ARRAYAGG(total ORDER BY total DESC) AS order_totals,
    COUNT(*) AS order_count,
    SUM(total) AS lifetime_value
FROM orders
GROUP BY customer_id;

-- JSON_OBJECTAGG: create a map of order_id to total:
SELECT JSON_OBJECTAGG(id, total) AS id_to_total FROM orders;
-- Returns: {"1": 150.00, "2": 300.00, "3": 75.00, "4": 450.00}

-- JSON_CONTAINS: find orders tagged as 'express':
SELECT id, customer_id, total
FROM orders
WHERE JSON_CONTAINS(tags, '"express"');
-- Returns: orders 1, 3, 4 (those with "express" in their tags array)

-- JSON_CONTAINS with a nested path:
SELECT id FROM orders
WHERE JSON_CONTAINS(tags, '"priority"');
-- Returns: orders 1 and 4
```

**Explanation:**
`JSON_ARRAYAGG(total ORDER BY total DESC)` produces a JSON array of totals per customer, sorted in descending order within the array. This is useful for building API responses or reports that need arrays of related values. `JSON_OBJECTAGG(id, total)` creates a JSON object where each order's id maps to its total — useful for lookup maps. `JSON_CONTAINS(tags, '"express"')` searches the `tags` JSON array for the string "express" — the value argument must itself be valid JSON, so the string "express" must be quoted within the JSON string (hence `'"express"'`). For production use on large tables, creating a multi-valued index on the tags array allows `MEMBER OF` operator queries to use an index instead of scanning the full table.
