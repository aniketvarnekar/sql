# Pivoting Data with Conditional Aggregation

## Learning Objectives

By the end of this section you should be able to:
- Explain why PostgreSQL has no native `PIVOT` keyword and what standard technique replaces it
- Write a query that turns many rows into one row-per-group with one column per category, using `CASE WHEN` inside an aggregate function
- Use PostgreSQL's `FILTER (WHERE ...)` clause as a cleaner alternative to `CASE WHEN` inside an aggregate
- Explain what the `crosstab()` function does and when it's worth reaching for over hand-written conditional aggregation
- Recognize when data should be left in its original "long" form rather than pivoted at all

## Prerequisites

This topic depends on **Module 9 — Aggregation**, specifically [Aggregate Functions](../09-aggregation/01-aggregate-functions.md) (you need `SUM`, `COUNT`, and `GROUP BY` to already feel routine), and on **Module 8 — Functions and Expressions**' `CASE` expressions topic (conditional logic inside a query). Nothing in this topic is new syntax — it is two tools you already have, combined in a new way.

## Motivation

Data is almost always *stored* in "long" form — one row per fact (one row per sale, one row per event, one row per measurement) — because that shape is easy to insert into, easy to constrain, and easy to keep normalized. But data is almost always *reported* in "wide" form — one row per entity, with a column per category (one row per year, with a column for each month; one row per product, with a column for each region's sales). Every spreadsheet, dashboard, and finance report you've ever looked at is in wide form. The gap between how data is stored and how it needs to be presented is the exact problem this topic solves.

## Problem Statement

Suppose you keep a `monthly_sales` table — one row per year/month/amount, the natural, long-form way to store this data as it arrives, a new row appended each month:

```sql
CREATE TABLE monthly_sales (
    sale_year  INTEGER NOT NULL,
    sale_month INTEGER NOT NULL CHECK (sale_month BETWEEN 1 AND 12),
    amount     NUMERIC(12, 2) NOT NULL
);

INSERT INTO monthly_sales (sale_year, sale_month, amount) VALUES
    (2025, 1, 18400.00), (2025, 2, 15200.00), (2025, 3, 21300.00),
    (2025, 4, 19800.00), (2025, 5, 22750.00), (2025, 6, 24100.00),
    (2026, 1, 20950.00), (2026, 2, 17600.00), (2026, 3, 23400.00);
```

A plain query returns exactly this long shape:

```sql
SELECT sale_year, sale_month, amount FROM monthly_sales ORDER BY sale_year, sale_month;
```

```
 sale_year | sale_month | amount
-----------+------------+----------
      2025 |          1 | 18400.00
      2025 |          2 | 15200.00
      2025 |          3 | 21300.00
      2025 |          4 | 19800.00
      2025 |          5 | 22750.00
      2025 |          6 | 24100.00
      2026 |          1 | 20950.00
      2026 |          2 | 17600.00
      2026 |          3 | 23400.00
(9 rows)
```

But what a finance team actually wants for a year-over-year comparison report is one row per year, with a column per month:

```
 sale_year |   jan    |   feb    |   mar    |   apr    |   may    |   jun
-----------+----------+----------+----------+----------+----------+----------
      2025 | 18400.00 | 15200.00 | 21300.00 | 19800.00 | 22750.00 | 24100.00
      2026 | 20950.00 | 17600.00 | 23400.00 |          |          |
```

There is no single `SELECT` clause that produces this shape by simply describing "give me this table, but sideways" — you have to build it, and that's exactly what this topic teaches.

## Concept

### Why PostgreSQL Has No `PIVOT` Keyword

Some database products — SQL Server and Oracle among them — provide a dedicated `PIVOT` clause that performs exactly this row-to-column transformation with special syntax. PostgreSQL deliberately does not. This isn't an oversight: PostgreSQL's philosophy is to keep the core language small and let general-purpose tools it already has (aggregate functions, `CASE`) compose to solve the same problem, rather than adding a special-purpose keyword for one specific shape-transformation. The vendor differences here are covered in depth in Module 22 (SQL Across Databases) — for this course, the technique below is the one you'll use everywhere.

### The `CASE WHEN` Inside an Aggregate Pattern

The core trick: put a `CASE` expression *inside* an aggregate function's argument. For each row, the `CASE` expression evaluates to either the row's value (if it belongs to this column's category) or `NULL` (if it doesn't) — and every aggregate function in SQL silently ignores `NULL` values. The result is that `SUM(CASE WHEN sale_month = 1 THEN amount END)` sums *only* the rows where the month is January, across the whole group, and produces one column per month:

```sql
SELECT
    sale_year,
    SUM(CASE WHEN sale_month = 1 THEN amount END) AS jan,
    SUM(CASE WHEN sale_month = 2 THEN amount END) AS feb,
    SUM(CASE WHEN sale_month = 3 THEN amount END) AS mar,
    SUM(CASE WHEN sale_month = 4 THEN amount END) AS apr,
    SUM(CASE WHEN sale_month = 5 THEN amount END) AS may,
    SUM(CASE WHEN sale_month = 6 THEN amount END) AS jun
FROM monthly_sales
GROUP BY sale_year
ORDER BY sale_year;
```

```
 sale_year |   jan    |   feb    |   mar    |   apr    |   may    |   jun
-----------+----------+----------+----------+----------+----------+----------
      2025 | 18400.00 | 15200.00 | 21300.00 | 19800.00 | 22750.00 | 24100.00
      2026 | 20950.00 | 17600.00 | 23400.00 |          |          |
(2 rows)
```

Notice `GROUP BY sale_year` is still doing exactly what it always does — collapsing many rows per year into one row per year. The only new idea is that instead of one plain `SUM(amount)` per group, there are six *conditional* sums, each looking at a different, mutually exclusive slice of that group's rows. Every row that doesn't match a given month's `CASE` condition contributes `NULL` to that column's sum, which the aggregate simply skips — that's why 2026's `apr`, `may`, `jun` columns come back blank (`NULL`): there are genuinely no rows for those months yet, so every input to those particular sums was `NULL`.

The same pattern works for counting rather than summing. To count how many orders fell into each category per customer, using `COUNT` instead of `SUM`:

```sql
SELECT
    customer,
    COUNT(CASE WHEN category = 'Electronics' THEN 1 END) AS electronics_orders,
    COUNT(CASE WHEN category = 'Furniture' THEN 1 END) AS furniture_orders
FROM orders
GROUP BY customer
ORDER BY customer;
```

```
 customer | electronics_orders | furniture_orders
----------+---------------------+-------------------
 Diego    |                   1 |                 1
 Elena    |                   1 |                 2
 Marcus   |                   2 |                 1
 Priya    |                   2 |                 2
 Sofia    |                   1 |                 1
(5 rows)
```

`COUNT(expression)` counts non-`NULL` values, so `COUNT(CASE WHEN category = 'Electronics' THEN 1 END)` counts exactly the rows where the category matched. A common and easy mistake here is writing `COUNT(CASE WHEN ... THEN 1 ELSE 0 END)` instead — that puts a `0` (not `NULL`) into every non-matching row, and `COUNT` counts `0` the same as any other non-`NULL` value, so it silently returns the *total* row count for every column instead of the category-specific count. See Common Mistakes below.

### PostgreSQL's Cleaner Alternative: `FILTER (WHERE ...)`

PostgreSQL supports a standard-SQL clause, `FILTER`, that expresses the exact same idea more directly, without writing out a `CASE` expression at all:

```sql
SELECT
    sale_year,
    SUM(amount) FILTER (WHERE sale_month = 1) AS jan,
    SUM(amount) FILTER (WHERE sale_month = 2) AS feb,
    SUM(amount) FILTER (WHERE sale_month = 3) AS mar
FROM monthly_sales
GROUP BY sale_year
ORDER BY sale_year;
```

```
 sale_year |   jan    |   feb    |   mar
-----------+----------+----------+----------
      2025 | 18400.00 | 15200.00 | 21300.00
      2026 | 20950.00 | 17600.00 | 23400.00
(2 rows)
```

`FILTER (WHERE condition)` attaches a per-aggregate filter directly, and reads far more naturally than `CASE WHEN ... THEN ... END` buried inside the aggregate's parentheses — it's the same result, produced by a syntax purpose-built for this. `FILTER` is not part of the SQL standard's earliest versions but is a well-supported PostgreSQL feature; the `CASE WHEN` form is more universally portable across databases (Module 22), which is why both are worth knowing.

### The Fixed-Columns Limitation

Both approaches share the same constraint: **you must know the category values (the months, in this example) when you write the query.** A query's result always has a fixed, predetermined set of columns — you cannot ask SQL to "add one column automatically for each distinct value that happens to exist in the data," because the shape of a query's result is part of the query's definition, not something computed at runtime from unknown future data (this connects back to the relational model's expectation, from Module 2, that a query result is itself a well-defined relation with a fixed column set). If the set of categories is genuinely unbounded or user-driven (e.g., an admin can create new product categories at any time), the SQL itself must be regenerated — typically by application code building the column list dynamically — rather than a single static query somehow discovering new columns on its own.

