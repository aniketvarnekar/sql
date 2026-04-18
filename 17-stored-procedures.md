# Stored Procedures

A stored procedure is a named block of SQL code stored in the database that can be called by name. Stored procedures encapsulate business logic, reduce network round trips by executing multiple statements server-side, and can accept parameters and return results. They are one of MySQL's mechanisms for moving logic closer to the data.

## Creating and Calling Procedures

Before writing a procedure, you must change the statement delimiter. MySQL normally uses `;` to end statements, but the procedure body contains statements that also end with `;`. The `DELIMITER` command changes the delimiter to something else (commonly `//` or `$$`) so that MySQL doesn't interpret the inner semicolons as the end of the `CREATE PROCEDURE` statement:

```sql
DELIMITER //

CREATE PROCEDURE get_orders_by_customer(IN p_customer_id INT UNSIGNED)
BEGIN
    SELECT id, total, status, order_date
    FROM orders
    WHERE customer_id = p_customer_id
    ORDER BY order_date DESC;
END //

DELIMITER ;
```

Call a stored procedure with `CALL`:

```sql
CALL get_orders_by_customer(42);
```

To remove a procedure:

```sql
DROP PROCEDURE IF EXISTS get_orders_by_customer;
```

To see existing procedures:

```sql
SHOW PROCEDURE STATUS WHERE db = 'your_database';
SHOW CREATE PROCEDURE get_orders_by_customer;
```

## Parameters: IN, OUT, INOUT

Stored procedures support three parameter modes.

`IN` parameters pass a value into the procedure. The procedure cannot modify the caller's variable:

```sql
DELIMITER //
CREATE PROCEDURE apply_discount(IN p_order_id INT, IN p_discount DECIMAL(5,2))
BEGIN
    UPDATE orders SET total = total * (1 - p_discount / 100) WHERE id = p_order_id;
END //
DELIMITER ;

CALL apply_discount(101, 10); -- Apply 10% discount to order 101
```

`OUT` parameters return a value from the procedure to the caller. The caller provides a variable that the procedure populates:

```sql
DELIMITER //
CREATE PROCEDURE get_order_total(IN p_order_id INT, OUT p_total DECIMAL(10,2))
BEGIN
    SELECT total INTO p_total FROM orders WHERE id = p_order_id;
END //
DELIMITER ;

CALL get_order_total(101, @total);
SELECT @total AS order_total;
```

`INOUT` parameters pass a value in and allow the procedure to modify and return it:

```sql
DELIMITER //
CREATE PROCEDURE double_value(INOUT p_value INT)
BEGIN
    SET p_value = p_value * 2;
END //
DELIMITER ;

SET @num = 5;
CALL double_value(@num);
SELECT @num; -- Returns 10
```

## Variables: DECLARE and SET

Local variables inside a procedure are declared with `DECLARE` at the beginning of the `BEGIN...END` block, before any other statements:

```sql
DELIMITER //
CREATE PROCEDURE summarize_orders(IN p_customer_id INT)
BEGIN
    DECLARE v_order_count INT DEFAULT 0;
    DECLARE v_total_spent DECIMAL(10,2) DEFAULT 0.00;
    DECLARE v_last_order DATE;

    SELECT COUNT(*), SUM(total), MAX(order_date)
    INTO v_order_count, v_total_spent, v_last_order
    FROM orders
    WHERE customer_id = p_customer_id;

    SELECT
        p_customer_id AS customer_id,
        v_order_count AS order_count,
        v_total_spent AS total_spent,
        v_last_order AS last_order_date;
END //
DELIMITER ;
```

`SET` assigns a value to a variable. `SELECT ... INTO variable` populates a variable from a query result.

## Conditional Logic: IF, ELSEIF, ELSE

```sql
DELIMITER //
CREATE PROCEDURE classify_order(IN p_total DECIMAL(10,2), OUT p_tier VARCHAR(20))
BEGIN
    IF p_total >= 1000 THEN
        SET p_tier = 'Premium';
    ELSEIF p_total >= 500 THEN
        SET p_tier = 'Standard';
    ELSEIF p_total >= 100 THEN
        SET p_tier = 'Basic';
    ELSE
        SET p_tier = 'Micro';
    END IF;
END //
DELIMITER ;

CALL classify_order(750.00, @tier);
SELECT @tier; -- 'Standard'
```

