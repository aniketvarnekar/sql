# Module 16 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The OVER Clause and PARTITION BY** — what a window function is, how it differs from `GROUP BY` by keeping every row, and how `PARTITION BY` and `ORDER BY` scope and shape a window's calculation
- [x] **ROW_NUMBER, RANK, and DENSE_RANK** — the three core ranking functions, their differing tie behavior, and the top-N-per-group query pattern
- [x] **NTILE and Distribution Functions** — bucketing rows into N roughly equal groups with `NTILE`, why it can split tied values, and relative standing via `PERCENT_RANK`/`CUME_DIST`
- [x] **LAG and LEAD** — reading a previous or next row's value directly, with offset and default-value arguments, replacing self-joins for row-to-neighbor comparisons
- [x] **Running Totals and Moving Averages** — explicit frame clauses (`ROWS BETWEEN ... AND ...`), and PostgreSQL's default frame behavior with and without `ORDER BY`

## Practical Connections

- **A sales dashboard showing each transaction alongside its running total for the day** relies directly on `SUM(...) OVER (... ORDER BY ...)` with a running-total frame — exactly the pattern built up across Topics 1 and 5 — rather than a separate aggregate query joined back to the detail rows.
- **A leaderboard or standings page ("rank 1 through N, with ties handled correctly")** is a direct, real-world instance of the `RANK`/`DENSE_RANK` distinction from Topic 2 — competitive rankings and "tier" displays genuinely need different tie-breaking behavior, and picking the wrong one produces a leaderboard that looks subtly broken to its users.
- **A financial report flagging "accounts whose balance changed by more than 10% since last statement"** is a direct application of `LAG` from Topic 4 — computed for millions of account rows in a single pass, with no self-join required.
- **Customer segmentation into spend tiers (e.g., "top 20% of customers by lifetime spend")** commonly uses `NTILE` or `PERCENT_RANK` from Topic 3, applied across an entire customer table, to drive targeted reporting or pricing logic without a separate, manually maintained cutoff table.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Window function vs. `GROUP BY` aggregation | A window function keeps every input row and attaches a computed value to each; `GROUP BY` collapses rows into one per group, discarding row-level detail. |
| `ORDER BY` inside `OVER(...)` vs. the query's trailing `ORDER BY` | The one inside `OVER(...)` defines the sequence used to compute the window function's value (and can change its default frame); the trailing one only controls the final display order of the result set. |
| `RANK()` vs. `DENSE_RANK()` | Both give tied rows equal ranks; `RANK()` then skips ahead by the size of the tie for the next distinct value, while `DENSE_RANK()` never skips — its maximum equals the count of distinct values, not the count of rows. |
| `NTILE` vs. `PERCENT_RANK`/`CUME_DIST` | `NTILE` buckets by row position and can split tied values across adjacent buckets; `PERCENT_RANK`/`CUME_DIST` are peer-group aware and always give tied values identical results. |
| `LAG`/`LEAD` vs. a self-join | Both can compare a row to a neighboring row, but `LAG`/`LEAD` do it as a direct, declarative positional lookup within a window, with no extra join or join condition to construct or get wrong. |
| Default frame with vs. without `ORDER BY` in `OVER(...)` | Without `ORDER BY`, the default frame is the entire partition (a flat, repeated group total); with `ORDER BY` present but no explicit frame, the default becomes a running slice — the same aggregate function can produce very different results depending purely on whether `ORDER BY` was added. |
| `ROWS` vs. `RANGE` frame mode | `ROWS` counts physical row positions; `RANGE` groups tied `ORDER BY` values into a shared peer group with an identical result. They behave identically only when the `ORDER BY` column has no ties. |

## What's Next

This module gave you the ability to compute rankings, bucket distributions, compare neighboring rows, and accumulate running totals — all while preserving full row-level detail, something `GROUP BY` alone could never do. Several topics in this module (top-N-per-group filtering, in particular) already leaned on a `WITH ... AS (...)` common table expression to work around the rule that window functions can't be filtered directly in `WHERE`. **Module 17 — CTEs and Recursion** picks up exactly there: it covers the `WITH` clause in full depth, including how to build multi-step queries that are easier to read than deeply nested subqueries, and how to write recursive CTEs to query hierarchical and graph-shaped data — a capability no window function alone can provide.
