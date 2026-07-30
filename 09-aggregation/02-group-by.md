# GROUP BY

## Learning Objectives

By the end of this section you should be able to:
- Explain what `GROUP BY` does to a result set before aggregate functions are applied
- Write queries that combine `GROUP BY` with `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX` to produce a per-group breakdown
- Group by more than one column and correctly interpret the resulting rows
- State and apply the rule that every non-aggregated column in `SELECT` must appear in `GROUP BY`

## Prerequisites

- [Aggregate Functions](01-aggregate-functions.md) — `GROUP BY` is meaningless without an aggregate function to apply per group; you need to already be comfortable with `SUM`, `COUNT`, `AVG`, `MIN`, and `MAX` and their `NULL`-handling before combining them with grouping.

## Motivation

Topic 1 gave you exactly one number for the entire `orders` table: one total revenue, one total order count. Real questions are almost never that flat. "What's our total revenue *by region*?" "How many orders did *each customer* place?" These are still aggregate questions — you still want `SUM` and `COUNT` — but you want one answer *per bucket*, not one answer for the whole table. `GROUP BY` is the clause that defines what a "bucket" is.

## Problem Statement

Suppose a sales director asks: "Break down our total revenue by region." Using only what Topic 1 taught you, your best option is something like this, run once per region:

```sql
SELECT SUM(quantity * unit_price) AS east_revenue FROM orders WHERE region = 'East';
SELECT SUM(quantity * unit_price) AS west_revenue FROM orders WHERE region = 'West';
SELECT SUM(quantity * unit_price) AS north_revenue FROM orders WHERE region = 'North';
SELECT SUM(quantity * unit_price) AS south_revenue FROM orders WHERE region = 'South';
```

This has real problems beyond just being tedious: you have to already know every distinct region value in advance to write these queries, the database has to scan the table four separate times, and if a fifth region is added tomorrow, your four hardcoded queries silently miss it entirely — with no error, no warning, just a wrong report. What you actually want is a single instruction: "group the rows by region, and give me the total for each group, whatever those groups turn out to be." That instruction is `GROUP BY`.

## Concept

### What `GROUP BY` Does

> `GROUP BY column_name` gathers all rows that share the same value in `column_name` into one **group**, and any aggregate function in the `SELECT` list is then computed **once per group** instead of once for the whole table.

```sql
SELECT region, SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region
ORDER BY region;
```

```
 region | total_revenue
--------+----------------
 East   |        3623.00
 North  |        1270.00
 South  |        1580.00
 West   |        1258.00
(4 rows)
```

Instead of one row for the entire table, you now get **one row per distinct value of `region`** — four rows, because there are four distinct regions in the data. Each row's `total_revenue` is the `SUM` computed using *only the rows belonging to that group*. Adding a `COUNT(*)` to the same query shows how many orders fed into each region's total:

```sql
SELECT
    region,
    COUNT(*)                    AS order_count,
    SUM(quantity * unit_price)  AS total_revenue,
    ROUND(AVG(quantity * unit_price), 2) AS avg_order_value
FROM orders
GROUP BY region
ORDER BY region;
```

```
 region | order_count | total_revenue | avg_order_value
--------+---------------+-----------------+-------------------
 East   |             6 |        3623.00 |           603.83
 North  |             3 |        1270.00 |           423.33
 South  |             3 |        1580.00 |           526.67
 West   |             4 |        1258.00 |           314.50
(4 rows)
```

East has 6 orders totaling $3,623.00, averaging $603.83 per order — and every other row is that same computation, independently, for its own group. One query, one scan, and it automatically adapts if a fifth region shows up tomorrow — no query rewrite required.

### Grouping by a Different Column Gives a Different Breakdown

The exact same shape of query, grouped by `category` instead of `region`, answers a completely different question:

```sql
SELECT
    category,
    COUNT(*)                   AS order_count,
    SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY category
ORDER BY category;
```

```
 category         | order_count | total_revenue
-------------------+---------------+-----------------
 Electronics       |             7 |        4928.00
 Furniture         |             5 |        2350.00
 Office Supplies   |             4 |         453.00
(3 rows)
```