`CASE` is an alternative to `IF/ELSEIF` for matching against specific values:

```sql
DELIMITER //
CREATE PROCEDURE describe_status(IN p_status VARCHAR(20), OUT p_description VARCHAR(100))
BEGIN
    CASE p_status
        WHEN 'pending' THEN SET p_description = 'Awaiting processing';
        WHEN 'completed' THEN SET p_description = 'Order fulfilled';
        WHEN 'cancelled' THEN SET p_description = 'Order cancelled by customer';
        ELSE SET p_description = 'Unknown status';
    END CASE;
END //
DELIMITER ;
```

## Loops: LOOP, WHILE, REPEAT

`WHILE` executes its body while a condition is true:

```sql
DELIMITER //
CREATE PROCEDURE count_down(IN p_start INT)
BEGIN
    DECLARE v_counter INT DEFAULT p_start;
    WHILE v_counter > 0 DO
        -- In a real procedure, you'd do something useful here
        SET v_counter = v_counter - 1;
    END WHILE;
END //
DELIMITER ;
```

`REPEAT` executes its body at least once, then checks the condition:

```sql
DELIMITER //
CREATE PROCEDURE repeat_example()
BEGIN
    DECLARE v_i INT DEFAULT 0;
    REPEAT
        SET v_i = v_i + 1;
    UNTIL v_i >= 5
    END REPEAT;
END //
DELIMITER ;
```

`LOOP` with `LEAVE` provides explicit loop control (similar to a `while(true)` with `break`):

```sql
DELIMITER //
CREATE PROCEDURE loop_example()
BEGIN
    DECLARE v_i INT DEFAULT 0;
    my_loop: LOOP
        SET v_i = v_i + 1;
        IF v_i >= 5 THEN
            LEAVE my_loop;
        END IF;
    END LOOP my_loop;
END //
DELIMITER ;
```

`ITERATE` is the equivalent of `continue` — it skips to the next iteration.

## Error Handling with DECLARE ... HANDLER

Stored procedures can catch SQL errors and exceptions using handlers:

```sql
DELIMITER //
CREATE PROCEDURE safe_insert_customer(
    IN p_email VARCHAR(255),
    IN p_name VARCHAR(200),
    OUT p_success BOOLEAN
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        SET p_success = FALSE;
        ROLLBACK;
    END;

    START TRANSACTION;
    INSERT INTO customers (email, name) VALUES (p_email, p_name);
    COMMIT;
    SET p_success = TRUE;
END //
DELIMITER ;
```

`DECLARE EXIT HANDLER FOR SQLEXCEPTION` catches any SQL exception. `EXIT` means the handler exits the current block after executing. `CONTINUE` would let execution resume after the statement that caused the error. `FOR SQLEXCEPTION` matches any unhandled SQL error; you can also handle specific SQLSTATE codes:

```sql
DECLARE CONTINUE HANDLER FOR SQLSTATE '23000'
BEGIN
    -- Handle duplicate key violation specifically
    SET p_message = 'Duplicate key error';
END;
```

## Practice Problems

### Problem 1
Create a stored procedure called `transfer_funds` that accepts a source account id, destination account id, and transfer amount. It should deduct the amount from the source and add it to the destination within a transaction, and return an error message if the source account has insufficient funds.

**Schema used:**
```sql
CREATE TABLE accounts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    owner_name VARCHAR(200) NOT NULL,
    balance DECIMAL(12, 2) NOT NULL DEFAULT 0.00
);

INSERT INTO accounts (owner_name, balance) VALUES
    ('Alice', 1000.00),
    ('Bob', 500.00);
```

**Your query:**
```sql
-- Write the stored procedure.
```

