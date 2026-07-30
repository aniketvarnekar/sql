# Categories of SQL Commands

## Learning Objectives

By the end of this section you should be able to:
- Name the five categories of SQL commands and what each is responsible for
- Correctly classify any given SQL statement into its category
- Explain why this categorization exists and how it maps to the rest of this course

## Prerequisites

- [What Is SQL?](02-what-is-sql.md) — you need to know SQL is a language for defining, manipulating, and querying data before its commands can be meaningfully grouped.

## Motivation

SQL has dozens of keywords and statement types. Without a mental grouping, they feel like an unordered pile to memorize. Once you see that every SQL statement falls into one of five clear buckets — each with a distinct *purpose* — the entire language becomes far easier to navigate, and this categorization directly maps onto how this course's modules are organized.

## Problem Statement

If someone hands you an unfamiliar SQL statement, how do you quickly reason about *what kind of thing* it does before even reading it in detail? Is it changing the shape of your data storage? Changing the actual data? Just asking a question? Changing who can access it? Without a category to slot it into, every new statement is evaluated from scratch.

## Concept

SQL commands are grouped into five categories, based on what they act upon:

| Category | Full name | Purpose | Example commands |
|---|---|---|---|
| **DDL** | Data Definition Language | Defines/changes the *structure* of data (tables, columns, constraints) | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** | Data Manipulation Language | Changes the actual *data* inside tables | `INSERT`, `UPDATE`, `DELETE` |
| **DQL** | Data Query Language | Reads/retrieves data without changing it | `SELECT` |
| **DCL** | Data Control Language | Controls *who* can do what (permissions) | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | Groups statements into all-or-nothing units | `BEGIN`/`START TRANSACTION`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

Some textbooks fold DQL into DML (treating `SELECT` as just another data operation) — you'll see both conventions. This course treats DQL as its own category because *querying* data and *manipulating* data are conceptually very different operations (one changes nothing, the other does), and separating them makes the mental model cleaner.

### DDL — Data Definition Language

