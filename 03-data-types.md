# Data Types

Choosing the right data type for each column is one of the most consequential decisions in database design. The data type determines what values a column can hold, how much storage each value requires, and what operations can be performed efficiently. Using the wrong data type leads to wasted storage, incorrect calculations, or queries that cannot use indexes effectively.

## Numeric Types

### Integer Types

MySQL offers several integer types that differ only in their storage size and the range of values they can hold.

`TINYINT` uses 1 byte of storage and holds values from -128 to 127 (signed) or 0 to 255 (unsigned).

`SMALLINT` uses 2 bytes and holds values from -32,768 to 32,767 (signed) or 0 to 65,535 (unsigned).

`MEDIUMINT` uses 3 bytes and holds values from -8,388,608 to 8,388,607 (signed).

`INT` (also written as `INTEGER`) uses 4 bytes and holds values from approximately -2.1 billion to 2.1 billion (signed) or 0 to approximately 4.3 billion (unsigned). This is the most commonly used integer type and the right default for most id columns and counts.

`BIGINT` uses 8 bytes and holds values from approximately -9.2 quintillion to 9.2 quintillion. Use this when you have a genuine need for numbers this large — for example, a global unique identifier or a financial system that tracks quantities in the smallest fractional units.

The `UNSIGNED` modifier restricts a column to non-negative values and doubles the maximum positive range. Use it when negative values are logically impossible, such as for a `quantity` or `age` column.

```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    quantity SMALLINT UNSIGNED NOT NULL DEFAULT 0,
    sort_order TINYINT DEFAULT 0
);
```

A historical note: MySQL allowed specifying a display width for integers, like `INT(11)`. This display width did not affect the storage size or range — it was a hint to some client tools for how to pad the display. MySQL 8.0.17 deprecated this syntax and it was removed in MySQL 8.0.19. Do not use it in new code.

### Decimal and Floating-Point Types

`DECIMAL(precision, scale)` (also written as `NUMERIC`) stores exact numeric values. `precision` is the total number of significant digits, and `scale` is the number of digits after the decimal point. For example, `DECIMAL(10, 2)` can store numbers up to 99,999,999.99. Use `DECIMAL` for monetary values, measurements, and any situation where exact arithmetic is required.

`FLOAT` is a 4-byte single-precision floating-point number. `DOUBLE` is an 8-byte double-precision floating-point number. Both are approximate — they cannot represent all decimal fractions exactly. The value 0.1 stored as a `FLOAT` is actually something like 0.100000001490116119384765625 internally. This makes them unsuitable for financial calculations but acceptable for scientific measurements where a small margin of error is tolerable.

```sql
CREATE TABLE prices (
    amount DECIMAL(12, 2) NOT NULL,   -- exact: use for money
    measurement DOUBLE,                -- approximate: use for physics
    temperature FLOAT                  -- approximate: acceptable for sensor data
);
```

## String Types

### CHAR and VARCHAR

`CHAR(n)` stores a fixed-length string of exactly `n` characters, padding with spaces if the stored value is shorter. `n` can be between 1 and 255. Because the length is fixed, `CHAR` is slightly faster to read and write for columns where values are always the same length, such as a 2-letter country code or a fixed-format code.

`VARCHAR(n)` stores a variable-length string of up to `n` characters (up to 65,535 bytes in a row, subject to the row format and character set). MySQL stores the actual length alongside the value, so a `VARCHAR(255)` column storing the value `'hi'` uses only 3 bytes (2 for the characters plus 1 for the length prefix), not 255. Use `VARCHAR` for most string columns where values vary in length.

```sql
CREATE TABLE countries (
    code CHAR(2) NOT NULL,        -- always exactly 2 characters
    name VARCHAR(100) NOT NULL    -- variable length
);
```

### TEXT Types

`TEXT` (and its variants `TINYTEXT`, `MEDIUMTEXT`, `LONGTEXT`) are designed for large text values. Unlike `VARCHAR`, a `TEXT` column's contents are stored outside the main row when they are large enough, and you cannot specify a default value for a `TEXT` column. Use `TEXT` for blog post bodies, comments, and other long-form content.

