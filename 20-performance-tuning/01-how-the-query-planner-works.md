# How the Query Planner Works

## Learning Objectives

By the end of this section you should be able to:
- Describe, in detail, what happens to a SQL statement between the moment you send it and the moment rows come back
- Explain what "cost-based optimization" means and read the cost units the planner itself uses
- Explain what table statistics are, how `ANALYZE` produces them, and why stale statistics lead directly to bad plans
- Describe, at a conceptual level, the three join algorithms PostgreSQL can choose between and when each tends to win
- Explain why join order matters and how the planner decides it instead of you

## Prerequisites

- [What Is SQL?](../01-introduction/02-what-is-sql.md) — this topic assumes you remember the brief parser/planner/executor preview from that topic's "Internal Working" section; this topic is that preview's full expansion.
- Module 13 — Indexes ([overview](../13-indexes/00-module-overview.md)) — you need to already understand what an index is and what a sequential scan versus an index scan means, since this topic is about *how the planner chooses between them*, not what they are.
- Module 10 — Joins & Set Operations ([overview](../10-joins-and-set-operations/00-module-overview.md)) — you need to be comfortable writing a multi-table `JOIN`, since this topic explains how that same join gets executed physically.

## Motivation

You have been writing SQL for nineteen modules now, and at no point have you told the database *how* to fetch anything — only *what* you want. That gap between what you asked for and how it actually gets computed is filled by a piece of software inside PostgreSQL called the query planner (also called the optimizer). Every single query you have ever run, no matter how simple, passed through it. Most of the time it makes a good decision invisibly and you never think about it. When it makes a *bad* decision — and it sometimes will, for reasons that are entirely understandable once you know how it works — a query that should take milliseconds can suddenly take minutes, with no error, no warning, and no obvious cause in the SQL text itself. You cannot fix what you cannot see, and you cannot see it without understanding what the planner is trying to do in the first place.

## Problem Statement

Consider a single, simple-looking query:

```sql
SELECT o.order_id, c.name, o.order_date
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE c.city = 'Austin'
ORDER BY o.order_date DESC
LIMIT 20;
```

To answer this, PostgreSQL has an enormous number of *possible* ways to actually do the work, all producing the same correct result:

- It could scan every row of `customers`, checking `city = 'Austin'` one by one, or it could use an index on `city` (if one exists) to jump straight to matching rows.
- It could scan every row of `orders`, checking each one's `customer_id` against the matching customers, or it could use an index on `orders.customer_id`.
- It could start from `customers` and, for each matching customer, look up their orders — or it could start from `orders` and, for each order, look up its customer. The result is identical either way; the amount of work is not.
- It could join the two tables using a nested loop, a hash join, or a merge join — three genuinely different physical strategies, described later in this topic, all logically equivalent.
- It could sort all matching rows by `order_date` and then take the top 20 — or, if an index already exists in that order, it could walk the index in order and stop after 20 rows without sorting anything at all.

None of this is visible in the SQL text. The SQL only describes the *what*. Something inside the database has to decide the *how*, for every single query, every single time — and it has to make that decision in milliseconds, before a single row has even been read.

## Concept

### The Full Pipeline

Module 1 previewed a simplified three-step pipeline (parser → planner → executor). The real pipeline has one more explicit stage worth naming:

```
   Your SQL text
        │
        ▼
   1. Parser            — checks syntax, resolves table/column names against the catalog,
        │                   produces a parse tree
        ▼
   2. Rewriter           — expands views and rules into their underlying definitions
        │                   (not relevant to most queries; matters once views appear)
        ▼
   3. Planner/Optimizer  — generates candidate execution plans, estimates the *cost*
        │                   of each using table statistics, picks the cheapest one
        ▼
   4. Executor           — carries out the chosen plan, actually reading/writing data
        │                   and computing joins, filters, sorts, aggregates
        ▼
   Result set
```

