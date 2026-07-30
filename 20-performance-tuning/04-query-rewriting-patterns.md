# Query Rewriting Patterns

## Learning Objectives

By the end of this section you should be able to:
- Rewrite a correlated subquery as an equivalent join and explain why the rewrite is often (though not always) faster
- Explain the concrete cost of `SELECT *` in production code, beyond style preference
- Define sargability and identify (and fix) a `WHERE` clause that defeats index usage by wrapping an indexed column in a function
- Explain why `OFFSET`-based pagination gets progressively slower at higher page numbers, and how keyset (cursor) pagination avoids that cost

## Prerequisites

- Module 11 — Subqueries, specifically correlated subqueries — this topic assumes you already know what makes a subquery "correlated" (it references a column from the outer query) versus independent, and revisits that exact pattern here.
- [How the Query Planner Works](01-how-the-query-planner-works.md) and [Index Usage and Selectivity](03-index-usage-and-selectivity.md) — the reasoning behind several rewrites in this topic (sargability, join rewrites) depends directly on the cost model and selectivity concepts from those topics.
- Module 7 — Querying Basics, specifically `LIMIT` and `ORDER BY` — this topic's pagination discussion assumes fluency with both.

## Motivation

Two queries can express the exact same logical request — "give me these rows" — and differ enormously in how efficiently the planner can execute them, because some phrasings expose more useful information to the planner (about join relationships, about which columns actually matter, about what an index can directly satisfy) than others. This topic is a toolbox of specific, common rewrites: the same shape of problem, restated in a way the planner handles dramatically better, with no change in the logical result.

## Problem Statement

Four completely ordinary-looking queries, each hiding a performance problem invisible from the SQL text alone:

1. A correlated subquery checking, for every customer, whether they have any orders — fine on a small table, and increasingly expensive as `customers` grows, because the subquery conceptually re-runs once per outer row.
2. A reporting query that does `SELECT *` against a `customers` table that, over the years, has grown a large `notes` text column and a `profile_picture` bytea column nobody asked for.
3. A query filtering `WHERE EXTRACT(YEAR FROM order_date) = 2026` against a table with a perfectly good index on `order_date` — that the planner cannot use at all.
4. A "page 500 of results" query using `OFFSET 10000 LIMIT 20` that takes noticeably longer than "page 1."

None of these are wrong SQL. All four run and return correct results. All four also have a straightforward rewrite that keeps the same result while being measurably cheaper to execute.

## Concept

### Rewriting Correlated Subqueries as Joins

A **correlated subquery** references a column from its enclosing (outer) query, meaning it is conceptually re-evaluated once per outer row:

```sql
-- Correlated subquery: "does this customer have at least one order?"
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

PostgreSQL's planner is often smart enough to transform a simple `EXISTS` subquery like this one into an efficient **semi-join** internally, without you doing anything — so this particular form is not automatically a performance problem. The pattern becomes genuinely risky once the correlated subquery appears in the `SELECT` list itself, forcing a true per-row re-evaluation that the planner has far less freedom to optimize away:

```sql
-- Correlated scalar subquery in the SELECT list — evaluated once per outer row
SELECT c.customer_id, c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count
FROM customers c;
```

```
 Seq Scan on customers c  (cost=0.00..185000.00 rows=50000 width=36)
   SubPlan 1
     ->  Aggregate  (cost=3.65..3.66 rows=1 width=8)
           ->  Seq Scan on orders o  (cost=0.00..3.64 rows=4 width=0)
                 Filter: (customer_id = c.customer_id)
```

Every one of the 50,000 outer rows triggers its own inner scan of `orders`, filtered by that row's `customer_id` — the `SubPlan` is executed once per outer row. Rewritten as a join with aggregation, the same result is produced by scanning `orders` and `customers` once each and grouping:

```sql
SELECT c.customer_id, c.name, COALESCE(o.order_count, 0) AS order_count
FROM customers c
LEFT JOIN (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY customer_id
) o ON o.customer_id = c.customer_id;
```

```
 Hash Left Join  (cost=1350.00..2100.00 rows=50000 width=44)
   Hash Cond: (c.customer_id = o.customer_id)
   ->  Seq Scan on customers c  (cost=0.00..750.00 rows=50000 width=36)
   ->  Hash  (cost=1200.00..1200.00 rows=12000 width=12)
         ->  HashAggregate  (cost=1050.00..1170.00 rows=12000 width=12)
               Group Key: customer_id
               ->  Seq Scan on orders  (cost=0.00..900.00 rows=20000 width=8)
