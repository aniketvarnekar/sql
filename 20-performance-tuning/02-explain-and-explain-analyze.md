# EXPLAIN and EXPLAIN ANALYZE in Depth

## Learning Objectives

By the end of this section you should be able to:
- Read a multi-join, multi-aggregate execution plan node by node, including nested indentation
- Explain precisely what `EXPLAIN` shows versus what `EXPLAIN ANALYZE` shows, and why the difference matters
- Use `EXPLAIN (ANALYZE, BUFFERS)` and interpret its buffer statistics
- Identify an estimated-vs-actual row mismatch in a plan and connect it back to stale statistics
- List concrete, actionable red flags to scan for in any execution plan

## Prerequisites

- [How the Query Planner Works](01-how-the-query-planner-works.md) — you need the cost model, statistics, and join algorithm concepts from that topic before a plan's contents mean anything; this topic is about *reading the planner's own report of the decisions that topic described*.
- Module 13 — Indexes ([overview](../13-indexes/00-module-overview.md)), specifically its introduction to `EXPLAIN`, `Seq Scan`, and `Index Scan` — this topic assumes you can already read a single-table plan and extends that skill to realistic multi-table, multi-aggregate queries.

## Motivation

`EXPLAIN` is the single most valuable tool in this entire module, and arguably in all of practical SQL work. Every claim the previous topic made about cost estimation, statistics, and join algorithms is directly observable, for any query you write, by asking the database to show its own plan. Without it, tuning a slow query is guesswork — trying random rewrites and seeing if they feel faster. With it, tuning becomes a diagnostic process: you can see exactly which operation is expensive, exactly how many rows the planner expected versus how many actually showed up, and exactly where a plan's assumptions broke down.

## Problem Statement

Suppose the following query, using the customers/orders/order_items schema from Module 10, has started taking several seconds where it used to return instantly:

```sql
SELECT c.city, COUNT(*) AS order_count, SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY c.city
ORDER BY total_revenue DESC
LIMIT 10;
```

The SQL hasn't changed. Nobody edited this query. But something about the *data* or the *plan* clearly has. Staring at the SQL text alone gives you no way to find out which join, which filter, or which sort is the bottleneck — you need to see what the database actually did, step by step, with real timings attached.

## Concept

### `EXPLAIN` vs. `EXPLAIN ANALYZE`

`EXPLAIN` alone asks the planner to produce a plan and show it to you, **without running the query**. Every number you see is an estimate.

```sql
EXPLAIN
SELECT c.city, COUNT(*) AS order_count, SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY c.city
ORDER BY total_revenue DESC
LIMIT 10;
```

```
 Limit  (cost=48213.10..48213.13 rows=10 width=48)
   ->  Sort  (cost=48213.10..48215.85 rows=1100 width=48)
         Sort Key: (sum((oi.quantity * oi.unit_price))) DESC
         ->  HashAggregate  (cost=48160.22..48182.72 rows=1100 width=48)
               Group Key: c.city
               ->  Hash Join  (cost=3120.00..47210.00 rows=190000 width=24)
                     Hash Cond: (oi.order_id = o.order_id)
                     ->  Seq Scan on order_items oi  (cost=0.00..38500.00 rows=1900000 width=16)
                     ->  Hash  (cost=2870.00..2870.00 rows=20000 width=16)
                           ->  Hash Join  (cost=95.00..2870.00 rows=20000 width=16)
                                 Hash Cond: (o.customer_id = c.customer_id)
                                 ->  Seq Scan on orders o  (cost=0.00..2650.00 rows=20000 width=12)
                                       Filter: (status = 'completed'::text)
                                 ->  Hash  (cost=60.00..60.00 rows=2800 width=12)
                                       ->  Seq Scan on customers c  (cost=0.00..60.00 rows=2800 width=12)
```

Everything here — `rows=1900000`, `rows=20000`, every cost range — is the planner's *estimate*, computed from statistics alone, before any row has actually been touched. `EXPLAIN` is cheap to run (it never executes the query) and is safe to use even on a slow `UPDATE` or `DELETE`, since nothing is actually modified.

