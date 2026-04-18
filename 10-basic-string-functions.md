# Basic String Functions

MySQL includes a rich library of string functions for manipulating, inspecting, and transforming text values. These functions are used extensively in `SELECT` lists, `WHERE` conditions, and `ORDER BY` clauses. Understanding the core string functions gives you the ability to clean and format data directly in SQL without post-processing in application code.

## Concatenation: CONCAT and CONCAT_WS

`CONCAT(str1, str2, ...)` joins two or more strings into one. If any argument is `NULL`, the result is `NULL`:

```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;
-- 'Alice' + ' ' + 'Smith' = 'Alice Smith'

SELECT CONCAT('Order #', id) AS order_label FROM orders;
```

`CONCAT_WS(separator, str1, str2, ...)` joins strings with a separator between each one. Unlike `CONCAT`, it ignores `NULL` arguments rather than returning `NULL`:

```sql
SELECT CONCAT_WS(', ', city, state, country) AS address FROM locations;
-- 'New York, NY, USA'

-- If state is NULL: 'New York, USA'  (NULL is skipped, not included)
```

`CONCAT_WS` is the safer choice when any of the arguments might be `NULL`, which is common with optional fields like middle names or address lines.

## String Length: LENGTH and CHAR_LENGTH

`LENGTH(str)` returns the number of bytes in a string, not the number of characters. For multi-byte character sets like UTF-8, a single character can be 1 to 4 bytes:

```sql
SELECT LENGTH('hello');        -- 5 (5 bytes)
SELECT LENGTH('héllo');        -- 6 (the é character is 2 bytes in UTF-8)
SELECT LENGTH('emoji: 😀');    -- 11 (emoji is 4 bytes in UTF-8)
```

`CHAR_LENGTH(str)` (also written as `CHARACTER_LENGTH`) returns the number of characters, regardless of how many bytes each character uses:

```sql
SELECT CHAR_LENGTH('hello');       -- 5
SELECT CHAR_LENGTH('héllo');       -- 5
SELECT CHAR_LENGTH('emoji: 😀');   -- 8
```

For applications working with multilingual text or emoji, always use `CHAR_LENGTH` to count characters. Use `LENGTH` when you care about actual byte size, such as when checking whether a value fits within a binary field.

## Case Conversion: UPPER and LOWER

`UPPER(str)` converts all characters to uppercase. `LOWER(str)` converts all characters to lowercase:

```sql
SELECT UPPER('hello world');   -- 'HELLO WORLD'
SELECT LOWER('Hello World');   -- 'hello world'

-- Normalize email addresses for comparison:
SELECT * FROM users WHERE LOWER(email) = LOWER('User@Example.COM');
```

Using `LOWER()` or `UPPER()` in a `WHERE` clause forces MySQL to compute the function for every row during the query, which prevents index use on the column. For frequently queried columns like `email`, it is better to store the value in a consistent case (always lowercase) than to convert at query time.

## Whitespace Trimming: TRIM, LTRIM, RTRIM

`TRIM(str)` removes leading and trailing spaces from a string:

```sql
SELECT TRIM('  hello  ');   -- 'hello'
```

`LTRIM(str)` removes only leading (left) spaces. `RTRIM(str)` removes only trailing (right) spaces:

```sql
SELECT LTRIM('  hello  ');  -- 'hello  '
SELECT RTRIM('  hello  ');  -- '  hello'
```

`TRIM` has an extended form that removes specific characters:

```sql
SELECT TRIM(LEADING '0' FROM '000123');   -- '123'
SELECT TRIM(TRAILING '.' FROM '3.14.');   -- '3.14'
SELECT TRIM(BOTH 'x' FROM 'xxxhelloxxx'); -- 'hello'
```

Trimming is useful for cleaning up user-entered data, parsing fixed-format strings, and normalizing imported data.

## Substring Extraction: SUBSTRING, LEFT, RIGHT