```

The rewritten version reads each table exactly once, aggregates `orders` a single time, and joins the two results together — replacing 50,000 tiny repeated subquery executions with two single-pass scans and one hash join. The `LEFT JOIN` (rather than an inner join) is essential here to preserve customers with zero orders, exactly mirroring what the original correlated subquery did (returning `0` via `COUNT(*)` over an empty match set) — a detail worth double-checking whenever converting this pattern, since an ordinary `INNER JOIN` would silently drop customers with no orders at all.

### Avoiding `SELECT *` in Production Queries

`SELECT *` is convenient for ad hoc, exploratory queries (as noted back in Module 1), but carries concrete costs once embedded in production code:

- **More data transferred than needed.** If `customers` has grown a large `notes` text column or a `bytea` profile picture, `SELECT *` fetches and transmits that data even when the calling code only ever reads `name` and `email` — real network and memory cost, multiplied by every row returned.
- **Prevents index-only scans.** Module 13 covers index-only scans: if a query needs only a few columns, and an index (potentially a covering index with an `INCLUDE` clause) contains all of them, the planner can satisfy the entire query from the index alone, without ever touching the table's main storage. `SELECT *` guarantees every column is needed, which usually forces a full table lookup even when a covering index exists.
- **Silent breakage when the table's shape changes.** Application code that unpacks a `SELECT *` result by column position (rather than by name) breaks silently and confusingly the moment a column is added, dropped, or reordered — a risk that naming exact columns eliminates entirely, since the result shape only changes when the query itself is edited.

```sql
-- Fetches every column, including large ones the caller never uses
SELECT * FROM customers WHERE customer_id = 501;

-- Fetches, and transmits, only what's actually needed
SELECT customer_id, name, email FROM customers WHERE customer_id = 501;
```

### Sargability: Avoiding Functions on Indexed Columns

A condition is **sargable** ("Search ARGument ABLE") if the planner can evaluate it directly using an index, without first computing a function over every row. Wrapping an indexed column in a function on the *left-hand side* of a comparison is the single most common way to accidentally make a condition non-sargable:

```sql
CREATE INDEX idx_orders_order_date ON orders (order_date);

-- Non-sargable: the function must be evaluated on every row before comparison,
-- so the index on the raw order_date column cannot be used
EXPLAIN SELECT * FROM orders WHERE EXTRACT(YEAR FROM order_date) = 2026;
```

```
 Seq Scan on orders  (cost=0.00..2650.00 rows=8341 width=32)
   Filter: (EXTRACT(year FROM order_date) = '2026'::numeric)
```

The index stores raw `order_date` values, not the result of `EXTRACT(YEAR FROM ...)` applied to them — so, exactly as in Topic 3's expression-index discussion, the planner has no way to use it as written. Rewriting the condition to compare the raw column directly against a range restores sargability:

```sql
-- Sargable: compares the raw, indexed column directly against literal bounds
EXPLAIN SELECT * FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01';
```

```
 Index Scan using idx_orders_order_date on orders  (cost=0.29..410.55 rows=8341 width=32)
   Index Cond: ((order_date >= '2026-01-01'::date) AND (order_date < '2027-01-01'::date))
```

The general rule: keep the indexed column bare on one side of the comparison, and push any necessary computation to the *literal* side instead, wherever that's logically possible. `WHERE salary * 1.1 > 100000` (non-sargable, function on the column) can become `WHERE salary > 100000 / 1.1` (sargable, computation on the constant). When the transformation genuinely isn't possible — a case-insensitive text match is a common example, where the comparison inherently needs to happen on a transformed value — an **expression index** on that exact transformation (Topic 3) is the correct fix, rather than abandoning the function.

### Pagination Performance: `OFFSET` vs. Keyset Pagination

A common way to paginate results is `LIMIT`/`OFFSET`:

```sql
-- "Page 1" — cheap
SELECT order_id, order_date FROM orders ORDER BY order_id LIMIT 20 OFFSET 0;

-- "Page 500" — the same query shape, a much bigger OFFSET
SELECT order_id, order_date FROM orders ORDER BY order_id LIMIT 20 OFFSET 10000;
```

`OFFSET` does not let the database "skip ahead" for free. To return rows 10,001–10,020, PostgreSQL must still generate the first 10,020 rows in sorted order and then discard the first 10,000 — the discarded rows are fully computed, just not returned:

```
 Limit  (cost=850.30..851.99 rows=20 width=16) (actual time=42.883..42.901 rows=20 loops=1)
   ->  Index Scan using orders_pkey on orders  (cost=0.29..2500.00 rows=30000 width=16) (actual time=0.020..42.210 rows=10020 loops=1)