**Solution:**
```sql
DELIMITER //

CREATE PROCEDURE transfer_funds(
    IN p_from_id INT UNSIGNED,
    IN p_to_id INT UNSIGNED,
    IN p_amount DECIMAL(12, 2),
    OUT p_result VARCHAR(100)
)
BEGIN
    DECLARE v_balance DECIMAL(12, 2);

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_result = 'Transfer failed due to an error';
    END;

    START TRANSACTION;

    SELECT balance INTO v_balance FROM accounts WHERE id = p_from_id FOR UPDATE;

    IF v_balance < p_amount THEN
        ROLLBACK;
        SET p_result = 'Insufficient funds';
    ELSE
        UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_id;
        UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_id;
        COMMIT;
        SET p_result = 'Transfer successful';
    END IF;
END //

DELIMITER ;

-- Test:
CALL transfer_funds(1, 2, 200.00, @result);
SELECT @result;
SELECT * FROM accounts;
```

**Explanation:**
`FOR UPDATE` locks the source account row during the transaction, preventing concurrent transfers from reading a stale balance. The balance check happens inside the transaction to avoid a time-of-check/time-of-use race condition. If the balance is insufficient, we `ROLLBACK` and set the output parameter to an error message. If the transfer succeeds, we `COMMIT`. The `EXIT HANDLER FOR SQLEXCEPTION` catches unexpected errors and rolls back to avoid partial transfers (money leaving one account without arriving in the other).

### Problem 2
Create a stored procedure that generates a report of monthly order counts and totals for a given year, using a cursor to iterate over months or a loop with numbered months.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    total DECIMAL(10, 2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO orders (total, order_date) VALUES
    (100.00, '2023-01-15'), (200.00, '2023-01-28'),
    (300.00, '2023-02-10'), (150.00, '2023-03-05');
```

**Your query:**
```sql
-- Write the stored procedure.
```

**Solution:**
```sql
DELIMITER //

CREATE PROCEDURE monthly_report(IN p_year INT)
BEGIN
    SELECT
        MONTH(order_date) AS month_number,
        MONTHNAME(order_date) AS month_name,
        COUNT(*) AS order_count,
        ROUND(SUM(total), 2) AS monthly_total
    FROM orders
    WHERE YEAR(order_date) = p_year
    GROUP BY MONTH(order_date), MONTHNAME(order_date)
    ORDER BY month_number;
END //

DELIMITER ;

CALL monthly_report(2023);
```

**Explanation:**
This procedure accepts a year parameter and returns a grouped summary. Stored procedures are not limited to complex procedural logic — they can encapsulate even simple parameterized queries. Calling `CALL monthly_report(2023)` is cleaner and more maintainable than copying the query everywhere it is needed. If the report format needs to change, you update it in one place.

### Problem 3
Create a stored procedure that accepts a category name and a price adjustment percentage (positive to increase, negative to decrease) and updates the prices of all products in that category. It should return the number of rows affected and validate that the adjustment percentage is between -50 and 50.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(100) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);

INSERT INTO products (name, category, price) VALUES
    ('Book A', 'Books', 15.00), ('Book B', 'Books', 20.00),
    ('Widget', 'Hardware', 9.99);
```

**Your query:**
```sql
-- Write the stored procedure.
```

**Solution:**
```sql
DELIMITER //

CREATE PROCEDURE adjust_category_prices(
    IN p_category VARCHAR(100),
    IN p_adjustment_pct DECIMAL(5, 2),
    OUT p_rows_affected INT
)
BEGIN
    IF p_adjustment_pct < -50 OR p_adjustment_pct > 50 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Adjustment percentage must be between -50 and 50';
    END IF;

    UPDATE products
    SET price = ROUND(price * (1 + p_adjustment_pct / 100), 2)
    WHERE category = p_category;

    SET p_rows_affected = ROW_COUNT();
END //

DELIMITER ;

CALL adjust_category_prices('Books', 10, @affected);
SELECT @affected; -- 2
SELECT * FROM products WHERE category = 'Books';
```

**Explanation:**
`SIGNAL SQLSTATE '45000'` is MySQL's mechanism for raising a custom error from within a procedure. SQLSTATE '45000' is the conventional code for user-defined exceptions. If the adjustment is out of range, the `SIGNAL` causes the procedure to raise an error before modifying any data. `ROW_COUNT()` returns the number of rows affected by the most recent `UPDATE`, `INSERT`, or `DELETE`. `SET p_rows_affected = ROW_COUNT()` captures this value for the caller.
