# SQL Server (T-SQL) Differences

## Learning Objectives

By the end of this section you should be able to:
- Translate PostgreSQL's `LIMIT` into SQL Server's `TOP` and its standard-SQL `OFFSET`/`FETCH` alternative
- Explain how SQL Server's `IDENTITY` column property compares to PostgreSQL's `SERIAL`/`GENERATED ALWAYS AS IDENTITY`
- Contrast square-bracket identifier quoting with PostgreSQL's double-quote quoting
- Compare `GETDATE()` with PostgreSQL's `NOW()`, and `+` string concatenation with PostgreSQL's `||`
- Describe, at a high level, what T-SQL is and how it extends standard SQL with procedural constructs
- Translate a PostgreSQL `ON CONFLICT` upsert into SQL Server's `MERGE` statement

## Prerequisites

- [MySQL Differences](01-mysql-differences.md) — establishes the "same concept, different keyword and default behavior" pattern this topic continues to apply to a second database.
- [Primary Keys](../05-constraints-and-keys/03-primary-keys.md) — needed to compare `IDENTITY` against `SERIAL`.
- [INSERT](../06-modifying-data/01-insert.md) — the baseline `ON CONFLICT` upsert this topic translates into `MERGE`.

## Motivation

Microsoft SQL Server is the dominant relational database inside a huge share of enterprise, government, and .NET-based software environments — if your career takes you into a large established company rather than a startup running PostgreSQL or MySQL, there is a strong chance the production database on the other end of your queries is SQL Server. Its dialect, T-SQL (Transact-SQL), diverges from PostgreSQL in more places than MySQL does, including in some very frequently-used everyday statements, so recognizing its specific vocabulary is directly useful rather than academic.

## Problem Statement

Suppose you're handed this PostgreSQL query, written using techniques from earlier in this course, and asked to produce the SQL Server equivalent:

```sql
SELECT id, first_name || ' ' || last_name AS full_name, hire_date
FROM employees
WHERE hire_date >= NOW() - INTERVAL '30 days'
ORDER BY hire_date DESC
LIMIT 10;
```

