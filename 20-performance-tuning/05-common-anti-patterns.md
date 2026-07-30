# Common Anti-Patterns

## Learning Objectives

By the end of this section you should be able to:
- Describe the N+1 query problem at the data-access level and explain how batching fixes it
- Explain why an implicit type mismatch between a column and a literal can silently disable index usage
- Explain why adding more indexes is not a free performance improvement, and what it costs on writes
- Explain the risk of an unbounded result set in a production query path and how to guard against it

## Prerequisites

- Module 13 — Indexes ([overview](../13-indexes/00-module-overview.md)) — this topic assumes you understand what an index costs to maintain, which the over-indexing discussion below builds on directly.
- [How the Query Planner Works](01-how-the-query-planner-works.md) and [Index Usage and Selectivity](03-index-usage-and-selectivity.md) — the type-casting discussion here is a specific instance of the sargability and index-usage reasoning from those topics.
- Module 3 — Data Types, specifically comfort with the idea that every column has a strict, declared type — the type-casting anti-pattern below is entirely about what happens when a comparison's two sides don't share that type.

## Motivation

Some performance problems live in a single query's structure (Topic 4). Others live in the broader *pattern* of how an application talks to its database — habits that look completely harmless in a single instance, and only reveal their cost when repeated at scale, under load, or against a growing table. This topic catalogs four such patterns. None of them produce an error. All four are common enough, in real systems of every size, that recognizing them on sight is one of the highest-leverage skills this module teaches.

## Problem Statement

Four scenarios, each syntactically unremarkable:

1. A reporting task first fetches a list of 200 customer IDs, then — for each ID, one at a time — runs a separate query to fetch that customer's most recent order. 200 customers means 201 total round-trips to the database for one report.
2. A `customer_id` column is declared `INTEGER`, but a piece of code compares it against the text value `'501'` instead of the number `501`. The query still returns the correct row. It just doesn't use the index it should.
3. A table has accumulated fourteen indexes over the years, one added every time a new report needed a new filter column, and the table's `INSERT` throughput has quietly degraded to a fraction of what it once was.
4. A query with no `LIMIT` clause, written against a table with 200 rows in development, is deployed unchanged to production, where the same table now has 40 million rows.

Every one of these "worked" when first written. Every one becomes a serious, sometimes catastrophic, real-world performance problem as data or load grows — and every one is fixable once recognized.

## Concept

### The N+1 Query Problem

The **N+1 query problem** occurs when retrieving related data is done as one query to fetch an initial set of N rows, followed by N *additional* separate queries — one per row of that initial result — instead of a single additional query that retrieves all the related data at once. It is a pattern of *data access*, not any particular syntax, and can happen in raw, hand-written SQL just as easily as anywhere else.

```sql
-- Query 1: fetch 200 customer IDs
SELECT customer_id FROM customers WHERE city = 'Austin';

-- Then, for EACH of the 200 IDs returned above, a separate query is issued:
SELECT order_id, order_date FROM orders
WHERE customer_id = 501 ORDER BY order_date DESC LIMIT 1;

SELECT order_id, order_date FROM orders
WHERE customer_id = 502 ORDER BY order_date DESC LIMIT 1;

-- ...198 more, one per remaining customer_id
```

This is 201 total round-trips to the database to answer one logical question ("each Austin customer's most recent order"). Each individual query might be fast — a few milliseconds — but the *round-trip* itself (network latency, connection overhead, query parsing and planning for each separate statement) is paid 200 extra times, and that fixed per-query overhead, multiplied by N, frequently dominates the actual work being done.

The fix is to replace the N repeated queries with a single query that retrieves everything needed for the entire initial set at once, typically using a join or a window function:

```sql
-- One query: each Austin customer's most recent order, in a single round-trip
SELECT DISTINCT ON (c.customer_id)
       c.customer_id, o.order_id, o.order_date
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.city = 'Austin'
ORDER BY c.customer_id, o.order_date DESC;
```

This single statement is planned once, executed once, and transfers exactly the rows needed — no matter whether the Austin customer count is 200 or 200,000, it remains one round-trip. The N+1 pattern is easy to miss precisely because each of the individual per-row queries, examined in isolation, looks completely fine — the problem is only visible when you count *how many times* that "fine" query actually runs for one logical operation.

### Implicit Type Casting Killing Index Usage

