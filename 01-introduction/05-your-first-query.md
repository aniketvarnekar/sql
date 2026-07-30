# Your First Query

## Learning Objectives

By the end of this section you should be able to:
- Create a table, insert rows into it, and query it back
- Explain, keyword by keyword, what each statement in that sequence does
- Recognize and avoid the most common beginner mistakes when writing first queries

## Prerequisites

- [Setting Up PostgreSQL](04-setting-up-postgresql.md) — you need a running, connected `psql` session to follow along.
- [Categories of SQL Commands](03-categories-of-sql-commands.md) — helps you recognize which category each statement below belongs to.

## Motivation

Every concept so far has been conceptual. This topic is your first hands-on win: you will create a real table, put real data in it, and ask a real question of that data — and understand precisely what every word in every statement did. This small, complete loop (define → insert → query) is the seed of everything the rest of this course expands on.

## Problem Statement

You want to track a small list of employees — their names, departments, and salaries — and be able to ask questions like "who works in Sales?" There's no data or structure yet; you're starting from a completely empty database.

## Concept

### Step 1 — Define the Structure (DDL)

Before any data can exist, a table's shape must be defined: what columns it has, and what type of data each column holds.

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    department TEXT,
    salary NUMERIC
);
```

Breaking this down keyword by keyword:

| Piece | Meaning |
|---|---|
| `CREATE TABLE employees` | Create a new table named `employees`. |
| `id SERIAL PRIMARY KEY` | A column named `id`, of type `SERIAL` (PostgreSQL's auto-incrementing integer — it fills itself in with 1, 2, 3... automatically), marked as the `PRIMARY KEY` (a value that must be unique and non-null for every row — full explanation in Module 5). |
| `name TEXT NOT NULL` | A column named `name` holding text, which can never be left empty (`NOT NULL` — Module 5 covers constraints in depth). |
| `department TEXT` | A column named `department` holding text, allowed to be empty (no `NOT NULL`, so it's optional). |
| `salary NUMERIC` | A column named `salary` holding an exact numeric value (Module 3 covers why `NUMERIC` is preferred over floating-point types for things like money). |

This is a **DDL** statement (Topic 3) — it defines structure, not data. After running it, the `employees` table exists but is completely empty.

### Step 2 — Insert Data (DML)

Now put some actual rows into that structure:

```sql
INSERT INTO employees (name, department, salary) VALUES
    ('Asha', 'Sales', 85000),
    ('Ben', 'Engineering', 95000),
    ('Chen', 'Sales', 78000);
```

Breaking this down:

| Piece | Meaning |
|---|---|
| `INSERT INTO employees (name, department, salary)` | Insert into the `employees` table, providing values for these three named columns. |
| `VALUES (...), (...), (...)` | Three separate rows, each a parenthesized list of values matching the column order given above. |

Notice `id` isn't mentioned at all — because it's `SERIAL`, PostgreSQL fills it in automatically (1, 2, 3) without you specifying it. This is a **DML** statement (Topic 3) — it changes data, not structure.

### Step 3 — Query the Data (DQL)

Now ask a question of that data:

```sql
SELECT name, salary
FROM employees
WHERE department = 'Sales'
ORDER BY salary DESC;
```

Breaking this down:

| Piece | Meaning |
|---|---|
| `SELECT name, salary` | Return only these two columns for each matching row (not every column). |
| `FROM employees` | Look in the `employees` table. |
| `WHERE department = 'Sales'` | Only include rows where the `department` column equals the text `'Sales'` (single-quoted, since it's a text value — Module 3 covers this in depth). |
| `ORDER BY salary DESC` | Sort the resulting rows by `salary`, highest first (`DESC` = descending; `ASC`, ascending, is the default if omitted). |

Running this against the three rows inserted above returns:

```
 name | salary
------+--------
 Asha |  85000
 Chen |  78000
(2 rows)
```

Notice Ben is excluded entirely — he's in Engineering, not Sales — and the two Sales rows are sorted by salary, descending. This is a **DQL** statement (Topic 3) — it read data, changing nothing.

### The Full Beginner-to-Result Loop

```
CREATE TABLE   →   define the shape (once, usually)
      │
      ▼
   INSERT     →   populate it with data (repeatedly, over time)
      │
      ▼
   SELECT     →   ask questions of it (repeatedly, as often as needed)
