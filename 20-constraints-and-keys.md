# Constraints and Keys

Constraints are rules defined at the table level that MySQL enforces automatically on every `INSERT`, `UPDATE`, and `DELETE`. They are the database's built-in mechanism for maintaining data integrity — ensuring that invalid data is rejected before it can corrupt your records. Understanding and using constraints correctly reduces the need for defensive application-level validation and catches data quality problems at the earliest possible point.

## PRIMARY KEY

A primary key uniquely identifies each row in a table. It combines a `NOT NULL` constraint and a `UNIQUE` constraint. Every InnoDB table should have a primary key; if none is defined, InnoDB creates a hidden internal one.

```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);
```

A composite primary key uniquely identifies rows by a combination of columns:

```sql
CREATE TABLE order_items (
    order_id INT UNSIGNED NOT NULL,
    product_id INT UNSIGNED NOT NULL,
    quantity INT NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

Each table can have only one primary key. Choose a primary key that will never change for the life of the row. Auto-incrementing integer surrogate keys (`id INT AUTO_INCREMENT`) are the most common choice because they are stable, compact, and easy to reference in foreign keys.

## FOREIGN KEY and Referential Integrity

A foreign key creates an enforced link between a column in one table and the primary key (or unique key) of another table:

```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

With this constraint, MySQL will:
- Reject any `INSERT` or `UPDATE` that sets `customer_id` to a value that does not exist in `customers.id`
- Control what happens to `orders` rows when a referenced `customers` row is deleted or its `id` is updated, based on the `ON DELETE` and `ON UPDATE` actions

Foreign keys require the referenced column to be indexed (MySQL creates the index automatically on the referenced side if needed, and you should manually index the referencing side — `customer_id` in this case — for join performance).

Both tables must use the InnoDB storage engine for foreign keys to be enforced. MyISAM tables accept the syntax but silently ignore foreign key constraints.

### Referential Actions

`ON DELETE` and `ON UPDATE` control what happens to the child row when the referenced parent row changes:

```sql
FOREIGN KEY (customer_id) REFERENCES customers(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE
```

- `RESTRICT` (default): Reject the delete or update if any child rows reference the parent. The operation fails with an error.
- `NO ACTION`: In MySQL, identical to `RESTRICT` in practice. The standard SQL difference is about deferred checking, which InnoDB does not support.
- `CASCADE`: Automatically delete or update the child row to match the parent. `ON DELETE CASCADE` deletes orders when their customer is deleted; `ON UPDATE CASCADE` updates `customer_id` in orders when the customer's `id` changes.
- `SET NULL`: Set the foreign key column to `NULL` when the parent is deleted or updated. Requires the foreign key column to allow `NULL`.
- `SET DEFAULT`: Set the foreign key column to its default value. MySQL supports this syntactically but InnoDB does not honor it — use `SET NULL` instead.

```sql
-- Cascade deletes: delete orders when customer is deleted
FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE

-- Set to NULL: orphan orders remain but with no customer
FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE SET NULL

-- Restrict: prevent customer deletion if they have orders
FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
```

To disable foreign key checks temporarily (useful during bulk data imports):

```sql
SET foreign_key_checks = 0;
-- ... load data ...
SET foreign_key_checks = 1;
```

Re-enabling foreign key checks does not retroactively validate existing data — it only enforces constraints on new modifications.

## UNIQUE Constraint

`UNIQUE` ensures that no two rows have the same value in the specified column or combination of columns. Unlike `PRIMARY KEY`, a column can be `UNIQUE` and still allow `NULL` — and multiple rows can have `NULL` without violating uniqueness, because `NULL` is not considered equal to any value.

```sql
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(100) NOT NULL UNIQUE
);
```

Multi-column unique constraints:

```sql
CREATE TABLE team_memberships (
    team_id INT UNSIGNED NOT NULL,
    user_id INT UNSIGNED NOT NULL,
    UNIQUE KEY unique_team_user (team_id, user_id)
);
```

Adding a unique constraint to an existing table:

