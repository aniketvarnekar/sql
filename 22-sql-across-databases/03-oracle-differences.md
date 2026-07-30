# Oracle Differences

## Learning Objectives

By the end of this section you should be able to:
- Translate PostgreSQL's `LIMIT` into Oracle's `ROWNUM` pattern and its modern `FETCH FIRST` syntax
- Explain why Oracle historically required an explicit `NEXTVAL` call against a sequence, in contrast to PostgreSQL's auto-generated identity columns
- Explain what the `DUAL` table is and why it exists
- Identify `VARCHAR2` as Oracle's string type and explain how it differs in name (and subtly in behavior) from PostgreSQL's `VARCHAR`
- Describe PL/SQL as Oracle's procedural extension and contrast it with PostgreSQL's PL/pgSQL
- Translate a PostgreSQL `ON CONFLICT` upsert into Oracle's `MERGE` statement

## Prerequisites

- [SQL Server (T-SQL) Differences](02-sql-server-differences.md) — introduces the `OFFSET`/`FETCH` standard-SQL pagination form that Oracle's modern syntax also uses, and the `MERGE` upsert pattern this topic reuses for Oracle.
- [Primary Keys](../05-constraints-and-keys/03-primary-keys.md) — needed to contrast PostgreSQL's auto-generated identity mechanisms with Oracle's historical sequence-and-`NEXTVAL` pattern.
- Module 18 (Procedures, Functions & Triggers) — needed to meaningfully contrast PostgreSQL's PL/pgSQL with Oracle's PL/SQL as two different vendor procedural extensions serving the same purpose.

## Motivation

Oracle Database has, for decades, been the database of choice for large enterprises, banks, telecoms, and government systems with the deepest pockets for licensing and the highest bar for reliability at massive scale. Oracle also predates several standard-SQL features you've learned in this course by many years, which means it accumulated its own vocabulary and idioms for problems PostgreSQL solves more directly — and, notably, some of Oracle's oldest idioms (like `ROWNUM`) are still in widespread production use today even though Oracle itself now also supports the modern standard-SQL alternative. Recognizing both the old and new forms is genuinely useful, since you will encounter both in real Oracle codebases.

## Problem Statement

You're handed a decades-old Oracle query during a migration project:

```sql
SELECT * FROM (
    SELECT e.*, ROWNUM AS rn
    FROM employees e
    ORDER BY salary DESC
) WHERE rn <= 5;
```

At first glance this looks nothing like the PostgreSQL you know: `SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 5;`. Why does the Oracle version need a nested subquery, a mysterious `ROWNUM`, and a `WHERE` clause on the *outside* of the ordering, instead of a single trailing keyword? Understanding `ROWNUM`'s quirky behavior — and knowing that modern Oracle also accepts a much more familiar `FETCH FIRST` form — is exactly the kind of gap this topic closes.

## Concept

### Row Limiting: `ROWNUM` vs. `FETCH FIRST`

Oracle's original mechanism for limiting rows is the pseudo-column `ROWNUM`, which numbers rows **as they are produced by the query**, before any `ORDER BY` in the same query level is applied. This ordering quirk is the single most common Oracle beginner trap:

```sql
-- WRONG on Oracle: this does NOT return the 5 highest-paid employees
SELECT name, salary
FROM employees
WHERE ROWNUM <= 5
ORDER BY salary DESC;
```

This query filters to *some arbitrary* 5 rows first (whichever 5 the database happens to produce first, entirely unrelated to salary) and only *then* sorts those 5 by salary — it does not return the 5 highest earners. The correct historical pattern wraps the ordered query in a subquery, so `ROWNUM` is applied to already-sorted rows:

```sql
-- Correct historical Oracle pattern
SELECT name, salary FROM (
    SELECT name, salary
    FROM employees
    ORDER BY salary DESC
) WHERE ROWNUM <= 5;
```

Modern Oracle (12c and later) supports the same standard-SQL `OFFSET`/`FETCH` syntax introduced for SQL Server in Topic 2 of this module, which avoids the `ROWNUM` ordering trap entirely:

```sql
-- Modern Oracle (12c+) — equivalent to PostgreSQL's LIMIT 5
SELECT name, salary
FROM employees
ORDER BY salary DESC
FETCH FIRST 5 ROWS ONLY;

-- With pagination (skip 10, take 5)
SELECT name, salary
FROM employees
ORDER BY salary DESC
OFFSET 10 ROWS FETCH NEXT 5 ROWS ONLY;
```