```

Notice the underlying index scan actually produces **10,020** rows before the `Limit` node throws away the first 10,000 — the further into the results a page is, the more wasted work every single request performs, even though the caller only ever sees 20 rows at a time. On a large table, "page 1" and "page 500" of the same query are not equally cheap, and the cost grows roughly linearly with the offset.

**Keyset pagination** (also called cursor-based or "seek" pagination) avoids this entirely by remembering the *last row's key* from the previous page and asking for rows strictly after it, rather than asking the database to count and discard a fixed number of rows every time:

```sql
-- First page: no prior key yet
SELECT order_id, order_date FROM orders ORDER BY order_id LIMIT 20;

-- Suppose the last row on that page had order_id = 20.
-- Next page: continue strictly after the last key seen, instead of using OFFSET
SELECT order_id, order_date FROM orders
WHERE order_id > 20
ORDER BY order_id
LIMIT 20;
```

```
 Limit  (cost=0.29..8.51 rows=20 width=16) (actual time=0.018..0.041 rows=20 loops=1)
   ->  Index Scan using orders_pkey on orders  (cost=0.29..2500.00 rows=30000 width=16) (actual time=0.017..0.036 rows=20 loops=1)
         Index Cond: (order_id > 20)
```

Because `order_id` is indexed, this query jumps directly to the first row past `20` and reads only the 20 rows actually needed — page 500 costs virtually the same as page 1, since there is no accumulating "discard" cost tied to how far into the result set the page happens to be.

| | `OFFSET`/`LIMIT` | Keyset (cursor) pagination |
|---|---|---|
| Cost at low page numbers | Cheap | Cheap |
| Cost at high page numbers | Grows roughly linearly with offset | Stays roughly constant |
| Supports "jump to arbitrary page N" | Yes, directly | Not directly — only "next"/"previous" relative to a known key |
| Stable under concurrent inserts/deletes | No — rows can shift between pages if the underlying data changes between requests | Yes — the key already used is unaffected by later inserts/deletes elsewhere |
| Requires an index on the ordering/key column | Helpful but not strictly required | Effectively required for the performance benefit to materialize |

Keyset pagination isn't a strict upgrade in every respect — it requires the caller to carry forward the last-seen key rather than a simple page number, and it doesn't naturally support jumping straight to an arbitrary page in the middle. For a "next page" style interface (feeds, infinite scroll, API pagination over large tables) it is almost always the better choice; for a small, bounded result set, or a UI that genuinely needs numbered page links, plain `OFFSET` remains perfectly reasonable.

## Internal Working (Preview)

Each rewrite in this topic works by giving the planner *more direct information* to reason about, rather than by "tricking" it:

- The join rewrite lets the planner apply its ordinary single-pass join and aggregation machinery (Topic 1) instead of re-planning and re-executing a tiny subplan per outer row.
- Naming exact columns (avoiding `SELECT *`) lets the planner consider index-only scans (Module 13) as a genuinely viable, cheaper access path.
- Sargable conditions let the planner apply an index's ordered structure directly to the comparison, instead of being forced to evaluate a function against every row before any comparison can happen at all.
- Keyset pagination turns "skip N rows" (which has no shortcut — the database must still generate and discard them) into "start from this specific key" (which an index can jump straight to), eliminating the discarded work entirely.

## Real-World Analogy

`OFFSET` pagination is like being told "walk into the library, count past the first 10,000 books on the shelf in order, then hand me the next 20" — every single request restarts the count from zero, no matter how many times you've already done this. Keyset pagination is like leaving a bookmark in the exact spot you stopped last time and saying "continue from right after my bookmark" — you never recount anything you've already passed. Both eventually retrieve the same 20 books; one repeats a linearly growing amount of pointless counting work every time, the other does not.

## Why These Rewrites Work the Way They Do

Every rewrite in this topic is really the same underlying idea applied to a different symptom: the planner can only optimize based on what the query *literally expresses*. A correlated scalar subquery literally expresses "re-evaluate this per outer row," even when a mathematically equivalent join-and-aggregate formulation would let the planner batch the work. A function wrapped around an indexed column literally expresses "compute this for every row first," even when the same condition could be expressed as a comparison against the raw, indexed value. `OFFSET N` literally expresses "generate and discard the first N," even when a keyset condition would let the index skip straight past them. None of this is the planner failing to be clever enough — it is the planner correctly executing exactly what was asked, and the fix in each case is to ask a differently-shaped, logically equivalent question that exposes the more efficient path.

## Advantages

- **Same logical result, meaningfully less work** — every rewrite in this topic preserves the query's correctness while removing repeated, unnecessary, or unusable computation.
- **Works with the planner rather than against it** — none of these rewrites require hints or forcing specific plans; they simply present the same request in a form the existing cost-based machinery already handles well.
- **Compounding benefit at scale** — the gap between the original and rewritten forms widens as tables grow, meaning these patterns matter most exactly where performance matters most.

## Disadvantages / Limitations

- **Not every correlated subquery needs rewriting** — PostgreSQL's planner already transforms many `EXISTS`/`IN` correlated subqueries into efficient semi-joins automatically; blindly rewriting every subquery into a join adds complexity without guaranteed benefit, so verify with `EXPLAIN` first.
- **Keyset pagination has real interface limitations** — it cannot directly answer "show me page 47," only "show me the next page after this key," which is a genuine functional trade-off, not just a syntax difference.
- **Sargability fixes aren't always possible** — some genuinely necessary transformations (case-insensitive search, searching inside a larger string) cannot be algebraically moved to the constant side of a comparison, and require an expression index (Topic 3) rather than a pure query rewrite.

## Best Practices

- Rewrite a correlated scalar subquery in a `SELECT` list into a joined, pre-aggregated derived table whenever `EXPLAIN` shows it re-executing per outer row — but confirm with `EXPLAIN` before and after, rather than rewriting reflexively.
- Name exact columns in any query embedded in application code, reserving `SELECT *` for genuinely ad hoc, throwaway exploration.
- Keep indexed columns bare in comparisons; push necessary computation to the literal/constant side, or create an expression index when the computation cannot be moved.
- Default to keyset pagination for any interface with unbounded, forward-scrolling result sets (feeds, exports, API listings over large tables); reserve `OFFSET`-based paging for small, bounded, or genuinely random-access page needs.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Rewriting every correlated subquery into a join on principle | PostgreSQL already optimizes many `EXISTS`/`IN` subqueries internally; the real target is a scalar subquery in the `SELECT` list, verified with `EXPLAIN` to actually be re-executing per row. |
| `SELECT *` inside application code that only reads two of the table's twelve columns | Transfers and processes unnecessary data on every call, and forecloses index-only scans that naming exact columns would allow. |
| `WHERE LOWER(status) = 'active'` against a plain index on `status`, when the data is already stored consistently in lowercase | Wraps an indexed column in a needless function, defeating index usage for no actual benefit, since the transformation wasn't required in the first place. |
| Using `OFFSET` to paginate a feed that users scroll through indefinitely | Cost grows with page depth, so a user scrolling far enough experiences steadily worsening latency; keyset pagination keeps every page equally cheap. |

## Interview Questions

1. **Q: Why is a correlated scalar subquery in a `SELECT` list often slower than an equivalent `LEFT JOIN` with pre-aggregation?**
   A: The scalar subquery is conceptually re-executed once for every row of the outer query, each execution independently scanning and filtering the inner table. A `LEFT JOIN` against a pre-aggregated derived table instead scans and aggregates the inner table exactly once, then joins the single aggregated result to the outer table in one pass — replacing N small repeated operations with two single-pass operations plus one join.

2. **Q: What does it mean for a `WHERE` condition to be "sargable," and give an example of turning a non-sargable condition into a sargable one.**
   A: A sargable condition is one the planner can evaluate directly using an index, without first computing a function over every row. `WHERE EXTRACT(YEAR FROM order_date) = 2026` is non-sargable, since it must apply a function to every row's `order_date` before comparing. Rewriting it as `WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01'` compares the raw, indexed column directly against literal bounds, making it sargable and eligible for an index scan.