`SUBSTRING(str, pos, len)` (also written as `SUBSTR`) extracts a portion of a string. `pos` is the starting position (1-indexed, not 0-indexed). `len` is the number of characters to extract:

```sql
SELECT SUBSTRING('Hello World', 7, 5);  -- 'World'
SELECT SUBSTRING('Hello World', 7);     -- 'World' (to end of string)
SELECT SUBSTRING('Hello World', -5);    -- 'World' (negative pos counts from end)
```

`LEFT(str, n)` returns the leftmost `n` characters. `RIGHT(str, n)` returns the rightmost `n` characters:

```sql
SELECT LEFT('Hello World', 5);   -- 'Hello'
SELECT RIGHT('Hello World', 5);  -- 'World'

-- Extract year from a date string:
SELECT LEFT('2024-03-15', 4);    -- '2024'
```

## String Replacement: REPLACE

`REPLACE(str, from_str, to_str)` replaces all occurrences of `from_str` with `to_str` in `str`. It is case-sensitive:

```sql
SELECT REPLACE('Hello World', 'World', 'MySQL');  -- 'Hello MySQL'
SELECT REPLACE('aababab', 'ab', 'X');              -- 'aXXX'

-- Clean up phone numbers:
SELECT REPLACE(REPLACE(REPLACE(phone, '-', ''), '(', ''), ')', '') AS clean_phone
FROM contacts;
```

## Finding Position: INSTR and LOCATE

`INSTR(str, substr)` returns the position (1-indexed) of the first occurrence of `substr` in `str`, or 0 if not found:

```sql
SELECT INSTR('Hello World', 'World');   -- 7
SELECT INSTR('Hello World', 'xyz');     -- 0
SELECT INSTR('abcabc', 'bc');           -- 2 (first occurrence)
```

`LOCATE(substr, str)` is similar but with arguments in the opposite order, and accepts an optional starting position:

```sql
SELECT LOCATE('World', 'Hello World');     -- 7
SELECT LOCATE('bc', 'abcabc', 3);          -- 5 (search starting at position 3)
```

Use `INSTR` or `LOCATE > 0` as a more flexible alternative to `LIKE` when you need to know where a substring appears, not just whether it appears.

## Additional Useful Functions

`REPEAT(str, count)` repeats a string `count` times:

```sql
SELECT REPEAT('ab', 3);   -- 'ababab'
```

`REVERSE(str)` reverses the characters of a string:

```sql
SELECT REVERSE('hello');  -- 'olleh'
```

`LPAD(str, len, padstr)` and `RPAD(str, len, padstr)` pad a string to a target length with a specified character:

```sql
SELECT LPAD('42', 6, '0');   -- '000042'  (zero-padding a number)
SELECT RPAD('hello', 10, '.'); -- 'hello.....'
```

`FORMAT(number, decimals)` formats a number with commas as thousands separators:

```sql
SELECT FORMAT(1234567.891, 2);   -- '1,234,567.89'
```

## Performance Considerations

Applying string functions to indexed columns in a `WHERE` clause prevents MySQL from using the index on that column:

```sql
-- Cannot use index on 'name':
SELECT * FROM customers WHERE UPPER(name) = 'ALICE SMITH';

-- Can use index on 'name':
SELECT * FROM customers WHERE name = 'Alice Smith';
```

For case-insensitive searches, store data in a normalized case, or use a case-insensitive collation (which MySQL's default already is). For substring searches (`LIKE '%pattern%'`), use a `FULLTEXT` index for large tables rather than relying on string functions in `WHERE`.

## Practice Problems

### Problem 1
Write a query against a `customers` table that returns a single column called `display_name` which combines `first_name` and `last_name` with a space, converts the result to uppercase, and trims any extra whitespace from the individual name parts before combining them.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100)
);

