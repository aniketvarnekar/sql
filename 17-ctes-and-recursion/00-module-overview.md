# Module 17 — CTEs & Recursion

## Module Goal

By the end of this module, you will be able to break a complex query into clearly named, sequential steps using the `WITH` clause, understand how PostgreSQL actually executes a Common Table Expression (and when it optimizes one away entirely), and write `WITH RECURSIVE` queries that walk hierarchies (like an org chart) and graphs (like a route network) of arbitrary, unknown depth — something no plain `JOIN` can do. This unlocks an entire category of real-world reporting problems (management chains, category trees, bill-of-materials explosions, reachability/shortest-path questions) that are otherwise impossible to express in a single SQL statement.

## Topics Covered in This Module

1. **[The WITH Clause](01-the-with-clause.md)** — Defining a named, temporary result set at the top of a query; basic syntax; chaining multiple CTEs where later ones reference earlier ones; readability benefits of naming intermediate steps.
2. **[CTEs vs. Subqueries](02-ctes-vs-subqueries.md)** — When a CTE communicates intent more clearly than a nested subquery (especially when the same logic is reused multiple times); PostgreSQL's materialization behavior before and after version 12, and how to force it with `MATERIALIZED`/`NOT MATERIALIZED`; when a plain derived table is still the simpler choice.
3. **[Recursive CTEs](03-recursive-ctes.md)** — `WITH RECURSIVE` syntax; the anchor member and recursive member; `UNION ALL` vs. `UNION` inside a recursive CTE; a fully worked hierarchical (org chart) example traversed both downward and upward; a fully worked graph (route network) traversal example; guarding against infinite recursion.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 11 (Subqueries)** — a CTE is, at its core, a named subquery; this module constantly compares CTEs against scalar subqueries, correlated subqueries, and derived tables, so you need to already be fluent with what a subquery is and how a derived table works before contrasting them here.
- **Module 10 (Joins & Set Operations)** — every non-trivial example in this module, and every recursive CTE without exception, joins the CTE's result against a base table (or against itself, in the recursive case). You need to already be comfortable reading and writing `JOIN ... ON` conditions, and you'll recognize the same explicit `JOIN...ON` style taught in [Inner Join](../10-joins-and-set-operations/01-inner-join.md).
- **Module 09 (Aggregation)** — several worked examples in this module compute a `GROUP BY` aggregate first and then join back to it; you should already be comfortable with `GROUP BY` and aggregate functions like `AVG` and `COUNT`.
- **Module 05 (Constraints & Keys)**, specifically the concept of a foreign key referencing the same table it's declared on (a self-referencing foreign key), is used to model the manager/employee hierarchy in this module's recursive examples — see [Foreign Keys and Referential Integrity](../05-constraints-and-keys/04-foreign-keys-and-referential-integrity.md).

## How to Study This Module

Read Topic 1 hands-on and type out both examples yourself — the `WITH` syntax it teaches is the foundation every later topic (and much of the rest of this course, whenever a query gets non-trivial) builds on directly. Topic 2 is more conceptual and is best absorbed slowly; it explains something that trips up even experienced SQL writers (that a CTE is not automatically a performance optimization), so don't skim the materialization discussion even if the syntax feels simple. Topic 3 is the payoff of the module and the most heavily tested topic on SQL interviews — it is dense, so trace through each worked example's iterations by hand (literally write out what each iteration's rows look like on paper) before moving on, rather than just reading the final query and result. Revisit Topic 3's "guarding against infinite recursion" section every time you write a new recursive query in the future, even after you're experienced with the syntax — it's the single most common way a recursive CTE goes wrong in practice.
