# Introduction to Databases

A database is an organized collection of structured information stored so that it can be easily accessed, managed, and updated. Understanding why databases exist, how they differ from simpler tools like spreadsheets, and where MySQL fits in the broader landscape of data storage systems is the essential foundation before writing a single line of SQL.

## Why Databases Instead of Spreadsheets

Spreadsheets like Excel or Google Sheets are excellent tools for small datasets, ad hoc analysis, and human-readable reports. However, they break down quickly in production software systems. A spreadsheet has no concept of concurrent users — if two people edit the same file simultaneously, you get conflicts or corruption. There is no way to enforce that a value in one column must match a value in another file, no built-in mechanism to prevent duplicate records, and no reliable way to handle millions of rows without performance degradation.

Databases solve all of these problems. They are designed from the ground up to allow many users to read and write simultaneously without corrupting data, to enforce rules about what data is valid, to maintain relationships between different sets of data, and to retrieve specific records efficiently even from tables containing hundreds of millions of rows. When you build an application that needs to persist data reliably — whether it is a simple blog or a global e-commerce platform — a database is the right tool.

## Relational vs Non-Relational Databases

Databases fall into two broad families: relational and non-relational (often called NoSQL).

A relational database stores data in tables. Each table has a fixed set of columns (also called fields or attributes), and each row in the table is one record. Tables can be related to each other through shared keys. For example, an `orders` table might contain a `customer_id` column that refers to the `id` column of a `customers` table. The database enforces this relationship and can efficiently retrieve all orders for a given customer by joining the two tables. MySQL, PostgreSQL, Oracle, and SQL Server are all relational databases.

Non-relational databases take a different approach. They store data as documents (MongoDB), key-value pairs (Redis), wide-column structures (Cassandra), or graphs (Neo4j). They are often chosen when the data structure is irregular or unpredictable, when you need to scale horizontally across many machines with minimal configuration, or when you need to store and retrieve data at extreme speed with a simple access pattern. The trade-off is that non-relational databases typically sacrifice some of the consistency guarantees and relational features that make relational databases reliable for complex, interconnected data.

For most application development, particularly when your data has clear structure and relationships, a relational database is the right starting point. MySQL is the most widely deployed relational database in the world and is an excellent choice for the vast majority of applications.

## What MySQL Is

MySQL is an open-source relational database management system (RDBMS). It was originally developed by a Swedish company called MySQL AB in 1995, acquired by Sun Microsystems in 2008, and then passed to Oracle when Oracle acquired Sun in 2010. Despite Oracle's ownership, MySQL remains open-source under the GNU General Public License, and a community fork called MariaDB was created in 2009 to ensure a fully open-source alternative would always be available.

MySQL is the "M" in the LAMP stack (Linux, Apache, MySQL, PHP/Python/Perl), which powered the early web and continues to power an enormous portion of it today. It is used by companies including Facebook (historically), Twitter, GitHub, Airbnb, and Netflix. Its combination of reliability, performance, broad language support, and large community makes it one of the most practical databases to learn.

MySQL 8.x, which these notes cover, introduced significant improvements over MySQL 5.x: native support for window functions, common table expressions (CTEs), descending indexes, improved JSON support, and roles for access control. If you are maintaining older MySQL 5.x systems, be aware that many features in later sections of these notes will not be available.

## Key Terms

Understanding database vocabulary precisely will prevent confusion as you read these notes and work with MySQL in practice.

A **table** is the fundamental structure in a relational database. It organizes data into rows and columns, similar to a spreadsheet tab, but with strict rules about what data each column can contain.

A **row** (also called a record or tuple) is a single entry in a table. In a `users` table, each row represents one user.

A **column** (also called a field or attribute) defines one piece of information that every row in the table has. In a `users` table, columns might include `id`, `email`, `created_at`, and `username`. Every column has a data type that restricts what values it can hold.

A **schema** has two related meanings in MySQL. At the broadest level, a schema is the overall structure of a database — the definition of all its tables, columns, relationships, and constraints. MySQL also uses the word "schema" interchangeably with "database" as a container for a collection of tables. You will see `SHOW SCHEMAS;` and `SHOW DATABASES;` produce identical results in MySQL.

A **query** is a request you make to the database, written in SQL (Structured Query Language). Queries can retrieve data (`SELECT`), add data (`INSERT`), modify data (`UPDATE`), delete data (`DELETE`), or change the database structure (`CREATE`, `ALTER`, `DROP`). SQL is a declarative language: you describe what you want, and the database engine figures out how to retrieve or modify the data efficiently.

A **primary key** is a column or combination of columns that uniquely identifies each row in a table. No two rows in a table can have the same primary key value, and no primary key column can be NULL. A `users` table typically has an `id` column as its primary key.

A **foreign key** is a column in one table that references the primary key of another table. It enforces referential integrity — meaning the database will prevent you from creating a row that references a non-existent record, and it controls what happens when the referenced record is deleted or updated.