`EXPLAIN ANALYZE` *actually runs the query* and augments every node with real measured numbers: actual rows produced, actual time taken, and actual number of loops:

```sql
EXPLAIN ANALYZE
SELECT c.city, COUNT(*) AS order_count, SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY c.city
ORDER BY total_revenue DESC
LIMIT 10;
```

```
 Limit  (cost=48213.10..48213.13 rows=10 width=48) (actual time=4210.442..4210.448 rows=10 loops=1)
   ->  Sort  (cost=48213.10..48215.85 rows=1100 width=48) (actual time=4210.440..4210.443 rows=10 loops=1)
         Sort Key: (sum((oi.quantity * oi.unit_price))) DESC
         Sort Method: top-N heapsort  Memory: 27kB
         ->  HashAggregate  (cost=48160.22..48182.72 rows=1100 width=48) (actual time=4195.117..4198.900 rows=42 loops=1)
               Group Key: c.city
               Batches: 1  Memory Usage: 585kB
               ->  Hash Join  (cost=3120.00..47210.00 rows=190000 width=24) (actual time=88.203..3980.511 rows=812340 loops=1)
                     Hash Cond: (oi.order_id = o.order_id)
                     ->  Seq Scan on order_items oi  (cost=0.00..38500.00 rows=1900000 width=16) (actual time=0.011..1602.334 rows=1900000 loops=1)
                     ->  Hash  (cost=2870.00..2870.00 rows=20000 width=16) (actual time=87.912..87.913 rows=8341 loops=1)
                           Buckets: 32768  Batches: 1  Memory Usage: 493kB
                           ->  Hash Join  (cost=95.00..2870.00 rows=20000 width=16) (actual time=0.041..85.220 rows=8341 loops=1)
                                 Hash Cond: (o.customer_id = c.customer_id)
                                 ->  Seq Scan on orders o  (cost=0.00..2650.00 rows=20000 width=12) (actual time=0.007..38.552 rows=8341 loops=1)
                                       Filter: (status = 'completed'::text)
                                       Rows Removed by Filter: 11659
                                 ->  Hash  (cost=60.00..60.00 rows=2800 width=12) (actual time=0.021..0.022 rows=2800 loops=1)
                                       ->  Seq Scan on customers c  (cost=0.00..60.00 rows=2800 width=12) (actual time=0.005..0.011 rows=2800 loops=1)
 Planning Time: 0.312 ms
 Execution Time: 4213.107 ms
```

Two very different things are visible now that plain `EXPLAIN` could never show: the query took **4.2 seconds** in `Execution Time`, and the outer `Hash Join` produced **812,340 actual rows** against an estimate of only 190,000 — a more than 4x underestimate. Both facts matter, and reading them correctly is the rest of this topic.

### Reading a Plan Tree

A plan is a **tree**, printed with indentation showing parent/child relationships. Execution logically happens from the innermost (most indented) nodes outward — the deepest nodes read raw data, and each parent node consumes its children's output:

```
Limit                              ← outermost: apply LIMIT 10
  Sort                             ← sort all grouped rows by total_revenue
    HashAggregate                  ← group by city, compute SUM/COUNT
      Hash Join (oi.order_id=o.order_id)   ← join order_items to (orders ⋈ customers)
        Seq Scan on order_items    ← read every row of order_items
        Hash                       ← build hash table from the inner join's result
          Hash Join (o.customer_id=c.customer_id)  ← join orders to customers
            Seq Scan on orders (Filter: status='completed')
            Hash
              Seq Scan on customers
```

Every node has a **node type** (`Seq Scan`, `Hash Join`, `HashAggregate`, `Sort`, `Limit`, and many others), an estimated cost range (`cost=start..total`), an estimated row count and average row width, and — only under `ANALYZE` — actual timing and row counts. `cost=0.00..38500.00` on the `order_items` scan means: `0.00` is the estimated cost to produce the *first* row (essentially instant for a plain sequential scan), and `38500.00` is the estimated cost to produce *all* rows.

### `EXPLAIN (ANALYZE, BUFFERS)`