INSERT INTO customers (first_name, last_name) VALUES
    ('  Alice  ', 'Smith'),
    ('Bob', '  Jones  '),
    ('Carol', 'Lee');
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT UPPER(CONCAT_WS(' ', TRIM(first_name), TRIM(last_name))) AS display_name
FROM customers;
```

**Explanation:**
`TRIM` is applied to each name part individually before concatenation, removing any surrounding spaces from the raw data. `CONCAT_WS(' ', ...)` joins the trimmed parts with a single space separator — and crucially, if either name were NULL, `CONCAT_WS` would skip the NULL rather than returning NULL for the whole expression. Finally, `UPPER` converts the result to uppercase. The functions nest cleanly because each returns a string that the outer function can accept.

### Problem 2
A `products` table stores SKUs in the format 'CAT-SUBCAT-NNNN' (e.g., 'ELE-COM-0042'). Write a query that extracts and returns the category part (before the first hyphen), the subcategory part (between the two hyphens), and the numeric code (after the second hyphen) as separate columns.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sku VARCHAR(50) NOT NULL,
    name VARCHAR(200) NOT NULL
);

INSERT INTO products (sku, name) VALUES
    ('ELE-COM-0042', 'Computer Monitor'),
    ('CLO-SHI-0103', 'Cotton T-Shirt'),
    ('FUR-CHA-0007', 'Office Chair');
```

**Your query:**
```sql
-- Write the SELECT that extracts category, subcategory, and code.
```

**Solution:**
```sql
SELECT
    sku,
    SUBSTRING(sku, 1, 3) AS category,
    SUBSTRING(sku, 5, 3) AS subcategory,
    RIGHT(sku, 4) AS code
FROM products;

-- Alternatively, using LOCATE for more robustness:
SELECT
    sku,
    LEFT(sku, LOCATE('-', sku) - 1) AS category,
    SUBSTRING(sku,
        LOCATE('-', sku) + 1,
        LOCATE('-', sku, LOCATE('-', sku) + 1) - LOCATE('-', sku) - 1
    ) AS subcategory,
    RIGHT(sku, CHAR_LENGTH(sku) - LOCATE('-', sku, LOCATE('-', sku) + 1)) AS code
FROM products;
```

**Explanation:**
The first approach works when the format is strictly fixed (3 characters, hyphen, 3 characters, hyphen, 4 characters). `SUBSTRING(sku, 1, 3)` extracts the first 3 characters (category), `SUBSTRING(sku, 5, 3)` starts after 'ELE-' (positions 1-4) and takes 3 characters (subcategory), and `RIGHT(sku, 4)` takes the last 4 characters (the numeric code). The second approach using `LOCATE` is more robust if the part lengths might vary — it finds the actual positions of the hyphens dynamically. For production use with strict SKU formats, the simpler positional approach is clear and fast.

### Problem 3
Write a query that finds all products whose name contains the word 'Pro' (case-insensitive) and returns the product name, the position of 'Pro' within the name, and the name with 'Pro' replaced by 'Professional'.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

INSERT INTO products (name) VALUES
    ('Widget Pro'),
    ('Pro Display Monitor'),
    ('Gadget Lite'),
    ('ProMax Camera'),
    ('Basic Keyboard');
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    name,
    INSTR(LOWER(name), 'pro') AS pro_position,
    REPLACE(name, 'Pro', 'Professional') AS expanded_name
FROM products
WHERE LOWER(name) LIKE '%pro%';
```

**Explanation:**
`WHERE LOWER(name) LIKE '%pro%'` finds rows where the name contains 'pro' in any case. `INSTR(LOWER(name), 'pro')` returns the 1-based position of 'pro' in the lowercased name — applying `LOWER` to both sides makes the position search case-insensitive. `REPLACE(name, 'Pro', 'Professional')` replaces 'Pro' with 'Professional'. Note that `REPLACE` is case-sensitive, so it replaces 'Pro' exactly — 'ProMax' becomes 'ProfessionalMax' but 'PRO' would not be replaced. The query returns Widget Pro, Pro Display Monitor, and ProMax Camera but not Gadget Lite or Basic Keyboard.
