# Functions

A stored function (also called a user-defined function or UDF in the context of stored routines) is a named block of SQL code that takes zero or more input parameters and returns a single scalar value. Unlike stored procedures, functions can be called directly within SQL expressions — in `SELECT` lists, `WHERE` clauses, `ORDER BY`, and anywhere a scalar value is valid.

## Stored Functions vs Stored Procedures

The key differences are:

- A function always returns exactly one scalar value. A procedure can return zero or more result sets and multiple `OUT` parameters.
- A function is called inline within a SQL expression: `SELECT my_func(id) FROM table`. A procedure is called with `CALL my_proc(args)` and cannot be embedded in expressions.
- A function cannot run `COMMIT` or `ROLLBACK` and cannot call procedures that do. Procedures have no such restriction.
- Functions that are `DETERMINISTIC` and do not modify data are safe for use in generated column definitions and expression indexes.

Use a function when you want to encapsulate a calculation or transformation that returns a single value and that you want to use inline in queries. Use a procedure when you need to execute multiple statements, return multiple result sets, or manage transactions.

## Creating a Function

The `DELIMITER` trick is required here just as with stored procedures:

```sql
DELIMITER //

CREATE FUNCTION calculate_tax(price DECIMAL(10,2), tax_rate DECIMAL(5,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    RETURN ROUND(price * tax_rate / 100, 2);
END //

DELIMITER ;
```

The function is then usable anywhere a scalar value is expected:

```sql
SELECT name, price, calculate_tax(price, 8.25) AS tax FROM products;
SELECT * FROM products WHERE price + calculate_tax(price, 8.25) < 50.00;
```

## DETERMINISTIC vs NOT DETERMINISTIC

Every stored function must be declared as either `DETERMINISTIC` or `NOT DETERMINISTIC`. This declaration affects replication and query optimization.

A `DETERMINISTIC` function always returns the same output for the same input — calling `add_tax(100, 10)` always returns 10. A `NOT DETERMINISTIC` function may return different values on different calls, even with the same inputs — for example, a function that uses `NOW()` or `RAND()`.

In MySQL, if you omit the declaration, the default is `NOT DETERMINISTIC`. If your function is genuinely deterministic, declare it so — MySQL can make better optimization decisions and replication is more reliable.

If binary logging is enabled (as it is on most production servers), you must declare one of: `DETERMINISTIC`, `NO SQL`, or `READS SQL DATA` when creating functions, or you will receive an error:

```sql
-- For functions that read from tables but don't modify data:
READS SQL DATA

-- For functions that don't touch the database at all (pure computation):
NO SQL

-- For functions that could return different values on different calls:
NOT DETERMINISTIC
```

## Returning Values

A function must include at least one `RETURN` statement that returns a value of the declared return type. `RETURN` immediately exits the function:

```sql
DELIMITER //

CREATE FUNCTION classify_score(score INT)
RETURNS VARCHAR(20)
DETERMINISTIC
NO SQL
BEGIN
    IF score >= 90 THEN
        RETURN 'A';
    ELSEIF score >= 80 THEN
        RETURN 'B';
    ELSEIF score >= 70 THEN
        RETURN 'C';
    ELSEIF score >= 60 THEN
        RETURN 'D';
    ELSE
        RETURN 'F';
    END IF;
END //

DELIMITER ;

SELECT student_name, exam_score, classify_score(exam_score) AS grade
FROM exam_results;
```

## Functions That Read from Tables

A function can contain `SELECT` statements to read data from tables:

```sql
DELIMITER //

CREATE FUNCTION get_customer_name(p_customer_id INT UNSIGNED)
RETURNS VARCHAR(200)
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE v_name VARCHAR(200);
    SELECT name INTO v_name FROM customers WHERE id = p_customer_id;
    RETURN COALESCE(v_name, 'Unknown');
END //

DELIMITER ;

SELECT id, get_customer_name(customer_id) AS customer FROM orders;
```

Be cautious with functions that query tables — if called in a `SELECT` that returns many rows, the function executes once per row. This can cause significant performance problems for large result sets, similar to the correlated subquery problem. Consider whether the same result can be achieved with a join.

## Dropping a Function

```sql
DROP FUNCTION IF EXISTS calculate_tax;
```

## Viewing Function Definitions

```sql
SHOW CREATE FUNCTION calculate_tax;
SHOW FUNCTION STATUS WHERE db = 'your_database';
```

## Using Functions Inside Queries

Functions integrate naturally with all query contexts:

```sql
-- In SELECT:
SELECT name, price, calculate_tax(price, 7.5) AS tax FROM products;

-- In WHERE:
SELECT name FROM products WHERE calculate_tax(price, 7.5) < 5.00;

-- In ORDER BY:
SELECT name, price FROM products ORDER BY calculate_tax(price, 7.5) DESC;

-- In expressions:
SELECT name, price + calculate_tax(price, 7.5) AS total_with_tax FROM products;

-- In INSERT:
INSERT INTO order_summary (order_id, tax_amount)
SELECT id, calculate_tax(total, 8.25) FROM orders WHERE status = 'completed';
```

## Common Mistakes

Declaring a function `DETERMINISTIC` when it is not is a correctness bug — MySQL may cache results that should vary, leading to wrong query results, especially with binary log-based replication. Be honest about the function's behavior.

Using a function on an indexed column in a `WHERE` clause prevents index use:

```sql
-- Cannot use index on created_at:
SELECT * FROM orders WHERE fiscal_quarter(created_at) = 1;

-- Better: compute the date range and use it directly:
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-03-31';
```

## Practice Problems

### Problem 1
Create a function called `full_name` that accepts `first_name` and `last_name` as `VARCHAR(100)` parameters and returns the full name formatted as 'Last, First'. Handle the case where either name might be NULL by returning only the non-NULL part.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100)
);

INSERT INTO employees (first_name, last_name) VALUES
    ('Alice', 'Smith'),
    ('Bob', NULL),
    (NULL, 'Jones'),
    ('Carol', 'Lee');
```

**Your query:**
```sql
-- Write the CREATE FUNCTION and a SELECT that uses it.
```

**Solution:**
```sql
DELIMITER //

CREATE FUNCTION full_name(p_first VARCHAR(100), p_last VARCHAR(100))
RETURNS VARCHAR(205)
DETERMINISTIC
NO SQL
BEGIN
    IF p_last IS NULL AND p_first IS NULL THEN
        RETURN NULL;
    ELSEIF p_last IS NULL THEN
        RETURN p_first;
    ELSEIF p_first IS NULL THEN
        RETURN p_last;
    ELSE
        RETURN CONCAT(p_last, ', ', p_first);
    END IF;
END //

DELIMITER ;

SELECT id, full_name(first_name, last_name) AS formatted_name FROM employees;
```

**Explanation:**
The function handles four cases: both NULL (returns NULL), only first name (returns first), only last name (returns last), and both present (returns 'Last, First'). The return type `VARCHAR(205)` accommodates up to 100 + 2 + 100 + 3 characters (first + ', ' + last, plus safety margin). `NO SQL` is correct because this function contains no SQL statements — only control flow logic. `DETERMINISTIC` is correct because the same inputs always produce the same output.

### Problem 2
Create a function called `days_since` that takes a `DATE` parameter and returns the number of days elapsed since that date as an `INT`. Then use it to find all customers who registered more than 365 days ago.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    registered_on DATE NOT NULL
);

INSERT INTO customers (name, registered_on) VALUES
    ('Alice', '2022-01-15'),
    ('Bob', '2023-06-20'),
    ('Carol', '2021-11-01'),
    ('Dave', '2024-03-10');
```

**Your query:**
```sql
-- Write the function and the query that uses it.
```

**Solution:**
```sql
DELIMITER //

CREATE FUNCTION days_since(p_date DATE)
RETURNS INT
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    RETURN DATEDIFF(CURDATE(), p_date);
END //

DELIMITER ;

-- Find customers registered more than 365 days ago:
SELECT id, name, registered_on, days_since(registered_on) AS days_ago
FROM customers
WHERE days_since(registered_on) > 365
ORDER BY days_ago DESC;
```

**Explanation:**
`DATEDIFF(CURDATE(), p_date)` returns the number of days between today and the given date. The function is `NOT DETERMINISTIC` because `CURDATE()` changes every day — calling the function on '2022-01-15' returns a different result today than it did a year ago. `READS SQL DATA` is not strictly necessary (the function only reads a built-in function, not tables), but `NOT DETERMINISTIC` is the correct and required declaration. Note that using this function in a `WHERE` clause prevents index use on `registered_on`. For performance on large tables, the direct approach `WHERE registered_on < DATE_SUB(CURDATE(), INTERVAL 365 DAY)` is better.

### Problem 3
Create a function called `discount_price` that takes a price and a customer tier ('bronze', 'silver', 'gold', 'platinum') and returns the discounted price. Bronze: 0% discount, Silver: 5%, Gold: 10%, Platinum: 20%. Then use it to generate a price list.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    base_price DECIMAL(8, 2) NOT NULL
);

INSERT INTO products (name, base_price) VALUES
    ('Widget', 50.00),
    ('Gadget', 120.00),
    ('Doohickey', 30.00);
```

**Your query:**
```sql
-- Write the function and a query that shows prices for all four tiers.
```

**Solution:**
```sql
DELIMITER //

CREATE FUNCTION discount_price(p_price DECIMAL(10,2), p_tier VARCHAR(20))
RETURNS DECIMAL(10,2)
DETERMINISTIC
NO SQL
BEGIN
    DECLARE v_discount DECIMAL(5,4) DEFAULT 0;

    CASE p_tier
        WHEN 'bronze'   THEN SET v_discount = 0.00;
        WHEN 'silver'   THEN SET v_discount = 0.05;
        WHEN 'gold'     THEN SET v_discount = 0.10;
        WHEN 'platinum' THEN SET v_discount = 0.20;
        ELSE SET v_discount = 0.00;
    END CASE;

    RETURN ROUND(p_price * (1 - v_discount), 2);
END //

DELIMITER ;

-- Price list for all tiers:
SELECT
    name,
    base_price,
    discount_price(base_price, 'bronze')   AS bronze_price,
    discount_price(base_price, 'silver')   AS silver_price,
    discount_price(base_price, 'gold')     AS gold_price,
    discount_price(base_price, 'platinum') AS platinum_price
FROM products;
```

**Explanation:**
The `CASE` statement maps tier names to discount factors stored in `v_discount`. The `ELSE` clause ensures that an unrecognized tier receives a 0% discount rather than causing an error. `ROUND(..., 2)` keeps the price to two decimal places. The query calls the function four times per row — once per tier — which is fine for a small table but would be worth replacing with a join for large datasets.