`EXPLAIN` accepts a parenthesized option list, and the two most important options to combine are `ANALYZE` and `BUFFERS`:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.order_id, o.order_date
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.order_date DESC
LIMIT 20;
```

```
 Limit  (cost=1245.33..1245.38 rows=20 width=12) (actual time=8.912..8.918 rows=20 loops=1)
   Buffers: shared hit=812 read=340
   ->  Sort  (cost=1245.33..1266.18 rows=8341 width=12) (actual time=8.910..8.913 rows=20 loops=1)
         Sort Key: order_date DESC
         Sort Method: top-N heapsort  Memory: 27kB
         Buffers: shared hit=812 read=340
         ->  Seq Scan on orders o  (cost=0.00..2650.00 rows=8341 width=12) (actual time=0.014..7.220 rows=8341 loops=1)
               Filter: (status = 'completed'::text)
               Rows Removed by Filter: 11659
               Buffers: shared hit=812 read=340
 Planning Time: 0.098 ms
 Execution Time: 8.951 ms
```

`Buffers` reports how many 8KB pages were touched: `shared hit` is pages found already in PostgreSQL's shared buffer cache (fast — no disk access needed), and `shared read` is pages that had to be fetched from disk (or the operating system's file cache) because they weren't cached. A plan with a large `shared read` relative to `shared hit`, especially if it recurs on repeated runs of the same query, points to a working set that doesn't comfortably fit in memory — useful information no cost estimate alone can tell you, since it's a real measurement of what actually happened on this run.

### Estimated vs. Actual Rows: The Sign of Stale Statistics

This is the single most important comparison to make in any `EXPLAIN ANALYZE` output. Every node shows both an estimated row count (from the plain `EXPLAIN` portion, e.g. `rows=190000`) and an actual row count (`actual ... rows=812340`). A large, systematic gap between the two is the clearest possible symptom that the planner's statistics no longer reflect the real data:

```
->  Hash Join (cost=3120.00..47210.00 rows=190000 width=24) (actual time=88.203..3980.511 rows=812340 loops=1)
```

Estimated 190,000 rows; actually produced 812,340 — over 4 times as many. Because the planner chose its join algorithm, memory allocation, and join order based on the *smaller* estimate, this join might have been built with the assumption that a hash table would be small and fit comfortably in `work_mem`. When 4x more rows than expected actually flow through, the hash table can spill to disk (visible as `Batches` increasing beyond 1 in the `HashAggregate`/`Hash` node, each additional batch meaning some of the hashing had to be redone against temporary files), turning what should have been an in-memory operation into a disk-bound one. The fix is rarely to rewrite the SQL — it is almost always to run `ANALYZE` on the underlying tables (or investigate *why* autovacuum hasn't kept up) so the next plan is computed against accurate numbers.

A small, occasional estimate/actual gap (say, estimated 1,000 rows, actual 1,150) is completely normal — statistics are a sample, not an exact count, and some imprecision is expected and harmless. It is a *large, systematic* gap — an order of magnitude or more, especially one that recurs consistently rather than being a one-off sampling artifact — that is the actionable signal.

### Actionable Red Flags in a Plan

| Red flag | What it usually means |
|---|---|
| Large estimated-vs-actual row mismatch (an order of magnitude or more) | Stale or insufficient statistics; run `ANALYZE`, or increase the column's statistics target for skewed data |
| `Seq Scan` on a large table with a highly selective `Filter` and a large `Rows Removed by Filter` | A usable index may be missing, or an existing index isn't being chosen — cross-check with Topic 3 |
| `Nested Loop` with a very high `loops` count on the inner node | The outer input turned out much larger than expected, and the nested loop's inner side is being re-scanned an enormous number of times — often traceable back to the same stale-statistics problem |
| `Sort Method: external merge  Disk: ...kB` instead of `quicksort`/`top-N heapsort` in memory | The sort spilled to disk because it didn't fit in `work_mem`; a very large data volume, an unexpectedly high row estimate, or a low `work_mem` setting are the usual causes |
| `Hash Join` / `HashAggregate` showing `Batches` greater than 1 | The hash table didn't fit in memory and had to be built across multiple disk-backed passes — often downstream of an underestimated row count |
| A large gap between `Planning Time` and `Execution Time` where `Planning Time` is unexpectedly high | Often a sign of a very complex query (many joins, many partitions) where planning itself, not execution, is the bottleneck |
| High `shared read` relative to `shared hit`, recurring across repeated runs | The data being accessed doesn't fit comfortably in the database's cache, so repeated disk access is a genuine, ongoing cost rather than a one-time cold-cache effect |

## Internal Working (Deep Dive)

When `EXPLAIN ANALYZE` runs, the executor doesn't just execute the chosen plan — it wraps every node with instrumentation that records, per node: a high-resolution timestamp on entry and exit (aggregated into `actual time=first_row_ms..last_row_ms`), a running count of rows returned, and a count of how many times the node was invoked (`loops`) — relevant because a node nested inside a `Nested Loop` can be executed once *per outer row*, and the reported `actual time` and `rows` are already an average per loop, not a total, which is why multiplying `actual rows × loops` (not just reading `actual rows` alone) is necessary to get the true total row count produced by an inner node under a nested loop. This instrumentation has a small overhead of its own — `EXPLAIN ANALYZE` typically runs a query slightly slower than running it plainly, because of the timing calls — which is worth remembering when using it to measure a query whose absolute speed matters down to the millisecond on very hot, high-frequency paths.

## Real-World Analogy

Plain `EXPLAIN` is like asking an architect for the blueprint of a building before it's built — you can see the intended structure and a projected budget, but nothing about how construction will actually go. `EXPLAIN ANALYZE` is like getting that same blueprint back *after* construction, annotated everywhere the actual cost or time overran the original estimate — "budgeted 2 weeks for the foundation, actually took 5" is exactly analogous to "estimated 190,000 rows, actually produced 812,340." The blueprint alone tells you the plan; the annotated, as-built version tells you where the plan's assumptions were wrong, which is precisely where you should focus your attention if you want to prevent it happening again.

## Why EXPLAIN Was Designed This Way

Because the planner's decisions are invisible by default (a deliberate consequence of SQL being declarative, established in Module 1), some mechanism has to exist for a human to inspect them when something needs debugging — otherwise the entire cost-based optimization process would be an unauditable black box. `EXPLAIN` exposes the plan without side effects (safe on any statement, including destructive ones); `EXPLAIN ANALYZE` goes further and actually executes the statement to compare estimate against reality, deliberately kept as an explicit opt-in (rather than the default) because *actually running* a slow `UPDATE`/`DELETE` just to inspect its plan can itself be dangerous or slow — you must consciously choose to pay that cost.

## Advantages

- **Zero-risk inspection** — plain `EXPLAIN` never executes anything, so it's always safe to run against any query, including ones you haven't fully vetted yet.
- **Ground truth when combined with `ANALYZE`** — real timings and real row counts remove all guesswork about where time is actually being spent.
- **Granular, per-node detail** — a slow query's bottleneck is rarely "the whole query"; `EXPLAIN ANALYZE` pinpoints the exact node responsible.

## Disadvantages / Limitations

- **`EXPLAIN ANALYZE` actually runs the query** — for a slow `UPDATE`, `DELETE`, or `INSERT ... SELECT`, this means the write genuinely happens; wrapping it in a transaction you intend to `ROLLBACK` is standard practice when you only want to inspect the plan.
- **Instrumentation overhead** — the timing calls added under `ANALYZE` introduce measurable overhead of their own, especially for queries with a very large number of rows/nodes, meaning the reported execution time can be somewhat inflated versus a plain, uninstrumented run.
- **A single run is one data point** — caching effects (a "warm" versus "cold" buffer cache), concurrent load on the server, and natural variance mean one `EXPLAIN ANALYZE` run is a sample, not a guarantee; running it more than once, and being mindful of whether the cache was warm, gives a more reliable picture.

## Best Practices

- Reach for `EXPLAIN (ANALYZE, BUFFERS)` as the default combination for real investigation, rather than plain `EXPLAIN` alone — the buffer information is nearly free to add and frequently explains a slowdown that row counts alone don't.
- When investigating a destructive statement, wrap it in `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;` so the plan is inspected against real execution without the write actually persisting.
- Always compare estimated rows to actual rows at every node, not just the outermost one — a small mismatch near the leaves of the tree can compound into a huge mismatch by the time it reaches a join several levels up.
- Re-run `EXPLAIN ANALYZE` more than once when investigating a one-off slow result, to separate a genuine, repeatable problem from a cold-cache or transient-load artifact.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Treating `EXPLAIN` (without `ANALYZE`) numbers as real timings | Plain `EXPLAIN` never executes the query — every number shown is a pre-execution estimate, not a measurement. |
| Running `EXPLAIN ANALYZE` directly on a production `DELETE`/`UPDATE` to "just see the plan" | This actually executes the statement and persists its effects; wrap it in a transaction you roll back if you only want to inspect the plan. |
| Reading only the outermost `Limit`/top node's timing and ignoring inner nodes | The outermost node's `actual time` includes all of its children's time — the actual bottleneck is usually a specific inner node, not the whole query as an undifferentiated block. |
| Ignoring the `loops` value on a node nested inside a `Nested Loop` | `actual rows` and `actual time` under a node with `loops > 1` are averages *per loop*, not totals — the real total row count is `actual rows × loops`. |

## Interview Questions

1. **Q: What is the fundamental difference between `EXPLAIN` and `EXPLAIN ANALYZE`?**
   A: `EXPLAIN` shows the planner's chosen plan and its estimated costs and row counts without executing the query at all. `EXPLAIN ANALYZE` actually executes the query and augments each plan node with real measured timing, actual row counts, and loop counts, allowing a direct comparison between what the planner expected and what really happened.

2. **Q: You see `rows=500` in a plan node's estimate and `actual rows=48000` under `EXPLAIN ANALYZE`. What does this indicate, and what's the typical remedy?**
   A: A large estimated-vs-actual row mismatch like this points to inaccurate table statistics — the planner made its access-path and join-algorithm decisions based on an estimate roughly 100 times too low. The typical remedy is running `ANALYZE` on the underlying table(s) so subsequent plans are computed against accurate, current statistics, rather than rewriting the query itself.

3. **Q: Why would you want to run `EXPLAIN (ANALYZE, BUFFERS)` instead of just `EXPLAIN ANALYZE`?**
   A: `BUFFERS` adds a report, per node, of how many pages were found already cached (`shared hit`) versus had to be read from disk or the OS file cache (`shared read`). This reveals whether slowness is due to genuine disk I/O for data that doesn't fit in cache, information that row counts and timings alone don't directly show.

4. **Q: Why is it considered risky to run `EXPLAIN ANALYZE` directly on a slow `DELETE` statement in production?**
   A: Unlike plain `EXPLAIN`, `EXPLAIN ANALYZE` actually executes the statement to gather real timing data — for a `DELETE`, that means the rows are genuinely deleted and the change persists once committed. The safe approach is to run it inside an explicit transaction and roll back afterward, so the plan can be inspected without the deletion actually taking effect.

## Summary

- `EXPLAIN` shows the planner's chosen plan and its cost/row estimates without running the query; `EXPLAIN ANALYZE` actually executes it and adds real timing, row counts, and loop counts.
- A plan is read as a tree: innermost nodes read raw data first, and each parent node consumes and further processes its children's output.
- `EXPLAIN (ANALYZE, BUFFERS)` additionally reports how much of the data was already cached (`shared hit`) versus read from disk (`shared read`), exposing genuine I/O cost.
- A large, systematic mismatch between a node's estimated and actual row count is the clearest sign that table statistics are stale, and the fix is almost always to run `ANALYZE`, not to rewrite the query.
- Concrete red flags to scan every plan for include large row mismatches, unindexed sequential scans with high filtered-row counts, disk-spilling sorts and hash joins, and high `loops` counts on nested-loop inner nodes.
- The next topic, Index Usage and Selectivity, explains precisely *why* the planner sometimes rejects an index you expected it to use — a decision you now have the tools to observe directly in a real plan.
