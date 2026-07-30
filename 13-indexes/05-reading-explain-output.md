# Reading EXPLAIN Output

## Learning Objectives

By the end of this section you should be able to:
- Explain the difference between `EXPLAIN` and `EXPLAIN ANALYZE`, including why `EXPLAIN ANALYZE` actually executes the query
- Read a simple query plan tree, understanding execution order
- Distinguish a Seq Scan, an Index Scan, and an Index Only Scan in plan output
- Interpret the cost estimates a plan reports (startup cost, total cost, rows, width) at a working level
- Read a before/after pair of plans and identify exactly what changed after adding an index

## Prerequisites

- [What Is an Index?](01-what-is-an-index.md), [B-Tree and Composite Indexes](02-b-tree-and-composite-indexes.md), and [Covering Indexes and Index-Only Scans](04-covering-indexes-and-index-only-scans.md) — this topic assumes you already know what a Seq Scan, Index Scan, and Index Only Scan conceptually *are*; here you'll learn to recognize each of them in the database's own output and read the numbers that accompany them.

## Motivation

Every previous topic in this module has described what PostgreSQL does *conceptually* — scans the whole table, uses an index, skips the heap. But so far, all of that has been taken on faith. `EXPLAIN` is how you stop taking it on faith: it is PostgreSQL directly telling you, in its own words, exactly what plan it chose for a specific query and, with `EXPLAIN ANALYZE`, exactly how long each step actually took. Once you can read `EXPLAIN` output confidently, "is my index actually helping?" stops being a guess and becomes something you can verify in seconds, for any query, for the rest of your work with SQL.

## Problem Statement

Suppose the `orders` table has grown to a million rows, and a report runs:

```sql
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id = 8842;
```

It feels slow. Is it slow because there's no useful index? Because there is an index but PostgreSQL isn't using it? Because the index exists and is used, but the table is just enormous and even indexed access takes real time at that scale? Without a way to see the plan the database actually chose, these are three different problems requiring three completely different fixes, and you'd be guessing at which one applies. `EXPLAIN` removes the guesswork.

## Concept

### `EXPLAIN` vs. `EXPLAIN ANALYZE`

```sql
EXPLAIN
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id = 8842;
```

`EXPLAIN` alone asks the query planner what it *would* do, using its cost-based estimates — it does **not** actually run the query. This is safe to run against any statement, including an `UPDATE` or `DELETE`, since nothing is executed and no data changes.

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id = 8842;
```

`EXPLAIN ANALYZE` actually **executes** the query, timing each step of the real plan and reporting both the planner's original estimates and the real, measured numbers side by side. This is far more informative — estimates can be wrong, especially on tables the planner doesn't have accurate statistics for — but it comes with an important caution: because it truly executes the statement, running `EXPLAIN ANALYZE` on an `INSERT`, `UPDATE`, or `DELETE` genuinely performs that change. A common safe habit is to wrap such a statement in a transaction and roll it back (Module 14 covers transactions in full):

```sql
BEGIN;
EXPLAIN ANALYZE
UPDATE orders SET status = 'shipped' WHERE customer_id = 8842;
ROLLBACK;
```

For a plain `SELECT`, this concern doesn't apply — reading data has no side effect to worry about.

### Reading a Plan Tree

Here is realistic `EXPLAIN` output for the query above, on a 1,000,000-row `orders` table with **no index** on `customer_id`:

```
                                QUERY PLAN
-----------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..21834.00 rows=5 width=24)
   Filter: (customer_id = 8842)
```

And here is `EXPLAIN ANALYZE` for the same query and same table:

```
                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..21834.00 rows=5 width=24) (actual time=0.031..142.558 rows=4 loops=1)
   Filter: (customer_id = 8842)
   Rows Removed by Filter: 999996
 Planning Time: 0.112 ms
 Execution Time: 142.601 ms
