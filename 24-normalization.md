# Normalization

Normalization is the process of organizing a relational database schema to reduce data redundancy and improve data integrity. It proceeds through a series of "normal forms," each eliminating a specific type of anomaly. Understanding normalization guides you toward schemas that are correct by design — and understanding denormalization tells you when to deliberately break those rules for performance.

## Purpose of Normalization

An unnormalized database stores the same piece of information in multiple places. When that information changes, every copy must be updated — and if any are missed, the database contains contradictory facts. This is called an update anomaly. Similarly, deleting a row might inadvertently destroy the only record of a fact that should be preserved (a deletion anomaly), and inserting a new fact might require inventing placeholder values for other columns that aren't yet known (an insertion anomaly).

Normalization eliminates these anomalies by ensuring each fact is stored in exactly one place.

## First Normal Form (1NF)

A table is in 1NF when:
- Every column holds atomic (indivisible) values — no lists, arrays, or sets in a single column
- Every row is unique (there is a primary key)
- Each column holds values of a single type

A violation of 1NF — storing a comma-separated list in a column — is one of the most common database design mistakes:

```sql
-- VIOLATES 1NF: phone_numbers contains a list
CREATE TABLE contacts_bad (
    id INT PRIMARY KEY,
    name VARCHAR(200),
    phone_numbers VARCHAR(500)  -- '555-1234, 555-5678, 555-9012'
);
```

The correct 1NF design separates the repeating group into its own table:

```sql
-- 1NF compliant:
CREATE TABLE contacts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE contact_phones (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    contact_id INT UNSIGNED NOT NULL,
    phone_number VARCHAR(50) NOT NULL,
    FOREIGN KEY (contact_id) REFERENCES contacts(id)
);
```

Now each phone number is a separate row, and any number of phone numbers can be stored without changing the schema.

## Second Normal Form (2NF)

A table is in 2NF when:
- It is already in 1NF
- Every non-key column is fully functionally dependent on the entire primary key (not just a part of it)

2NF only applies to tables with composite primary keys. A violation occurs when a non-key column depends on only part of the composite key:

```sql
-- VIOLATES 2NF: product_name depends only on product_id, not on (order_id, product_id)
CREATE TABLE order_items_bad (
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    product_name VARCHAR(200),  -- depends only on product_id
    unit_price DECIMAL(8,2),    -- depends only on product_id
    quantity INT NOT NULL,       -- depends on (order_id, product_id) -- OK
    PRIMARY KEY (order_id, product_id)
);
```

`product_name` and `unit_price` depend only on `product_id`, not on the full `(order_id, product_id)` key. If the product's name changes, you must update every row in `order_items_bad` that references it. The 2NF fix separates the product information:

```sql
-- 2NF compliant:
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    unit_price DECIMAL(8,2) NOT NULL
);

CREATE TABLE order_items (
    order_id INT UNSIGNED NOT NULL,
    product_id INT UNSIGNED NOT NULL,
    quantity INT NOT NULL,
    -- unit_price stored here at order time to preserve historical price:
    price_at_purchase DECIMAL(8,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

Note that `price_at_purchase` is legitimately in `order_items` — it records the price at the time of purchase, which may differ from the current product price.

## Third Normal Form (3NF)

A table is in 3NF when:
- It is already in 2NF
- No non-key column is transitively dependent on the primary key through another non-key column

A transitive dependency means: A (primary key) → B (non-key) → C (non-key). Column C depends on B, not directly on A:

```sql
-- VIOLATES 3NF: zip_code → city → state (city depends on zip, not on employee_id)
CREATE TABLE employees_bad (
    id INT PRIMARY KEY,
    name VARCHAR(200),
    zip_code CHAR(5),
    city VARCHAR(100),     -- determined by zip_code, not by id
    state CHAR(2)          -- determined by zip_code, not by id
);
```

If the city or state for a zip code changes, you must update every employee with that zip code. The 3NF fix separates the zip code data:

```sql
-- 3NF compliant:
CREATE TABLE zip_codes (
    zip_code CHAR(5) PRIMARY KEY,
    city VARCHAR(100) NOT NULL,
    state CHAR(2) NOT NULL
);

CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    zip_code CHAR(5),
    FOREIGN KEY (zip_code) REFERENCES zip_codes(zip_code)
);
```

Now city and state are stored once per zip code, not duplicated across every employee.

## Boyce-Codd Normal Form (BCNF)

BCNF is a slightly stronger version of 3NF. A table is in BCNF when, for every functional dependency A → B, A is a superkey (a set of columns that uniquely identifies a row). Most tables that are in 3NF are also in BCNF; the exceptions are unusual schema designs where non-key columns determine parts of a composite key. BCNF violations are relatively rare in practice and beyond the scope of most application database design.

## Denormalization: When and Why

Denormalization is the deliberate reversal of normalization — introducing controlled redundancy to improve query performance. It is an optimization tool to reach for only after measuring a real performance problem.

Common denormalization patterns:

Storing a computed aggregate in the parent table to avoid expensive joins:

```sql
-- Instead of always counting: SELECT COUNT(*) FROM orders WHERE customer_id = 1
-- Store it: customers.order_count
ALTER TABLE customers ADD COLUMN order_count INT NOT NULL DEFAULT 0;
UPDATE customers c
SET order_count = (SELECT COUNT(*) FROM orders WHERE customer_id = c.id);
-- Maintain with triggers or application logic on every INSERT/DELETE of orders
```

Duplicating a frequently read column to enable covering indexes:

```sql
-- If order queries always join to customers.name, denormalize name into orders:
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(200);
-- Maintain with triggers or application logic
```

## Practical Trade-offs in Production

Strict normalization (3NF) is the right starting point for most schemas. It produces correct, consistent data and makes it easier to reason about data integrity. However, production systems at scale often make targeted denormalization choices:

- Reporting tables and data warehouses routinely denormalize heavily to avoid join overhead on analytical queries over billions of rows.
- OLTP (transactional) systems typically stay closer to 3NF and use indexes and query optimization before resorting to denormalization.
- Generated columns in MySQL 8 allow storing computed values that MySQL maintains automatically, providing some denormalization benefits without the consistency risk of application-maintained redundancy.

The guiding principle is: normalize first for correctness, then denormalize only where you have measured a concrete performance problem that simpler measures (indexes, query rewriting) cannot solve.

## Practice Problems

### Problem 1
A developer designed the following table. Identify which normal form it violates and redesign it to correct the violation.

**Schema used:**
```sql
CREATE TABLE student_courses_bad (
    student_id INT NOT NULL,
    student_name VARCHAR(200),
    course_id INT NOT NULL,
    course_name VARCHAR(200),
    instructor_name VARCHAR(200),
    grade CHAR(1),
    PRIMARY KEY (student_id, course_id)
);
```

**Your query:**
```sql
-- Identify the normal form violation and write the corrected schema.
```

**Solution:**
```sql
-- ANALYSIS:
-- Violates 2NF:
--   student_name depends only on student_id (partial dependency)
--   course_name and instructor_name depend only on course_id (partial dependency)
--   grade depends on (student_id, course_id) -- correctly placed
--
-- Also potentially violates 3NF if instructor_name is determined by course_id
-- and course_id is not a primary key by itself.

