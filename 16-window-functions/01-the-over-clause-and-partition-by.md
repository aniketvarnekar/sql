# The OVER Clause and PARTITION BY

## Learning Objectives

By the end of this section you should be able to:
- Explain, precisely, what a window function is and how it differs from an aggregate function used with `GROUP BY`
- Write a query using `OVER()` to compute a value across a set of related rows without collapsing them
- Use `PARTITION BY` to scope a window calculation to groups within a result set, while keeping every row
- Explain what `ORDER BY` means inside `OVER(...)`, and how it differs from a query's trailing `ORDER BY`
- Predict, correctly, whether a given window function call will return one constant value per group or a row-by-row changing value

## Prerequisites

- This topic assumes you're comfortable with grouped aggregation — `GROUP BY` together with aggregate functions like `SUM`, `AVG`, and `COUNT` (Module 9 — Aggregation). Window functions reuse those exact same aggregate functions; this topic is entirely about the *different way* they get applied.
- This topic assumes you're comfortable with `ORDER BY` for sorting a result set (Module 7 — Querying Basics). `OVER(...)` reuses the `ORDER BY` keyword for a distinct purpose, and mixing up the two is the most common early mistake with window functions — this topic addresses it head-on.
- No prior topic within this module — this is the first topic.

## Motivation

Almost every real reporting question mixes two things that, so far, you've had to choose between: the detail of an individual row, and a summary computed across a group of rows. "Show me each sale, and also what percentage of that salesperson's total this one sale represents" needs *both* — the individual row and the group aggregate, side by side, on the same line. `GROUP BY` cannot do this: the moment you aggregate, the individual rows are gone. Window functions exist specifically to close this gap, and once you see the pattern, you'll find a use for it in nearly every non-trivial query you write from here on.

## Problem Statement

Suppose you run a small sales team and you're tracking monthly performance in a table with one row per salesperson per month. You want to know, for every single sale record, what that salesperson's *total* sales are — printed right next to the individual sale, so you can eyeball how big a chunk of their total any given month was.

With only `GROUP BY`, you have two unsatisfying options:
- Run `GROUP BY salesperson` with `SUM(amount)` — but this collapses four months of detail per salesperson down to one row each, destroying exactly the row-level detail you wanted to keep.
- Run the detail query and the aggregate query separately, then manually (or via a self-join) match them back up — extra work, an extra join, and a second query to keep in sync with the first.

Neither is good. What you actually want is a way to say: "for every row, compute a total *as if you had grouped*, but don't actually throw any rows away." That is precisely what a window function does.

## Concept

### The Running Example: `monthly_sales`

Every topic in this module reuses the same table, so let's set it up once here.

```sql
CREATE TABLE monthly_sales (
    salesperson TEXT    NOT NULL,
    region      TEXT    NOT NULL,
    month       DATE    NOT NULL,
    amount      NUMERIC NOT NULL
);

INSERT INTO monthly_sales (salesperson, region, month, amount) VALUES
    ('Asha',  'East', '2024-01-01', 12000),
    ('Asha',  'East', '2024-02-01', 15000),
    ('Asha',  'East', '2024-03-01', 11000),
    ('Asha',  'East', '2024-04-01', 18000),
    ('Ben',   'East', '2024-01-01',  9000),
    ('Ben',   'East', '2024-02-01',  9500),
    ('Ben',   'East', '2024-03-01', 11000),
    ('Ben',   'East', '2024-04-01', 13000),
    ('Chen',  'West', '2024-01-01', 20000),
    ('Chen',  'West', '2024-02-01', 17000),
    ('Chen',  'West', '2024-03-01', 19000),
    ('Chen',  'West', '2024-04-01', 21000),
    ('Deepa', 'West', '2024-01-01',  8000),
    ('Deepa', 'West', '2024-02-01',  8000),
    ('Deepa', 'West', '2024-03-01', 10000),
    ('Deepa', 'West', '2024-04-01',  9000);
```

Four salespeople, two regions, four months each — 16 rows total. Notice Asha and Ben both sold exactly 11000 in March — that tie will matter in the next topic. Deepa's January and February are both 8000 — that will matter for Topic 4.

### What `GROUP BY` Does (for Contrast)