- `TINYTEXT`: up to 255 bytes
- `TEXT`: up to 65,535 bytes (~64 KB)
- `MEDIUMTEXT`: up to 16,777,215 bytes (~16 MB)
- `LONGTEXT`: up to 4,294,967,295 bytes (~4 GB)

### ENUM

`ENUM('value1', 'value2', ...)` defines a column that can only contain one of a predefined set of string values. MySQL stores the value internally as a small integer (1 or 2 bytes), making it compact and fast to compare. The list of valid values is fixed at table creation time.

```sql
CREATE TABLE orders (
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') NOT NULL DEFAULT 'pending'
);
```

`ENUM` is convenient for columns with a small, fixed set of allowed values. However, adding a new value requires an `ALTER TABLE`, which can be slow on large tables. For high-change lists of values, a separate lookup table with a foreign key is more flexible.

## Date and Time Types

`DATE` stores a calendar date with no time component, in the format `YYYY-MM-DD`. The range is '1000-01-01' to '9999-12-31'.

`TIME` stores a time of day (or a duration) in the format `HH:MM:SS`. The range is '-838:59:59' to '838:59:59', making it suitable for time durations as well as times of day.

`DATETIME` stores both a date and a time in the format `YYYY-MM-DD HH:MM:SS`. The range is '1000-01-01 00:00:00' to '9999-12-31 23:59:59'. MySQL stores `DATETIME` without time zone information — the value you put in is the value you get back, regardless of the server's time zone setting.

`TIMESTAMP` also stores a date and time, but with an important difference: MySQL converts `TIMESTAMP` values to UTC when storing and converts back to the current session time zone when retrieving. The range is '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC. `TIMESTAMP` is useful for recording when a record was created or modified, as the stored UTC value can be correctly displayed in any time zone. However, the year 2038 limitation is a real concern for long-lived systems.

`YEAR` stores a 4-digit year, from 1901 to 2155. Use it when you only need the year, such as for a `published_year` on a books table.

```sql
CREATE TABLE events (
    id INT PRIMARY KEY AUTO_INCREMENT,
    event_date DATE NOT NULL,
    start_time TIME NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

The `ON UPDATE CURRENT_TIMESTAMP` modifier automatically sets the column to the current timestamp whenever the row is updated, which is a very common pattern for `updated_at` columns.

## Boolean Handling in MySQL

MySQL does not have a native `BOOLEAN` or `BOOL` data type. Instead, it uses `TINYINT(1)` as a synonym for boolean values. When you declare a column as `BOOLEAN` or `BOOL`, MySQL silently converts it to `TINYINT(1)`. The value `1` is treated as true and `0` as false, though any non-zero value is also truthy.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    is_admin BOOL NOT NULL DEFAULT FALSE
);

-- The above is identical to:
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    is_active TINYINT(1) NOT NULL DEFAULT 1,
    is_admin TINYINT(1) NOT NULL DEFAULT 0
);
```

You can use the literals `TRUE` and `FALSE` in queries, but MySQL converts them to `1` and `0` internally. Be aware that a `TINYINT(1)` column can technically store any value from -128 to 127 — MySQL does not enforce that only 0 and 1 are stored. Application code must maintain this constraint.

## Choosing the Right Data Type

The goal of type selection is to use the smallest type that can correctly represent all possible values and that supports the operations you need. Smaller types use less storage, fit more rows in a buffer page, and can make indexes more efficient.

Use `INT` for most id columns, but consider `BIGINT` if you expect more than 2 billion records. Use `VARCHAR` for most text columns and `CHAR` only when values are always a fixed length. Use `DECIMAL` for money and `FLOAT`/`DOUBLE` for scientific measurements. Use `TIMESTAMP` for created/updated audit columns, `DATETIME` when you need dates beyond 2038, and `DATE` when you only need the date without the time.

Avoid storing numbers in `VARCHAR` columns. Arithmetic on `VARCHAR` values is slower, sorting is alphabetical rather than numeric (so '10' sorts before '9'), and indexes are far less efficient. Similarly, avoid storing dates as `VARCHAR` strings — you lose all date arithmetic, sorting, and indexing benefits.

## Practice Problems

