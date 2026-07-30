# Module 06 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **INSERT** — single-row and multi-row `INSERT`, `INSERT ... SELECT` for copying/deriving data, the `RETURNING` clause, and exactly what happens (and what error is raised) when a `NOT NULL`, `CHECK`, `UNIQUE`, or foreign key constraint is violated.
- [x] **UPDATE** — the `SET` clause, updating multiple columns atomically, `SET` expressions and `WHERE` conditions referencing a row's own columns, `UPDATE ... FROM` for cross-table updates, and the danger of an omitted `WHERE`.
- [x] **DELETE** — `DELETE ... WHERE`, the catastrophic effect of omitting `WHERE`, `DELETE ... USING` for cross-table deletes, `RETURNING` with `DELETE`, and how foreign key `ON DELETE` actions from Module 5 activate during deletion.
- [x] **TRUNCATE vs. DELETE** — the speed difference (page deallocation vs. row-by-row logging), `WHERE` support (`DELETE` only), trigger-firing behavior, `RESTART IDENTITY` for resetting `SERIAL` sequences, and PostgreSQL's transactional/rollback-safe behavior for both.
- [x] **UPSERT with ON CONFLICT** — the insert-or-update problem, `DO NOTHING` vs. `DO UPDATE`, the `EXCLUDED` pseudo-table, and why this avoids the race condition inherent in check-then-insert application logic.

## Practical Connections

- **Any "save" button in any application** — a user profile edit, a shopping cart update, a settings change — is, underneath, an `UPDATE` (or an upsert) executed with a precise `WHERE` clause identifying exactly one row, relying on the constraints from Module 5 to guarantee the result never violates the application's data rules.
- **Nightly batch jobs and data pipelines** that reload staging tables from an external feed routinely choose `TRUNCATE` over `DELETE` specifically for the performance difference at scale, and often follow it with `INSERT ... SELECT` or bulk inserts to repopulate the table from source data.
- **Any system that syncs external state into a database** — inventory counts, cache tables, deduplicated event logs — leans on `INSERT ... ON CONFLICT DO UPDATE` to correctly handle "this might already exist" without introducing race conditions, which becomes critical the moment more than one process can write concurrently.
- **Every dangerous production incident involving "we deleted the wrong data"** almost always traces back to exactly the pattern warned about twice in this module: an `UPDATE` or `DELETE` executed without a `WHERE` clause that was correctly scoped and verified beforehand — the single habit of testing a `WHERE` clause as a `SELECT` first prevents the overwhelming majority of these.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| `DELETE` (no `WHERE`) vs. `TRUNCATE` | Both empty a table entirely, but `DELETE` removes rows one by one (DML, supports triggers, `RETURNING`, and `WHERE` when present) while `TRUNCATE` deallocates storage pages wholesale (DDL, far faster, no `WHERE`, no row-level triggers by default). |
| `UPDATE ... FROM` vs. `DELETE ... USING` | Both let a statement reference a second table to decide which rows are affected — `UPDATE ... FROM` uses that second table's values to compute new column values; `DELETE ... USING` uses it purely to help filter which rows to remove, never to compute a value. |
| `ON CONFLICT DO NOTHING` vs. `ON CONFLICT DO UPDATE` | `DO NOTHING` silently discards a conflicting insert, leaving the existing row untouched; `DO UPDATE` modifies the existing row using a `SET` clause, performing a true upsert. |
| A `SERIAL` sequence "resetting" vs. simply having empty data | Emptying a table (via `DELETE` or plain `TRUNCATE`) does not reset the underlying sequence that generates new `SERIAL` values — only `TRUNCATE ... RESTART IDENTITY` explicitly resets the counter back to its starting value. |
| `EXCLUDED.column` vs. `table_name.column` inside `DO UPDATE SET` | `EXCLUDED.column` is the incoming value that was about to be inserted; `table_name.column` (e.g. `inventory.quantity`) is the value already stored in the existing, conflicting row — conflating the two produces either an unintended overwrite or an unintended no-op. |
| A `RESTRICT` foreign key rejecting a `DELETE` vs. a `CHECK`/`UNIQUE` constraint rejecting an `INSERT`/`UPDATE` | Both are constraint enforcement from Module 5, but a `RESTRICT` rejection on `DELETE` is about a *referenced* row still being *needed* elsewhere, while a `CHECK`/`UNIQUE` rejection on `INSERT`/`UPDATE` is about the row's *own* values being invalid or duplicated. |

## What's Next

Module 06 gave you full command over changing data inside a table: adding rows (`INSERT`), changing them (`UPDATE`), removing them (`DELETE` and `TRUNCATE`), and handling the common insert-or-update case safely under concurrency (`ON CONFLICT`) — all while relying on Module 5's constraints to guarantee that data never ends up in an invalid state, no matter which of these statements touches it. Every example in this module used a `WHERE` clause, a filter, or a join-like condition somewhat informally, without yet exploring the full depth of how rows are selected and shaped. **Module 07 — Querying Basics** picks that thread up properly: the full syntax and logical processing order of `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`, `DISTINCT`, and the comparison/logical operators that make filtering precise — the foundation the next several modules (Aggregation, Joins, Subqueries) build on directly.