```

Reading this from the top:

- **`Seq Scan on orders`** — the operation being performed: a full sequential scan of the `orders` table (exactly the full table scan described in Topic 1).
- **`Filter: (customer_id = 8842)`** — the `WHERE` condition being checked against every row read.
- **`Rows Removed by Filter: 999996`** — out of 1,000,000 rows read, 999,996 were checked and discarded because they didn't match — direct, concrete evidence of the "check every row" cost from Topic 1.
- **`Planning Time`** — how long the planner spent deciding on this plan (usually tiny).
- **`Execution Time`** — how long actually running the query took, end to end: 142.6 milliseconds here, to find 4 matching rows out of a million.

A query tree with more than one step is read from the **innermost (most indented) node outward**, and execution conceptually flows **bottom-up**: the deepest node runs first, and its output feeds the node above it. A plan joining two tables, for example, has scan nodes for each table nested beneath a join node — the scans happen first, and the join operates on their results.

### Seq Scan vs. Index Scan vs. Index Only Scan

Now suppose an index is added:

```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

Re-running `EXPLAIN ANALYZE` on the identical query:

```
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_orders_customer_id on orders  (cost=0.42..8.60 rows=5 width=24) (actual time=0.019..0.023 rows=4 loops=1)
   Index Cond: (customer_id = 8842)
 Planning Time: 0.098 ms
 Execution Time: 0.041 ms
```

The plan changed shape entirely:

| Before (no index) | After (index added) |
|---|---|
| `Seq Scan on orders` | `Index Scan using idx_orders_customer_id on orders` |
| `Filter: (customer_id = 8842)` (checked against every row) | `Index Cond: (customer_id = 8842)` (used directly by the index to jump to matching entries) |
| Cost estimate: `21834.00` total | Cost estimate: `8.60` total |
| Execution time: `142.601 ms` | Execution time: `0.041 ms` |

`Index Cond`, rather than `Filter`, is the tell-tale sign the index is actually being used to narrow the search itself, rather than merely being scanned incidentally. This is the same underlying mechanism described conceptually in Topics 1 and 2, now visible directly in the database's own explanation.