```sql
ALTER TABLE users ADD UNIQUE KEY unique_email (email);
```

## CHECK Constraint

`CHECK` constraints validate that column values satisfy a boolean expression. They were introduced as fully enforced constraints in MySQL 8.0.16 — earlier versions parsed the syntax but silently ignored it:

```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(8, 2) NOT NULL,
    discount_percent DECIMAL(5, 2) NOT NULL DEFAULT 0,
    stock_quantity INT NOT NULL DEFAULT 0,
    CONSTRAINT chk_price CHECK (price > 0),
    CONSTRAINT chk_discount CHECK (discount_percent BETWEEN 0 AND 100),
    CONSTRAINT chk_stock CHECK (stock_quantity >= 0)
);
```

The constraint name (e.g., `chk_price`) is optional but makes error messages more readable. A `CHECK` constraint can reference multiple columns:

```sql
CONSTRAINT chk_date_range CHECK (end_date >= start_date)
```

Attempting to insert or update a row that violates a `CHECK` constraint raises error 3819:

```sql
INSERT INTO products (name, price, discount_percent) VALUES ('Widget', -5.00, 0);
-- Error: Check constraint 'chk_price' is violated.
```

## NOT NULL

`NOT NULL` prevents a column from accepting `NULL`. This is the simplest and most fundamental constraint:

```sql
CREATE TABLE contacts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50)           -- allows NULL (optional field)
);
```

In MySQL's strict mode (`STRICT_TRANS_TABLES`, enabled by default in MySQL 8), attempting to insert a `NULL` into a `NOT NULL` column with no default raises an error. Without strict mode, MySQL would silently insert an empty string or 0, which is a data quality hazard.

## DEFAULT

`DEFAULT` specifies the value used when a column is not included in an `INSERT` statement. Without a default, omitting a `NOT NULL` column from `INSERT` causes an error:

```sql
CREATE TABLE events (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    participants INT NOT NULL DEFAULT 0,
    is_public BOOLEAN NOT NULL DEFAULT TRUE
);
```

Starting in MySQL 8.0.13, `DEFAULT` can use parenthesized expressions:

```sql
created_on DATE NOT NULL DEFAULT (CURRENT_DATE)
expiry_date DATE NOT NULL DEFAULT (DATE_ADD(CURRENT_DATE, INTERVAL 30 DAY))
```

## Practice Problems

### Problem 1
Create a `bookings` table that enforces the following rules: each booking must reference a valid room and a valid guest, the check-in date must be before the check-out date, the total price must be non-negative, and the same room cannot be double-booked for overlapping dates (handle the uniqueness only at the column level for now with a suitable unique key).

**Schema used:**
```sql
CREATE TABLE rooms (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    room_number VARCHAR(10) NOT NULL UNIQUE,
    capacity INT NOT NULL
);

CREATE TABLE guests (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);
```

**Your query:**
```sql
-- Write the CREATE TABLE for bookings.
```

**Solution:**
```sql
CREATE TABLE bookings (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    room_id INT UNSIGNED NOT NULL,
    guest_id INT UNSIGNED NOT NULL,
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_bookings_room FOREIGN KEY (room_id) REFERENCES rooms(id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT fk_bookings_guest FOREIGN KEY (guest_id) REFERENCES guests(id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT chk_booking_dates CHECK (check_out > check_in),
    CONSTRAINT chk_booking_price CHECK (total_price >= 0)
);
```

**Explanation:**
Foreign keys on `room_id` and `guest_id` with `ON DELETE RESTRICT` prevent bookings from becoming orphaned when a room or guest is deleted — you must reassign or cancel the booking first. `ON UPDATE CASCADE` means if a room's id changes (rare with surrogate keys), the booking's `room_id` updates automatically. The `CHECK (check_out > check_in)` constraint prevents logically impossible bookings. True overlapping booking prevention requires application-level logic or a scheduled check, since SQL constraints cannot easily express "no overlapping date ranges among multiple rows." An `INDEX (room_id, check_in, check_out)` would help queries that look for room availability.