Every column has a declared, strict type (Module 3). When a comparison's two sides don't share a type, PostgreSQL must apply an implicit cast to make the comparison valid — and depending on which side gets cast, this can silently disable an otherwise perfectly good index, in exactly the same way a function wrapped around a column does (Topic 4's sargability discussion).

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    customer_id INTEGER,
    order_date DATE
);
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

-- customer_id is INTEGER; '501' is a text literal
EXPLAIN SELECT * FROM orders WHERE customer_id = '501';
```

```
 Index Scan using idx_orders_customer_id on orders  (cost=0.29..8.31 rows=1 width=24)
   Index Cond: (customer_id = 501)
```

In this particular case PostgreSQL is able to resolve the text literal `'501'` to the integer `501` at parse time, before planning even begins, so the index is still used — PostgreSQL's cast resolution between a typed column and an untyped literal is often forgiving. The genuinely dangerous version is comparing two *columns* of different types, or a column against an already-typed value of a different type, where the cast cannot be resolved away and must instead be applied to every row of the indexed column at execution time:

```sql
-- order_reference is stored as TEXT, e.g. '00501', with leading zeros preserved
CREATE INDEX idx_orders_reference ON orders (order_reference);

-- Comparing a text column against a numeric literal forces a cast
-- of every row's order_reference to numeric before comparison
EXPLAIN SELECT * FROM orders WHERE order_reference::numeric = 501;
```

```
 Seq Scan on orders  (cost=0.00..2650.00 rows=1 width=24)
   Filter: ((order_reference)::numeric = '501'::numeric)
