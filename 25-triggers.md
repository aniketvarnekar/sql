# Triggers

A trigger is a named database object that automatically executes a block of SQL code in response to a specific event on a table — an `INSERT`, `UPDATE`, or `DELETE`. Triggers run silently, without any explicit call from application code, making them powerful for enforcing cross-table consistency rules and maintaining audit records. They are also one of the most easily misused features in MySQL, so understanding their limitations is as important as knowing their syntax.

## What Triggers Are and When to Use Them

Triggers are appropriate when you need logic to execute automatically every time certain data changes, regardless of which application or user made the change. Common legitimate use cases include:

- Maintaining a denormalized aggregate (such as an order count on a customer table)
- Writing to an audit or history table whenever a sensitive record changes
- Enforcing cross-table business rules that cannot be expressed as a simple constraint
- Synchronizing related data that would otherwise require application-level coordination

Triggers are less appropriate for complex business logic, sending notifications, or anything that should be visible and testable from application code. Hidden side effects in triggers make systems harder to debug and understand.

## CREATE TRIGGER

```sql
CREATE TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON table_name
FOR EACH ROW
BEGIN
    -- trigger body
END;
```

The `DELIMITER` change is required when defining a trigger in the CLI, just as with stored procedures:

```sql
DELIMITER //

CREATE TRIGGER after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    UPDATE customers
    SET order_count = order_count + 1
    WHERE id = NEW.customer_id;
END //

DELIMITER ;
```

## BEFORE and AFTER Triggers

`BEFORE` triggers fire before the row modification occurs. They can modify the values being written (using `SET NEW.column = ...`) and can raise errors to prevent the operation. Use `BEFORE` triggers for validation and data transformation.

`AFTER` triggers fire after the row modification has been committed to the table. They cannot modify the row being inserted/updated (the modification already happened), but they can perform secondary actions like writing to an audit table or updating aggregate counts. Use `AFTER` triggers for side effects that depend on the change having succeeded.

## INSERT, UPDATE, DELETE Triggers

A trigger is tied to exactly one event type. You need separate triggers for `INSERT`, `UPDATE`, and `DELETE` if you want to handle all three.

```sql
DELIMITER //

CREATE TRIGGER before_order_update
BEFORE UPDATE ON orders
FOR EACH ROW
BEGIN
    IF NEW.total < 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Order total cannot be negative';
    END IF;
END //

CREATE TRIGGER after_order_delete
AFTER DELETE ON orders
FOR EACH ROW
BEGIN
    UPDATE customers
    SET order_count = order_count - 1
    WHERE id = OLD.customer_id;
END //

DELIMITER ;
```

## OLD and NEW Row References

Inside trigger bodies, `NEW` and `OLD` are pseudo-records that refer to the row being modified:

- In `INSERT` triggers: `NEW` contains the values being inserted. `OLD` does not exist.
- In `UPDATE` triggers: `NEW` contains the new values, `OLD` contains the original values before the update.
- In `DELETE` triggers: `OLD` contains the values of the row being deleted. `NEW` does not exist.

```sql
DELIMITER //

CREATE TRIGGER after_salary_change
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
    IF OLD.salary != NEW.salary THEN
        INSERT INTO salary_audit (employee_id, old_salary, new_salary, changed_at)
        VALUES (NEW.id, OLD.salary, NEW.salary, NOW());
    END IF;
END //

DELIMITER ;
```

In `BEFORE INSERT` and `BEFORE UPDATE` triggers, you can modify the `NEW` values before they are written:

```sql
DELIMITER //

CREATE TRIGGER before_user_insert
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
    -- Normalize email to lowercase before storing
    SET NEW.email = LOWER(NEW.email);
    -- Set created_at if not provided
    IF NEW.created_at IS NULL THEN
        SET NEW.created_at = NOW();
    END IF;
END //

DELIMITER ;
```

## Dropping a Trigger

```sql
DROP TRIGGER IF EXISTS after_order_insert;
```

To view existing triggers:

```sql
SHOW TRIGGERS FROM database_name;
SHOW CREATE TRIGGER after_order_insert;
```

## Limitations and Risks of Triggers

**Triggers cannot call stored procedures that use transactions** (`COMMIT`, `ROLLBACK`) — they run within the triggering statement's transaction.

**Triggers cannot fire other triggers** — MySQL does not support cascading triggers where a trigger on table A fires a trigger on table B. (The trigger body can modify table B, but if table B has its own triggers, those do fire — and MySQL does support this cascade. However, direct trigger-on-trigger nesting is limited and can produce infinite loops if not designed carefully.)

**Triggers are invisible in application code.** A developer reading the application code for an order insertion will not see the trigger updating the customer's `order_count`. This hidden behavior makes debugging harder and can surprise new team members.

**Performance impact:** Triggers add overhead to every affected DML statement. A trigger that writes to an audit table on every `UPDATE` doubles the write work for that table. On high-volume tables, this can be significant.

**Triggers do not fire for TRUNCATE.** `TRUNCATE TABLE` removes all rows without firing `DELETE` triggers. If your triggers maintain audit logs or aggregate counts, `TRUNCATE` will silently bypass them.

**Triggers cannot be disabled temporarily** without dropping them, which makes data migrations that use bulk loads more complicated.