| Aspect | PostgreSQL `LIMIT` | Oracle `ROWNUM` (legacy) | Oracle `FETCH FIRST` (modern) |
|---|---|---|---|
| Applies before or after `ORDER BY` in the same query? | After | **Before** — a common source of bugs if not wrapped in a subquery | After (correct by construction) |
| Requires a subquery to combine with sorting correctly? | No | Yes, effectively always | No |
| Standard SQL? | Close, widely supported | No — Oracle-specific pseudo-column | Yes — ANSI SQL:2008, same form used by SQL Server |

Even in modern Oracle databases, you will frequently encounter the `ROWNUM`-subquery pattern in existing production code, because it long predates `FETCH FIRST`'s availability — recognizing it, and its ordering trap, remains a practical necessity.

### Sequences and `NEXTVAL`: Explicit Generation vs. Automatic Identity

PostgreSQL's `SERIAL` and `GENERATED ALWAYS AS IDENTITY` (Module 5) make a column auto-generate its next value invisibly, as a side effect of `INSERT`. Oracle's traditional model makes this explicit: you first create a standalone **sequence** object, then call its `NEXTVAL` pseudo-column yourself inside every `INSERT`:

```sql
-- Oracle (traditional pattern)
CREATE SEQUENCE employees_seq START WITH 1 INCREMENT BY 1;

CREATE TABLE employees (
    id     NUMBER PRIMARY KEY,
    name   VARCHAR2(100) NOT NULL
);

INSERT INTO employees (id, name) VALUES (employees_seq.NEXTVAL, 'Asha');
```

Compare this to PostgreSQL, where the sequence exists but is invoked *for* you:

```sql
-- PostgreSQL
CREATE TABLE employees (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

INSERT INTO employees (name) VALUES ('Asha');  -- id is filled in automatically
```

Modern Oracle (12c and later) also supports `GENERATED ALWAYS AS IDENTITY`, syntactically almost identical to PostgreSQL's modern form — closing most of this historical gap:

```sql
-- Modern Oracle (12c+) — much closer to PostgreSQL
CREATE TABLE employees (
    id   NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR2(100) NOT NULL
);
```