```sql
SELECT salesperson, SUM(amount) AS total_sales
FROM monthly_sales
GROUP BY salesperson
ORDER BY salesperson;
```

```
 salesperson | total_sales
-------------+-------------
 Asha        |       56000
 Ben         |       42500
 Chen        |       77000
 Deepa       |       35000
(4 rows)
```

Sixteen rows went in; four rows came out. This is exactly what `GROUP BY` is *for* — but it means the individual months (12000, 15000, 11000, 18000 for Asha) are gone from the result. There is no way to get them back in this same query.

### What a Window Function Does Instead

A **window function** computes a value across a *set of rows related to the current row* — called its **window** — but returns that value **for every row**, without collapsing anything. The set of related rows never shrinks; only the *value computed from them* is attached to each row.

```sql
SELECT
    salesperson,
    region,
    month,
    amount,
    SUM(amount) OVER (PARTITION BY salesperson) AS salesperson_total,
    ROUND(AVG(amount) OVER (PARTITION BY salesperson), 2) AS salesperson_avg
FROM monthly_sales
ORDER BY salesperson, month;
```

```
 salesperson | region |   month    | amount | salesperson_total | salesperson_avg
-------------+--------+------------+--------+--------------------+-----------------
 Asha        | East   | 2024-01-01 |  12000 |              56000 |        14000.00
 Asha        | East   | 2024-02-01 |  15000 |              56000 |        14000.00
 Asha        | East   | 2024-03-01 |  11000 |              56000 |        14000.00
 Asha        | East   | 2024-04-01 |  18000 |              56000 |        14000.00
 Ben         | East   | 2024-01-01 |   9000 |              42500 |        10625.00
 Ben         | East   | 2024-02-01 |   9500 |              42500 |        10625.00
 Ben         | East   | 2024-03-01 |  11000 |              42500 |        10625.00
 Ben         | East   | 2024-04-01 |  13000 |              42500 |        10625.00
 Chen        | West   | 2024-01-01 |  20000 |              77000 |        19250.00
 Chen        | West   | 2024-02-01 |  17000 |              77000 |        19250.00
 Chen        | West   | 2024-03-01 |  19000 |              77000 |        19250.00
 Chen        | West   | 2024-04-01 |  21000 |              77000 |        19250.00
 Deepa       | West   | 2024-01-01 |   8000 |              35000 |         8750.00
 Deepa       | West   | 2024-02-01 |   8000 |              35000 |         8750.00
 Deepa       | West   | 2024-03-01 |  10000 |              35000 |         8750.00
 Deepa       | West   | 2024-04-01 |   9000 |              35000 |         8750.00
(16 rows)
```

All 16 rows survive. The `salesperson_total` column shows exactly the same numbers `GROUP BY` computed above (56000, 42500, 77000, 35000) — but repeated on every row belonging to that salesperson, next to the individual month's amount. This is the entire point of a window function: **group-level insight, without losing row-level detail.**

### Anatomy of `OVER(...)`

```sql
SUM(amount) OVER (PARTITION BY salesperson)
```

| Piece | Meaning |
|---|---|
| `SUM(amount)` | An ordinary aggregate function — the same one `GROUP BY` would use. |
| `OVER (...)` | Turns `SUM(amount)` into a **window function call** — "compute this across a window of rows, one result per row, don't collapse." |
| `PARTITION BY salesperson` | Defines the window's boundaries: rows are grouped by `salesperson`, and each row only "sees" the other rows sharing its own salesperson. |

Without any `OVER(...)`, `SUM(amount)` in a plain `SELECT` (with no `GROUP BY`) would need every other column also aggregated or grouped — that's ordinary aggregation rules (Module 9). `OVER(...)` is what tells PostgreSQL "don't fold this into a normal aggregate — evaluate it as a window function instead," which is a fundamentally different execution path, covered in Internal Working below.

### `PARTITION BY` — Windows Within Groups

`PARTITION BY` does for window functions what `GROUP BY` does for aggregation: it splits the rows into groups. The crucial difference is that `PARTITION BY` **never removes rows** — it only defines, for each row, which *other* rows count as part of its window.

You can partition by any column, including ones unrelated to the column you're aggregating:

```sql
SELECT
    salesperson,
    region,
    month,
    amount,
    SUM(amount) OVER (PARTITION BY region) AS region_total
FROM monthly_sales
ORDER BY region, salesperson, month;
```

```
 salesperson | region |   month    | amount | region_total
-------------+--------+------------+--------+--------------
 Asha        | East   | 2024-01-01 |  12000 |        98500
 Asha        | East   | 2024-02-01 |  15000 |        98500
 Asha        | East   | 2024-03-01 |  11000 |        98500
 Asha        | East   | 2024-04-01 |  18000 |        98500
 Ben         | East   | 2024-01-01 |   9000 |        98500
 Ben         | East   | 2024-02-01 |   9500 |        98500
 Ben         | East   | 2024-03-01 |  11000 |        98500
 Ben         | East   | 2024-04-01 |  13000 |        98500
 Chen        | West   | 2024-01-01 |  20000 |       112000
 Chen        | West   | 2024-02-01 |  17000 |       112000
 Chen        | West   | 2024-03-01 |  19000 |       112000
 Chen        | West   | 2024-04-01 |  21000 |       112000
 Deepa       | West   | 2024-01-01 |   8000 |       112000
 Deepa       | West   | 2024-02-01 |   8000 |       112000
 Deepa       | West   | 2024-03-01 |  10000 |       112000
 Deepa       | West   | 2024-04-01 |   9000 |       112000
(16 rows)
```

Now every East row shows the East region's combined total (98500 = Asha's 56000 + Ben's 42500), and every West row shows the West total (112000 = Chen's 77000 + Deepa's 35000) — again, without dropping a single row. Omitting `PARTITION BY` entirely treats the *whole result set* as one giant window — every row would show the same grand total across all 16 rows.

### `ORDER BY` Inside `OVER(...)` — A First Look

So far, every window in this topic has been unordered — `SUM`/`AVG` just added up (or averaged) every row in the partition, in no particular sequence, because summing and averaging don't care about order. But `OVER(...)` also accepts its own `ORDER BY`, and adding one changes the *meaning* of the calculation, not just the row order:

```sql
SELECT
    salesperson,
    month,
    amount,
    SUM(amount) OVER (PARTITION BY salesperson ORDER BY month) AS running_total
FROM monthly_sales
ORDER BY salesperson, month;
```

```
 salesperson |   month    | amount | running_total
-------------+------------+--------+----------------
 Asha        | 2024-01-01 |  12000 |          12000
 Asha        | 2024-02-01 |  15000 |          27000
 Asha        | 2024-03-01 |  11000 |          38000
 Asha        | 2024-04-01 |  18000 |          56000
 Ben         | 2024-01-01 |   9000 |           9000
 Ben         | 2024-02-01 |   9500 |          18500
 Ben         | 2024-03-01 |  11000 |          29500
 Ben         | 2024-04-01 |  13000 |          42500
 Chen        | 2024-01-01 |  20000 |          20000
 Chen        | 2024-02-01 |  17000 |          37000
 Chen        | 2024-03-01 |  19000 |          56000
 Chen        | 2024-04-01 |  21000 |          77000
 Deepa       | 2024-01-01 |   8000 |           8000
 Deepa       | 2024-02-01 |   8000 |          16000
 Deepa       | 2024-03-01 |  10000 |          26000
 Deepa       | 2024-04-01 |   9000 |          35000
(16 rows)
```

Compare this to the very first example in this topic: identical `SUM(amount)`, identical `PARTITION BY salesperson` — but adding `ORDER BY month` turned a flat per-group *total* into a **running total**. This is not a coincidence of formatting; it's a real behavioral change, because `ORDER BY` inside `OVER(...)` changes the default **frame** — the exact subset of the partition each row's calculation includes. Topic 5 of this module (Running Totals and Moving Averages) is dedicated entirely to explaining frames precisely; for now, just internalize this: **`ORDER BY` inside `OVER(...)` is not for sorting your output** (that's still the query's trailing `ORDER BY`) — **it changes what gets computed.**

### `ORDER BY` Inside `OVER()` vs. the Query's Own `ORDER BY`

These are two entirely different things that happen to share a keyword:

| | `ORDER BY` inside `OVER(...)` | `ORDER BY` at the end of the query |
|---|---|---|
| Purpose | Defines the *sequence* used to compute the window function's value (e.g., which rows count as "so far" for a running total) | Defines the *display order* of the final result set |
| Affects | The *values* the window function produces | Only the *row order* shown to you |
| Required? | Only for order-sensitive window functions (running totals, `LAG`/`LEAD`, ranking) | Optional, but without it PostgreSQL gives no ordering guarantee at all (Module 7) |

You will very often see both in the same query, doing genuinely different jobs — as in the running total example above, where `ORDER BY month` inside `OVER()` drives the cumulative sum, and the trailing `ORDER BY salesperson, month` just makes the printed rows easy to read top to bottom.

## Internal Working (Deep Dive)

When PostgreSQL executes a query containing a window function, it runs a distinct step, conceptually *after* `WHERE` and `GROUP BY` have already reduced the row set, but *before* the final `ORDER BY` and `SELECT`'s column list are applied to produce the displayed result:

```
   rows after FROM / WHERE / GROUP BY / HAVING
                    │
                    ▼
        split rows into partitions
      (one partition per PARTITION BY value;
       the whole result set is one partition
             if PARTITION BY is omitted)
                    │
                    ▼
     within each partition, sort rows by
     the OVER(...)'s own ORDER BY, if given
                    │
                    ▼
   for each row, determine its frame (the exact
   subset of its partition the function reads —
   default: whole partition, or a running slice if
   ORDER BY is present — Topic 5 covers this fully)
                    │
                    ▼
   evaluate the window function once per row,
   over that row's frame — attach the result as
   an extra column, without removing any row
                    │
                    ▼
        query's own SELECT list, ORDER BY,
             LIMIT applied as normal
```

This is why window functions cannot appear in a `WHERE` clause directly — `WHERE` is evaluated *before* window functions are computed, so at the point `WHERE` runs, the window function's result doesn't exist yet (this exact restriction, and how to work around it, is covered in the next topic).

## Real-World Analogy

Picture a teacher with a spreadsheet listing every student's score on four quizzes — one row per student per quiz. The teacher wants to add a column next to every single row showing that *student's* average across all four quizzes — without deleting any of the individual quiz rows.

That's `PARTITION BY student`: the teacher isn't collapsing the sheet down to one row per student (that would be `GROUP BY`, and it would throw away which quiz was which). Instead, for every row, the teacher writes in "this student's average" — a value computed *from* a group of related rows, attached back onto each individual row in that group, leaving every quiz score visible. Window functions are that column formula, applied automatically by the database instead of by hand.

## Why the OVER Clause Was Designed This Way

Module 01 established that SQL is a *declarative*, set-based language: you describe the shape of the result you want, not the steps to compute it. Before window functions were added to the SQL standard (in SQL:1999), getting "detail row plus group aggregate" in one result required either a **self-join** (join a table to a `GROUP BY` summary of itself) or fetching detail and aggregate separately and combining them in application code — both of which worked, but both broke the clean declarative style: a self-join for this purpose isn't describing a real relationship between two different things, it's a workaround for a missing language feature.

`OVER(...)` was designed to close that gap directly: it lets you declare "compute this aggregate, across this window of related rows" as a first-class part of a single `SELECT`, keeping the query set-based and avoiding an artificial join. This is precisely the same "what, not how" philosophy from Module 01 — you say *what* window of rows a value should be computed from, and the query planner handles doing so efficiently.

## Advantages

- **No self-joins required.** What used to require joining a table to its own aggregated summary is now a single, flat `SELECT` — simpler to write, and typically faster, since the database can compute it in one pass instead of a join.
- **Detail and summary in one result set.** You get individual rows and group-level context simultaneously, which is exactly what most real reports (dashboards, statements, leaderboards) need.
- **Composable.** Multiple window functions with different `PARTITION BY`/`ORDER BY` clauses can appear side by side in the same query, each computing a different perspective on the same rows.

## Disadvantages / Limitations

- **Cannot be used in `WHERE` or `HAVING` directly.** Because window functions are evaluated after those clauses (see Internal Working above), filtering on a window function's result requires wrapping the query in a subquery or CTE — an extra layer of nesting compared to filtering on a plain column (the next topic shows this pattern in full).
- **Can be easy to misread at first.** Seeing `ORDER BY` appear both inside `OVER(...)` and at the end of the same query, meaning two different things, trips up almost everyone the first few times.
- **Cost is real, even though row count doesn't change.** A window function still requires PostgreSQL to sort/partition data internally; on very large tables with many distinct partitions, this is genuine computational work, not a free column to add casually (Module 20 — Performance Tuning covers reading a query's execution plan in depth).