Run as-is against SQL Server, this fails at multiple points: `||` is not string concatenation in T-SQL (it's a bitwise-OR-like operator in some contexts and simply invalid here), `NOW()` does not exist, `INTERVAL '30 days'` is not valid T-SQL syntax, and `LIMIT` does not exist at all. Every one of these has a direct, learnable T-SQL equivalent — this topic builds that translation table.

## Concept

### Row Limiting: `TOP` vs. `LIMIT`

PostgreSQL uses `LIMIT` (optionally with `OFFSET`) at the *end* of a query. SQL Server's classic mechanism, `TOP`, sits immediately after `SELECT`, at the *start*:

```sql
-- PostgreSQL
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 5;

-- SQL Server (TOP)
SELECT TOP 5 name, salary FROM employees ORDER BY salary DESC;
```

`TOP` alone has no direct concept of "skip N rows first" the way `OFFSET` does. For pagination, modern SQL Server (2012 and later) supports the standard-SQL `OFFSET`/`FETCH` syntax instead, which maps much more directly onto PostgreSQL's `LIMIT ... OFFSET ...`:

```sql
-- PostgreSQL
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 5 OFFSET 10;

-- SQL Server (OFFSET / FETCH — requires ORDER BY)
SELECT name, salary
FROM employees
ORDER BY salary DESC
OFFSET 10 ROWS FETCH NEXT 5 ROWS ONLY;
```

| Aspect | PostgreSQL `LIMIT`/`OFFSET` | SQL Server `TOP` | SQL Server `OFFSET`/`FETCH` |
|---|---|---|---|
| Position in query | End | Right after `SELECT` | End |
| Requires `ORDER BY`? | No (result order undefined without it) | No | **Yes** — a hard requirement of the syntax |
| Supports skipping rows (pagination) | Yes, via `OFFSET` | No, not directly | Yes, via `OFFSET ... ROWS` |
| Standard SQL? | Close to it, widely supported | No — SQL Server-specific | Yes — this is the ANSI SQL:2008 standard form |

Because `OFFSET`/`FETCH` is the standard-SQL form, it is the version most worth learning if you expect to move between multiple databases — Oracle's modern syntax (Topic 3 of this module) uses the identical `OFFSET ... FETCH NEXT ... ROWS ONLY` structure.

### `IDENTITY` Columns

SQL Server's equivalent of `SERIAL` is the `IDENTITY` column property, specified with a seed (starting value) and an increment:

```sql
-- PostgreSQL
CREATE TABLE employees (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

-- SQL Server
CREATE TABLE employees (
    id   INT IDENTITY(1,1) PRIMARY KEY,
    name NVARCHAR(100) NOT NULL
);
```

`IDENTITY(1,1)` means "start at 1, increase by 1 each time" — either number can be changed (e.g., `IDENTITY(1000,5)` starts at 1000 and increments by 5). Conceptually this is extremely close to PostgreSQL's modern `GENERATED ALWAYS AS IDENTITY` syntax (both use the word "identity" and both express the same "seed and increment" idea), which is a useful mnemonic bridge if you learned PostgreSQL's identity-column form rather than the older `SERIAL` shorthand.

### Identifier Quoting: Square Brackets vs. Double Quotes

Where PostgreSQL uses double quotes for unusually-named identifiers, SQL Server uses square brackets:

```sql
-- PostgreSQL
SELECT "order", "Customer Name" FROM "Orders";

-- SQL Server
SELECT [order], [Customer Name] FROM [Orders];
```

SQL Server does also accept double quotes for identifiers under a specific session setting (`SET QUOTED_IDENTIFIER ON`, which is the default in most modern client tools), but square brackets are the idiomatic, always-safe form you will see in essentially all real T-SQL code and generated scripts.

### Current Date/Time: `GETDATE()` vs. `NOW()`

```sql
-- PostgreSQL
SELECT NOW();

-- SQL Server
SELECT GETDATE();
```

Both return the current date and time from the server. SQL Server also offers `SYSDATETIME()` for a higher-precision variant and `GETUTCDATE()` for the current UTC time, mirroring PostgreSQL's own family of date/time functions covered in Module 3 — the underlying need (know "now," in various precisions and time zones) is universal; only the function name differs.

### String Concatenation: `+` vs. `||`

```sql
-- PostgreSQL
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- SQL Server
SELECT first_name + ' ' + last_name AS full_name FROM employees;
```

This is a genuinely dangerous divergence, not just a cosmetic one: `+` is also the arithmetic addition operator in both databases. In PostgreSQL, using `||` for concatenation and `+` for addition keeps the two operations visually and syntactically distinct. In SQL Server, `+` does double duty — the same operator means "add" for numbers and "concatenate" for strings, and mixing a `NULL` or a numeric column into a `+`-based string expression can silently produce a `NULL` result or an unexpected type-conversion error rather than the concatenated string you intended.

```
-- SQL Server: a NULL anywhere in a + chain silently makes the whole result NULL
SELECT 'Hello, ' + NULL + '!';
```
```
(No column name)
----------------
NULL
```

PostgreSQL's `||` behaves the same way with `NULL` (any `NULL` operand makes the whole concatenation `NULL`), so this specific pitfall isn't unique to SQL Server — but the *reuse* of `+` for both arithmetic and string concatenation is what makes T-SQL expressions easier to misread at a glance.

### T-SQL's Procedural Extensions, Briefly

Just as PostgreSQL extends standard SQL with the procedural language PL/pgSQL (Module 18) for writing functions, procedures, and trigger bodies, SQL Server extends standard SQL with its own procedural dialect, **T-SQL**, used the same way — for control flow (`IF`/`WHILE`), local variables (`DECLARE @x INT`), and stored procedure/trigger bodies:

```sql
-- A tiny T-SQL procedure body, for orientation only
CREATE PROCEDURE GetHighEarners
AS
BEGIN
    SELECT name, salary FROM employees WHERE salary > 100000;
END;
```

The concept — a database-native procedural language for packaging logic that runs *inside* the database, close to the data — is identical to what Module 18 taught with PL/pgSQL; only the specific syntax (`@variable` prefixes, `BEGIN`/`END` blocks used more pervasively, `GO` as a batch separator in client tools) differs.

### Upsert: `MERGE` vs. `ON CONFLICT`

PostgreSQL's `INSERT ... ON CONFLICT` (Module 6) is a single, purpose-built statement for upserts. SQL Server's `MERGE` statement is more general — it can combine insert, update, and delete logic against a target table based on a join with a source, and covers upsert as one of its use cases:

```sql
-- PostgreSQL
INSERT INTO customers (id, email)
VALUES (1, 'asha@example.com')
ON CONFLICT (id) DO UPDATE SET email = EXCLUDED.email;

-- SQL Server
MERGE customers AS target
USING (VALUES (1, 'asha@example.com')) AS source (id, email)
ON target.id = source.id
WHEN MATCHED THEN
    UPDATE SET email = source.email
WHEN NOT MATCHED THEN
    INSERT (id, email) VALUES (source.id, source.email);
```

| Aspect | PostgreSQL `ON CONFLICT` | SQL Server `MERGE` |
|---|---|---|
| Scope | Purpose-built for one row (or one batch) vs. one conflict target | General-purpose: can insert, update, *and* delete in a single statement against an arbitrary source |
| Readability for a simple upsert | Compact, single clause | More verbose — requires an explicit `USING ... ON ...` join and separate `WHEN MATCHED`/`WHEN NOT MATCHED` branches |
| Source of incoming data | The `VALUES`/`SELECT` in the `INSERT` itself | Any table, view, or derived result set named after `USING` |

`MERGE` is considerably more powerful in scope than `ON CONFLICT` — it is really a general "reconcile a target table against a source" tool that happens to cover upsert as a special case — but for the everyday "insert, or update if it already exists" scenario, it takes noticeably more code to express the same intent.

## Internal Working

The `OFFSET`/`FETCH` requirement that `ORDER BY` must be present is not an arbitrary restriction — it reflects something genuinely true of every database's execution model, including PostgreSQL's: "give me rows 11 through 15" is a meaningless request unless the rows have a defined order first. PostgreSQL's `LIMIT`/`OFFSET` *permits* you to omit `ORDER BY` (the database will simply return whatever rows it encounters first internally, in no reliable order), while SQL Server's `OFFSET`/`FETCH` syntax refuses to compile at all without one. Functionally, this makes SQL Server's version the stricter, arguably safer form — it prevents you from ever writing a pagination query whose page contents are undefined between runs, a subtle correctness bug PostgreSQL's more permissive syntax allows you to accidentally write.

```
PostgreSQL:  LIMIT/OFFSET without ORDER BY   →  compiles, runs, result order undefined
SQL Server:  OFFSET/FETCH without ORDER BY   →  compile-time error, forces correctness
```

## Real-World Analogy

Think of `LIMIT`/`OFFSET` versus `TOP`/`OFFSET`-`FETCH` like two different countries' approaches to a queue-numbering system at a service counter. One country's ticket machine (`LIMIT`) lets you request "give me the next 5 people" without first confirming everyone is standing in a defined line — it will hand you *some* 5 people, but which 5 depends on how they happened to be standing. The other country's machine (`OFFSET`/`FETCH`) simply refuses to dispense a ticket at all until the queue has been explicitly ordered — a stricter but more reliable guarantee that "person 11 through 15 in line" actually means something consistent every time you ask.

## Why These Differences Exist

SQL Server's `TOP` predates the SQL:2008 standard's `OFFSET`/`FETCH` clause by many years — it was Microsoft's own early solution to the same "limit how many rows come back" problem PostgreSQL solved with `LIMIT`, and it stuck around for backward compatibility even after the standard form was added. The reuse of `+` for both arithmetic and string concatenation traces back to T-SQL's older heritage (shared conceptual roots with Sybase SQL Server), where a single overloaded operator was considered acceptable economy of syntax; PostgreSQL's `||` (borrowed directly from the ANSI SQL standard's own concatenation operator) reflects a design choice to keep the two operations visually distinct precisely to avoid the `NULL`-propagation and type-ambiguity confusion described above. In both cases, the differences are historical accidents of when and by whom each database was designed, not evidence that one approach is universally correct.

