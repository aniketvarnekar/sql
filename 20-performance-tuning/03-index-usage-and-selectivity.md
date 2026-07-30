# Index Usage and Selectivity

## Learning Objectives

By the end of this section you should be able to:
- Explain precisely why the planner sometimes chooses a sequential scan over an existing, applicable index
- Define selectivity and estimate it for a simple condition given basic statistics
- Create an expression (function-based) index and explain when a plain index cannot substitute for one
- Create a partial index and identify the situations where it helps and where it doesn't

## Prerequisites

- Module 13 — Indexes ([overview](../13-indexes/00-module-overview.md)), in full — this topic assumes you already know what a B-tree index is and how `CREATE INDEX` works, and focuses entirely on the planner's *decision* of whether to use one.
- [How the Query Planner Works](01-how-the-query-planner-works.md) — you need the cost model and statistics concepts from that topic; selectivity is really just a specific, named application of the row-count estimates discussed there.

## Motivation

Almost everyone who learns about indexes eventually hits the same confusing moment: they add an index on exactly the column their slow query filters on, run the query again, and the planner still uses a sequential scan. This isn't a bug, and it isn't the planner being obstinate — it's the planner correctly recognizing that, for this particular column and this particular condition, the index would actually make things *slower*. Understanding why requires the concept this topic is built around: selectivity.

## Problem Statement

A `customers` table has 50,000 rows and a `status` column with only two possible values, `'active'` and `'inactive'`, where 48,000 rows are `'active'` and 2,000 are `'inactive'`. An index is created:

```sql
CREATE INDEX idx_customers_status ON customers (status);
```

Querying for the *rare* value uses the index, exactly as expected:

```sql
EXPLAIN SELECT * FROM customers WHERE status = 'inactive';
```

```
 Index Scan using idx_customers_status on customers  (cost=0.29..85.71 rows=2000 width=64)
   Index Cond: (status = 'inactive'::text)
```

But querying for the *common* value ignores the index entirely, even though it exists and is perfectly applicable:

```sql
EXPLAIN SELECT * FROM customers WHERE status = 'active';
```

```
 Seq Scan on customers  (cost=0.00..1250.00 rows=48000 width=64)
   Filter: (status = 'active'::text)
```

Same table, same index, same column, same operator — a different value produces a different plan. The index wasn't "broken" for the second query. It was correctly avoided.

## Concept

### Selectivity, Defined

**Selectivity** is the fraction of a table's rows that satisfy a given condition — a number between 0 and 1 (or, described informally, "how rare a match is"). A condition matching 2,000 rows out of 50,000 has a selectivity of `2000 / 50000 = 0.04` (4% — highly selective, meaning it filters *out* the vast majority of rows). A condition matching 48,000 out of 50,000 has a selectivity of `48000 / 50000 = 0.96` (96% — very poorly selective, meaning it filters out almost nothing).

The planner estimates selectivity for every condition using the statistics from `ANALYZE` (most common values, histograms, distinct-value counts) described in Topic 1, then uses that estimate to decide which access path is cheaper.

### Why Low Selectivity Favors a Sequential Scan

