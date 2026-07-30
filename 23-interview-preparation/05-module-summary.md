# Module 23 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Core Conceptual Questions — Ranked & Synthesized** — the course's highest-signal conceptual questions, reorganized by cross-cutting theme (relational fundamentals, constraints/keys, querying/joins/aggregation, subqueries/CTEs, indexes/performance, transactions/concurrency, normalization/design, security) instead of module order, each with an interview-length model answer and a pointer back to its source module.
- [x] **Query-Writing Problems** — eleven query-writing prompts of increasing difficulty, each with a small schema, a plain-English problem statement, and a complete worked PostgreSQL solution naming the specific technique (window functions, anti-joins, correlated subqueries, recursive CTEs, conditional aggregation, gaps-and-islands) used to solve it.
- [x] **Schema Design Questions** — five realistic "design a schema for X" prompts (e-commerce, library lending, ride-booking, a blogging platform, a banking ledger), each worked from entity identification through cardinality reasoning to complete `CREATE TABLE` statements, with explicit normalization and constraint reasoning behind every design choice.
- [x] **Mock Interview Walkthrough** — a full narrated transcript of a conceptual warm-up followed by a query-writing problem talked through step by step, plus concrete guidance on presenting a SQL solution out loud and the self-sabotage patterns that undermine an otherwise-correct answer.

## Practical Connections

- A candidate who narrates their reasoning through a live query problem — clarifying assumptions, naming the technique, tracing through an example — consistently reads as stronger to an interviewer than one who silently produces the same correct query, because the interview is measuring how you think, not just what you type.
- An engineer asked for an ad hoc report by a non-technical stakeholder faces exactly the same "clarify the ambiguous request before writing anything" discipline rehearsed in the mock interview — "most recent orders" is ambiguous in exactly the same way whether it's an interview prompt or a Slack message from a product manager.
- A team standardizing how it reviews new table designs leans on the same entity-cardinality-then-constraints process used throughout the schema-design topic — deciding what should be its own table, what needs a `UNIQUE` constraint versus a `CHECK` constraint, and where a composite key is doing real work, applies identically in a design review as it does in an interview.
- Debugging a real production slow query and explaining that investigation clearly to a interview panel, as in the mock interview's closing behavioral question, draws on the exact same `EXPLAIN ANALYZE`-first habit (Module 20) that resolves the same problem on a live system.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| `NOT IN` vs. `NOT EXISTS` | `NOT IN` silently returns zero rows if its subquery's result list contains a single `NULL`; `NOT EXISTS` has no such failure mode and is the safer default for anti-join patterns. |
| `RANK()` vs. `DENSE_RANK()` | `RANK()` skips subsequent rank numbers after a tie; `DENSE_RANK()` never skips a number — the distinction matters directly for "Nth-highest value" problems whenever ties are possible. |
| Correlated vs. non-correlated subquery | A non-correlated subquery runs independently and produces one reusable result; a correlated subquery references a column from the outer row and is conceptually re-evaluated per outer row. |
| Natural key vs. surrogate key | A natural key is a real-world attribute assumed to be unique; a surrogate key is a database-generated identifier with no business meaning, chosen for stability when a natural key might change. |
| 2NF vs. 3NF | 2NF eliminates partial dependencies on a composite key; 3NF goes further and eliminates transitive dependencies between two non-key columns. |
| `READ COMMITTED` vs. `SERIALIZABLE` | `READ COMMITTED` only rules out dirty reads; `SERIALIZABLE` rules out every concurrency anomaly by guaranteeing an outcome equivalent to some one-at-a-time ordering of transactions. |

## What's Next

This is the final module of the course. There is no Module 24 — Modules 01 through 23 together form the complete arc from "what is a database" through advanced querying, design, performance, security, and finally interview readiness. If a specific interview question in this module exposed a gap — a technique you couldn't quite reconstruct, or a "why" you couldn't answer past the surface level — the right next step isn't a new module, it's going back to that specific topic's own **Interview Questions** section (every topic file in Modules 01 through 22 ends with one) and its full **Concept** section above it, where the material is explained once at full depth rather than in this module's compressed, interview-length form. Treat this module as the map back to exactly where the deeper explanation lives, not as a replacement for it.