## Practice Problems

### Problem 1
You are designing a database for a library. The library tracks books, authors, and which members have borrowed which books. Identify at least three tables you would need, the key columns in each, and describe two relationships between the tables.

**Schema used:**
```sql
-- No schema needed for this conceptual problem.
-- The answer is a written description.
```

**Your query:**
```sql
-- Write your table designs here as CREATE TABLE statements.
```

**Solution:**
```sql
CREATE TABLE authors (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    birth_year YEAR
);

CREATE TABLE books (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(300) NOT NULL,
    author_id INT NOT NULL,
    isbn VARCHAR(20) UNIQUE,
    published_year YEAR,
    FOREIGN KEY (author_id) REFERENCES authors(id)
);

CREATE TABLE members (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(200) UNIQUE NOT NULL,
    joined_date DATE NOT NULL
);

CREATE TABLE borrowings (
    id INT PRIMARY KEY AUTO_INCREMENT,
    book_id INT NOT NULL,
    member_id INT NOT NULL,
    borrowed_on DATE NOT NULL,
    returned_on DATE,
    FOREIGN KEY (book_id) REFERENCES books(id),
    FOREIGN KEY (member_id) REFERENCES members(id)
);
```

**Explanation:**
The library needs at least four tables. `authors` holds author information independently of books so that an author can have many books without repeating their name and birth year. `books` references `authors` through a foreign key, capturing the one-to-many relationship between authors and books. `members` stores library member data. `borrowings` is a junction table that captures the many-to-many relationship between members and books — a member can borrow many books, and a book can be borrowed by many members over time. The `returned_on` column being nullable allows us to distinguish books that are still out on loan (NULL) from those that have been returned.

### Problem 2
Explain in your own words why a relational database is more appropriate than a spreadsheet for tracking orders for an online store that processes 10,000 orders per day across 50,000 products and 200,000 customers.

**Schema used:**
```sql
-- No schema needed. This is a conceptual problem.
```

**Your query:**
```sql
-- Write your reasoning here as a comment.
```

**Solution:**
```sql
-- At 10,000 orders per day, the orders table will grow by 3.65 million rows per year.
-- Spreadsheets become sluggish and unreliable well before this scale.
--
-- Key reasons a relational database is the right choice:
-- 1. Concurrency: thousands of customers can place orders simultaneously
--    without corrupting data. A spreadsheet cannot handle this.
-- 2. Referential integrity: a foreign key from orders to customers ensures
--    you cannot place an order for a non-existent customer.
-- 3. Normalization: product names, prices, and descriptions live in one place
--    (the products table) and are referenced by id in orders, avoiding
--    duplication and update anomalies.
-- 4. Query power: SQL lets you instantly answer questions like
--    "which products were ordered most in Q3" across millions of rows.
-- 5. Transactions: if charging a card and creating an order record must
--    both succeed or both fail, a database transaction guarantees this.
```

**Explanation:**
A spreadsheet forces you to duplicate data (copying product names into every order row), has no mechanism to prevent an order from referencing a product that does not exist, cannot handle concurrent writes from thousands of users, and performs poorly as the dataset grows into the millions of rows. A relational database handles all of these problems natively: foreign keys enforce consistency, transactions ensure atomicity, indexes keep queries fast, and the engine manages concurrent access through locking and transaction isolation.

### Problem 3
What is the difference between a schema and a database in MySQL? Look up how MySQL handles the `SHOW SCHEMAS` and `SHOW DATABASES` commands and explain why they produce the same result.

**Schema used:**
```sql
-- No schema needed.
```

**Your query:**
```sql
SHOW DATABASES;
SHOW SCHEMAS;
```

**Solution:**
```sql
-- Both commands produce identical output in MySQL.
-- Example output:
-- +--------------------+
-- | Database           |
-- +--------------------+
-- | information_schema |
-- | mysql              |
-- | performance_schema |
-- | sys                |
-- | your_database_name |
-- +--------------------+

-- In standard SQL theory, a "schema" is a namespace within a database
-- that groups objects like tables and views. In MySQL's implementation,
-- the concepts of "schema" and "database" are merged into one level:
-- a database IS a schema. MySQL chose to make SCHEMA a synonym for
-- DATABASE to improve compatibility with the SQL standard and with
-- other database systems like PostgreSQL that use the term "schema".
```

**Explanation:**
In standard SQL and in databases like PostgreSQL, a schema is a layer beneath the database — you can have multiple schemas within one database, each with their own tables. MySQL collapses this hierarchy: a MySQL "database" is equivalent to a "schema" in the standard sense. This is why `CREATE DATABASE` and `CREATE SCHEMA` are synonyms in MySQL, and why `SHOW DATABASES` and `SHOW SCHEMAS` return the same result. When migrating SQL code from PostgreSQL to MySQL, this is one of the structural differences you will need to account for.