Defines the **structure** (schema) that your data lives in — creating and altering tables, columns, and constraints. Think of this as building the shelves before you put anything on them.

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    salary NUMERIC
);
```

Covered in depth in Module 4 (Database & Table Design) and Module 5 (Constraints & Keys).

### DML — Data Manipulation Language

Changes the **data itself** — the rows sitting inside a table whose structure DDL already defined.

```sql
INSERT INTO employees (name, salary) VALUES ('Asha', 85000);
UPDATE employees SET salary = 90000 WHERE name = 'Asha';
DELETE FROM employees WHERE name = 'Asha';
```

Covered in depth in Module 6 (Modifying Data).

### DQL — Data Query Language

**Reads** data without modifying anything. In practice, this is a single keyword — `SELECT` — but it is the single most powerful and heavily used statement in all of SQL, with an enormous amount of depth (filtering, sorting, joining, aggregating, subqueries...).

```sql
SELECT name, salary FROM employees WHERE salary > 80000 ORDER BY salary DESC;
```

Covered across the majority of this course: Modules 7 through 17.

### DCL — Data Control Language

Controls **permissions** — who is allowed to do what to which data. This is how a DBMS enforces that, say, only certain users can delete data, while others can only read it.

```sql
GRANT SELECT ON employees TO reporting_user;
REVOKE DELETE ON employees FROM reporting_user;
```

Covered in depth in Module 19 (Security & Access Control).

### TCL — Transaction Control Language

Groups multiple statements together so they succeed or fail **as a single unit** — critical for operations like "transfer money from account A to account B," where you never want only *half* of that to happen.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Covered in depth in Module 14 (Transactions & Concurrency).

### A Quick Classification Table

| Statement | Category |
|---|---|
| `CREATE TABLE` | DDL |
| `ALTER TABLE` | DDL |
| `DROP TABLE` | DDL |
| `TRUNCATE TABLE` | DDL (though it deletes all rows — it's classified as DDL because, in most databases, it resets the table's storage rather than deleting row-by-row; Module 6 explains this nuance) |
| `INSERT` | DML |
| `UPDATE` | DML |
| `DELETE` | DML |
| `SELECT` | DQL |
| `GRANT` | DCL |
| `REVOKE` | DCL |
| `COMMIT` | TCL |
| `ROLLBACK` | TCL |

## Internal Working (Preview)

The DBMS doesn't just use this categorization for teaching purposes — it's functionally meaningful internally too:

- **DDL** statements typically trigger changes to the database's internal *catalog* (metadata tables that describe your schema) and, in most systems, run with stronger locking since they change structure everyone depends on.
- **DML** statements go through the transaction and concurrency machinery (Module 14) since they change shared data other users might be reading.
- **DQL** statements go through the query planner/optimizer (Topic 2) since their entire job is retrieval efficiency, not modification.
- **DCL** statements modify permission metadata, checked by the DBMS before *any* other statement from a given user is allowed to execute.

## Real-World Analogy

Think of a company office building:

- **DDL** is construction — building rooms, walls, and labeled shelves (the *structure* data will live in).
- **DML** is moving boxes in and out of those rooms (changing the *contents*).
- **DQL** is someone walking around reading labels and taking notes, without touching or moving anything.
- **DCL** is the building's keycard system — deciding who is allowed into which rooms.
- **TCL** is a rule like "if you're moving furniture between two rooms, either finish moving everything, or if you get interrupted, put it all back — never leave it half-moved."

## Why This Categorization Exists

The categorization exists because these operations have fundamentally different *concerns*: structure vs. content vs. read-only inspection vs. permissions vs. atomicity guarantees. Database systems are architected around exactly this separation internally (a distinct catalog/metadata subsystem, a distinct query planner, a distinct permission-checking layer, a distinct transaction manager) — so understanding the categories isn't just a study mnemonic, it mirrors how the DBMS itself is actually built.

## Advantages of This Design

- **Clear mental model** — any SQL statement you encounter can be quickly classified by what it fundamentally does.
- **Maps directly to permissions** — real DBMSs let you grant permissions *by category* (e.g., a user can run DML but not DDL), which only makes sense because the categorization is meaningful, not arbitrary.
- **Maps directly to how this course — and most SQL education — is structured**, module by module.

## Disadvantages / Limitations

- **The boundaries aren't always crisp** — as noted above, `TRUNCATE` behaves like data deletion (DML-ish) but is classified as DDL in most systems because of how it's implemented internally. Some sources classify DQL as part of DML. Treat the categorization as a very useful mental model, not a rigid law of physics.

## Best Practices

- When learning a new SQL feature, first ask "which category is this?" — it immediately tells you whether it's about structure, data, reading, permissions, or atomicity, which shapes what else you should be thinking about (e.g., DDL changes are often harder to safely undo than DML changes).
- When granting database permissions in real systems (Module 19), think in terms of these categories — e.g., "this application user should only ever need DML and DQL, never DDL or DCL."

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "SELECT is a type of DML." | Many textbooks do group it that way, but this course (and many others) treats it as its own category, DQL, since it's fundamentally read-only and conceptually distinct from statements that change data. Either convention is defensible — just be consistent and know both exist. |
| "TRUNCATE is DML because it deletes data." | It's conventionally classified as DDL because of how most databases implement it internally (deallocating storage rather than row-by-row deletion) — covered in full in Module 6. |
| "DCL and TCL are the same thing." | DCL is about *permissions* (who can do what). TCL is about *atomicity* (grouping statements to succeed/fail together). They solve completely different problems. |

## Interview Questions

1. **Q: What are the five categories of SQL commands, and what does each control?**
   A: DDL (structure — CREATE/ALTER/DROP), DML (data — INSERT/UPDATE/DELETE), DQL (reading data — SELECT), DCL (permissions — GRANT/REVOKE), and TCL (transaction grouping — COMMIT/ROLLBACK).

2. **Q: Why is TRUNCATE typically classified as DDL rather than DML, even though it removes data?**
   A: Because in most database implementations, TRUNCATE works by deallocating the table's storage pages rather than deleting rows individually and logging each deletion — an implementation detail that aligns it with DDL's structural nature rather than DML's row-by-row semantics.

3. **Q: Give an example of a real-world reason the DDL/DML/DCL separation matters operationally.**
   A: Permission management — a production application's database user is commonly granted only DML and DQL permissions (so it can read and modify data) but explicitly denied DDL permissions (so a bug or compromised credential can't drop or alter tables) and denied DCL (so it can't grant itself more access).

## Summary

- SQL statements are grouped into five categories by what they act on: **DDL** (structure), **DML** (data), **DQL** (reading data), **DCL** (permissions), **TCL** (transaction grouping).
- This isn't just a teaching mnemonic — it reflects genuinely distinct subsystems inside a real DBMS (catalog/metadata, storage engine, query planner, permission system, transaction manager).
- This course's modules are organized largely along these same lines, so recognizing a statement's category tells you roughly which module governs it.
- Next: Topic 4 gets you a real PostgreSQL instance running so you can start executing these commands yourself.
