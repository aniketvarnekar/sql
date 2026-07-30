# The Complete SQL Course — From First Principles to Mastery

> A self-contained SQL and relational database learning resource. No prior database knowledge assumed.
> Written to replace books, tutorials, and documentation for learning SQL deeply.

## Who this is for

- Complete beginners to databases (but you should already know what a variable, loop, and function are, in *any* programming language, or at least be comfortable with basic logical thinking)
- College students studying databases for coursework or exams
- Developers who know some SQL but never learned the "why" behind it
- Engineers preparing for technical interviews
- Anyone who wants to understand SQL from the ground up — not just write queries, but *understand* why relational databases work the way they do

## Philosophy

Most SQL resources teach you **syntax**. This course teaches you **reasoning**.

For every concept, we don't just show you how to write it — we explain:

| Question | Why it matters |
|---|---|
| **Why does this exist?** | Every SQL feature was added to solve a real, painful data problem. If you don't know the problem, the solution feels arbitrary and you'll forget it. |
| **What problem does it solve?** | Concepts stick when tied to a concrete pain point. |
| **How does the database engine handle this internally?** | SQL hides a lot behind the query planner and storage engine. Understanding that machinery makes debugging, performance tuning, and interviews dramatically easier. |
| **When should I use it? When shouldn't I?** | Knowing a feature exists is useless without judgment about when to reach for it. |
| **What are the trade-offs?** | Every design decision in database design is a trade-off. Nothing is free. |

We follow this learning loop for every topic:

```
   Problem
      │
      ▼
    Need
      │
      ▼
  Concept
      │
      ▼
Implementation
      │
      ▼
  Example
      │
      ▼
Internal Working
      │
      ▼
 Advantages
      │
      ▼
 Limitations
      │
      ▼
Best Practices
```

## How this course is generated

This course is built **one module at a time**. At the start of a module, every topic it will cover is listed up front. At the end of a module, every topic is checked off, and exercises + interview questions are included. The next module only begins when you ask for it (type **"Continue"**).

This keeps each module focused, complete, and reviewable before moving on — rather than dumping an entire course at once.

## A note on which database this course uses

SQL is a *standard* (ANSI/ISO SQL), but every real database (PostgreSQL, MySQL, SQL Server, Oracle, SQLite, ...) implements it with small differences. To keep examples concrete and actually runnable, this course uses **PostgreSQL** as its primary, reference database — it is free, closely follows the SQL standard, and supports nearly every feature this course covers natively.

Every topic's examples will run correctly in PostgreSQL exactly as written. Module 22 is dedicated entirely to *where and how* other databases (MySQL, SQL Server, Oracle, SQLite) differ, so you are never caught off guard moving to a different system.

## Course Structure (23 Modules)

| # | Module | Status | What you'll learn |
|---|---|---|---|
| 01 | [Introduction](01-introduction/) | ✅ Complete | What a database and SQL are, SQL command categories, setting up PostgreSQL, your first query |
| 02 | [Relational Model](02-relational-model/) | ✅ Complete | Tables, rows, columns, relations, schemas, the theory behind relational databases |
| 03 | [Data Types](03-data-types/) | ✅ Complete | Numeric, string, date/time, boolean, and NULL semantics |
| 04 | [Database & Table Design](04-database-and-table-design/) | ✅ Complete | CREATE/ALTER/DROP for databases and tables |
| 05 | [Constraints & Keys](05-constraints-and-keys/) | ✅ Complete | Primary keys, foreign keys, UNIQUE, CHECK, referential integrity |
| 06 | [Modifying Data](06-modifying-data/) | ✅ Complete | INSERT, UPDATE, DELETE, TRUNCATE, UPSERT |
| 07 | [Querying Basics](07-querying-basics/) | ✅ Complete | SELECT, WHERE, ORDER BY, LIMIT, DISTINCT, operators |
| 08 | [Functions & Expressions](08-functions-and-expressions/) | ✅ Complete | String/numeric/date functions, CASE, CAST, NULL-handling functions |
| 09 | [Aggregation](09-aggregation/) | ✅ Complete | COUNT/SUM/AVG/MIN/MAX, GROUP BY, HAVING, ROLLUP/CUBE |
| 10 | [Joins & Set Operations](10-joins-and-set-operations/) | ✅ Complete | INNER/LEFT/RIGHT/FULL/CROSS/SELF joins, UNION/INTERSECT/EXCEPT |
| 11 | [Subqueries](11-subqueries/) | ✅ Complete | Scalar, correlated subqueries, EXISTS, ANY/ALL, derived tables |
| 12 | [Views](12-views/) | ✅ Complete | CREATE VIEW, updatable views, materialized views |
| 13 | [Indexes](13-indexes/) | ✅ Complete | B-tree indexes, clustered vs non-clustered, reading EXPLAIN |
| 14 | [Transactions & Concurrency](14-transactions-and-concurrency/) | ✅ Complete | ACID, COMMIT/ROLLBACK, isolation levels, locks, deadlocks |
| 15 | [Normalization & Design](15-normalization-and-design/) | ✅ Complete | Functional dependencies, 1NF–BCNF, ER modeling, denormalization |
| 16 | [Window Functions](16-window-functions/) | ✅ Complete | OVER, PARTITION BY, ROW_NUMBER, RANK, LAG/LEAD, running totals |
| 17 | [CTEs & Recursion](17-ctes-and-recursion/) | ✅ Complete | WITH clause, recursive CTEs, hierarchies and graphs |
| 18 | [Procedures, Functions & Triggers](18-procedures-functions-triggers/) | ✅ Complete | Stored procedures, user-defined functions, triggers |
| 19 | [Security & Access Control](19-security-and-access-control/) | ✅ Complete | GRANT/REVOKE, roles, SQL injection and parameterization |
| 20 | [Performance Tuning](20-performance-tuning/) | ✅ Complete | Execution plans, EXPLAIN ANALYZE, query rewriting, anti-patterns |
| 21 | [Advanced SQL](21-advanced-sql/) | ✅ Complete | Pivoting data, JSON columns, sequences, temp tables, partitioning |
| 22 | [SQL Across Databases](22-sql-across-databases/) | ✅ Complete | Where MySQL, SQL Server, Oracle, and SQLite diverge from the standard |
| 23 | [Interview Preparation](23-interview-preparation/) | ✅ Complete | Consolidated interview questions, query-writing drills, schema design questions |

## How to read this course

1. Go in order. Module 4 assumes Module 2 and 3. Module 15 assumes almost everything before it.
2. Type out every query example yourself against a real PostgreSQL instance. Don't just read it. Muscle memory and seeing real output matters.
3. Attempt exercises *before* checking answers.
4. When a topic references a diagram or table layout, actually trace through it with your finger/cursor — don't skim it.
5. Revisit the "Why" sections even if you already know "How" — most people can write working SQL but can't explain *why* it's structured this way, and that gap shows up in interviews and in bad schema designs.