## Best Practices

- Always ask "do I want to collapse rows, or keep every row while adding group-level context?" before choosing between `GROUP BY` and a window function — they solve related but distinct problems, and reaching for the wrong one either destroys detail you needed or overcomplicates a query that just needed a simple aggregate.
- When a window function's `OVER(...)` includes `ORDER BY`, be deliberate about it — know whether you want a full-partition total (no `ORDER BY`) or a running/ordered calculation (`ORDER BY` present), since, as shown above, adding it silently changes the result, not just its formatting.
- Give partition columns real thought — `PARTITION BY` with the wrong column (or none at all, when you meant to have one) is the single most common bug with window functions: it silently changes which rows are "related" to which.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Trying to use a window function's result inside `WHERE`, e.g. `WHERE SUM(amount) OVER (...) > 50000` | Window functions are evaluated after `WHERE` runs, so its result doesn't exist yet at that point — this must be filtered in an outer query/CTE wrapping the window function (covered in Topic 2). |
| Forgetting `PARTITION BY` entirely when a per-group value was intended | Omitting `PARTITION BY` treats the *entire result set* as a single window — you'll get one grand total repeated on every row, not a per-group total. |
| Assuming `OVER (PARTITION BY x)` behaves like `GROUP BY x` | `PARTITION BY` never removes rows from the result — the row count in equals the row count out. `GROUP BY` collapses rows into one per group. They compute similar aggregates but produce structurally different results. |
| Thinking `ORDER BY` inside `OVER(...)` only affects display order | It changes the window's default frame (Topic 5) and therefore the *computed value* itself, not merely the order rows are printed in. |

## Interview Questions

1. **Q: What is the fundamental difference between a window function and an aggregate function used with `GROUP BY`?**
   A: An aggregate function with `GROUP BY` collapses multiple rows into one row per group, discarding individual row detail. A window function, using `OVER(...)`, computes a value across a set of related rows (a "window") but returns that value on every original row — no rows are removed. Both can use the same underlying functions (`SUM`, `AVG`, `COUNT`), but the shape of the result is fundamentally different.

2. **Q: What does `PARTITION BY` do inside an `OVER(...)` clause?**
   A: It splits the rows into groups, analogous to `GROUP BY`, but purely for the purpose of scoping the window function's calculation — every row still appears in the final result, only "seeing" other rows that share its partition value when the function computes its result.

3. **Q: Why can't you filter on a window function's result using `WHERE`?**
   A: Because of SQL's logical processing order, `WHERE` is evaluated before window functions are computed — at the time `WHERE` runs, the window function's output doesn't exist yet. To filter on it, you must compute the window function in a subquery or CTE, then filter in an outer query against that already-computed column.

4. **Q: What's the effect of adding `ORDER BY` inside an `OVER(...)` clause that didn't have one before?**
   A: It changes the window's default frame from the entire partition to a running slice (from the start of the partition through the current row, by default), which changes an aggregate like `SUM` from a flat per-group total into a running/cumulative total — a real change in computed values, not just a display change.

## Summary

- A **window function** computes a value across a set of related rows (its *window*) without collapsing them — every input row survives in the output, unlike `GROUP BY`.
- `OVER(...)` is what turns an ordinary aggregate function into a window function call.
- `PARTITION BY` scopes the window to groups within the result set, analogous to `GROUP BY`, but without removing any rows.
- `ORDER BY` inside `OVER(...)` is a different thing from the query's trailing `ORDER BY` — it changes the window's default frame and therefore the computed value itself, not merely the display order.
- Window functions run logically after `WHERE`/`GROUP BY`/`HAVING` but before the final `SELECT`/`ORDER BY` — which is exactly why they can't be referenced directly inside `WHERE` or `HAVING`.
- The next topic builds directly on this foundation with `ROW_NUMBER`, `RANK`, and `DENSE_RANK` — three of the most common window functions used in real reporting queries.