### Problem 2
Demonstrate what happens when you try to delete a parent row with and without `ON DELETE CASCADE`. Show both the error case and the cascade case.

**Schema used:**
```sql
CREATE TABLE categories (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE products_restrict (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category_id INT UNSIGNED NOT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE RESTRICT
);

CREATE TABLE products_cascade (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category_id INT UNSIGNED NOT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE
);

INSERT INTO categories (id, name) VALUES (1, 'Electronics');
INSERT INTO products_restrict (name, category_id) VALUES ('Laptop', 1);
INSERT INTO products_cascade (name, category_id) VALUES ('Phone', 1);
```

**Your query:**
```sql
-- Show RESTRICT behavior and CASCADE behavior.
```

**Solution:**
```sql
-- RESTRICT: delete fails because a child row exists
DELETE FROM categories WHERE id = 1;
-- Error: Cannot delete or update a parent row: a foreign key constraint fails
-- (`products_restrict`, CONSTRAINT ... REFERENCES `categories` (`id`))

-- To delete with RESTRICT, you must first delete the child rows:
DELETE FROM products_restrict WHERE category_id = 1;
DELETE FROM categories WHERE id = 1;

-- CASCADE: deletes both parent and related children automatically
-- Re-insert for demonstration:
INSERT INTO categories (id, name) VALUES (2, 'Books');
INSERT INTO products_cascade (name, category_id) VALUES ('Novel', 2);

DELETE FROM categories WHERE id = 2;
-- Both the category AND the product are deleted automatically.

SELECT * FROM products_cascade WHERE category_id = 2;
-- (empty - the product was deleted by cascade)
```

**Explanation:**
`ON DELETE RESTRICT` (the default) protects parent rows from being deleted while child rows exist. This is the safer choice when orphaned child rows would be a problem — it forces the caller to explicitly deal with the children first. `ON DELETE CASCADE` automates the cleanup but can be dangerous: a single `DELETE` can silently remove large amounts of related data across multiple tables. Use `CASCADE` only when you have thoroughly considered the downstream effects and tested them. Avoid `CASCADE` chains deeper than two levels — they become hard to reason about and can cause unexpected data loss.

### Problem 3
Add a CHECK constraint to an existing table and demonstrate that it is enforced. Also show how to view all constraints on a table.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    age INT,
    salary DECIMAL(10, 2)
);

INSERT INTO employees (name, age, salary) VALUES ('Alice', 30, 75000);
```

**Your query:**
```sql
-- Add CHECK constraints and test them.
```

**Solution:**
```sql
-- Add CHECK constraints to existing table:
ALTER TABLE employees
    ADD CONSTRAINT chk_age CHECK (age BETWEEN 18 AND 100),
    ADD CONSTRAINT chk_salary CHECK (salary >= 0);

-- Valid insert succeeds:
INSERT INTO employees (name, age, salary) VALUES ('Bob', 25, 60000);

-- Invalid age fails:
INSERT INTO employees (name, age, salary) VALUES ('Carol', 15, 45000);
-- Error: Check constraint 'chk_age' is violated.

-- Invalid salary fails:
INSERT INTO employees (name, age, salary) VALUES ('Dave', 35, -100);
-- Error: Check constraint 'chk_salary' is violated.

-- View all constraints on the table:
SELECT
    constraint_name,
    constraint_type,
    check_clause
FROM information_schema.table_constraints tc
LEFT JOIN information_schema.check_constraints cc
    ON cc.constraint_name = tc.constraint_name
WHERE tc.table_schema = DATABASE()
  AND tc.table_name = 'employees';
```

**Explanation:**
`ALTER TABLE ... ADD CONSTRAINT` adds constraints to an existing table. MySQL checks whether existing data satisfies the new constraint before adding it — if existing rows violate the condition, the `ALTER TABLE` fails. In this example, Alice (age 30, salary 75000) satisfies both constraints, so they are added successfully. The `information_schema.table_constraints` and `information_schema.check_constraints` tables contain metadata about all constraints in the database, making them useful for inspecting what rules are in place without reading `CREATE TABLE` statements directly.