## Practice Problems

### Problem 1
Create an audit trigger that logs changes to an `employees` table's salary into a `salary_history` table whenever the salary is updated.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    salary DECIMAL(10, 2) NOT NULL
);

CREATE TABLE salary_history (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    employee_id INT UNSIGNED NOT NULL,
    old_salary DECIMAL(10, 2) NOT NULL,
    new_salary DECIMAL(10, 2) NOT NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (employee_id) REFERENCES employees(id)
);

INSERT INTO employees (id, name, salary) VALUES (1, 'Alice', 70000), (2, 'Bob', 85000);
```

**Your query:**
```sql
-- Write the trigger, then demonstrate it with an UPDATE.
```

**Solution:**
```sql
DELIMITER //

CREATE TRIGGER after_salary_update
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
    IF OLD.salary != NEW.salary THEN
        INSERT INTO salary_history (employee_id, old_salary, new_salary)
        VALUES (NEW.id, OLD.salary, NEW.salary);
    END IF;
END //

DELIMITER ;

-- Test the trigger:
UPDATE employees SET salary = 75000 WHERE id = 1;
UPDATE employees SET salary = 90000 WHERE id = 2;
UPDATE employees SET name = 'Alice Smith' WHERE id = 1; -- No salary change: no audit row

SELECT * FROM salary_history;
-- Returns 2 rows: Alice's raise to 75000, Bob's raise to 90000
```

**Explanation:**
The `IF OLD.salary != NEW.salary` condition prevents writing an audit row when the UPDATE doesn't actually change the salary — for example, when only the name changes. `NEW.id` and `OLD.salary`/`NEW.salary` are available because this is an `AFTER UPDATE` trigger. The `changed_at` column uses `DEFAULT CURRENT_TIMESTAMP`, so no explicit value is needed. The trigger fires once per updated row — if `UPDATE employees SET salary = salary * 1.1` updates 100 rows, the trigger fires 100 times.

### Problem 2
Create a `BEFORE INSERT` trigger that normalizes data before it is stored: trim whitespace from a `name` column and convert `email` to lowercase.

**Schema used:**
```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);
```

**Your query:**
```sql
-- Write the BEFORE INSERT trigger and test it.
```

**Solution:**
```sql
DELIMITER //

CREATE TRIGGER before_user_insert
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
    SET NEW.name = TRIM(NEW.name);
    SET NEW.email = LOWER(TRIM(NEW.email));
END //

DELIMITER ;

-- Test:
INSERT INTO users (name, email) VALUES ('  Alice Smith  ', '  Alice@EXAMPLE.COM  ');

SELECT id, name, email FROM users;
-- Returns: id=1, name='Alice Smith', email='alice@example.com'
```

**Explanation:**
In a `BEFORE INSERT` trigger, `SET NEW.column = value` modifies the value before it is written to the table. `TRIM(NEW.name)` removes leading and trailing whitespace. `LOWER(TRIM(NEW.email))` first trims whitespace and then converts to lowercase. The stored value is the cleaned version — applications that insert sloppy data will have it silently corrected. A `BEFORE UPDATE` trigger with the same logic would handle updates as well (create separately). This is a reasonable use of a trigger because the normalization is a data integrity concern that should always apply, regardless of which application inserts the data.

### Problem 3
Create triggers that maintain a denormalized `total_orders` count on a `customers` table, incrementing it on INSERT and decrementing it on DELETE from the `orders` table.

**Schema used:**
```sql
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    total_orders INT NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

INSERT INTO customers (id, name) VALUES (1, 'Alice'), (2, 'Bob');
```

**Your query:**
```sql
-- Write both triggers and verify the count stays correct.
```

**Solution:**
```sql
DELIMITER //

CREATE TRIGGER after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    UPDATE customers SET total_orders = total_orders + 1 WHERE id = NEW.customer_id;
END //

CREATE TRIGGER after_order_delete
AFTER DELETE ON orders
FOR EACH ROW
BEGIN
    UPDATE customers SET total_orders = total_orders - 1 WHERE id = OLD.customer_id;
END //

DELIMITER ;

-- Test:
INSERT INTO orders (customer_id, total) VALUES (1, 150.00);
INSERT INTO orders (customer_id, total) VALUES (1, 300.00);
INSERT INTO orders (customer_id, total) VALUES (2, 75.00);
SELECT id, name, total_orders FROM customers;
-- Alice: 2, Bob: 1

DELETE FROM orders WHERE id = 1;
SELECT id, name, total_orders FROM customers;
-- Alice: 1, Bob: 1
```

**Explanation:**
The `AFTER INSERT` trigger fires after each order row is successfully inserted and increments the customer's `total_orders`. The `AFTER DELETE` trigger decrements it. Using `AFTER` (not `BEFORE`) ensures the trigger only fires when the DML operation actually succeeds — a failed insert (e.g., due to a constraint violation) does not increment the count. The main risk with this pattern is consistency: bulk operations that bypass triggers (like `TRUNCATE orders`) will leave `total_orders` out of sync. A periodic reconciliation query (`UPDATE customers c SET total_orders = (SELECT COUNT(*) FROM orders WHERE customer_id = c.id)`) can correct drift.