Stage 3 — the planner — is the subject of this entire module. It does not "run" your query in any sense; it reasons about your query and produces a **plan**: a tree of physical operations (scan this table this way, join these two results that way, sort the output like this) that the executor will carry out mechanically. You can see this plan yourself without running the query at all, using `EXPLAIN` (the subject of the next topic).

### Cost-Based Optimization

PostgreSQL's planner is a **cost-based optimizer**. For every candidate way of executing your query, it computes an estimated numeric *cost* — an abstract unit, not a measurement of actual seconds — and (for query shapes small enough to fully enumerate) picks the plan with the lowest total cost. The cost model is built from a small number of tunable constants that represent how expensive different kinds of work are relative to each other:

| Constant | Default | Represents |
|---|---|---|
| `seq_page_cost` | 1.0 | Cost of reading one page (block) sequentially from disk |
| `random_page_cost` | 4.0 | Cost of reading one page via a random (non-sequential) access — historically much slower on spinning disks, still non-zero on SSDs due to reduced prefetching efficiency |
| `cpu_tuple_cost` | 0.01 | Cost of processing one row (tuple) in memory |
| `cpu_index_tuple_cost` | 0.005 | Cost of processing one index entry |
| `cpu_operator_cost` | 0.0025 | Cost of evaluating one operator or function call per row |

Every plan the planner considers gets a cost expressed roughly as `(pages read × page cost) + (rows processed × per-row cost)`. A sequential scan of a 10,000-page table costs roughly `10,000 × 1.0 = 10,000` in page cost alone, plus a per-row charge for every row examined. An index scan that only needs to read 50 pages via random access costs roughly `50 × 4.0 = 200` in page cost, plus a much smaller per-row charge, because far fewer rows are ever touched. This is why an index scan usually wins for a highly selective filter (Topic 3 formalizes "selective") and a sequential scan usually wins when most of the table's rows will be needed anyway — the fixed random-access penalty per page stops paying for itself.

Crucially: **none of this is a measurement**. It's an *estimate*, computed before a single row is actually read, using statistics the database keeps about your data. That's the next, and arguably most important, idea in this topic.

### Table Statistics and `ANALYZE`

The planner's cost estimates are only as good as its knowledge of your data's *shape* — how many rows a table has, how many distinct values a column has, how values are distributed, how correlated a column's physical storage order is with its logical value order. PostgreSQL keeps this knowledge in a system catalog called `pg_statistic` (readable in a friendlier form via the view `pg_stats`), and it is populated by the `ANALYZE` command:

```sql
ANALYZE orders;
```

`ANALYZE` takes a random sample of rows from the table and computes, per column:

- **`n_distinct`** — an estimate of how many distinct values the column has (or the exact count, for small tables).
- **Most Common Values (MCV) list** — the most frequently occurring values and their frequencies, e.g. `status` might record that `'completed'` accounts for 82% of rows.
- **Histogram bounds** — for the remaining, less common values, a set of bucket boundaries that let the planner estimate "what fraction of rows have `order_date` between these two dates" without storing every value.
- **Correlation** — how closely the column's on-disk physical order matches its logical sort order, which affects whether an index scan is likely to require lots of random disk access or can proceed almost sequentially.

You can inspect this directly:

```sql
SELECT attname, n_distinct, most_common_vals, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

```
 attname | n_distinct |        most_common_vals         | correlation
---------+------------+----------------------------------+-------------
 status  |          4 | {completed,pending,cancelled,returned} |        0.91