`GROUP BY` doesn't care *what* the column means — it only cares that rows sharing an equal value belong together. The same table, the same aggregate expressions, grouped on a different column, produces a completely different (but equally valid) breakdown.

### Grouping by Multiple Columns

You can group by more than one column at once, separated by commas. This creates one group **per unique combination** of the listed columns' values — a finer-grained breakdown than either column alone:

```sql
SELECT
    region,
    category,
    COUNT(*)                   AS order_count,
    SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region, category
ORDER BY region, category;
```

```
 region | category         | order_count | total_revenue
--------+------------------+---------------+-----------------
 East   | Electronics      |             3 |        2498.00
 East   | Furniture        |             2 |        1000.00
 East   | Office Supplies  |             1 |         125.00
 North  | Electronics      |             1 |         800.00
 North  | Furniture        |             1 |         350.00
 North  | Office Supplies  |             1 |         120.00
 South  | Electronics      |             1 |         880.00
 South  | Furniture        |             1 |         600.00
 South  | Office Supplies  |             1 |         100.00
 West   | Electronics      |             2 |         750.00
 West   | Furniture        |             1 |         400.00
 West   | Office Supplies  |             1 |         108.00
(12 rows)
```

Each row now represents one *(region, category)* pair — East's total of $3,623.00 from the region-only breakdown has been split further into its three category components ($2,498.00 + $1,000.00 + $125.00 = $3,623.00, exactly matching). Grouping by more columns always produces a result that is at least as detailed (and usually has at least as many rows) as grouping by any single one of those columns alone — a fact Topic 4 builds on directly when it introduces `ROLLUP` and `CUBE` for generating several of these breakdowns, at different levels of detail, in one query.

### The Rule: Non-Aggregated Columns Must Appear in `GROUP BY`

This is the single most important mechanical rule in this topic, and the one that produces the most beginner errors. Consider this attempt:

```sql
SELECT region, customer, SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region;
```

```
ERROR:  column "orders.customer" must appear in the GROUP BY clause or be used in an aggregate function
LINE 1: SELECT region, customer, SUM(quantity * unit_price) AS tot...
                       ^
```