3. **Q: Why does `OFFSET 10000 LIMIT 20` get slower as the offset grows, even though the same number of rows (20) is always returned?**
   A: `OFFSET` does not skip work — the database must still generate all rows up through the offset in sorted order and then discard them, only returning the rows after that point. As the offset grows, the amount of generated-then-discarded work grows with it, even though the final returned row count never changes.

4. **Q: How does keyset (cursor) pagination avoid the cost that plain `OFFSET` pagination has at high page numbers?**
   A: Instead of asking the database to skip a growing number of rows, keyset pagination asks for rows strictly after the last key seen on the previous page (e.g., `WHERE order_id > 20 ORDER BY order_id LIMIT 20`). Given an index on the ordering column, the database can jump directly to that key and read only the next page's worth of rows, with no accumulating discarded work regardless of how deep into the result set the page is.

## Summary

- A correlated scalar subquery in a `SELECT` list is conceptually re-executed once per outer row; rewriting it as a `LEFT JOIN` against a pre-aggregated derived table replaces that repetition with a single-pass join.
- `SELECT *` in production code transfers unneeded data and blocks index-only scans; naming exact columns avoids both costs.
- Sargability means a condition can be evaluated directly by an index; wrapping an indexed column in a function on the comparison's left-hand side breaks it, and pushing the computation to the literal side (or building an expression index) restores it.
- `OFFSET`-based pagination cost grows with page depth because rows up to the offset are generated and discarded every time; keyset pagination avoids this by seeking directly to the last-seen key, keeping every page's cost roughly constant.
- The next topic, Common Anti-Patterns, catalogs several broader, structural data-access habits — beyond individual query phrasing — that quietly degrade performance at scale.