-- Corrected 3NF schema:
CREATE TABLE students (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE instructors (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE courses (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    instructor_id INT UNSIGNED NOT NULL,
    FOREIGN KEY (instructor_id) REFERENCES instructors(id)
);

CREATE TABLE enrollments (
    student_id INT UNSIGNED NOT NULL,
    course_id INT UNSIGNED NOT NULL,
    grade CHAR(1),
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);
```

**Explanation:**
The original table has partial dependencies: `student_name` depends only on `student_id`, not on the full composite key `(student_id, course_id)`. Similarly, `course_name` and `instructor_name` depend only on `course_id`. This violates 2NF. The fix separates students, courses, and instructors into their own tables. `enrollments` contains only `student_id`, `course_id`, and `grade` — the attributes that genuinely depend on the full composite key. The instructor is stored in `courses` because one instructor teaches each course, not stored per enrollment.

### Problem 2
Show a concrete example of an update anomaly in a denormalized table and demonstrate how the normalized version prevents it.

**Schema used:**
```sql
-- Denormalized (redundant city stored with each order):
CREATE TABLE orders_denormalized (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    customer_city VARCHAR(100),
    total DECIMAL(10, 2) NOT NULL
);

INSERT INTO orders_denormalized (customer_id, customer_city, total) VALUES
    (1, 'New York', 150.00),
    (1, 'New York', 300.00),
    (1, 'New York', 75.00);
```

**Your query:**
```sql
-- Show the update anomaly and the normalized fix.
```

**Solution:**
```sql
-- Update anomaly: customer 1 moves to Boston.
-- Must update every order row for this customer (3 rows here, could be thousands):
UPDATE orders_denormalized SET customer_city = 'Boston' WHERE customer_id = 1;
-- If this UPDATE is interrupted or partially applied, rows become inconsistent.

-- Normalized version: city is stored once in customers
CREATE TABLE customers (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    city VARCHAR(100)
);

CREATE TABLE orders_normalized (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

INSERT INTO customers (id, name, city) VALUES (1, 'Alice', 'New York');
INSERT INTO orders_normalized (customer_id, total) VALUES (1, 150.00), (1, 300.00), (1, 75.00);

-- Customer moves: update once, all queries immediately reflect the new city:
UPDATE customers SET city = 'Boston' WHERE id = 1;

-- All orders for customer 1 now show Boston via the join -- no order rows touched:
SELECT o.id, c.city, o.total
FROM orders_normalized o JOIN customers c ON o.customer_id = c.id;
```

**Explanation:**
In the denormalized version, changing a customer's city requires updating potentially thousands of order rows — and any failure during that update leaves the database in an inconsistent state where some orders show the old city and some show the new one. In the normalized version, the city is stored exactly once in the `customers` table. A single `UPDATE` to `customers` is immediately reflected in every query that joins orders to customers — no order rows are touched at all. This is the core benefit of normalization: a single source of truth for each fact.

### Problem 3
Design a normalized schema for a library system where books can have multiple authors, belong to a genre, and be borrowed by members. Ensure the schema is at least in 3NF.

**Schema used:**
```sql
-- No pre-existing schema. Design from scratch.
```

**Your query:**
```sql
-- Write the CREATE TABLE statements for a 3NF library schema.
```

**Solution:**
```sql
CREATE TABLE genres (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE authors (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    birth_year YEAR
);

CREATE TABLE books (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    genre_id INT UNSIGNED NOT NULL,
    isbn VARCHAR(20) UNIQUE,
    published_year YEAR,
    FOREIGN KEY (genre_id) REFERENCES genres(id)
);

CREATE TABLE book_authors (
    book_id INT UNSIGNED NOT NULL,
    author_id INT UNSIGNED NOT NULL,
    author_order TINYINT NOT NULL DEFAULT 1,
    PRIMARY KEY (book_id, author_id),
    FOREIGN KEY (book_id) REFERENCES books(id),
    FOREIGN KEY (author_id) REFERENCES authors(id)
);

CREATE TABLE members (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    joined_date DATE NOT NULL DEFAULT (CURRENT_DATE)
);

CREATE TABLE borrowings (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    book_id INT UNSIGNED NOT NULL,
    member_id INT UNSIGNED NOT NULL,
    borrowed_on DATE NOT NULL,
    due_date DATE NOT NULL,
    returned_on DATE,
    FOREIGN KEY (book_id) REFERENCES books(id),
    FOREIGN KEY (member_id) REFERENCES members(id)
);
```

**Explanation:**
Each table stores one type of entity. `genres` is separate from `books` so a genre name is stored once — changing it updates all books automatically. `book_authors` is a junction table that handles the many-to-many relationship between books and authors. `author_order` allows recording which author is listed first, second, etc. `borrowings` links members to books over time, with `returned_on` being NULL for currently borrowed books. Every non-key column depends only on the primary key of its table, satisfying 3NF. No computed or derivable data is stored (e.g., no "is_overdue" column — that is computed from `due_date` and `returned_on` at query time).
