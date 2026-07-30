# Module 13 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **What Is an Index?** — the full-table-scan problem at scale, an index as a separate ordered lookup structure, `CREATE INDEX`/`DROP INDEX` syntax, the storage and write-speed cost of every index, and when to add (or not add) one.
- [x] **B-Tree and Composite Indexes** — how a B-tree stays balanced, sorted, and logarithmic in lookup cost, why it's PostgreSQL's default, multi-column indexes, and the leftmost-prefix rule governing when they can actually be used.
- [x] **Clustered vs. Non-Clustered Indexes** — the general clustered/non-clustered distinction, the fact that every PostgreSQL index is non-clustered (heap plus pointer), and the `CLUSTER` command as a one-time, unmaintained physical reorder.
- [x] **Covering Indexes and Index-Only Scans** — an index containing every column a query needs, PostgreSQL's `INCLUDE` clause, and why skipping the heap entirely can dramatically speed up read-heavy queries.
- [x] **Reading EXPLAIN Output** — `EXPLAIN` vs. `EXPLAIN ANALYZE`, reading a plan tree, Seq Scan vs. Index Scan vs. Index Only Scan, cost estimate meaning, and a worked before/after example.

## Practical Connections

- A reporting dashboard querying millions of order rows by customer, date range, or status relies entirely on the concepts in this module — without the right indexes, the exact same SQL that answers instantly on a development database with a thousand rows can take seconds or minutes once real production data volume is reached.
- An e-commerce checkout flow that repeatedly looks up a customer's account by email, or an order by its ID, depends on those columns being indexed (frequently automatically, via a `UNIQUE` constraint or primary key, as Topic 1 connected back to Module 5) — every one of those lookups is exactly the kind of high-frequency, high-selectivity access pattern an index is built for.
- A high-traffic application under heavy concurrent write load (constant `INSERT`s of new orders, events, or log rows) has to weigh Topic 1's write-speed cost directly — every extra index on that table is extra work on every single write, which is why production schema decisions about indexing are never "just add it" but a genuine, considered trade-off.
- A slow query reported by users in production is diagnosed, in real practice, exactly the way Topic 5 taught: reach for `EXPLAIN ANALYZE` first, identify whether the plan is scanning far more rows than necessary, and only then decide whether a new index, a composite index reordering, or a covering index is the right fix.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Index vs. constraint | A constraint (`PRIMARY KEY`, `UNIQUE`, Module 5) is a *rule* about what data is allowed; an index is the *structure* PostgreSQL uses internally to enforce that rule efficiently — PostgreSQL creates an index automatically to back a primary key or `UNIQUE` constraint, but an index by itself enforces no rule at all. |
| `(a, b)` composite index vs. `(b, a)` composite index | These are two different physically sorted structures; each only accelerates queries whose conditions start from its own leftmost column (Topic 2) — they are not interchangeable. |
| Clustered index (general concept) vs. PostgreSQL's indexes | A true clustered index physically stores table data in index order; every PostgreSQL index, including the primary key's, is non-clustered — it stores sorted keys with pointers into an unordered heap (Topic 3). |
| `CLUSTER` command vs. a maintained clustered index | `CLUSTER` performs a one-time physical reorder of a table's rows to match an index; it is not kept up to date afterward, unlike a genuinely maintained clustered index in database products that have one (Topic 3). |
| Index Scan vs. Index Only Scan | An Index Scan uses the index to find matching rows but still fetches each row from the heap for any column not present in the index; an Index Only Scan answers entirely from the index itself, with no heap fetch needed (Topic 4, confirmed via `Heap Fetches: 0` in `EXPLAIN ANALYZE`, Topic 5). |
| `EXPLAIN` vs. `EXPLAIN ANALYZE` | `EXPLAIN` shows the planner's estimated plan without running the query; `EXPLAIN ANALYZE` actually executes it and reports real, measured timing alongside the estimates — the latter has genuine side effects on data-modifying statements (Topic 5). |

## What's Next

This module gave you the tools to understand and control how quickly a query finds its data: what an index is, the B-tree structure behind it, PostgreSQL's non-clustered storage model, how far an index can be extended to cover a query entirely, and how to directly verify all of this with `EXPLAIN`. Every topic so far — from Module 5's constraints through this module's indexes — has assumed that a single statement runs safely and predictably on its own. **Module 14 — Transactions & Concurrency** removes that assumption: it covers what happens when multiple statements need to succeed or fail together (`COMMIT`/`ROLLBACK`), the ACID guarantees a database provides, isolation levels, and how concurrent access from multiple users at once is coordinated with locks — including the deeper mechanics behind PostgreSQL's MVCC model that this module's discussion of the heap and `CLUSTER` already began to touch on.