The important conceptual point survives regardless of era: **a sequence is a genuinely separate database object** in both PostgreSQL and Oracle (unlike MySQL's table-metadata-based `AUTO_INCREMENT`, Topic 1 of this module) — Oracle's older pattern simply required you to name and call it explicitly, rather than having a shorthand column type wire that call in for you automatically.

### The `DUAL` Table

Oracle requires every `SELECT` to name a table in a `FROM` clause — there is no way to `SELECT` a bare expression with nothing to select *from*. To work around this for evaluating an expression with no real table involved (e.g., checking the current date, or evaluating simple arithmetic), Oracle provides a special built-in one-row, one-column table named `DUAL`:

```sql
-- Oracle
SELECT SYSDATE FROM DUAL;
SELECT 1 + 1 FROM DUAL;
```

PostgreSQL has no such requirement — `FROM` is entirely optional when you're not actually reading from a table:

```sql
-- PostgreSQL — no DUAL needed
SELECT NOW();
SELECT 1 + 1;
```

```
 ?column?
----------
        2
(1 row)
```

`DUAL` is purely a syntactic accommodation for Oracle's stricter grammar rule ("every `SELECT` needs a `FROM`") — it holds no meaningful data of its own (conceptually, one row, one column, used only as a placeholder target).

### `VARCHAR2`

Oracle's variable-length string type is spelled `VARCHAR2`, not `VARCHAR`:

```sql
-- Oracle
CREATE TABLE customers (
    name VARCHAR2(100)
);

-- PostgreSQL
CREATE TABLE customers (
    name VARCHAR(100)
);
```

Oracle does technically also have a type called `VARCHAR`, but Oracle's own documentation has, for a long time, recommended never using it and always using `VARCHAR2` instead — `VARCHAR`'s exact semantics have been left officially reserved for potential future redefinition by Oracle, making `VARCHAR2` the only stable, safe choice in real Oracle schemas. There is also a subtler behavioral difference worth knowing: historically, Oracle treats an empty string (`''`) as equivalent to `NULL` for `VARCHAR2` columns — a genuinely surprising divergence from both the SQL standard and PostgreSQL, where an empty string and `NULL` are always distinct values (Module 3 covers `NULL` semantics in depth).

### PL/SQL: Oracle's Procedural Extension

Just as PostgreSQL extends standard SQL with PL/pgSQL (Module 18) for functions, procedures, and triggers, Oracle extends it with its own procedural language, **PL/SQL** — one of the oldest and most mature vendor procedural SQL extensions, predating PL/pgSQL by many years (PL/pgSQL's own design was, in fact, directly influenced by PL/SQL):

```sql
-- A tiny PL/SQL procedure, for orientation only
CREATE OR REPLACE PROCEDURE give_raise (
    emp_id IN NUMBER,
    amount IN NUMBER
) AS
BEGIN
    UPDATE employees SET salary = salary + amount WHERE id = emp_id;
    COMMIT;
END;
/
```

The family resemblance to PL/pgSQL is strong on purpose: `BEGIN`/`END` blocks, `IN`/`OUT` parameters, and imperative constructs like `IF`/`LOOP` layered over standard SQL statements. The differences are mostly surface-level syntax (Oracle's `/` statement terminator in many client tools, its specific exception-handling clause names) rather than conceptual.

### Upsert: `MERGE`

Oracle is, in fact, the database that originally introduced the `MERGE` statement to SQL — SQL Server's `MERGE` (Topic 2 of this module) was modeled after Oracle's. The syntax is close to identical to the SQL Server version already shown:

```sql
-- Oracle
MERGE INTO customers target
USING (SELECT 1 AS id, 'asha@example.com' AS email FROM DUAL) source
ON (target.id = source.id)
WHEN MATCHED THEN
    UPDATE SET email = source.email
WHEN NOT MATCHED THEN
    INSERT (id, email) VALUES (source.id, source.email);
```

Note the `FROM DUAL` inside the `USING` clause's inline source — a direct consequence of Oracle's "every `SELECT` needs a `FROM`" rule discussed above, showing up in a completely different context.

## Internal Working

`ROWNUM`'s "before `ORDER BY`" quirk is a direct, visible consequence of the logical order in which a `SELECT` is processed. Oracle assigns `ROWNUM` values to rows as they are produced by the query's `FROM`/`WHERE` evaluation — before any requested sort has been applied:

```
 FROM/WHERE evaluated → rows numbered by ROWNUM as produced → (then, separately) ORDER BY sorts
                              ↑
                 ROWNUM is fixed to rows HERE, not after sorting
```

This is exactly why filtering `WHERE ROWNUM <= 5` in the same query level as an `ORDER BY` filters the *wrong* 5 rows — the numbering already happened before the sort. Wrapping the sorted query in a subquery forces a fresh, separate query level: the inner query fully sorts its rows and returns them as a materialized-in-effect result, and only then does the outer query's own `ROWNUM` (or `FETCH FIRST`, which has no such ordering trap by construction) apply to already-sorted data.

## Real-World Analogy

`ROWNUM`'s trap is like a raffle where numbered tickets are handed out to people as they enter a room, and only afterward does someone announce "line up by height." If you asked for "tickets 1 through 5" *before* the height-based lineup happened, you'd get five essentially random people, not the five shortest (or tallest). To get the five shortest, everyone has to line up by height *first*, and only then should ticket numbers 1 through 5 be handed out. Oracle's subquery-wrapping pattern is exactly this: sort first (inside), then number and filter (outside).

## Why These Differences Exist

Oracle Database's core architecture dates to the late 1970s, making it one of the oldest commercial relational database products in continuous use — many of its idioms (`ROWNUM`, mandatory sequences with explicit `NEXTVAL`, `DUAL`, `VARCHAR2`) were established well before the SQL standard later codified more ergonomic equivalents (`FETCH FIRST`, auto-generated identity columns), and Oracle has maintained backward compatibility with its original forms for decades rather than removing them, even after adopting the modern standard alternatives alongside them. `DUAL` in particular is a direct consequence of a stricter early grammar decision — that a `SELECT` must always specify a `FROM` — which PostgreSQL simply never adopted as a requirement. None of this reflects "outdated" engineering; it reflects the practical reality that a multi-decade-old, still actively used product accumulates historical syntax it cannot simply delete without breaking enormous amounts of existing production code.

## Advantages

- **Recognizing the `ROWNUM` pattern is directly useful for reading legacy production code** — an enormous amount of real-world Oracle SQL still uses it, predating widespread `FETCH FIRST` adoption.
- **`FETCH FIRST`/`OFFSET` knowledge from this topic and Topic 2 transfers directly** — it is the same standard-SQL syntax in both SQL Server and modern Oracle, making it one of the most portable idioms across this entire module.
- **Understanding PL/SQL deepens your appreciation of PL/pgSQL** (Module 18) — seeing the language PostgreSQL's own procedural extension was directly inspired by clarifies *why* PL/pgSQL is shaped the way it is.

## Disadvantages / Limitations

- **The `ROWNUM` ordering trap is a genuine, easy-to-miss correctness bug**, not merely an unfamiliar syntax — code that looks superficially reasonable can silently return the wrong rows.
- **`DUAL` and mandatory sequence-and-`NEXTVAL` calls add verbosity** for very simple operations (evaluating a single expression, generating one key) compared to PostgreSQL's more permissive, shorthand-friendly syntax.
- **The empty-string-as-`NULL` behavior for `VARCHAR2`** is a subtle correctness hazard when porting logic from a database (like PostgreSQL) where empty string and `NULL` are always distinct.

## Best Practices

- Always wrap an `ORDER BY`-then-`ROWNUM` query in a subquery on legacy Oracle systems (or, on modern Oracle, simply use `FETCH FIRST` instead and avoid the pattern entirely).
- Prefer `GENERATED ALWAYS AS IDENTITY` over explicit sequence-and-`NEXTVAL` calls in new Oracle 12c+ schemas — it is both less error-prone and closer to how PostgreSQL, this course's reference dialect, already expresses the same idea.
- When porting `VARCHAR2` logic that relies on distinguishing an empty string from `NULL`, test explicitly against Oracle's actual behavior rather than assuming standard-SQL semantics — this is one of the module's sharper, less-obvious gotchas.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `WHERE ROWNUM <= 5 ORDER BY salary DESC` in one query level and expecting the top 5 salaries | `ROWNUM` is assigned before `ORDER BY` sorts the rows at that same query level, so the filter applies to arbitrary rows, not the highest earners — the sorted query must be wrapped in a subquery first. |
| Forgetting `FROM DUAL` when evaluating a bare expression in Oracle | Oracle requires every `SELECT` to have a `FROM` clause; `DUAL` exists specifically to satisfy that requirement when no real table is involved. |
| Assuming an empty string behaves identically to PostgreSQL when stored in an Oracle `VARCHAR2` column | Oracle has historically treated `''` as equivalent to `NULL` for this type — a genuine semantic divergence, not just a naming difference, that can silently change the results of `IS NULL` checks and comparisons. |
| Treating `VARCHAR` and `VARCHAR2` as interchangeable in Oracle | Oracle documentation reserves `VARCHAR`'s exact behavior for potential future change and recommends `VARCHAR2` exclusively — using `VARCHAR` risks depending on semantics Oracle has explicitly not committed to. |

## Interview Questions

1. **Q: Why does `SELECT * FROM employees WHERE ROWNUM <= 5 ORDER BY salary DESC` not return the 5 highest-paid employees on Oracle?**
   A: `ROWNUM` is assigned to rows as they are produced by the query, before `ORDER BY` sorts them within that same query level — so the `WHERE ROWNUM <= 5` filter is applied to an arbitrary 5 rows, not the sorted top 5. The correct approach sorts the data in an inner subquery first, then applies `ROWNUM` filtering (or `FETCH FIRST`) in an outer query against the already-sorted result.

2. **Q: What is the `DUAL` table in Oracle, and why does PostgreSQL not need an equivalent?**
   A: `DUAL` is a built-in, one-row, one-column table Oracle provides so that expressions with no real table source (like `SELECT SYSDATE FROM DUAL`) can still satisfy Oracle's grammar rule that every `SELECT` must include a `FROM` clause. PostgreSQL has no such rule — `FROM` is optional, so `SELECT NOW();` is valid on its own.

3. **Q: How does Oracle's traditional approach to auto-generated primary keys differ from PostgreSQL's `SERIAL`?**
   A: Oracle's traditional pattern requires creating a standalone `SEQUENCE` object and explicitly calling `sequence_name.NEXTVAL` inside every `INSERT` that needs a new key value. PostgreSQL's `SERIAL` (or modern `GENERATED ALWAYS AS IDENTITY`, which modern Oracle 12c+ also now supports) wires that same sequence call in automatically as the column's default, so the value is generated silently without the `INSERT` statement needing to reference it explicitly.

## Summary

- Oracle's legacy `ROWNUM` numbers rows before `ORDER BY` sorts them at the same query level, requiring a subquery to correctly combine sorting with row limiting; modern Oracle's `FETCH FIRST`/`OFFSET` (shared with SQL Server) avoids this trap entirely.
- Oracle's traditional key-generation pattern uses an explicit `SEQUENCE` object and a manual `NEXTVAL` call; modern Oracle also supports `GENERATED ALWAYS AS IDENTITY`, much closer to PostgreSQL's approach.
- `DUAL` is Oracle's placeholder one-row table, needed only because Oracle requires every `SELECT` to have a `FROM` clause — PostgreSQL has no such requirement.
- Oracle's string type is `VARCHAR2` (not `VARCHAR`), and it has historically treated empty strings as equivalent to `NULL` — a real semantic gap from PostgreSQL, not just a spelling difference.
- PL/SQL is Oracle's procedural extension, one of the oldest of its kind and a direct influence on PostgreSQL's own PL/pgSQL (Module 18) — the family resemblance between the two is intentional.
- Oracle originated the `MERGE` statement for upserts and reconciliation; its syntax is nearly identical to the SQL Server `MERGE` covered in Topic 2.
