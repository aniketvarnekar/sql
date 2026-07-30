# Module 09 — Aggregation

## Module Goal

By the end of this module, you will be able to turn a table full of individual rows into meaningful summary numbers — totals, counts, averages, and breakdowns by category — using SQL's aggregate functions and the `GROUP BY` clause. You will understand precisely how `NULL` values interact with each aggregate function, why filtering an aggregate result requires a different clause (`HAVING`) than filtering raw rows (`WHERE`), and how to generate multi-level subtotal reports with `ROLLUP`, `CUBE`, and `GROUPING SETS` in a single query instead of stitching several queries together by hand. Aggregation is the mechanism behind nearly every dashboard, report, and "how many / how much" business question ever asked of a database — this module is where SQL stops just *retrieving* rows and starts *summarizing* them.

## Topics Covered in This Module

1. **[Aggregate Functions](01-aggregate-functions.md)** — `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT ...)`, and exactly how each one treats `NULL`.
2. **[GROUP BY](02-group-by.md)** — collapsing rows into per-value groups, combining `GROUP BY` with aggregate functions, grouping by multiple columns, and the rule governing which columns may appear in `SELECT`.
3. **[HAVING vs. WHERE](03-having-vs-where.md)** — why an aggregate result can't be filtered with `WHERE`, the logical order SQL actually processes a query in, and how to use both clauses together correctly.
4. **[ROLLUP and CUBE](04-rollup-and-cube.md)** — generating subtotals and grand totals declaratively with `GROUP BY ROLLUP`, `GROUP BY CUBE`, and the general-purpose `GROUPING SETS`.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 7 — Querying Basics**, especially `SELECT`, `WHERE`, and `ORDER BY`. Aggregation always builds on top of a query that already knows how to select and filter rows; every example in this module starts from a plain `SELECT ... FROM ... WHERE ...` you should already be comfortable writing before adding aggregation on top of it.
- **Module 8 — Functions and Expressions**, especially numeric expressions (e.g., `quantity * unit_price`) and `CASE`. Aggregate functions are frequently applied not to a raw column but to an *expression* built from one or more columns (`SUM(quantity * unit_price)` rather than `SUM(some_column)`), and `CASE` is a common building block inside an aggregate argument (covered again in this module in that exact context).
- **Module 1 — Introduction**, specifically the [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md) topic: everything in this module is `SELECT` (DQL) — no new command category is introduced, only new capability inside the query language you already know.

No new table-design or data-modification concepts are required — this module is entirely about *reading and summarizing* data that's already sitting in a table.

## How to Study This Module

Topic 1 (Aggregate Functions) and Topic 2 (`GROUP BY`) are the foundation of everything else in this module and, honestly, of a huge fraction of real-world SQL — read them slowly and actually run every example. Topic 3 (`HAVING` vs. `WHERE`) is short but is one of the most frequently misunderstood distinctions in SQL, and a favorite interview question — don't skim the logical processing order table, since it explains *why* the rule exists rather than asking you to memorize it as an arbitrary fact. Topic 4 (`ROLLUP`/`CUBE`/`GROUPING SETS`) is the most advanced topic in this module; it builds directly on Topics 1–3, so make sure `GROUP BY` and the concept of a "group" feel completely natural before starting it. All four topics share one running example — an `orders` table — so that the results of grouping and aggregating are easy to cross-check against each other as the module progresses.