```

The explicit cast on the column side (`order_reference::numeric`) makes this exactly as non-sargable as wrapping the column in any other function — the index on the raw text values cannot be consulted, because the comparison is no longer "is this raw value equal to X," it's "is the numeric conversion of this raw value equal to X," and the index has no way to answer the second question without inspecting every row. The fix mirrors Topic 4's sargability fix directly: compare like types without a cast on the column side — `WHERE order_reference = '00501'` — or, if the comparison genuinely must happen on a converted value, build an expression index on that exact conversion.

### Over-Indexing

Module 13 establishes that an index isn't free — it costs disk space and, critically, it costs **write overhead**, because every `INSERT`, `UPDATE` (of an indexed column), or `DELETE` must also update every index defined on that table, not just the underlying table row itself. A table with one index pays this cost once per write. A table with fourteen indexes pays it fourteen times per write:

```sql
-- Every one of these INSERTs must also add an entry to all fourteen indexes
-- defined on this table, even though the query only touches one row
INSERT INTO orders (customer_id, order_date, status, order_reference, ...)
VALUES (501, '2026-07-31', 'pending', '00999', ...);
```

Each additional index defined on a table is a genuine, ongoing tax on every write to that table, paid whether or not that particular write ever benefits from the index existing at all. This cost is easy to lose track of because indexes tend to accumulate gradually — one added for a report last quarter, another for a new filter last month, a third "just in case" — with nobody ever measuring the cumulative write-side cost of the full set. A table can absolutely have *too many* indexes: past a certain point, additional indexes stop meaningfully helping the read queries they were added for (many end up redundant with each other, or serving reports run once a month) while continuing to slow down every single write, all the time.

| Symptom | Likely cause |
|---|---|
| `INSERT`/`UPDATE` throughput on a table has degraded steadily over time, with no application logic change | Accumulated indexes, each adding write overhead |
| Several indexes on the same table share a leading column or heavily overlap | Redundant indexes maintaining largely duplicate information |
| An index hasn't been used by any query in a long time (checkable via PostgreSQL's `pg_stat_user_indexes` usage counters) | A candidate for removal — it's pure write-side cost with no offsetting read-side benefit |

The right number of indexes on a table is not "as many as possible" or "one per column that ever appears in a `WHERE` clause" — it's a deliberate, periodically revisited balance between the read queries that genuinely benefit and the write cost every index adds, regardless of whether that particular write ever touches a query that uses it.

### Unbounded Result Sets Without `LIMIT`

A query with no `LIMIT` clause returns *every* matching row, however many there turn out to be. On a 200-row development table, this is invisible — the whole table is 200 rows either way. On a production table with 40 million rows, the same query, unchanged, can attempt to return millions of rows in one response — consuming enormous memory on both the database and the application side, saturating network bandwidth, and often stalling or crashing whatever received it, all from a query that "worked fine" everywhere it was tested.

```sql
-- Fine on a 200-row development table; potentially catastrophic
-- on the same table with 40 million rows in production
SELECT * FROM orders WHERE status = 'completed' ORDER BY order_date DESC;
```

```sql
-- Bounded: caps the result set to a size any caller can safely handle,
-- combined with keyset pagination (Topic 4) if more than one page is needed
SELECT * FROM orders WHERE status = 'completed'
ORDER BY order_date DESC
LIMIT 100;
```

Any query embedded in a production code path that a user or an automated process can trigger — as opposed to a genuinely one-off, manually run analysis query — should have an explicit, deliberate bound on how many rows it can possibly return. This isn't only about performance in the abstract; it's about making the query's worst-case behavior predictable and safe regardless of how large the underlying table eventually grows, rather than relying on today's data volume staying small forever.

## Internal Working (Preview)

Each of these four anti-patterns produces normal, individually well-formed query plans — there is no single `EXPLAIN` red flag that says "this is an N+1 problem" or "this table is over-indexed," because each *individual* statement is unremarkable on its own:

- N+1 is visible only by counting *how many separate statements* are issued for one logical operation — typically observed in database query logs or connection-level statement counters, not in any single plan.
- The type-cast anti-pattern is visible in a single plan, as a `Filter` (rather than an `Index Cond`) wrapping a cast expression around the column — exactly the sargability signature from Topic 4.
- Over-indexing is visible in the table's catalog metadata (`\d tablename` in `psql`, or querying `pg_indexes`/`pg_stat_user_indexes`) showing a large number of indexes, several with heavily overlapping leading columns, combined with degraded write throughput observed independently of any single query's plan.
- Unbounded result sets are visible as a plan with no `Limit` node at the top at all — the plan is prepared to return however many rows match, however many that turns out to be.

## Real-World Analogy

The N+1 pattern is like a delivery service that, given a list of 200 addresses, sends a separate driver and a separate truck to each individual address one at a time, instead of loading all 200 packages onto a single truck and running one efficient route — the same total cargo delivered, at wildly different total cost, purely because of how the work was batched. Over-indexing is like maintaining fourteen separate cross-referenced card catalogs in a library for a collection that only ever gets searched three different ways — every new book added must now be filed into all fourteen catalogs, even though eleven of them are barely ever consulted. An unbounded query without `LIMIT` is like asking a librarian for "every book about history," in a small town library where that's forty books, versus in a national archive where that's forty million — the same request, catastrophically different in scale, if nobody ever bounds it.

## Why These Are Considered Anti-Patterns

All four patterns share the same underlying shape: something that is invisible or harmless at small scale becomes expensive or dangerous at large scale, and because these problems don't produce errors — the SQL is correct, the results are correct — they tend to survive testing and code review entirely undetected until real production data or real production load exposes them. They are called "anti-patterns" rather than "bugs" precisely because nothing about them is syntactically wrong; the risk is entirely about how a data-access habit behaves as the two variables that testing environments rarely match production on — data volume and concurrent load — actually grow.

## Advantages

This section is genuinely advantage-free from the reader's perspective — these are patterns to recognize and avoid, not techniques with a legitimate trade-off to weigh. The "advantage" side of this topic is simply: recognizing each pattern early is cheap; discovering it in a production incident is not.

## Disadvantages / Limitations

- **N+1 avoidance can, in rare cases, over-fetch.** A single joined query retrieving related data for every row of an initial set can occasionally fetch more data than strictly needed if only a small handful of the initial rows actually required the related data — batching is almost always still the right trade-off, but it is a trade-off, not a universal free win.
- **Removing an index carries real risk if usage isn't measured carefully.** An index that looks unused in a short observation window might still be essential for a rare but critical monthly or quarterly query; verify actual usage over a representative time period (via `pg_stat_user_indexes`) before removing anything.
- **A hard `LIMIT` without pagination can silently hide data from a legitimate caller who actually needed more than the limit.** Bounding a result set is about *safety*, not truncating correctness — any caller that might legitimately need more than one page must be given an explicit way to request subsequent pages (Topic 4's keyset pagination), not just a query that quietly caps out.

## Best Practices

- Whenever a report, page, or process needs related data for a set of rows, retrieve it with one batched query (a join, or a single query using `IN`/`= ANY`), never with a loop issuing one query per row.
- Keep comparisons type-matched: compare a column against a literal of the *same* declared type, and avoid casting the column side of any comparison; if a cast is genuinely unavoidable, build an expression index on it (Topic 3).
- Periodically audit a table's index list against actual usage statistics, and remove indexes that are redundant with another index or that no query has used in a representative observation window.
- Treat every query embedded in a path a user or automated process can trigger as needing an explicit `LIMIT`, paired with pagination (Topic 4) if the caller may legitimately need more than one page's worth of results.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Looping over an initial result set, issuing one additional query per row to fetch related data | Each additional query pays a fixed round-trip and planning cost N times over, when a single batched query pays that fixed cost once, regardless of how large N grows. |
| Comparing a `TEXT` column against a numeric literal, or casting the column side of a comparison | Forces a per-row conversion that the planner cannot resolve using an index built on the column's raw, uncast values, exactly like wrapping the column in any other function. |
| Adding a new index for every new report's filter column, without ever removing old ones | Each additional index is a permanent tax on every write to the table, whether or not that write's query ever uses the index — the set of indexes needs periodic pruning, not just periodic growth. |
| Shipping a query with no `LIMIT` because the development table was small | The query's worst-case cost and result size scale with the table's *actual* row count, which is dictated by production data volume, not by whatever happened to be loaded during development or testing. |

## Interview Questions

1. **Q: What is the N+1 query problem, described purely in terms of database access, and how do you fix it?**
   A: It's a pattern where retrieving related data for a set of N rows is done as one initial query plus N additional separate queries — one per row — instead of a single additional query retrieving all the related data at once. Each of the N extra queries individually looks fine, but the fixed overhead of a database round-trip is paid N extra times for one logical operation. The fix is to replace the per-row queries with a single batched query, typically a join or a query using `IN`/`= ANY` against the full set of keys at once.

2. **Q: Why might a query like `WHERE numeric_column = '42'` (comparing an integer column to a text literal) still use an index, while `WHERE text_column::numeric = 42` might not?**
   A: In the first case, PostgreSQL can typically resolve the text literal to the column's actual numeric type once, at parse time, without needing to transform the column's stored values at all — the index on the raw numeric values remains directly usable. In the second case, the cast is applied to the *column* itself, meaning every row's value must be converted before the comparison can happen, which the planner cannot do using an index built on the column's original, untransformed values — the same sargability problem as wrapping a column in any function.

3. **Q: Why isn't "add an index for every column that appears in a `WHERE` clause somewhere" a good indexing strategy?**
   A: Every index adds ongoing write-side overhead — every `INSERT`, and every `UPDATE` touching an indexed column, must also update that index, on every single write, regardless of whether that particular write's data is ever queried through the index. A table can accumulate more indexes than its actual read workload justifies, at which point additional indexes are pure write-side cost with diminishing or redundant read-side benefit — the right number of indexes is a deliberate balance, revisited periodically, not a one-way accumulation.

4. **Q: A query with no `LIMIT` performs fine in development but causes problems in production. What changed, and how do you prevent this category of problem going forward?**
   A: Nothing about the query changed — its worst-case result size scales with the actual number of matching rows in the table, which was small in development and is far larger in production. The general fix is to treat any query in a path reachable by a user or automated process as needing an explicit, deliberate bound (`LIMIT`), paired with pagination for callers that legitimately need more than one page, rather than relying on the table happening to stay small.

## Summary

- The **N+1 query problem** replaces one batched query with N+1 separate ones — a fixed per-query overhead paid N extra times for a single logical operation; the fix is always a single batched retrieval (a join or an `IN`/`= ANY` query).
- Comparing a column against a differently-typed value — or explicitly casting the column side of a comparison — can force a per-row conversion that defeats index usage, exactly like the sargability problem from Topic 4.
- Every index is a permanent, ongoing cost on every write to its table; accumulating indexes without periodically pruning unused or redundant ones degrades write throughput for read benefit that may no longer justify it.
- A production query path with no `LIMIT` has a worst-case cost and result size dictated by the table's actual row count, not by whatever happened to be loaded during development — always bound production-reachable queries explicitly.
- All four patterns are dangerous precisely because they produce no error and look correct in isolation; they only reveal their cost at real data volume or real concurrent load, which is exactly why deliberately watching for them matters.