Think about *why* this must be an error, not just that it is one. The query groups by `region` alone, so each output row represents an entire region — potentially several different customers' orders folded together (East alone has orders from both Priya and Sofia). If PostgreSQL allowed `customer` in the `SELECT` list unchanged, which customer's name should it print for the East row — Priya's or Sofia's? There is no single correct answer, because `customer` isn't a single value within that group — it's several different values PostgreSQL cannot arbitrarily pick one of. The rule exists to prevent exactly that ambiguity: **every column named in `SELECT` must either be listed in `GROUP BY` (guaranteeing it's a single, consistent value within each group) or wrapped in an aggregate function (which explicitly tells PostgreSQL how to reduce multiple values to one).**

The fix is to either add `customer` to `GROUP BY` (which changes the meaning — now you're grouping by region *and* customer combined, producing more, finer-grained rows) or aggregate it, e.g. `COUNT(DISTINCT customer)` (how many different customers ordered from that region) — a different, but valid, question:

```sql
SELECT region, COUNT(DISTINCT customer) AS distinct_customers, SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region
ORDER BY region;
```

```
 region | distinct_customers | total_revenue
--------+-----------------------+-----------------
 East   |                     2 |        3623.00
 North  |                     1 |        1270.00
 South  |                     1 |        1580.00
 West   |                     1 |        1258.00
(4 rows)
```

This is now unambiguous — East had contributions from 2 distinct customers (Priya and Sofia), while the other three regions each had exactly 1.

### Ordering Grouped Results

`ORDER BY` works after grouping exactly as you'd expect, and can reference either a grouping column or an aggregate alias:

```sql
SELECT region, SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region
ORDER BY total_revenue DESC;
```

```
 region | total_revenue
--------+----------------
 East   |        3623.00
 South  |        1580.00
 North  |        1270.00
 West   |        1258.00
(4 rows)
```

This is an extremely common reporting pattern: "show me regions ranked from highest to lowest revenue."

## Internal Working (Preview)

Conceptually, PostgreSQL executes a grouped aggregate query in two logical stages, though the physical strategy the query planner picks can vary (explored fully in Module 20 — Performance Tuning):

```
 All qualifying rows (after WHERE, previewed in Topic 3)
        │
        ▼
 Bucket rows by GROUP BY key(s)
   ┌─────────┬─────────┬─────────┬─────────┐
   │  East    │  North   │  South   │  West    │   ← one bucket per distinct
   │ 6 rows   │ 3 rows   │ 3 rows   │ 4 rows   │     combination of GROUP BY values
   └─────────┴─────────┴─────────┴─────────┘
        │           │          │          │
        ▼           ▼          ▼          ▼
   aggregate    aggregate   aggregate  aggregate    ← SUM/COUNT/AVG computed
   East's rows  North's     South's    West's         independently per bucket
        │           │          │          │
        ▼           ▼          ▼          ▼
      1 output row per bucket, emitted as the final result set
```

PostgreSQL typically implements this bucketing with one of two strategies, visible if you run `EXPLAIN` on a grouped query (a tool covered properly in Module 13 and Module 20): a **`HashAggregate`**, which builds an in-memory hash table keyed by the `GROUP BY` value(s) and updates each bucket's running totals as rows stream in (efficient when there aren't too many distinct groups), or a **`GroupAggregate`**, which first sorts the rows by the `GROUP BY` key(s) so that all rows belonging to the same group become physically adjacent, then sweeps through once, closing out and emitting each group's aggregate as soon as the next different key value is seen. Which strategy the planner picks is invisible to your SQL — the *result* is identical either way, another instance of SQL's declarative "what, not how" principle from Module 1.

## Real-World Analogy

Picture a large pile of mail dropped on a desk, addressed to different departments within a company. `GROUP BY department` is the act of sorting that pile into labeled bins — one bin per department — before anyone looks inside any bin. Only *after* the sorting is complete does anyone start counting: "how many letters are in the Sales bin," "how many in Engineering." You would never try to answer "how many letters for Sales" while the mail is still an undivided pile — you sort into bins first, then count each bin independently. `GROUP BY` is the sorting step; the aggregate functions in `SELECT` are the counting-each-bin step, and both happen for every bin without you writing one instruction per bin.

## Why GROUP BY Was Designed This Way

The relational model (Module 2) treats a table as an unordered set of rows with no inherent grouping — grouping is something you *ask for*, not a built-in property of the data. `GROUP BY` exists as a distinct, explicit clause (rather than, say, an implicit behavior of aggregate functions) because "which column(s) define a group" is a business question with no universal default answer — sometimes you want groups by region, sometimes by category, sometimes by both together. Making it an explicit clause keeps the query fully declarative: you state the grouping key you want, and the database figures out the most efficient physical strategy (hash-based or sort-based) to actually produce those groups, exactly as previewed above.

## Advantages

- **One query replaces N queries** — a single `GROUP BY` query answers "break this down by X" regardless of how many distinct values X has, and automatically adapts as new values appear in the data.
- **Single scan of the data** — computing all groups' aggregates happens in one pass, rather than one pass per group as the naive per-value `WHERE` approach from the Problem Statement would require.
- **Composability** — grouping by multiple columns lets you move smoothly between coarse and fine-grained breakdowns of the same underlying data with a one-line change to the `GROUP BY` clause.

## Disadvantages / Limitations

- **Result cardinality grows quickly with more grouping columns** — grouping by two columns with many distinct values each can produce a very large number of groups (in the worst case, the product of each column's distinct value count), which can surprise someone expecting a small, tidy report.
- **Grouping by a high-cardinality column defeats the purpose of aggregation** — grouping by, say, `order_id` (already unique per row) produces one group per row, meaning every aggregate simply reflects that single row's own value; this is a common accidental mistake rather than a deliberate design choice.
- **The "must appear in GROUP BY or be aggregated" rule can feel restrictive to beginners** coming from procedural code, where printing "any value from this bucket" would compile without complaint — SQL's insistence on an unambiguous per-group value is stricter, and deliberately so.

## Best Practices

- List grouping columns explicitly in the `SELECT` clause (as every example above does) so the output is self-describing — a report with numbers but no labels identifying which group each number belongs to is far less useful.
- Before writing a `GROUP BY`, ask "what does one row of my result represent?" — if the answer is "one region," group by region; if it's "one region and category combination," group by both. Being explicit about this up front prevents the ambiguous-column error entirely.
- Watch for accidentally grouping by a near-unique column (like an ID or a timestamp with second-level precision) — if every group ends up with a count of 1, aggregation isn't doing anything useful and the grouping column is probably wrong.
- Combine `GROUP BY` with `ORDER BY` on an aggregate alias (e.g., `ORDER BY total_revenue DESC`) whenever the report's purpose is to rank groups, rather than leaving the reader to eyeball an unordered list.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Selecting a non-aggregated column that isn't listed in `GROUP BY` | PostgreSQL cannot pick a single representative value for that column within a group that may contain several different values for it — this raises a "must appear in the GROUP BY clause or be used in an aggregate function" error. |
| Assuming `GROUP BY` alone (with no aggregate function) does something different from `SELECT DISTINCT` | With no aggregate function in the `SELECT` list, `GROUP BY` on those same columns produces the same distinct combinations `DISTINCT` would — but relying on `GROUP BY` for this is a misuse of the clause; if you don't need any aggregate, `DISTINCT` (Module 7) states the intent more clearly. |
| Expecting `GROUP BY region, category` and `GROUP BY category, region` to produce differently *ordered* output | The grouping itself is the same set of groups either way (order of columns in `GROUP BY` doesn't change which rows are grouped together) — only `ORDER BY` controls the final row order, and it must be specified explicitly regardless of the `GROUP BY` column order. |
| Trying to filter on an aggregate value using `WHERE` inside a grouped query | `WHERE` filters individual rows *before* grouping and aggregation happen — at that point, no aggregate value exists yet to filter on. This exact problem, and its correct fix, is the entire subject of Topic 3. |

## Interview Questions

1. **Q: What does `GROUP BY` actually do to a query's rows, conceptually, before any aggregate function runs?**
   A: It partitions all the qualifying rows into groups, where every row in a group shares the same value(s) for the column(s) listed in `GROUP BY`. Aggregate functions in the `SELECT` list are then computed independently, once per group, rather than once for the whole result set.

2. **Q: Why can't you `SELECT` a plain, non-aggregated column that isn't included in the `GROUP BY` clause?**
   A: Because that column may hold multiple different values within a single group, and there's no unambiguous single value PostgreSQL could return for it. Any column that isn't part of the grouping key must be reduced to one value per group via an aggregate function (`COUNT`, `SUM`, `MIN`, `MAX`, etc.) instead.

3. **Q: If you group by two columns instead of one, does the result generally have more or fewer rows, and why?**
   A: Generally more (or equal, never fewer) — grouping by two columns creates one group per unique *combination* of both columns' values, which is a finer partition than grouping by either column alone; each single-column group gets split further based on the second column's values within it.

4. **Q: What's the difference between `GROUP BY column` with no aggregate function in `SELECT`, and `SELECT DISTINCT column`?**
   A: For that narrow case, they return the same set of distinct values. But `GROUP BY` is designed for aggregation and is the wrong tool to reach for when you only want distinct values with no aggregate computation — `SELECT DISTINCT` states that intent directly and is the more conventional choice.

## Summary

- `GROUP BY column` partitions rows into groups sharing that column's value; any aggregate function in `SELECT` is then computed once per group instead of once for the entire table.
- Grouping by multiple columns (`GROUP BY a, b`) creates one group per unique combination of those columns' values — a finer breakdown than grouping by either column alone.
- Every non-aggregated column in the `SELECT` list must appear in `GROUP BY`, because any column outside the grouping key could hold multiple different values within one group, with no unambiguous single value to return.
- `ORDER BY` after a grouped query can reference grouping columns or aggregate aliases, and is commonly used to rank groups by their aggregate value.
- `GROUP BY` cannot itself filter on an aggregate result (e.g., "only regions with revenue over $2,000") — that requires `HAVING`, covered next in Topic 3.
