# Module 11 — Subqueries

## Module Goal

By the end of this module, you will be able to write queries that contain other queries nested inside them — as a single value, as a filter list, as a row-by-row existence check, or as an entire virtual table — and know precisely when each form is the right tool. You will also be able to look at a subquery-based query and its join-based equivalent side by side and reason about which one is clearer and safer, with a forward look at which is faster (Module 20). Subqueries are the bridge between the single-table querying you've done so far and the multi-step, multi-table reasoning that real reporting and analytics queries require every day.

## Topics Covered in This Module

1. **[Scalar Subqueries](01-scalar-subqueries.md)** — a subquery that collapses to exactly one row and one column, usable anywhere a single value is expected, and what happens when it accidentally returns more than one row.
2. **[Subqueries in WHERE and FROM](02-subqueries-in-where-and-from.md)** — using a subquery as a filter list with `IN`, and using a subquery as a derived table in `FROM`, including PostgreSQL's requirement that every derived table be aliased.
3. **[Correlated Subqueries](03-correlated-subqueries.md)** — subqueries that reference a column from the surrounding query and are conceptually re-evaluated once per outer row, contrasted with the non-correlated subqueries from Topics 1–2.
4. **[EXISTS and NOT EXISTS](04-exists-and-not-exists.md)** — checking for the mere presence of matching rows without caring about their values, why `EXISTS` typically outperforms `IN`, and the dangerous `NOT IN` + `NULL` trap.
5. **[ANY, ALL, and Derived Tables Recap](05-any-all-and-derived-tables.md)** — comparing a value against an entire set of results with `ANY`/`ALL`, plus a consolidated decision guide for subquery vs. join vs. CTE.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 7 (Querying Basics)** — this entire module assumes fluency with `SELECT`, `WHERE`, and comparison operators (`=`, `>`, `<`, `IN`, `BETWEEN`). Every subquery you'll write here ultimately plugs into a `WHERE` clause, a `SELECT` list, or a `FROM` clause you already know how to write on its own.
- **Module 9 (Aggregation)** — most scalar subqueries in this module wrap an aggregate function (`AVG`, `MAX`, `COUNT`, `SUM`) to collapse many rows down to the single value a scalar subquery requires. If `GROUP BY` and aggregate functions aren't solid yet, several examples here will feel like a leap.
- **Module 10 (Joins & Set Operations)** — a recurring theme of this module is that many subqueries can be rewritten as joins, and vice versa. You need working knowledge of `INNER JOIN` and `LEFT JOIN` to follow those comparisons, and to understand why a correlated subquery is sometimes just a join wearing a different syntax. This module also directly reuses and extends the `customers`/`orders` running schema Module 10 introduced.

## How to Study This Module

Read Topics 1 through 4 in strict order — they build one mental model in layers: a subquery producing exactly one value (Topic 1), a subquery producing a list or an entire virtual table (Topic 2), a subquery that changes its answer per outer row (Topic 3), and a subquery used purely to test existence (Topic 4). Topic 4's `NOT IN`/`NULL` trap is one of the most consequential gotchas in this entire course — real production bugs from this exact mistake are extremely common, so do not skim it. Topic 5 introduces `ANY`/`ALL` but is equally important as a deliberate look back across the whole module, consolidating when to reach for a subquery versus a join versus a CTE (the latter given full treatment in Module 17) — treat it as required reading, not an optional recap.
