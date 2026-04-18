# Creating Databases and Tables

The foundation of working with MySQL is creating the containers — databases and tables — that will hold your data. This file covers the full lifecycle of both: creation, modification, and deletion, along with the syntax for defining columns, constraints, and defaults at table creation time.

## Creating and Dropping Databases

A database in MySQL is a named container that groups related tables together. To create a new database:

```sql
CREATE DATABASE bookstore;
```

To avoid an error if the database already exists:

```sql
CREATE DATABASE IF NOT EXISTS bookstore;
```

You can specify the character set and collation at creation time. In MySQL 8.x, the default character set is `utf8mb4` and the default collation is `utf8mb4_0900_ai_ci`, which supports the full Unicode character set including emoji. This is the right choice for most applications:

```sql
CREATE DATABASE bookstore
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_0900_ai_ci;
```

To delete a database and all tables it contains:

```sql
DROP DATABASE bookstore;
DROP DATABASE IF EXISTS bookstore;
```

`DROP DATABASE` is irreversible — all data in all tables is permanently deleted. Use with extreme caution, especially in production.

To switch to a database as the active context:

```sql
USE bookstore;
```

## Creating Tables

The `CREATE TABLE` statement defines a new table by specifying its columns, their data types, and any constraints:

```sql
CREATE TABLE books (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    author_id INT UNSIGNED NOT NULL,
    isbn VARCHAR(20) UNIQUE,
    price DECIMAL(8, 2) NOT NULL,
    published_date DATE,
    in_stock BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

Use `IF NOT EXISTS` to skip creation if the table already exists without raising an error:

```sql
CREATE TABLE IF NOT EXISTS books (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(300) NOT NULL
);
```

### Column Definition Syntax

Each column definition follows the pattern:

```sql
column_name data_type [NOT NULL | NULL] [DEFAULT value] [AUTO_INCREMENT] [COMMENT 'text']
```

`NOT NULL` means the column cannot contain a NULL value. If you attempt to insert a row without providing a value for a `NOT NULL` column that has no default, MySQL raises an error. Designing columns as `NOT NULL` wherever logically appropriate makes your data cleaner and your queries simpler — you avoid constantly checking for NULL.

`DEFAULT value` sets the value used when no explicit value is provided during an `INSERT`. The default can be a literal value or, for certain types, a function like `CURRENT_TIMESTAMP`.

`AUTO_INCREMENT` is used on integer primary key columns to automatically assign the next sequential integer when a new row is inserted. MySQL tracks the next available value per table.

`COMMENT` lets you attach a description to a column, visible in `SHOW CREATE TABLE`:

```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT 'surrogate primary key',
    total DECIMAL(10, 2) NOT NULL COMMENT 'order total in USD'
);
```

### Defining Primary Keys

A primary key can be defined inline with the column:

```sql
id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY
```

Or as a separate table-level constraint, which is required for composite primary keys (primary keys made of more than one column):

```sql
CREATE TABLE order_items (
    order_id INT UNSIGNED NOT NULL,
    product_id INT UNSIGNED NOT NULL,
    quantity INT NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

### Defining Foreign Keys

Foreign keys are defined at the table level using the `REFERENCES` clause. They enforce referential integrity — preventing orphan rows in the child table:

```sql
CREATE TABLE books (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    author_id INT UNSIGNED NOT NULL,
    FOREIGN KEY (author_id) REFERENCES authors(id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```

Foreign key behavior on delete and update is covered in detail in the Constraints and Keys file.

## Dropping and Truncating Tables

To permanently delete a table and all its data:

```sql
DROP TABLE books;
DROP TABLE IF EXISTS books;
```

To delete all rows from a table without removing the table itself:

```sql
TRUNCATE TABLE books;
```

`TRUNCATE` is significantly faster than `DELETE FROM books` (with no `WHERE` clause) for large tables because it drops and recreates the table internally rather than deleting rows one by one. However, `TRUNCATE` cannot be rolled back in a transaction (in most storage engines), does not fire `DELETE` triggers, and resets `AUTO_INCREMENT` counters to 1. Use `DELETE` when you need any of those behaviors; use `TRUNCATE` when you just want to empty a table as fast as possible.

## Altering Tables

`ALTER TABLE` modifies the structure of an existing table. This is one of the most common and most impactful operations in a production database — on large tables, some alterations require rebuilding the entire table, which can take minutes or hours.

### Adding a Column

```sql
ALTER TABLE books ADD COLUMN page_count INT UNSIGNED;
ALTER TABLE books ADD COLUMN subtitle VARCHAR(200) AFTER title;
ALTER TABLE books ADD COLUMN sort_key INT FIRST;
```

`AFTER column_name` and `FIRST` control where in the column order the new column appears. This affects display order but has no impact on performance.

### Modifying a Column

To change a column's data type, nullability, or default:

```sql
ALTER TABLE books MODIFY COLUMN price DECIMAL(10, 2) NOT NULL DEFAULT 0.00;
```

To rename a column and optionally change its definition:

```sql
ALTER TABLE books RENAME COLUMN published_date TO publish_date;

-- Or change both name and definition at once:
ALTER TABLE books CHANGE COLUMN published_date publish_date DATE NOT NULL;
```

Note that `RENAME COLUMN` was introduced in MySQL 8.0. In MySQL 5.7 and earlier, you must use `CHANGE COLUMN`.

### Dropping a Column

```sql
ALTER TABLE books DROP COLUMN subtitle;
```

Dropping a column is irreversible and immediately destroys all data in that column. On large tables, this operation rebuilds the table.

### Renaming a Table

```sql
ALTER TABLE books RENAME TO library_books;
-- Or equivalently:
RENAME TABLE books TO library_books;
```

`RENAME TABLE` can rename multiple tables atomically in a single statement, which is useful for table swaps:

```sql
RENAME TABLE books TO books_old, books_new TO books;
```

### Viewing the Full Table Definition

To see the exact `CREATE TABLE` statement for an existing table, including all columns, indexes, and constraints:

```sql
SHOW CREATE TABLE books;
```

This is more complete than `DESCRIBE` and is the authoritative way to inspect a table's full definition.

## Practice Problems

### Problem 1
Create a database called `school` and within it create a `students` table with the following columns: an auto-incrementing integer primary key called `id`, a required `first_name` (up to 50 characters), a required `last_name` (up to 50 characters), a required `email` (up to 150 characters, unique), an `enrollment_date` that defaults to today's date, and a `is_graduated` boolean that defaults to false.

**Schema used:**
```sql
-- No pre-existing schema needed.
```

**Your query:**
```sql
-- Write the CREATE DATABASE and CREATE TABLE statements.
```

**Solution:**
```sql
CREATE DATABASE IF NOT EXISTS school
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_0900_ai_ci;

USE school;

CREATE TABLE students (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    enrollment_date DATE NOT NULL DEFAULT (CURRENT_DATE),
    is_graduated BOOLEAN NOT NULL DEFAULT FALSE
);
```

**Explanation:**
`DEFAULT (CURRENT_DATE)` uses a parenthesized expression as the default value, which is a MySQL 8.0.13+ feature. In earlier versions you would need to default `DATETIME` columns using `DEFAULT CURRENT_TIMESTAMP` and handle `DATE` defaults in application code. The `UNIQUE` keyword on `email` creates a unique index automatically, enforcing that no two students share an email address. Note that `UNIQUE` allows NULL values by default — multiple rows can have NULL in a unique column. If email is mandatory, the `NOT NULL` constraint handles that.

### Problem 2
A `students` table already exists. Add a `phone_number` column (up to 20 characters, nullable) after the `email` column, then rename the `is_graduated` column to `has_graduated`, and finally add a `notes` column of type `TEXT` at the end.

**Schema used:**
```sql
CREATE TABLE students (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    enrollment_date DATE NOT NULL DEFAULT (CURRENT_DATE),
    is_graduated BOOLEAN NOT NULL DEFAULT FALSE
);
```

**Your query:**
```sql
-- Write the ALTER TABLE statements.
```

**Solution:**
```sql
ALTER TABLE students
    ADD COLUMN phone_number VARCHAR(20) AFTER email;

ALTER TABLE students
    RENAME COLUMN is_graduated TO has_graduated;

ALTER TABLE students
    ADD COLUMN notes TEXT;
```

**Explanation:**
Each `ALTER TABLE` statement modifies the table structure. They can be combined in a single statement by separating operations with commas, which causes only one table rebuild: `ALTER TABLE students ADD COLUMN phone_number VARCHAR(20) AFTER email, RENAME COLUMN is_graduated TO has_graduated, ADD COLUMN notes TEXT;`. On large production tables, combining operations into a single `ALTER TABLE` reduces the number of full table rebuilds from three to one, significantly improving performance.

### Problem 3
Explain the difference between `DROP TABLE students` and `TRUNCATE TABLE students`, and describe a scenario where you would choose each.

**Schema used:**
```sql
CREATE TABLE students (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL
);

INSERT INTO students (first_name, last_name) VALUES
    ('Alice', 'Smith'),
    ('Bob', 'Jones');
```

**Your query:**
```sql
-- Demonstrate TRUNCATE (safe to run; the table still exists after):
TRUNCATE TABLE students;
SHOW TABLES;
SELECT * FROM students;

-- DROP removes the table entirely:
-- DROP TABLE students;
-- SHOW TABLES; -- students no longer appears
```

**Solution:**
```sql
-- TRUNCATE: removes all rows, keeps the table structure, resets AUTO_INCREMENT
TRUNCATE TABLE students;
-- Result: table exists, is empty, next INSERT will get id=1

-- DROP: removes the table structure and all data permanently
DROP TABLE students;
-- Result: table no longer exists; any query referencing it fails
```

**Explanation:**
Use `TRUNCATE` when you want to empty a table completely and start fresh — for example, clearing a staging table before reloading data from an ETL pipeline. The table structure, indexes, and constraints remain intact. Use `DROP` when you want to remove the table entirely, such as when removing a feature from an application and cleaning up the corresponding schema. The key operational difference is that `DROP` is final — nothing is left behind — while `TRUNCATE` leaves an empty, ready-to-use table. Also note that `TRUNCATE` resets the `AUTO_INCREMENT` counter to 1, which `DELETE FROM students` does not.