## Advantages

- **`OFFSET`/`FETCH` is standard SQL** — learning it in the SQL Server context transfers directly to Oracle's modern pagination syntax (Topic 3) and is close to PostgreSQL's own `OFFSET` clause, making it the single most portable pagination idiom across all four databases in this module.
- **`MERGE`'s generality is genuinely useful once you need it** — a single statement that can insert, update, and delete based on a comparison between a target and a source table covers reconciliation scenarios (e.g., syncing a nightly batch of changes) that would otherwise require three separate statements.
- **Recognizing T-SQL as "PL/pgSQL's cousin"** rather than a wholly alien language makes reading unfamiliar stored procedures and triggers in a SQL Server codebase far less intimidating.

## Disadvantages / Limitations

- **`TOP` without `OFFSET`/`FETCH` has no native pagination story** — skipping rows requires workarounds (commonly wrapping the query in a CTE with `ROW_NUMBER()`, previewed conceptually in Module 16/17) rather than a simple keyword.
- **`+` for string concatenation is a real footgun**, not just an unfamiliar keyword — a stray `NULL` silently nulls out an entire concatenated expression with no error raised.
- **`MERGE`'s verbosity for a simple upsert** can feel like overkill compared to PostgreSQL's single-clause `ON CONFLICT` when all you need is "insert or update one row."

