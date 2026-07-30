# Module 21 — Advanced SQL

## Module Goal

By the end of this module, you will be equipped with a set of practical, frequently-needed PostgreSQL features that don't belong to any single earlier theme (querying, joining, indexing, transactions) but show up constantly in real schemas and real interviews. You'll turn rows into columns for reporting with conditional aggregation, store and query flexible semi-structured data with `JSON`/`JSONB` without abandoning the relational model, understand the sequence machinery that actually powers every auto-incrementing primary key you've been using since Module 1, stage intermediate results safely with temporary tables, and split enormous tables into manageable physical partitions. None of these five topics depend on each other — each solves a distinct, self-contained problem — but all five depend heavily on modules you've already completed, and all five are exactly the kind of thing that separates "knows the syntax" from "has built real systems."

## Topics Covered in This Module

1. **[Pivoting Data with Conditional Aggregation](01-pivoting-with-conditional-aggregation.md)** — Turning rows into columns using `CASE WHEN` inside aggregate functions, PostgreSQL's `FILTER` clause, and the `crosstab()` function as an alternative.
2. **[JSON in SQL](02-json-in-sql.md)** — The `JSON` and `JSONB` types, querying with `->`/`->>`/`#>`, containment and existence operators, indexing `JSONB` with GIN, and when to normalize instead.
3. **[Sequences and Auto-Increment](03-sequences-and-auto-increment.md)** — `CREATE SEQUENCE`, how `SERIAL`/`BIGSERIAL`/`IDENTITY` are built on top of it, `nextval`/`currval`/`setval`, and why gaps in sequence values are normal.
4. **[Temporary Tables](04-temporary-tables.md)** — `CREATE TEMPORARY TABLE`, session/transaction-scoped lifetime, `ON COMMIT DROP`, and how temp tables differ from CTEs and views.
5. **[Table Partitioning](05-table-partitioning.md)** — Splitting one logical table into physical partitions (range, list, hash), query pruning, and the operational trade-offs of partitioning.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 3 — Data Types**, especially its coverage of numeric and string types and its early acknowledgment that `JSON`/`JSONB` exist as types (see the [Module 3 overview](../03-data-types/00-module-overview.md)): Topic 2 of this module is the deep dive that was deliberately deferred from there.
- **Module 5 — Constraints & Keys**, especially [Primary Keys](../05-constraints-and-keys/03-primary-keys.md) and its discussion of database-generated identifiers: Topic 3 of this module explains the machinery — sequences — that actually generates the values behind every `SERIAL` primary key you've declared since [Your First Query](../01-introduction/05-your-first-query.md).
- **Module 9 — Aggregation**, especially [Aggregate Functions](../09-aggregation/01-aggregate-functions.md) and `GROUP BY`, plus **Module 8 — Functions and Expressions**' `CASE` expressions: Topic 1 of this module combines these two tools — which you already know individually — in a new way to solve the row-to-column pivoting problem.
- **Module 13 — Indexes**, especially [What Is an Index?](../13-indexes/01-what-is-an-index.md) and [B-Tree and Composite Indexes](../13-indexes/02-b-tree-and-composite-indexes.md): Topic 2 of this module introduces GIN, a fundamentally different index structure from the B-tree you already know, built for exactly the kind of containment queries `JSONB` needs.
- **Module 17 — CTEs & Recursion**: Topic 4 of this module (temporary tables) is contrasted directly against the `WITH` clause you learned there — both let you name and stage an intermediate result, but they differ in persistence, materialization, and scope in ways worth knowing precisely.

## How to Study This Module

Each topic in this module is independent — you can read them in any order without losing continuity — but the numbering follows increasing structural weight. Topic 1 (pivoting) is the lightest: it's a query-writing pattern built entirely from tools you already have, not a new database feature. Topic 2 (JSON) deserves careful, hands-on reading — it introduces genuinely new operators and a new indexing strategy, and the "when should this actually be JSON vs. real columns" judgment call is one you'll be asked about in nearly every schema-design interview. Topic 3 (sequences) is short but conceptually important: understanding *why* sequences behave the way they do (atomic, non-transactional, gap-tolerant) will settle a question that confuses even experienced engineers the first time they see a gap in production IDs. Topic 4 (temporary tables) is best read side-by-side with your Module 17 notes on CTEs, since the comparison is the whole point. Topic 5 (partitioning) is the most operationally significant topic in this module — read it slowly, and pay particular attention to the unique-key restriction, since it's a real constraint that surprises people the first time they try to partition a table with an existing primary key.
