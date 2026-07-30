# Module 20 — Performance Tuning

## Module Goal

By the end of this module, you will be able to look at a slow query and actually diagnose *why* it's slow, rather than guessing. You will understand how the query planner turns your declarative SQL into a concrete execution strategy, how to read that strategy with `EXPLAIN` and `EXPLAIN ANALYZE`, why an index you created is sometimes ignored, how to rewrite queries into forms the planner can execute far more efficiently, and which common data-access patterns quietly destroy performance at scale even though they "work" on a small development database. Every prior module taught you how to express a correct question in SQL. This module teaches you how to tell whether the database is answering that question efficiently, and how to intervene when it isn't.

## Topics Covered in This Module

1. **[How the Query Planner Works](01-how-the-query-planner-works.md)** — the parser/planner/optimizer/executor pipeline in depth, cost-based optimization, table statistics and `ANALYZE`, and how the planner chooses join order and join algorithm.
2. **[EXPLAIN and EXPLAIN ANALYZE in Depth](02-explain-and-explain-analyze.md)** — reading a multi-join, multi-aggregate execution plan, `EXPLAIN (ANALYZE, BUFFERS)`, spotting estimated-vs-actual row mismatches, and the specific red flags worth hunting for in a plan.
3. **[Index Usage and Selectivity](03-index-usage-and-selectivity.md)** — why the planner sometimes rejects a perfectly good index in favor of a sequential scan, selectivity as a measurable concept, expression indexes, and partial indexes.
4. **[Query Rewriting Patterns](04-query-rewriting-patterns.md)** — rewriting correlated subqueries as joins, avoiding `SELECT *` in production code, sargability (why wrapping an indexed column in a function defeats the index), and `OFFSET` pagination's hidden cost versus keyset pagination.
5. **[Common Anti-Patterns](05-common-anti-patterns.md)** — the N+1 query problem explained at the pure data-access level, implicit type casting silently disabling index usage, over-indexing and its write-side cost, and unbounded result sets in production code paths.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 13 — Indexes** ([overview](../13-indexes/00-module-overview.md)), in full. This module assumes you already know what a B-tree index is, what a composite index's leftmost-prefix rule is, what an index-only scan is, and how to read a basic `EXPLAIN` plan for a single-table query. Everything here goes deeper: multi-table plans, the cost model behind the planner's choices, and the specific conditions under which an index gets rejected.
- **Module 11 — Subqueries**, specifically correlated subqueries. This module revisits that exact pattern and shows how (and why) to rewrite it as a join, so you should already be comfortable with what makes a subquery "correlated" versus independent.
- **Module 10 — Joins & Set Operations** ([overview](../10-joins-and-set-operations/00-module-overview.md)), specifically `INNER JOIN` and `LEFT JOIN`. The join algorithms discussed here (nested loop, hash join, merge join) are all different ways of physically executing the same logical join syntax you already know.
- **Module 1 — Introduction**, specifically the parser/planner/executor pipeline previewed in [What Is SQL?](../01-introduction/02-what-is-sql.md)'s "Internal Working" section. This module is where that brief preview becomes the real, detailed picture.
- **Module 7 — Querying Basics** and **Module 9 — Aggregation**, for general fluency with `WHERE`, `ORDER BY`, `LIMIT`, `GROUP BY`, and aggregate functions — the plans examined in this module filter, sort, limit, and aggregate constantly, and this module does not re-explain that syntax.

## How to Study This Module

Read Topics 1 and 2 first, and read them slowly — they are the conceptual and practical foundation for the rest of the module. Topic 1 gives you the mental model of what the planner is actually doing and why; Topic 2 gives you the tool (`EXPLAIN`/`EXPLAIN ANALYZE`) to see that model in action against your own queries. Do not skip running the examples yourself: performance tuning is a skill that is genuinely hard to absorb by reading alone, because the entire point is learning to notice small, specific discrepancies in real output. Topic 3 builds directly on Module 13 and explains the single most common point of confusion once people know indexes exist — "why isn't my index being used?" Topics 4 and 5 are the most immediately actionable in the module: Topic 4 is a toolbox of concrete rewrites you can apply to real queries today, and Topic 5 is a checklist of patterns to actively hunt for and eliminate in any codebase you work on. By the end of this module, `EXPLAIN` should be the first thing you reach for whenever a query feels slow — not a last resort after everything else has failed.