### `crosstab()` — An Alternative from the `tablefunc` Extension

PostgreSQL ships an optional extension, `tablefunc`, that provides a function called `crosstab()` specifically for this row-to-column transformation. It must be enabled once per database:

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;
```

`crosstab()` takes a SQL query (as a text string) that produces three columns — a row identifier, a category, and a value — and reshapes it into one row per identifier with one column per category:

```sql
SELECT * FROM crosstab(
    'SELECT sale_year, sale_month, amount FROM monthly_sales ORDER BY 1, 2',
    'SELECT DISTINCT sale_month FROM monthly_sales ORDER BY 1'
) AS pivoted (
    sale_year INTEGER,
    jan NUMERIC, feb NUMERIC, mar NUMERIC,
    apr NUMERIC, may NUMERIC, jun NUMERIC
);
```

```
 sale_year |   jan    |   feb    |   mar    |   apr    |   may    |   jun
-----------+----------+----------+----------+----------+----------+----------
      2025 | 18400.00 | 15200.00 | 21300.00 | 19800.00 | 22750.00 | 24100.00
      2026 | 20950.00 | 17600.00 | 23400.00 |          |          |
(2 rows)
```

`crosstab()` still requires you to declare the output column list (`AS pivoted (sale_year INTEGER, jan NUMERIC, ...)`) up front, for the exact same reason as above — it isn't truly "dynamic" either, it just saves you from hand-writing a long chain of `CASE WHEN`/`FILTER` expressions when the number of categories is large and fixed (say, pivoting by all 12 months, or by 50 U.S. states). For a small, stable number of categories, plain conditional aggregation is usually simpler to write, read, and maintain than installing and learning an extension; `crosstab()` earns its keep once the category list gets long enough that writing it out by hand becomes tedious.

## Internal Working (Preview)

For each row the query engine scans, the `CASE` (or `FILTER`) expression is evaluated first, independently, per column, per row:

```
Row: (sale_year=2025, sale_month=2, amount=15200.00)
        │
        ├─▶ jan expression:  CASE WHEN sale_month=1 THEN amount END  → NULL   (month is 2, not 1)
        ├─▶ feb expression:  CASE WHEN sale_month=2 THEN amount END  → 15200.00
        ├─▶ mar expression:  CASE WHEN sale_month=3 THEN amount END  → NULL
        ...
        ▼