```

Every module from here on is really just deepening one of these three steps: richer table definitions (Modules 4–5), richer data manipulation (Module 6), and dramatically richer querying (Modules 7 through 17 — this is where the vast majority of SQL's expressive power lives).

## Internal Working (Preview)

For the `SELECT` statement above, conceptually the database:
1. Locates the `employees` table (via its catalog/metadata — Topic 3's "Internal Working").
2. Evaluates the `WHERE` condition against each row to decide which rows qualify (in later modules, an index can make this step far faster than checking every row — Module 13).
3. Sorts the qualifying rows per `ORDER BY`.
4. Projects (keeps) only the requested columns (`name`, `salary`), discarding `id` and `department` from the output — even though they were used for filtering, they weren't asked for in the result.

This order — filter, then sort, then choose which columns to display — is close to, but not exactly, the order SQL statements are actually processed internally; Module 7 covers the precise logical processing order of a `SELECT` in full.

## Real-World Analogy

This three-step loop is like setting up a filing cabinet: first you label the drawers and folders (`CREATE TABLE` — defining structure), then you actually file documents into those folders (`INSERT` — adding data), and finally, whenever you need something, you go look through the folders for what matches your need (`SELECT` — querying) without ever needing to relabel the drawers again.

## Advantages of This Foundational Pattern

- **Separation of concerns** — defining structure, adding data, and asking questions are cleanly distinct operations, each with its own statement type.
- **Structure is defined once, reused forever** — you don't redefine what an "employee" looks like every time you add one or query them.
- **Querying is endlessly composable** — the same `employees` table can answer an unlimited variety of questions without ever touching its structure or its stored data.

## Limitations to Be Aware Of Early

- A table this simple (no relationships to other tables) is a toy example — real schemas involve multiple related tables (Module 2, Module 10 — Joins). This topic deliberately keeps things minimal so the core loop is crystal clear before complexity is introduced.
- `SERIAL` and exact syntax for auto-incrementing IDs differs across database vendors (Module 22 covers this) — the concept (an automatically assigned unique identifier) is universal, even where the keyword isn't.

## Best Practices

- Always type your SQL with clear line breaks between clauses (`SELECT` on one line, `FROM` on the next, `WHERE` on the next, etc.), as shown above — it's not required by the language, but it dramatically improves readability as queries grow more complex in later modules.
- Get in the habit of ending every statement with a semicolon — required by `psql` to know a statement is complete (Topic 4), and required by virtually every real SQL tool.
- Prefer naming exact columns in `SELECT` (`SELECT name, salary`) over `SELECT *` (select everything) once you're writing real, non-throwaway queries — explicit columns are more readable and more resilient to future table changes. (`SELECT *` is fine for quick, exploratory looks at data, which is why you'll still see it used casually throughout this course.)

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using double quotes for text values, e.g. `WHERE department = "Sales"` | In standard SQL, single quotes (`'Sales'`) are for string literals (actual data values); double quotes are reserved for quoting identifiers (table/column names) with unusual casing or characters. Mixing these up is a very common beginner error. |
| Forgetting `WHERE` and being surprised `UPDATE`/`DELETE` affected every row | Not demonstrated in this topic's example, but worth flagging early — Module 6 covers this danger in full. A `SELECT` without `WHERE` just returns every row (usually harmless); an `UPDATE`/`DELETE` without `WHERE` changes/removes every row (often catastrophic). |
| Assuming column order in the result matches the table's original column order | `SELECT name, salary` returns exactly `name` then `salary`, in that order — the output order is whatever you list in `SELECT`, not the table's underlying storage order. |

## Interview Questions

1. **Q: Walk through what happens, statement by statement, when you run `CREATE TABLE`, then `INSERT`, then `SELECT` against a brand-new table.**
   A: `CREATE TABLE` (DDL) defines the table's structure — its columns and their types — with no data yet. `INSERT` (DML) adds actual rows of data conforming to that structure. `SELECT` (DQL) reads and returns data matching specified conditions, without modifying anything, and can filter, sort, and choose which columns to display.

2. **Q: Why doesn't the `INSERT` statement in this topic's example mention the `id` column?**
   A: Because `id` was declared `SERIAL`, meaning PostgreSQL automatically generates the next sequential integer value for it on every insert — you don't (and normally shouldn't) supply it manually.

3. **Q: What's the difference between single quotes and double quotes in SQL?**
   A: Single quotes delimit string literal values (actual data, like `'Sales'`). Double quotes delimit identifiers — table or column names — and are only needed when a name has unusual casing, spaces, or reserved words; using double quotes around what should be a string value is a common and incorrect beginner mix-up.

## Summary

- The foundational SQL loop is: **`CREATE TABLE`** (define structure, DDL) → **`INSERT`** (add data, DML) → **`SELECT`** (ask questions, DQL).
- `CREATE TABLE` runs once to establish shape; `INSERT` runs repeatedly to add data over time; `SELECT` runs as often as needed to answer questions, without ever touching structure or existing data.
- Single quotes are for string values; double quotes are for unusually-named identifiers — don't confuse the two.
- Every module from here on deepens one of these three steps — Module 4 and 5 go deeper on table structure, Module 6 goes deeper on data manipulation, and Modules 7–17 go deep on the full power of querying.