An index scan works by: (1) walking the index to find matching entries, then (2) for each match, jumping to the actual table row the entry points to (unless the query is satisfied entirely by the index itself — see Module 13's coverage of index-only scans). Step 2 is typically a **random** disk access per matching row, because matching rows are scattered arbitrarily throughout the table's physical storage, not clustered together by `status` value.

If a condition matches 96% of the table, an index scan would need to perform *tens of thousands* of individual random-access row lookups — one per matching row — each one considerably more expensive per-page than reading the next page sequentially. A sequential scan, by contrast, reads every page of the table exactly once, in physical disk order, checking the condition on each row as it goes. For a low-selectivity condition (most rows match), reading almost the entire table anyway via cheap sequential access beats performing almost as many expensive random-access lookups. The break-even point isn't a fixed percentage — it depends on the relative costs (`random_page_cost` vs. `seq_page_cost` from Topic 1), the table's size, and how many rows fit per page — but the underlying reasoning is always this same trade-off.

### Small Tables: Selectivity Doesn't Even Matter

A second, simpler reason an index gets ignored: if a table is small enough that it fits in only a handful of disk pages, a sequential scan is already so cheap that no index could meaningfully beat it, regardless of selectivity.

```sql
CREATE TABLE departments (
    department_id SERIAL PRIMARY KEY,
    name TEXT
);
-- Only 12 rows total

CREATE INDEX idx_departments_name ON departments (name);

EXPLAIN SELECT * FROM departments WHERE name = 'Engineering';
```

```
 Seq Scan on departments  (cost=0.00..1.15 rows=1 width=36)
   Filter: (name = 'Engineering'::text)
```

Even though `name = 'Engineering'` matches only one row out of twelve — extremely selective — the planner still chooses a sequential scan, because scanning all 12 rows costs almost nothing, and the overhead of opening and traversing an index structure at all isn't worth paying for a table this small. This is not a mistake to "fix" — trying to force index usage on a tiny table would make things marginally *worse*, not better.

### Expression (Function-Based) Indexes

A standard index is built on a column's raw stored value. If a query instead filters on some *function* of a column, a plain index on that column cannot help, because the index stores the raw values, not the function's output:

```sql
-- A plain index on email cannot help this query, because the index
-- stores the raw, case-sensitive email values, not their lowercase form:
SELECT * FROM customers WHERE LOWER(email) = 'asha@example.com';
```

```
 Seq Scan on customers  (cost=0.00..1250.00 rows=1 width=64)
   Filter: (lower(email) = 'asha@example.com'::text)
```

PostgreSQL supports **expression indexes** — an index built not on a raw column, but on the *result of an expression* evaluated over that column:

```sql
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));

EXPLAIN SELECT * FROM customers WHERE LOWER(email) = 'asha@example.com';
```

```
 Index Scan using idx_customers_email_lower on customers  (cost=0.29..8.31 rows=1 width=64)
   Index Cond: (lower(email) = 'asha@example.com'::text)
```

PostgreSQL stores the precomputed `LOWER(email)` value for every row in the index itself, so a query using the *exact same expression* in its `WHERE` clause can use it directly. This is the deeper, general fix behind the "sargability" problem covered again in the next topic: an expression index lets you keep a function in your `WHERE` clause (when the function is genuinely necessary, such as case-insensitive matching) without giving up index usage — as long as the index expression and the query's expression match precisely.

### Partial Indexes

A **partial index** is an index built over only a subset of a table's rows, defined by a `WHERE` clause attached to the index itself (not to the query):

```sql
CREATE INDEX idx_orders_pending ON orders (order_date)
WHERE status = 'pending';
```

This index contains entries *only* for rows where `status = 'pending'`, entirely omitting the (likely much more numerous) `'completed'`, `'cancelled'`, and other rows. It helps precisely the queries whose `WHERE` clause is compatible with (implies) the index's own condition:

```sql
EXPLAIN SELECT * FROM orders
WHERE status = 'pending' AND order_date > '2026-07-01';
```

```
 Index Scan using idx_orders_pending on orders  (cost=0.29..12.40 rows=45 width=32)
   Index Cond: (order_date > '2026-07-01'::date)
```

Notice the plan doesn't even need a separate condition check for `status = 'pending'` in the output — it's implicitly guaranteed by which index was used, since every entry in that index already satisfies it. A query for `status = 'completed'` simply cannot use this index at all, because none of its entries would ever match; the planner would fall back to a full index (if one exists) or a sequential scan instead.

Partial indexes are especially valuable in exactly the low-selectivity scenario from the Problem Statement above, turned on its head: if `'pending'` orders are a *small* fraction of a large `orders` table (most orders eventually become `'completed'` or `'cancelled'`), a full index on `status` might get ignored for exactly the reasons already discussed — but a partial index scoped to just the rare, actively-relevant subset is small, cheap to maintain, and reliably selective for the exact queries an application actually runs against "still open" work.

| Situation | Best fit |
|---|---|
| Column has few distinct values, evenly spread, all frequently queried | Plain index rarely helps any of them — consider no index, or reconsider the query pattern |
| Column has a rare, frequently-filtered-for value, alongside a common one rarely filtered for | Partial index scoped to the rare value |
| Queries always filter on a *transformed* version of a column (`LOWER(...)`, `date_trunc(...)`, etc.) | Expression index on that exact transformation |
| Queries combine both — rare status *and* a transformed column | A partial index whose indexed expression is itself a function, e.g. `CREATE INDEX ... ON orders (date_trunc('day', order_date)) WHERE status = 'pending'` |

## Internal Working (Deep Dive)

When the planner considers whether to use an index for a condition like `status = 'active'`, it performs roughly this sequence:

```
1. Look up statistics for `status` from pg_statistic:
     - Is 'active' in the Most Common Values list? If so, use its recorded frequency directly.
     - If not, estimate using the histogram and/or n_distinct.
2. Compute estimated selectivity = estimated matching rows / total rows
3. Estimate cost of a Seq Scan:  ≈ total_pages × seq_page_cost + total_rows × cpu_tuple_cost
4. Estimate cost of an Index Scan: ≈ matching_pages × random_page_cost
                                      + matching_rows × (cpu_index_tuple_cost + cpu_tuple_cost)
5. Compare the two (and any other candidate access paths) and pick the cheapest
```

Selectivity is the single input that swings step 5's outcome the most dramatically, because it directly scales how many "matching pages/rows" the index scan estimate in step 4 has to account for — a highly selective condition keeps that number tiny; a poorly selective one inflates it until the fixed per-page random-access penalty overwhelms any benefit of not reading the whole table.

## Real-World Analogy

Think of an index like a book's alphabetical subject index at the back, versus just flipping through every page. If you're looking for a topic mentioned on only two pages out of four hundred, the index is an enormous time-saver — you look up one entry, flip directly to page 217, and you're done. But if you're looking for a topic mentioned on 380 of those 400 pages (say, a word so common it appears on nearly every page), consulting the index and flipping back and forth 380 separate times is absurd — you'd finish faster by just reading the book start to finish once. A sensible reader (the planner) picks whichever approach actually saves time for *this specific* topic (condition) in *this specific* book (table) — not the same approach every time regardless of how common the topic is.

## Why Selectivity-Aware Index Usage Was Designed This Way

An index is not free — it costs disk space, it costs write overhead to maintain (Topic 5 covers this cost in depth), and using one at query time is not automatically cheaper than not using one. A planner that mechanically used *any* available index whenever a matching column appeared in a `WHERE` clause, regardless of how many rows it actually filtered out, would frequently make queries *slower*, not faster — precisely the low-selectivity case in the Problem Statement. Cost-based, statistics-driven selectivity estimation exists specifically so the database can make this judgment call quantitatively, per query, per table, per actual data distribution — rather than following a blind rule that happens to be right often enough to seem trustworthy, but wrong exactly when it matters most (a very large table with a poorly selective condition).

## Advantages

- **Prevents self-inflicted slowdowns** — without selectivity-aware reasoning, a database could easily choose an index path that performs *worse* than simply reading the table, especially at scale.
- **Expression indexes extend indexing to computed values** — any deterministic expression, not just a raw column, can be indexed, closing an entire category of "the value I filter on isn't literally stored anywhere" problems.
- **Partial indexes shrink both storage and write overhead** — indexing only the rows that matter for a specific access pattern avoids paying an index-maintenance cost for rows nobody ever queries that way.

## Disadvantages / Limitations

- **Selectivity estimates can be wrong for correlated conditions** — a `WHERE` clause combining two columns whose values are related in the real world (e.g., `city = 'Austin' AND state = 'TX'`) may be estimated as far rarer than it actually is, because standard statistics assume independence between columns unless extended statistics are explicitly created.
- **Partial indexes only help queries whose condition matches (or is implied by) the index's own `WHERE` clause** — a partial index is not a general-purpose substitute for a full index; a differently-filtered query gets no benefit from it at all.
- **Expression indexes require an exact expression match** — `WHERE LOWER(email) = ...` benefits from an index on `LOWER(email)`, but a slightly different expression, such as `WHERE email ILIKE ...`, does not automatically benefit from the same index.

## Best Practices

- Before creating an index to speed up a specific `WHERE` condition, ask how selective that condition actually is on the real data — an index on a column where the target value covers more than roughly 5-10% of rows is often not worth creating for that condition alone.
- Reach for a partial index whenever a query pattern consistently targets a small, well-defined subset of a much larger table (open orders, unprocessed jobs, active users), rather than indexing the whole column.
- Reach for an expression index whenever a query condition wraps an indexed column in a deterministic function (case-normalization, date truncation, extracting part of a string) rather than trying to avoid the function in the query — the next topic discusses avoiding functions on indexed columns where the wrapping isn't actually necessary at all.
- Always verify a new index actually gets chosen with `EXPLAIN`, on realistically-sized data, rather than assuming its mere existence guarantees usage.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "The index isn't being used, so the index (or the database) is broken." | The planner rejecting an index for a poorly selective condition or a tiny table is a correct, cost-based decision, not a malfunction. |
| Creating a plain index on a column that's always queried through a function, e.g. `LOWER(col)` | A plain index stores raw values and cannot be used for a query filtering on a function of those values — an expression index on that exact function is required. |
| Assuming a partial index helps any query that touches the indexed column | A partial index only helps queries whose `WHERE` clause matches or logically implies the index's own `WHERE` clause; any other query on that column gets no benefit from it. |
| Testing index usage only on a small development database | Selectivity and table size both drive the planner's decision — a plan that (correctly) ignores an index on a 500-row dev table can (correctly) use that same index heavily once the table has 5 million rows in production. |

## Interview Questions

1. **Q: You add an index on a boolean column that's `true` for 90% of rows and `false` for 10%. A query filters `WHERE is_active = true`. Will the index be used? Why or why not?**
   A: Almost certainly not for that specific query. `true` is a poorly selective value here (90% of rows match), so an index scan would require a huge number of random-access row lookups, which the planner correctly estimates as more expensive than a single sequential scan. The same index would very likely be used for `WHERE is_active = false`, since that condition is far more selective (10% of rows).

2. **Q: What is a partial index, and give a realistic scenario where it clearly outperforms a full index on the same column.**
   A: A partial index is an index built over only the rows matching a `WHERE` condition defined on the index itself. It clearly outperforms a full index when a query pattern consistently targets a small, well-defined, frequently-filtered subset of a much larger table — for example, indexing only `WHERE status = 'pending'` on an `orders` table where the vast majority of historical orders are `'completed'`; the partial index stays small and highly selective for exactly the queries that matter, while a full index on `status` might be ignored entirely due to poor overall selectivity.

3. **Q: Why can't a standard index be used for a query like `WHERE UPPER(last_name) = 'SMITH'`, even if `last_name` is indexed?**
   A: A standard index stores the raw, unmodified column values. The query is filtering on the result of a function (`UPPER(last_name)`) applied to those values, which isn't what the index contains, so the planner cannot use it and falls back to a sequential scan with a filter. An expression index built specifically on `UPPER(last_name)` would resolve this, since it stores the precomputed function result as its indexed value.

## Summary

- **Selectivity** is the fraction of a table's rows a condition matches; the planner estimates it from `ANALYZE` statistics and uses it to decide whether an index scan or a sequential scan is cheaper.
- Low selectivity (a condition matching a large fraction of rows) and very small tables both routinely, and correctly, lead the planner to ignore an existing index in favor of a sequential scan.
- An **expression index** indexes the result of a function or expression rather than a raw column, enabling index usage for queries that filter on a transformed value.
- A **partial index** indexes only the subset of rows matching a `WHERE` condition on the index itself, and only benefits queries whose own condition matches or implies that same condition.
- The next topic, Query Rewriting Patterns, builds on this by showing concrete ways to reshape queries so the planner's decisions — including index usage — work in your favor more often.