Now suppose the query and index are adjusted so the index alone contains every needed column (Topic 4's covering index):

```sql
CREATE INDEX idx_orders_customer_covering
    ON orders (customer_id)
    INCLUDE (order_date, total_amount);
```

```sql
EXPLAIN ANALYZE
SELECT order_date, total_amount
FROM orders
WHERE customer_id = 8842;
```

```
                                                                 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using idx_orders_customer_covering on orders  (cost=0.42..8.44 rows=5 width=12) (actual time=0.015..0.018 rows=4 loops=1)
   Index Cond: (customer_id = 8842)
   Heap Fetches: 0
 Planning Time: 0.087 ms
 Execution Time: 0.033 ms
```

`Index Only Scan` (rather than plain `Index Scan`) confirms PostgreSQL never had to consult the heap, and `Heap Fetches: 0` confirms it directly — the exact mechanism Topic 4 described, now visible as a concrete, measured line in real output. (Had the visibility map not been up to date for some pages, you would instead see a non-zero `Heap Fetches` count, reflecting the caveat discussed in Topic 4.)

You will also frequently encounter a fourth, hybrid family of scan in real `EXPLAIN` output on larger tables: **`Bitmap Index Scan`** paired with **`Bitmap Heap Scan`**. This is a strategy PostgreSQL uses when many (but not all) rows match a condition — it first builds an in-memory "bitmap" of which heap pages contain matching rows using the index, then visits only those pages, in physical order, in a second step. It's a middle ground between a full `Seq Scan` and a plain `Index Scan`, and it's worth being able to recognize by name even though a full treatment of when the planner reaches for it belongs to a more advanced discussion (Module 20, Performance Tuning, covers query plan strategy selection at that depth).

### What the Cost Numbers Roughly Mean

The `cost=X..Y rows=N width=W` figures shown next to every plan node are the planner's own internal, unitless cost estimates — **not milliseconds** — computed before the query runs, based on table statistics:

| Figure | Rough meaning |
|---|---|
| **Startup cost** (`X` in `cost=X..Y`) | Estimated cost incurred before this step can produce its *first* row of output — for a `Seq Scan`, typically 0 (it can start returning rows immediately); for a step that must first fully sort or fully build something (like a `Sort` node), it's often high, since no output can be produced until that inner work finishes. |
| **Total cost** (`Y` in `cost=X..Y`) | Estimated cost to produce *all* of this step's output, assuming it runs to completion — this is the number most often compared between two candidate plans, and between "before" and "after" adding an index. |
| **`rows`** | The planner's *estimate* of how many rows this step will produce — compare this against the `rows=` figure inside `actual time=...` in `EXPLAIN ANALYZE` output to see how accurate the planner's guess was. |
| **`width`** | The planner's estimate of the average size, in bytes, of each row this step produces — wider rows generally cost more to move through later steps of the plan. |

These cost units are arbitrary and internally consistent (calibrated so that, roughly, reading one page from disk costs `1.0` unit by default) — they exist purely so the planner can *compare* candidate plans against each other and pick the cheapest one; they are not a prediction of wall-clock milliseconds. `EXPLAIN ANALYZE`'s `actual time=...` figures, by contrast, are genuinely measured milliseconds, from actually running the query.

## Internal Working (Preview)

Behind every `EXPLAIN` output is PostgreSQL's **query planner (optimizer)**: given a parsed query, it considers multiple possible ways to execute it (scan the whole table? use this index? use that one? join these two tables in which order?), estimates the cost of each candidate using statistics it maintains about the table (row counts, common value distributions, an so on — gathered via the `ANALYZE` command, which can be run manually or automatically), and picks the plan with the lowest estimated total cost. `EXPLAIN` shows you the winning plan directly; `EXPLAIN ANALYZE` additionally runs that winning plan and reports the real, measured results alongside the original estimate, letting you see exactly where the planner's guess and reality diverge, if they do.

## Real-World Analogy

`EXPLAIN` is like asking a route-planning app for directions without actually driving anywhere: it tells you the route it's chosen and its estimated travel time, based on its model of traffic and distance, before you've moved an inch. `EXPLAIN ANALYZE` is actually taking that drive with a stopwatch running, comparing the app's original estimate against how long each leg of the trip really took — sometimes confirming the estimate was accurate, sometimes revealing that a "quick shortcut" the planner favored on paper was actually slower in practice because of something the estimate didn't fully capture.

## Why EXPLAIN Was Designed This Way

SQL's declarative nature (established in Module 1) means you never tell the database *how* to execute a query — only *what* result you want. `EXPLAIN` exists precisely because that abstraction, while powerful, would otherwise be a black box: without some way to inspect the planner's actual chosen strategy, you'd have no way to verify whether an index you added is actually being used, or diagnose why a query is slow. `EXPLAIN` is the deliberate, built-in window into the "how," offered without forcing you to abandon the declarative "what" in your actual SQL — you get to keep writing plain `SELECT` statements while still having full visibility into their execution when you need it.

## Advantages

- **Removes guesswork from performance tuning** — instead of assuming an index helps, you can directly confirm whether it's used, and by how much, for any specific query.
- **`EXPLAIN` alone is completely safe** — it never executes anything, so it can be run freely against any statement, including data-modifying ones, without any risk.
- **`EXPLAIN ANALYZE` reveals planner misestimates** — comparing estimated `rows` against actual `rows` is one of the most common ways real production performance problems are diagnosed (a large gap usually means outdated table statistics).

## Disadvantages / Limitations

- **`EXPLAIN ANALYZE` has real side effects for data-modifying statements** — it genuinely executes an `INSERT`/`UPDATE`/`DELETE`, so running it carelessly against such a statement outside a rolled-back transaction actually changes data.
- **Reading plans well takes practice** — a plan involving several joins, sorts, and aggregates nested together can be visually dense, and correctly identifying which nested step is actually the bottleneck takes deliberate practice, not just a glance.
- **Plans can change as data and statistics change** — the same query can get a different plan tomorrow if the table has grown, shrunk, or been re-`ANALYZE`d, so a plan captured once isn't necessarily permanent evidence of how a query will always behave.

## Best Practices

- Reach for `EXPLAIN ANALYZE` (not just `EXPLAIN`) whenever you need to confirm real timing, not just the planner's estimate — the two can genuinely diverge.
- Always wrap `EXPLAIN ANALYZE` around a data-modifying statement in a transaction you intend to roll back, unless you specifically mean to perform that change.
- When comparing "before" and "after" adding an index, look at both the scan type (`Seq Scan` vs. `Index Scan` vs. `Index Only Scan`) and the total cost/execution time — a changed scan type with barely-changed timing on a tiny table is a sign the index isn't meaningfully helping yet (consistent with Topic 1's point about small tables).
- When `rows` (estimated) and the actual row count in `EXPLAIN ANALYZE` differ by a large factor, treat that as a signal the table's statistics may be stale, and consider running `ANALYZE` on the table.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Treating `EXPLAIN`'s cost numbers as milliseconds | They are arbitrary, internally-consistent planner units used to compare candidate plans against each other — only `EXPLAIN ANALYZE`'s `actual time=...` figures are real, measured milliseconds. |
| Running `EXPLAIN ANALYZE` on an `UPDATE` or `DELETE` in production without a transaction to roll back | `EXPLAIN ANALYZE` genuinely executes the statement — doing this on a data-modifying statement outside a transaction actually performs that change, not just a simulation of it. |
| Seeing `Filter` in plan output and assuming the index is being used to narrow the search | `Filter` means the condition is being checked row-by-row against whatever rows were already read (e.g., during a Seq Scan); `Index Cond` is the sign the index itself is being used to jump directly to matching entries. |
| Assuming a plan captured once will never change | Plans are chosen fresh based on current table statistics and estimated costs — a query's plan can and does change as a table's size or data distribution changes over time. |