### Problem 1
Design a table called `employees` with the following requirements: a unique integer id, a first name and last name (each up to 100 characters), an email address (up to 255 characters, must be unique), a salary stored to two decimal places and up to $9,999,999.99, a hire date, whether the employee is currently active, and a status that can only be 'full-time', 'part-time', or 'contractor'.

**Schema used:**
```sql
-- No pre-existing schema needed.
```

**Your query:**
```sql
-- Write the CREATE TABLE statement here.
```

**Solution:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    salary DECIMAL(9, 2) NOT NULL,
    hire_date DATE NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    employment_status ENUM('full-time', 'part-time', 'contractor') NOT NULL DEFAULT 'full-time'
);
```

**Explanation:**
`INT UNSIGNED AUTO_INCREMENT` is the standard pattern for a surrogate primary key on a table that won't exceed 4 billion rows. `VARCHAR(100)` is appropriate for names — it doesn't waste space on shorter names. `DECIMAL(9, 2)` can store values up to 9,999,999.99, which covers the stated maximum salary. `DATE` is correct for `hire_date` because we don't need the time of day. `BOOLEAN` maps to `TINYINT(1)` internally. `ENUM` is compact and self-documenting for a small fixed set of categories.

### Problem 2
A developer stored order totals as `FLOAT`. Write a query that demonstrates why this is a problem for financial data, and then show the correct approach using `DECIMAL`.

**Schema used:**
```sql
CREATE TABLE float_test (
    amount FLOAT
);

CREATE TABLE decimal_test (
    amount DECIMAL(10, 2)
);

INSERT INTO float_test VALUES (0.1), (0.2);
INSERT INTO decimal_test VALUES (0.1), (0.2);
```

**Your query:**
```sql
-- Show the sum of the two values in each table.
```

**Solution:**
```sql
-- With FLOAT (may show imprecision):
SELECT SUM(amount) AS float_sum FROM float_test;
-- Result might be: 0.30000001192092896

-- With DECIMAL (exact):
SELECT SUM(amount) AS decimal_sum FROM decimal_test;
-- Result is exactly: 0.30
```

**Explanation:**
Floating-point types cannot exactly represent many decimal fractions because they store values in binary, and many common decimal values (like 0.1) have no exact binary representation. The result of 0.1 + 0.2 in a `FLOAT` column may be something like 0.30000001192092896 rather than exactly 0.30. For financial data, even a tiny rounding error can accumulate across millions of transactions into a significant discrepancy. `DECIMAL` stores values as exact strings of decimal digits, so 0.1 + 0.2 is always exactly 0.30.

### Problem 3
Explain what happens when you store the timestamp '2038-02-01 00:00:00' in a `TIMESTAMP` column versus a `DATETIME` column in MySQL.

**Schema used:**
```sql
CREATE TABLE ts_test (
    created_timestamp TIMESTAMP NULL,
    created_datetime DATETIME NULL
);
```

**Your query:**
```sql
INSERT INTO ts_test VALUES ('2038-02-01 00:00:00', '2038-02-01 00:00:00');
SELECT * FROM ts_test;
```

**Solution:**
```sql
-- The TIMESTAMP column will produce an error or store 0000-00-00 00:00:00
-- because 2038-02-01 is beyond the TIMESTAMP range of 2038-01-19 03:14:07 UTC.
-- The DATETIME column will store the value correctly.

-- To safely store dates beyond 2038, use DATETIME:
CREATE TABLE future_events (
    event_name VARCHAR(200),
    event_at DATETIME NOT NULL   -- range: up to 9999-12-31
);

-- For audit timestamps that need timezone awareness AND stay within range,
-- use TIMESTAMP. For flexibility, use DATETIME and handle timezone in app code.
```

**Explanation:**
The `TIMESTAMP` type in MySQL is stored internally as the number of seconds since the Unix epoch (1970-01-01 00:00:00 UTC), using a 32-bit signed integer. The maximum value of a 32-bit signed integer corresponds to 2038-01-19 03:14:07 UTC — the so-called "Year 2038 Problem." Any `TIMESTAMP` value beyond this date cannot be stored. `DATETIME` does not have this limitation because it stores the date and time as a literal value without reference to the Unix epoch, supporting dates up to '9999-12-31 23:59:59'. For new systems with long expected lifetimes, prefer `DATETIME` for date columns where 2038 could be an issue.