## Best Practices

- Prefer `OFFSET`/`FETCH` over `TOP` for any pagination logic in SQL Server code you write — it forces (and documents) a defined `ORDER BY`, and it is the form most portable to other databases.
- When concatenating strings in T-SQL that might involve a nullable column, wrap nullable operands in `ISNULL(column, '')` (SQL Server's `COALESCE`-like function) to prevent an entire expression from collapsing to `NULL` unexpectedly.
- Reach for `MERGE` when you genuinely need to reconcile a target table against a whole source set (insert new, update changed, optionally delete missing) — for a single-row "insert or update" upsert, it is often more code than the problem needs, though it remains the standard SQL Server tool for it.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `SELECT TOP 5 ... OFFSET 10 ROWS` and expecting it to skip 10 rows | `TOP` has no `OFFSET` clause of its own — mixing the two is invalid syntax; use `OFFSET`/`FETCH` instead if skipping rows is needed. |
| Concatenating a nullable column with `+` and being surprised the whole result is `NULL` | `+` propagates `NULL` through the entire expression in T-SQL, exactly as `||` does in PostgreSQL — the surprise is usually because the column was assumed non-null; wrap it in `ISNULL()`/`COALESCE()` if it can be. |
| Assuming `MERGE` and `ON CONFLICT` are drop-in replacements for each other | `MERGE` requires an explicit source/target join and separate `WHEN MATCHED`/`WHEN NOT MATCHED` branches — it cannot be mechanically substituted keyword-for-keyword for a compact `ON CONFLICT` clause. |
| Using `OFFSET`/`FETCH` in SQL Server without an `ORDER BY` | The syntax requires `ORDER BY` and will raise a compile error without it — unlike PostgreSQL's `LIMIT`/`OFFSET`, which silently allows (but does not recommend) omitting it. |

## Interview Questions

1. **Q: How would you write a query in SQL Server to return rows 11 through 15 of a result set, ordered by salary descending?**
   A: Using the standard `OFFSET`/`FETCH` clause: `SELECT name, salary FROM employees ORDER BY salary DESC OFFSET 10 ROWS FETCH NEXT 5 ROWS ONLY;` — `TOP` alone cannot express the "skip 10" part, so `OFFSET`/`FETCH` is required for true pagination.

2. **Q: Why is `+` considered a riskier string concatenation operator in T-SQL than `||` is in PostgreSQL, even though both propagate `NULL` the same way?**
   A: Because `+` is overloaded to mean both numeric addition and string concatenation in T-SQL, a reader (or a subtle type-conversion bug) can misinterpret an expression's intent at a glance, and mixing a numeric and a string operand can trigger implicit conversions or errors that a dedicated concatenation operator like `||` never risks, since `||` only ever means concatenation.

3. **Q: What is the SQL Server equivalent of PostgreSQL's `INSERT ... ON CONFLICT DO UPDATE`, and how does its scope differ?**
   A: `MERGE`, using a `USING ... ON ...` join between the target table and a source, with `WHEN MATCHED THEN UPDATE` and `WHEN NOT MATCHED THEN INSERT` branches. Its scope is broader than `ON CONFLICT` — a single `MERGE` statement can also delete rows (`WHEN MATCHED ... THEN DELETE`) and reconcile against an arbitrary source table or query, not just a single incoming row.

## Summary

- SQL Server's classic `TOP` limits row count but has no built-in row-skipping; the standard-SQL `OFFSET`/`FETCH` clause (which requires `ORDER BY`) is the true, portable equivalent of PostgreSQL's `LIMIT ... OFFSET ...`.
- `IDENTITY(seed, increment)` is SQL Server's equivalent of `SERIAL`/`GENERATED ALWAYS AS IDENTITY` — same underlying concept, different keyword and configuration syntax.
- SQL Server quotes identifiers with square brackets (`[Order]`) rather than PostgreSQL's double quotes.
- `GETDATE()` replaces `NOW()`; `+` replaces `||` for string concatenation — and `+`'s dual role as both addition and concatenation is a genuine source of bugs, not just an unfamiliar spelling.
- T-SQL is SQL Server's procedural extension, playing the same role PL/pgSQL plays for PostgreSQL (Module 18) — a database-native language for functions, stored procedures, and triggers.
- `MERGE` is SQL Server's upsert mechanism, more general in scope than PostgreSQL's `ON CONFLICT` but noticeably more verbose for the simple "insert or update one row" case.
