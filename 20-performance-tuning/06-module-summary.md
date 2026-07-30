# Module 20 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **How the Query Planner Works** — the full parser/rewriter/planner/executor pipeline, cost-based optimization and its tunable constants, table statistics and `ANALYZE`, why stale statistics cause bad plans, and how join order and join algorithm (nested loop, hash join, merge join) are chosen.
- [x] **EXPLAIN and EXPLAIN ANALYZE in Depth** — reading a multi-join, multi-aggregate plan tree, `EXPLAIN (ANALYZE, BUFFERS)`, spotting estimated-vs-actual row mismatches as a sign of stale statistics, and a concrete checklist of actionable plan red flags.
- [x] **Index Usage and Selectivity** — why the planner sometimes rejects an existing index (low selectivity, small tables), selectivity as a measurable concept, expression indexes, and partial indexes.
- [x] **Query Rewriting Patterns** — rewriting correlated scalar subqueries as joins, avoiding `SELECT *` in production code, sargability and avoiding functions on indexed columns, and `OFFSET` pagination's cost versus keyset pagination.
- [x] **Common Anti-Patterns** — the N+1 query problem at the data-access level, implicit type casting defeating index usage, over-indexing's write-side cost, and unbounded result sets in production paths.
- [x] **Module Summary** — this consolidated recap.

## Practical Connections

- A dashboard querying millions of rows in real time relies on every idea in this module simultaneously: fresh statistics so the planner's row estimates are accurate, selective indexes chosen correctly over sequential scans, sargable filter conditions so those indexes are actually usable, and keyset pagination so scrolling deep into results doesn't degrade with depth.
- A batch data-loading job that inserts millions of rows overnight should always be followed by an explicit `ANALYZE`, precisely because the next morning's reporting queries will otherwise be planned against a stale, pre-load picture of the data — a direct, everyday application of Topic 1.
- A high-write system (an order-processing pipeline, an event-logging table) needs its index list periodically audited against actual usage, because every index silently taxes every write — the over-indexing anti-pattern from Topic 5 is easy to accumulate gradually and expensive to notice only after write throughput has already degraded.
- Any API or report that fetches "related data for a list of things" — a common shape in virtually every real application — is exactly where the N+1 anti-pattern hides, and exactly where a single batched join query (Topic 4, Topic 5) replaces a linearly growing number of round-trips with a constant number.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Estimated cost vs. actual execution time | Estimated cost is a unitless, pre-execution number derived from statistics and tunable cost constants; actual execution time is a real, measured duration only visible under `EXPLAIN ANALYZE`. |
| "No index used" vs. "index broken" | The planner correctly avoids an index for poorly selective conditions or tiny tables — a rejected index is usually a sound cost-based decision, not a malfunction. |
| Sargable vs. non-sargable conditions | A sargable condition compares a raw, indexed column directly against a value, usable by an index; a non-sargable condition wraps the column in a function or cast, forcing per-row evaluation before any index lookup can happen. |
| `OFFSET` pagination vs. keyset pagination | `OFFSET` generates and discards every row up to the offset on every request, growing costlier with page depth; keyset pagination seeks directly to a remembered key, keeping cost roughly constant regardless of depth. |
| Correlated subquery in `EXISTS`/`IN` vs. in a `SELECT` list | The former is frequently optimized automatically by the planner into an efficient semi-join; the latter is genuinely re-evaluated once per outer row and is the form most worth rewriting as a join. |
| Too few indexes vs. too many indexes | Too few indexes leaves selective, frequently-filtered queries stuck doing sequential scans; too many indexes taxes every write with maintenance overhead for indexes that may serve little or no read benefit — both are real, opposite failure modes. |

## What's Next

This module gave you the diagnostic and remedial toolkit for making SQL fast at real scale: understanding what the planner is actually deciding, seeing those decisions directly with `EXPLAIN`, reasoning about index usage through selectivity, and recognizing both query-level and pattern-level habits that quietly erode performance. **Module 21 — Advanced SQL** builds on this foundation by introducing features — pivoting data, JSON columns, sequences, temporary tables, and partitioning — that extend what you can express and store in a relational database beyond the core relational model covered so far, some of which (partitioning in particular) are themselves direct performance and scalability tools for very large tables.
