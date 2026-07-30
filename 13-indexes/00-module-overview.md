# Module 13 — Indexes

## Module Goal

By the end of this module, you will understand the single most important performance tool a relational database gives you: the index. You will know what an index actually is (a separate, ordered structure that points back to your table's rows), how PostgreSQL builds and uses one internally (the B-tree), how to design multi-column indexes correctly, how PostgreSQL's storage model differs from databases that maintain a true clustered index, how an index can be extended to satisfy an entire query without ever touching the table itself, and how to read the query planner's own explanation of what it's doing (`EXPLAIN`). Every `WHERE`, `JOIN`, and `ORDER BY` you have written since Module 7 has been silently relying on the database either scanning every row or, when an index exists and helps, skipping almost all of them — this module makes that difference visible and puts it under your control.

## Topics Covered in This Module

1. **[What Is an Index?](01-what-is-an-index.md)** — The full-table-scan problem at scale, an index as a separate ordered lookup structure, `CREATE INDEX` syntax, and why indexes are a trade-off, not a free win.
2. **[B-Tree and Composite Indexes](02-b-tree-and-composite-indexes.md)** — How PostgreSQL's default index structure is organized conceptually, multi-column indexes, and the leftmost-prefix rule that governs when a composite index can actually be used.
3. **[Clustered vs. Non-Clustered Indexes](03-clustered-vs-non-clustered-indexes.md)** — The general distinction between physically-ordered and separately-pointed-to data, why PostgreSQL has no true clustered index by default, and what the `CLUSTER` command actually does (and doesn't do).
4. **[Covering Indexes and Index-Only Scans](04-covering-indexes-and-index-only-scans.md)** — Building an index that contains everything a query needs so the table itself never has to be read, and PostgreSQL's `INCLUDE` clause.
5. **[Reading EXPLAIN Output](05-reading-explain-output.md)** — `EXPLAIN` vs. `EXPLAIN ANALYZE`, reading a plan tree, the difference between a Seq Scan, an Index Scan, and an Index Only Scan, and what the cost numbers mean.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 5 — Constraints & Keys**, especially its treatment of primary keys and `UNIQUE` constraints ([Module 5 overview](../05-constraints-and-keys/00-module-overview.md)): both of those constraints are *implemented* using an index under the hood in PostgreSQL. This module assumes you already understand what a primary key and a `UNIQUE` constraint guarantee logically; here you'll learn the storage structure that makes the guarantee enforceable and fast.
- **Module 7 — Querying Basics**, specifically `WHERE` filtering and `ORDER BY` sorting: these are exactly the two operations an index is built to accelerate. If you haven't internalized how a `WHERE` clause narrows down rows and how `ORDER BY` sorts them, the motivation for indexing won't land.
- **Module 9 — Aggregation** and **Module 10 — Joins & Set Operations**: `GROUP BY` and join conditions are the other two places indexes matter enormously. This module assumes you're comfortable writing a basic join and a basic `GROUP BY` query, since several examples filter, join, and aggregate against an indexed column.
- **Module 1 — Introduction**, specifically [Your First Query](../01-introduction/05-your-first-query.md): that topic's "Internal Working" section already previewed that an index can make row-filtering far faster than checking every row — this module is where that preview becomes a full explanation.

## How to Study This Module

Read Topics 1 and 2 first and in order — they are the conceptual backbone of the entire module. Topic 1 establishes *why* indexes exist and what they cost; Topic 2 establishes *how* PostgreSQL actually structures one and the leftmost-prefix rule, which is the single most common source of "I added an index and it still isn't being used" confusion in real practice. Topic 3 is shorter but deserves careful, precise reading — it's one of the places relational databases genuinely diverge from each other, and getting it slightly wrong (assuming PostgreSQL behaves like a database you've read about elsewhere) leads to real production misunderstandings. Topics 4 and 5 build directly on the first three: Topic 4 shows how far you can push an index (to the point where the table is never touched at all), and Topic 5 gives you the tool to actually verify, for your own queries, whether any of this is happening — treat Topic 5 as a skill you'll keep coming back to for the rest of the course, not a one-time read. By the end of this module, whenever a query feels slow, you should instinctively reach for `EXPLAIN` before reaching for a guess.
