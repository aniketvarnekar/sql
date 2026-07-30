# Module 17 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **The WITH Clause** — basic `WITH name AS (...)` syntax, chaining multiple CTEs where later ones reference earlier ones, and the readability case for naming intermediate steps instead of nesting subqueries
- [x] **CTEs vs. Subqueries** — when reuse of the same logic makes a CTE clearer than a subquery, PostgreSQL's materialization behavior before and after version 12, forcing behavior with `MATERIALIZED`/`NOT MATERIALIZED`, and when a plain derived table is still the simpler choice
- [x] **Recursive CTEs** — `WITH RECURSIVE` syntax, the anchor and recursive members, `UNION ALL` vs. `UNION` inside a recursive CTE, a fully worked org-chart hierarchy traversed both downward and upward, a fully worked cyclic route-network graph traversal, and explicit guards against infinite recursion

## Practical Connections

- **Multi-step reporting queries** — any dashboard or report that needs to compute an intermediate aggregate and then filter or join against it (department averages, running totals, cohort baselines) is exactly the pattern the `WITH` clause exists to make readable, whether or not the underlying data happens to be small or enormous.
- **Permission and access hierarchies** — a system that grants access transitively (a manager can approve anything their direct and indirect reports can approve; a folder inherits permissions from its parent folder, however many levels up) is precisely the kind of arbitrary-depth traversal that only `WITH RECURSIVE` (or repeated application-side queries, which are strictly worse) can express in one statement.
- **Bill-of-materials and category-tree explosions** — a manufacturing system computing "every component, and sub-component, that goes into this finished product" or a retail catalog computing "every product under this category, including subcategories of subcategories" both rely on the exact hierarchy-traversal pattern shown in this module's org-chart example, just with parts or categories instead of employees.
- **Network and reachability questions at scale** — "which servers can this one reach through the network," "which accounts are connected to this one through a chain of transactions," and similar reachability or shortest-path-style questions over graph-shaped data all reduce to the graph-traversal pattern demonstrated in this module's route-network example, guarded against cycles exactly the way real-world networks require.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| CTE vs. subquery | A CTE is a named result set defined once via `WITH`, usable anywhere later in the same statement (including more than once); a subquery is written inline at its point of use and, unless duplicated, can only be used in that one place. |
| CTE vs. view | A CTE exists only for the duration of the single statement that defines it; a view (Module 12) is a persistent, named database object that can be queried by any later, separate statement. |
| "Materialized" vs. "named" | Naming a computation (via `WITH`) is purely a readability choice; whether it's actually computed once and reused (materialized) or folded into the surrounding query (inlined) is a separate planner decision, independently controllable with `MATERIALIZED`/`NOT MATERIALIZED`. |
| `UNION` vs. `UNION ALL` inside a recursive CTE | `UNION ALL` keeps every row with no deduplication and is the typical, efficient default; `UNION` deduplicates after every iteration, which only protects against cycles that produce exact duplicate rows — it does not help once the recursive member accumulates per-row state like a path. |
| Anchor member vs. recursive member | The anchor member is a plain, non-recursive query that runs exactly once to seed the starting rows; the recursive member is the part that references the CTE itself and runs repeatedly, extending outward from the previous iteration's rows, until an iteration produces nothing new. |
| `WITH RECURSIVE`'s recursion vs. function-call recursion | `WITH RECURSIVE` is an iterative, set-based fixed-point process (each iteration processes a whole set of rows at once) — it is not the call-stack style of recursion used by recursive functions in a general-purpose language, which is instead the domain of stored procedures and functions (Module 18). |

## What's Next

This module gave you the tools to break a complex query into clearly named, readable steps, to reason precisely about whether that naming also changes performance, and — most importantly — to express hierarchical and graph-shaped questions that a plain `JOIN` simply cannot answer, regardless of how many times you repeat it. **Module 18 — Procedures, Functions & Triggers** builds on this by moving beyond single, self-contained `SELECT` statements entirely: you'll learn to package logic (including genuine, call-stack-style recursive logic) into stored procedures and user-defined functions that live in the database and can be invoked repeatedly, and to attach triggers that run automatically in response to data changes.