(1 row)
```

Every candidate plan's estimated *row count* — "how many rows will satisfy `WHERE status = 'pending'`?" — comes directly from this data, not from actually checking. If the statistics say `'pending'` is rare, the planner will estimate few matching rows and likely favor an index scan. If they say it's common, it will estimate many matching rows and likely favor a sequential scan instead — correctly recognizing that reading almost the whole table via random index lookups is *more* expensive than just reading it sequentially once.

### Why Stale Statistics Cause Bad Plans

Statistics are a **snapshot**, not a live view. They are only as current as the last time `ANALYZE` ran. PostgreSQL runs `ANALYZE` automatically in the background as part of **autovacuum** whenever a table has changed enough since its last analysis — but "enough" is a threshold, and there is always a window where the on-disk data has changed but the statistics have not caught up.

This becomes a real problem after any large, sudden change in the data's shape:

```sql
-- Table starts small and mostly 'pending'
INSERT INTO orders (customer_id, status, order_date)
SELECT (random() * 1000)::int, 'pending', now()
FROM generate_series(1, 100);

-- Statistics reflect: mostly 'pending', small table

-- A large batch job runs, inserting a million 'completed' orders
INSERT INTO orders (customer_id, status, order_date)
SELECT (random() * 1000)::int, 'completed', now() - (random() * interval '365 days')
FROM generate_series(1, 1000000);

-- Statistics have NOT been refreshed yet — the planner still believes
-- the table is small and 'pending' is the common case.
```

If a query now runs `WHERE status = 'completed'`, the planner — still working from the *old* statistics — may badly underestimate how many rows match, choose an index scan expecting to fetch a handful of rows, and instead end up performing a random-access index lookup for hundreds of thousands of rows one at a time. The correct plan, given the table's actual current shape, would have been a sequential scan. The SQL didn't change. The data did, and the planner's picture of the data didn't keep up. Running `ANALYZE orders;` immediately after a large bulk load is the direct fix, and is a standard step in any serious batch-loading procedure.

### Join Order and Join Algorithm Selection

A query joining several tables introduces two more decisions the planner must make, neither of which is expressed anywhere in your SQL: **which order to join the tables in**, and **which physical algorithm to use for each join**.

**Join order.** `FROM a JOIN b JOIN c` does not mean "join a to b, then join that to c" as a literal instruction — it means "these three tables must all be combined according to these conditions," and the planner is free to choose whichever order produces the least total work, as long as the logical result is identical. For a small number of tables, PostgreSQL exhaustively evaluates every possible join order and algorithm combination and picks the cheapest. Beyond a configurable threshold (`join_collapse_limit`, default 8), exhaustive search becomes too expensive to run for every query, and PostgreSQL switches to a genetic algorithm that heuristically searches for a good — not necessarily perfect — order instead.

**Join algorithm.** For each individual join in the chosen order, PostgreSQL picks between three physical strategies:

| Algorithm | How it works | Tends to win when |
|---|---|---|
| **Nested Loop** | For each row in the outer (driving) input, scan the inner input for matches — ideally using an index rather than a full scan of the inner side | The outer side is small, and the inner side has a usable index on the join column |
| **Hash Join** | Build an in-memory hash table from the smaller input, keyed on the join column, then scan the larger input once, probing the hash table for each row | Neither side is sorted, no useful index exists, and at least one side is small enough to fit reasonably in memory |
| **Merge Join** | If both inputs are already sorted on the join column (or can be cheaply sorted), walk both simultaneously, advancing whichever side is behind | Both inputs are large and already sorted (e.g., both scanned via an index that produces the join column in order) |

None of these is universally "best" — each is a different trade-off, and the planner picks based on estimated input sizes (from statistics, again) and what indexes/sort orders are available. A nested loop over a million-row inner table with no usable index is catastrophic; a nested loop over a 10-row outer table joined to an indexed million-row inner table is often the fastest option available. This is precisely why the same *logical* join can appear with a completely different plan on a small development database versus a large production one: the estimated row counts are different, so the cheapest physical strategy is different, even though the SQL is byte-for-byte identical.

## Internal Working (Deep Dive)

Putting the full decision process together, for a two-table join the planner conceptually does the following:

```
 1. Consult pg_statistic for both tables (rows, distinct values, MCVs, histograms)
 2. Estimate rows surviving each table's own WHERE conditions
 3. For each candidate access path per table:
       Seq Scan          — cost ≈ pages × seq_page_cost + rows × cpu_tuple_cost
       Index Scan        — cost ≈ (matching pages × random_page_cost) + (matching rows × cpu costs)
       Index Only Scan   — like Index Scan, but skips the table heap entirely if possible
 4. For each candidate join order × join algorithm combination:
       estimate combined cost using the input estimates from step 3
 5. Pick the single lowest total-cost complete plan
 6. Hand that plan tree to the executor, which runs it and returns rows