## Interview Questions

1. **Q: What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?**
   A: `EXPLAIN` shows the query planner's chosen plan and its cost estimates without actually running the query. `EXPLAIN ANALYZE` actually executes the query and reports real, measured timing and row counts alongside the original estimates, which is more informative but means it has genuine side effects for data-modifying statements.

2. **Q: In an `EXPLAIN` plan, what's the difference between seeing `Filter: (customer_id = 8842)` versus `Index Cond: (customer_id = 8842)`?**
   A: `Filter` means the condition is being checked against rows after they've already been read (typically during a sequential scan, checking every row one by one and discarding non-matches). `Index Cond` means the condition is being used directly by an index to locate matching entries efficiently, without reading every row in the table.

3. **Q: What do the `cost=X..Y` numbers in an `EXPLAIN` plan actually represent?**
   A: `X` is the estimated startup cost — the cost incurred before the step can return its first row — and `Y` is the estimated total cost to produce all of the step's output. Both are arbitrary, internally-consistent units used by the planner to compare candidate plans against each other; they are not literal time measurements.

4. **Q: You add an index intended to help a slow query, but `EXPLAIN` still shows a `Seq Scan`. What are two possible explanations?**
   A: One possibility is that the table (or the matching subset of rows) is small enough, or the matched fraction of rows large enough (low selectivity), that the planner correctly estimates a sequential scan as cheaper than using the index anyway (Topic 1's point about when indexes don't help). Another possibility is that the query doesn't actually match the index's leftmost-prefix requirements (Topic 2) — for example, filtering only on a non-leading column of a composite index, or applying a function to the column that the plain index can't be used to satisfy — in which case the index genuinely can't help that specific query as written.

## Summary

- **`EXPLAIN`** shows the planner's chosen plan and cost estimates without running the query; **`EXPLAIN ANALYZE`** actually executes it and adds real, measured timing — treat the latter as having genuine side effects on data-modifying statements.
- Read a plan tree from the **innermost, most indented node outward** — that's the actual execution order, bottom-up.
- **`Seq Scan`** reads every row and filters afterward (`Filter:`); **`Index Scan`** uses an index to jump to matching rows directly (`Index Cond:`) but may still fetch from the heap; **`Index Only Scan`** answers entirely from the index, confirmed by `Heap Fetches: 0`.
- **`cost=startup..total`**, **`rows`**, and **`width`** are the planner's own estimates (arbitrary units, not milliseconds) used to compare candidate plans; `EXPLAIN ANALYZE`'s `actual time=...` and `rows=` are real, measured values you can compare against those estimates.
- Adding a well-chosen index typically changes a plan from `Seq Scan` to `Index Scan` (or `Index Only Scan`), with a correspondingly large drop in both estimated cost and real execution time — exactly the kind of before/after change worth confirming directly with `EXPLAIN ANALYZE` rather than assuming.