Each column's running SUM() either adds this row's value, or (if NULL) simply skips it
```

Every row is examined once, but contributes to at most one of the many conditional columns for that category — the rest see `NULL` and are dropped from the sum by the aggregate's own `NULL`-skipping behavior (the same `NULL`-handling rule from [Aggregate Functions](../09-aggregation/01-aggregate-functions.md)). This is why a single pass over the table, with `GROUP BY sale_year`, is enough to compute every month's column simultaneously — there's no separate scan per month.

## Real-World Analogy

Picture a mail sorting facility. Letters arrive in one long, unsorted stream — that's your long-form table, one row (letter) at a time. A sorter stands at a wall of labeled bins, one bin per destination city. For each letter, the sorter checks its destination and drops it into exactly one bin — a letter destined for Chicago contributes nothing to the Boston bin, and vice versa. At the end of the shift, each bin holds a count (or a stack) belonging only to its own city. That's precisely what `SUM(CASE WHEN city = 'Chicago' THEN 1 END)` does for every row: check the condition, and either count it in this bin or contribute nothing at all.

## Why This Technique Was Designed This Way

SQL is declarative and set-based (Module 1): rather than adding an imperative "for each distinct value, create a column" instruction — which would require the engine to inspect the data's actual contents *before* it could even determine the shape of the query's result — the language instead lets you describe the result you want using tools that are already fully declarative and column-shape-fixed at query-definition time: an aggregate function and a conditional expression. This is consistent with the relational model's requirement (Module 2) that a query's result is itself a well-formed relation with a predetermined column list; conditional aggregation achieves the pivot entirely within that constraint, rather than needing an exception to it.

## Advantages

- **Uses only standard, portable SQL** — `CASE WHEN` inside an aggregate works, largely unchanged, on every major relational database, unlike vendor-specific `PIVOT` clauses.
- **No extension or special privilege required** — unlike `crosstab()`, which needs `tablefunc` enabled.
- **Full expressive control** — each column can compute something different (a sum in one, a count in another, an average in a third) within the same query, something a rigid `PIVOT` clause syntax often can't do as flexibly.
- **Composable with everything else you already know** — it's still an ordinary `GROUP BY` query, so `HAVING`, `ORDER BY`, joins, and window functions all still apply normally.

## Disadvantages / Limitations

- **Verbose for many categories** — pivoting by all 12 months or all 50 states means writing (or generating) that many `CASE`/`FILTER` expressions by hand.
- **The column list is fixed at query-write time** — a new category appearing in the data (a 13th "month," a new region) requires editing the query; nothing about this technique adapts to unknown future categories automatically.
- **Not ideal for further computation inside the database** — a wide, pivoted result is meant for final presentation (a report, a dashboard, an export); computing further aggregates *across* the pivoted columns (e.g., "total for the whole year") usually means adding the columns back together, which is more awkward than the long form's `SUM(amount)`.

## Best Practices

- Prefer `FILTER (WHERE ...)` over `CASE WHEN ... THEN ... END` in PostgreSQL when portability to another database isn't a concern — it's shorter and reads more clearly.
- Keep pivoted queries as a presentation-layer step, run last, rather than a form you keep querying further — the long form is almost always easier to filter, join, and re-aggregate.
- If the same pivot shape is needed repeatedly, wrap the query in a view (Module 12) so callers don't have to repeat the full `CASE`/`FILTER` list every time.
- When the number of categories is genuinely unbounded or user-controlled, generate the column list dynamically in application code (or accept `crosstab()`'s same limitation) rather than trying to force a variable-width result out of one static query — no single SQL statement can have a column count that depends on the data it's reading.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `COUNT(CASE WHEN condition THEN 1 ELSE 0 END)` instead of `COUNT(CASE WHEN condition THEN 1 END)` | The `ELSE 0` form produces `0` (not `NULL`) for non-matching rows, and `COUNT` counts `0` just like any other non-`NULL` value — this silently returns the *total* row count for every column instead of the category-specific count. |
| Forgetting `GROUP BY` still applies | Conditional aggregation doesn't remove the need to group rows — it changes *what* is being aggregated per group, not whether grouping is still required. |
| Trying to make the number of output columns depend on the data at query time | A query's result always has a fixed column list, declared when the query is written; no static SQL statement can decide "how many columns" based on what it finds while running. |
| Confusing pivoting (rows → columns) with its reverse, unpivoting (columns → rows) | They are opposite transformations; unpivoting long-form data back out of a wide table typically uses `UNION ALL` of several single-column selects, not conditional aggregation. |

## Interview Questions

1. **Q: PostgreSQL has no `PIVOT` keyword. How do you turn rows into columns anyway?**
   A: Use an aggregate function (commonly `SUM` or `COUNT`) with a `CASE WHEN` expression (or PostgreSQL's `FILTER (WHERE ...)` clause) as its argument, combined with `GROUP BY`. The conditional expression evaluates to the row's value for matching rows and `NULL` otherwise, and the aggregate ignores the `NULL`s, effectively summing/counting only the matching subset per group — one such expression per desired output column.

2. **Q: Why does `SUM(CASE WHEN month = 1 THEN amount END)` work correctly, but `COUNT(CASE WHEN month = 1 THEN amount ELSE 0 END)` would not, for counting January-only rows?**
   A: `SUM` ignores `NULL` inputs, so only matching rows contribute to the sum — non-matching rows contribute nothing. But `COUNT` counts any non-`NULL` value, including `0`; if the `ELSE` branch produces `0` instead of leaving it `NULL`, `COUNT` counts every row, not just the matching ones, giving a wrong (inflated) result.

3. **Q: What is `FILTER (WHERE ...)` and how does it relate to `CASE WHEN` inside an aggregate?**
   A: `FILTER (WHERE condition)` is a PostgreSQL clause attached directly to an aggregate function call that restricts which rows the aggregate considers, without needing a `CASE` expression at all. `SUM(amount) FILTER (WHERE month = 1)` is functionally equivalent to `SUM(CASE WHEN month = 1 THEN amount END)`, just more directly expressed.

4. **Q: When would you reach for `crosstab()` instead of writing `CASE WHEN`/`FILTER` by hand?**
   A: When the number of pivoted categories is large enough (say, dozens) that manually writing that many conditional expressions becomes tedious and error-prone. `crosstab()` (from the `tablefunc` extension) still requires declaring the output column list up front, so it doesn't remove the fixed-columns limitation — it just saves the repetitive typing.

## Summary

- PostgreSQL has no `PIVOT` keyword (unlike SQL Server or Oracle, covered in Module 22) — the standard technique is conditional aggregation: an aggregate function wrapped around a `CASE WHEN` expression, combined with `GROUP BY`.
- Aggregate functions ignore `NULL`, which is exactly what makes `SUM(CASE WHEN condition THEN value END)` work — non-matching rows contribute `NULL` and are silently skipped.
- PostgreSQL's `FILTER (WHERE ...)` clause expresses the identical idea more directly and is generally preferred when portability isn't a concern.
- Every pivoted query has a fixed column list decided at query-write time — there is no way for a single static query to produce a variable number of columns based on the data it reads.
- The `crosstab()` function (from the `tablefunc` extension) is a convenience for pivoting against a large, fixed set of categories, but shares the same fixed-output-column requirement.
- Next, Topic 2 covers `JSON`/`JSONB` — a completely different way PostgreSQL lets you handle data that doesn't fit neatly into fixed columns at all.
