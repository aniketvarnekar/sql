# Module 16 — Window Functions

## Module Goal

By the end of this module, you will be able to compute per-row rankings, running totals, moving averages, and row-to-row comparisons — all *without* collapsing your result set the way `GROUP BY` does. Window functions are one of the most powerful and most interview-tested features in modern SQL: they let you keep every row of detail while still asking group-level questions of it ("what rank is this sale within its salesperson's month?", "what's the running total through this row?", "how does this row compare to the previous one?"). This module unblocks nearly every realistic reporting query you'll be asked to write from here on, and it is a near-certain topic in any serious SQL technical interview.

## Topics Covered in This Module

1. **[The OVER Clause and PARTITION BY](01-the-over-clause-and-partition-by.md)** — what a window function actually is, how `OVER()` computes across related rows without collapsing them, and how `PARTITION BY` scopes that computation to groups.
2. **[ROW_NUMBER, RANK, and DENSE_RANK](02-row-number-rank-and-dense-rank.md)** — the three core ranking functions, how they differ specifically when values tie, and the classic "top-N per group" pattern.
3. **[NTILE and Distribution Functions](03-ntile-and-distribution-functions.md)** — bucketing rows into N roughly equal groups (e.g., quartiles) with `NTILE`, and measuring relative standing with `PERCENT_RANK` and `CUME_DIST`.
4. **[LAG and LEAD](04-lag-and-lead.md)** — reading a previous or next row's value directly, without a self-join, and comparing each row to its neighbor in an ordered sequence.
5. **[Running Totals and Moving Averages](05-running-totals-and-moving-averages.md)** — frame clauses (`ROWS BETWEEN ...`), cumulative sums, moving averages, and exactly what frame PostgreSQL uses by default when you do (and don't) specify one.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 9 — Aggregation**, especially `GROUP BY` and aggregate functions like `SUM`, `AVG`, `COUNT`. This entire module is defined in contrast to `GROUP BY`: a window function uses the exact same aggregate functions you already know, but applies them without collapsing rows. If you're not solid on what `GROUP BY` collapses away, the motivation for window functions won't land.
- **Module 7 — Querying Basics**, especially `ORDER BY` and the logical order in which a `SELECT` is processed. Window functions reuse the `ORDER BY` keyword inside a new clause (`OVER (... ORDER BY ...)`) with a different meaning, and understanding *when* window functions are evaluated relative to `WHERE`, `GROUP BY`, and the final `ORDER BY` is essential to avoid a very common class of mistake covered in this module.
- General comfort with `SELECT`, filtering, and basic expressions from Modules 1–8 is assumed throughout, as with every module from this point forward.

## How to Study This Module

Topic 1 is the conceptual foundation for the entire module — do not skim it, even if you're impatient to get to ranking or running totals. Every later topic is just "a specific function, placed inside the machinery Topic 1 teaches." Topics 2 and 3 (ranking and bucketing) are closely related and worth reading back to back, since they're often confused with each other. Topic 4 (`LAG`/`LEAD`) is comparatively simple once Topic 1 has landed. Topic 5 (running totals and moving averages) is the most technically dense topic in the module — it introduces frame clauses precisely, and it directly explains a subtlety hinted at in Topic 1 (why adding `ORDER BY` inside `OVER()` can silently change a total into a running total), so read it carefully even on a second pass. All six topics share one running example — a `monthly_sales` table of salespeople, regions, months, and sale amounts — so you can watch the *same* underlying data produce different, complementary results as each function is introduced.