```

Every one of these estimates traces back to statistics collected by `ANALYZE`. There is no step in this process that involves actually counting real rows before the query runs — that would defeat the purpose of estimating cost cheaply. The entire mechanism is a bet: that a random sample taken at `ANALYZE` time is representative enough of the table's current shape to make a good decision now.

## Real-World Analogy

Think of the planner like a logistics dispatcher at a delivery company who has never personally seen today's shipment, but has yesterday's manifest and a general sense of typical traffic patterns. Given an order ("deliver these five packages to these three addresses"), the dispatcher doesn't drive the route themselves — they choose a route and a vehicle based on estimates: how many packages, how far apart the addresses typically are, which roads are usually congested at this hour. If yesterday's manifest is a good approximation of today's, the dispatcher's chosen route is efficient. If today's shipment is wildly different from yesterday's — ten times as many packages, to a part of town rarely visited before — the dispatcher's estimate is wrong, and the chosen route (say, a small van instead of a truck) turns out to be a poor fit, even though the dispatcher followed a perfectly sound decision-making process using the information available. This is exactly the relationship between the planner's decisions and the freshness of `ANALYZE` statistics: the process is sound, but its output is only as good as its input.

## Why the Planner Was Designed This Way

SQL's entire value proposition, established in Module 1, is that you describe *what* you want and the database figures out *how* — freeing your queries from ever needing to change as data volume, hardware, or indexing strategy evolves. A cost-based optimizer is the mechanism that makes that promise real: instead of a fixed, rule-based strategy ("always use an index if one exists," which is frequently *wrong*, as Topic 3 shows), the planner reasons quantitatively about the *actual* current shape of your data and picks whatever is cheapest for *this* table, at *this* size, with *this* value distribution. The alternative — forcing you to hand-specify join order and access strategy in your SQL, the way older "navigational" systems required — would reintroduce exactly the physical-detail burden the relational model was invented to remove.

## Advantages

- **Adapts automatically as data grows** — the same query can go from a sequential scan on a tiny table to an index scan on a large one without any change to the SQL, because the cost estimates themselves shift as the statistics do.
- **Considers far more options than a human reasonably could** — for a five-table join, there are dozens of possible join orders and hundreds of order/algorithm combinations; the planner evaluates this space in milliseconds.
- **Improves over time for free** — successive PostgreSQL versions have shipped smarter cost models and better statistics without requiring any SQL rewrites from you.

## Disadvantages / Limitations

- **Estimates can be wrong** — the entire system rests on statistics being a reasonably accurate snapshot of the current data; stale statistics, small sample sizes, or correlated columns the planner doesn't model well can all produce a badly wrong estimate and, downstream, a badly wrong plan.
- **Cost is not the same as wall-clock time** — the cost units are an internal, relative measure calibrated by the tunable constants shown above; a "cheaper" plan by the model is *usually* faster in practice, but the model is an approximation of real hardware behavior, not a physical measurement.
- **Exhaustive search doesn't scale indefinitely** — beyond `join_collapse_limit` tables, the planner switches to a heuristic (genetic) search that is not guaranteed to find the true optimum, trading plan quality for planning speed on very large joins.

## Best Practices

- Run `ANALYZE` explicitly after any bulk load, bulk delete, or bulk update that meaningfully changes a table's size or value distribution, rather than waiting for autovacuum's automatic threshold to trigger.
- Don't assume a plan that was good on a small development or staging database will remain good in production — statistics-driven decisions are inherently size- and distribution-dependent, so always verify real query plans against realistically sized data before trusting performance conclusions.
- When a query is unexpectedly slow, form the habit of asking "does the planner's estimate of the row counts involved look right?" before assuming the SQL itself is wrong — Topic 2 gives you the tool to check this directly.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The database always uses an index if one exists on the filtered column." | The planner uses whichever access path is estimated to be cheapest for the actual data; a sequential scan can legitimately be cheaper than an index scan, especially for low-selectivity filters (Topic 3). |
| "The join order I write in `FROM a JOIN b JOIN c` is the order it actually executes in." | The planner is free to reorder joins as long as the logical result is unchanged, and typically will, based on cost estimates — the written order is a logical specification, not an execution instruction. |
| "Cost numbers in a plan represent milliseconds." | Cost is an internal, relative unit derived from the tunable cost constants, not a direct time measurement — `EXPLAIN ANALYZE` (Topic 2) is what shows actual elapsed time. |
| "Once I run `ANALYZE`, I never need to run it again." | Statistics go stale as data changes; while autovacuum reanalyzes automatically on a threshold, any sudden large change in a table's data benefits from an explicit `ANALYZE` rather than waiting. |

## Interview Questions

1. **Q: What is the difference between the query planner's estimated cost and the query's actual execution time?**
   A: Estimated cost is a unitless number computed *before* execution, derived from table statistics and a small set of tunable constants (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, etc.) representing the relative expense of different operations. Actual execution time is a real, measured wall-clock duration observed while the plan runs. The planner picks a plan based on estimated cost alone, since it must decide *before* running anything; `EXPLAIN ANALYZE` is required to see real, measured time.

2. **Q: Why can stale table statistics cause a good index to be ignored, or a bad plan to be chosen?**
   A: The planner's every cost estimate depends on statistics gathered by `ANALYZE` — row counts, value distributions, and distinctness. If the actual data has changed substantially since the last `ANALYZE` (e.g., after a large bulk load), the planner is reasoning about an outdated picture of the table and can badly misestimate how many rows a condition will match, leading it to choose an access path or join strategy that was appropriate for the old data shape but not the current one.

3. **Q: Explain, conceptually, the difference between a nested loop join, a hash join, and a merge join.**
   A: A nested loop join iterates the outer input row by row, searching the inner input (ideally via an index) for each match — efficient when the outer side is small. A hash join builds an in-memory hash table from the smaller input keyed on the join column, then probes it once per row of the larger input — efficient when neither side is pre-sorted and no useful index exists. A merge join walks two already-sorted inputs in lockstep, advancing whichever is behind — efficient when both sides are large and already ordered on the join key.

4. **Q: If two logically identical queries produce different execution plans on two different databases, is that a bug?**
   A: Not necessarily. The planner's chosen plan depends on the current table statistics — row counts, value distributions, index availability — which can legitimately differ between a small development database and a large production one, or between two production databases with different data shapes. Different plans for identical SQL are an expected consequence of cost-based optimization, not evidence of a bug.

## Summary

- SQL execution proceeds through a parser, a rewriter, a cost-based planner/optimizer, and an executor — the planner is the stage that decides *how* to fulfill a request, and this module is about understanding and influencing that stage.
- The planner is cost-based: it estimates a relative cost for every candidate plan using table statistics and a small set of tunable per-operation cost constants, then picks the cheapest.
- Table statistics come from `ANALYZE` (run automatically by autovacuum, or manually) and include row counts, most common values, histograms, and correlation; stale statistics are one of the most common real-world causes of a sudden, unexplained bad plan.
- For multi-table queries, the planner independently decides join order (which table to process first) and join algorithm (nested loop, hash join, or merge join) per join — none of this is fixed by the order you wrote your `FROM`/`JOIN` clauses in.
- The next topic, `EXPLAIN` and `EXPLAIN ANALYZE`, is the concrete tool for observing everything described conceptually here against your own real queries.
