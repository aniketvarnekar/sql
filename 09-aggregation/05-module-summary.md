# Module 09 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Aggregate Functions** — `COUNT(*)` vs. `COUNT(column)`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT ...)`, and how every aggregate function except `COUNT(*)` silently skips `NULL` values.
- [x] **GROUP BY** — collapsing rows sharing a column's value into groups, combining `GROUP BY` with aggregate functions, grouping by multiple columns, and the rule that non-aggregated `SELECT` columns must appear in `GROUP BY`.
- [x] **HAVING vs. WHERE** — why aggregate functions cannot appear in `WHERE`, SQL's logical processing order (`FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`), and worked examples using each clause for the job only it can do.
- [x] **ROLLUP and CUBE** — generating hierarchical subtotals with `ROLLUP`, full combinatorial subtotals with `CUBE`, `GROUPING SETS` as the general form behind both, and `GROUPING()` for telling a subtotal row's `NULL` apart from a real one.

## Practical Connections

- **Every dashboard showing "total revenue," "orders today," or "average response time"** is a `COUNT`/`SUM`/`AVG` computed over potentially millions of rows in a single database round trip — exactly the Topic 1 pattern, just at production scale.
- **Any report broken down "by region," "by customer tier," or "by product category"** is a direct application of Topic 2's `GROUP BY`, and the moment that report also needs a "top regions only" or "customers with more than N orders" filter, it's reaching for Topic 3's `HAVING`.
- **Financial statements, sales summaries, and executive dashboards that show line items with subtotals and a grand total on one page** — the exact shape a finance team has produced by hand or in a spreadsheet for decades — are what Topic 4's `ROLLUP`/`CUBE`/`GROUPING SETS` generate declaratively, in one query, instead of via several manually stitched-together reports.
- **A reporting query filtering to "this quarter" before grouping** (`WHERE order_date BETWEEN ...` combined with `GROUP BY region`) is the everyday, large-scale version of Topic 3's combined `WHERE`-then-`HAVING` example — narrowing millions of rows down to a relevant window before the (comparatively expensive) grouping work ever begins.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| `COUNT(*)` vs. `COUNT(column)` | `COUNT(*)` counts every row regardless of content; `COUNT(column)` counts only rows where that column is not `NULL`. They agree only when the column has no `NULL` values at all. |
| `AVG` ignoring `NULL` vs. treating it as zero | Every aggregate function except `COUNT(*)` excludes `NULL` rows from its computation entirely — it does not fold them in as zero. An `AVG` over a column with `NULL`s divides by the count of *non-`NULL`* rows, not the total row count. |
| `GROUP BY` vs. `SELECT DISTINCT` | With no aggregate function present, `GROUP BY` on a set of columns happens to produce the same rows as `DISTINCT` on those columns — but `GROUP BY` is designed for aggregation; reach for `DISTINCT` when no aggregate is actually needed. |
| `WHERE` vs. `HAVING` | `WHERE` filters individual rows before grouping/aggregation occurs and cannot reference an aggregate function. `HAVING` filters entire groups after their aggregate values have already been computed, and exists specifically to filter on those aggregate values. |
| A `ROLLUP`/`CUBE` subtotal row's `NULL` vs. a genuine stored `NULL` | The blank cell in a subtotal or grand-total row means "aggregated across every value of this column," not "value unknown." Use `GROUPING(column)` to distinguish the two — never assume from a blank cell alone. |
| `ROLLUP` vs. `CUBE` | `ROLLUP` produces a fixed hierarchy of subtotals (full detail, then progressively fewer grouping columns, left to right). `CUBE` produces subtotals for every possible combination of the listed columns — `CUBE`'s output is a superset of the equivalent `ROLLUP`'s. |

## What's Next

This module took you from a table full of individual rows to genuinely summarized, report-ready results — aggregate functions, per-group breakdowns with `GROUP BY`, correctly filtering those breakdowns with `HAVING`, and generating multi-level subtotal reports with `ROLLUP`, `CUBE`, and `GROUPING SETS`. Every example in this module deliberately stayed within a single table, computing summaries over rows that were already sitting together in `orders`. **Module 10 — Joins and Set Operations** builds directly on top of everything here: real questions rarely live inside one table alone (a customer's name might live in a separate `customers` table, a product's category in a separate `products` table), and Module 10 teaches you how to combine rows from multiple related tables *before* the exact aggregation and grouping techniques from this module get applied to the combined result.
