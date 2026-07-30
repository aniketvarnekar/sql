# HAVING vs. WHERE

## Learning Objectives

By the end of this section you should be able to:
- Explain why an aggregate result (like a `SUM` or `COUNT`) cannot be filtered using `WHERE`
- State SQL's logical processing order and use it to explain the `WHERE`/`HAVING` distinction from first principles, not as a memorized rule
- Write queries that correctly filter groups using `HAVING`
- Combine `WHERE` and `HAVING` in the same query, each doing the job only it can do

## Prerequisites

- [GROUP BY](02-group-by.md) — `HAVING` only makes sense in the context of groups produced by `GROUP BY`; you need to be comfortable with grouping and aggregate functions operating per group before filtering those groups is meaningful.

## Motivation

You now know how to produce a per-region or per-customer breakdown with `GROUP BY`. The very next question anyone asks of a breakdown is almost always "okay, but only show me the ones that matter" — regions above a revenue threshold, customers with more than a handful of orders, categories that underperformed. Your instinct, based on everything Module 7 taught about filtering with `WHERE`, will be to just add a `WHERE` condition on the aggregate. That instinct is wrong, and understanding precisely *why* it's wrong is one of the most valuable things you'll learn in this entire module — it comes up constantly in real work and is a near-guaranteed interview question.

## Problem Statement

Suppose you want only the regions whose total revenue exceeds $1,500. The natural-seeming attempt:

```sql
SELECT region, SUM(quantity * unit_price) AS total_revenue
FROM orders
WHERE SUM(quantity * unit_price) > 1500
GROUP BY region;
```

```
ERROR:  aggregate functions are not allowed in WHERE
LINE 3: WHERE SUM(quantity * unit_price) > 1500
              ^
```

This isn't an arbitrary restriction PostgreSQL invented to be difficult — it's a direct consequence of *when*, logically, each clause runs. `WHERE` runs before any grouping or aggregation happens at all. At the moment `WHERE` is evaluated, `SUM(quantity * unit_price)` for "the East region" doesn't exist yet as a value — the East region's rows haven't even been gathered into a group yet, let alone summed. You cannot filter on a number that hasn't been computed.

## Concept

### SQL's Logical Processing Order

SQL statements are *written* in one order (`SELECT`, `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`) but are not logically *processed* in that same order. The actual logical order — the order in which each clause's job conceptually happens, regardless of where it's typed — is:

```
1. FROM        — identify the source table(s)
2. WHERE       — filter individual rows, BEFORE any grouping happens
3. GROUP BY    — gather the surviving rows into groups
4. HAVING      — filter entire GROUPS, using aggregate values computed per group
5. SELECT      — compute the final output expressions (including aggregates) for each surviving row/group
6. ORDER BY    — sort the final result
7. LIMIT       — cut down to the requested number of rows
```

This single ordering explains the entire `WHERE`/`HAVING` distinction: **`WHERE` operates in step 2, before groups even exist. `HAVING` operates in step 4, after groups and their aggregate values already exist.** A condition that needs an aggregate value (`SUM(...) > 1500`) can only be evaluated in step 4 or later, because that's the earliest point at which the aggregate value has actually been computed — which is exactly why PostgreSQL refuses to let an aggregate function appear inside `WHERE` at all, rather than silently computing something meaningless.

### The Corrected Query, Using HAVING

```sql
SELECT region, SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region
HAVING SUM(quantity * unit_price) > 1500
ORDER BY region;
```

```
 region | total_revenue
--------+----------------
 East   |        3623.00
 South  |        1580.00
(2 rows)
```

North ($1,270.00) and West ($1,258.00) are excluded because their totals don't exceed $1,500 — but notice their rows were still fully computed (grouped and summed) before being filtered out. That's the mechanical difference in one sentence: `HAVING` filters *after* the aggregate values already exist, discarding whole groups whose aggregate doesn't satisfy the condition; `WHERE` filters individual rows before any of that computation has even started.

### Side-by-Side: The Same Table, Two Different Filters

To make the distinction concrete, compare filtering the same table with `WHERE` versus `HAVING`, applied to genuinely different things:

```sql
-- WHERE: keep only individual ROWS from a specific category, before any grouping
SELECT * FROM orders WHERE category = 'Electronics';
```

```
 order_id | customer | region | category    | quantity | unit_price | discount_code | order_date
----------+----------+--------+-------------+----------+------------+----------------+------------
        1 | Priya    | East   | Electronics |        2 |     250.00 | SPRING10       | 2024-01-05
        4 | Elena    | North  | Electronics |        1 |     800.00 |                | 2024-01-15
        6 | Marcus   | West   | Electronics |        3 |     150.00 |                | 2024-01-22
        7 | Sofia    | East   | Electronics |        1 |     999.00 |                | 2024-01-27
       10 | Diego    | South  | Electronics |        4 |     220.00 | WELCOME5       | 2024-02-11
       15 | Priya    | East   | Electronics |        1 |     999.00 |                | 2024-03-03
       16 | Marcus   | West   | Electronics |        2 |     150.00 | WELCOME5       | 2024-03-08
(7 rows)
```

```sql
-- HAVING: keep only GROUPS (customers) whose order COUNT exceeds a threshold, after grouping
SELECT customer, COUNT(*) AS order_count
FROM orders
GROUP BY customer
HAVING COUNT(*) > 3
ORDER BY customer;
```

```
 customer | order_count
----------+---------------
 Marcus   |             4
 Priya    |             4
(2 rows)
```

The first query never groups anything — it's a row-level question ("which individual orders are Electronics?") answered entirely by `WHERE`. The second query's condition (`COUNT(*) > 3`) is meaningless at the individual-row level — a single row doesn't have a "count" — it only makes sense once rows have been gathered into per-customer groups, which is exactly why it belongs in `HAVING`, evaluated after grouping.

### Using WHERE and HAVING Together

The two clauses are not mutually exclusive — most real reporting queries use both, each doing the job only it can do: `WHERE` narrows down the raw rows *before* the (potentially expensive) grouping and aggregation work happens, and `HAVING` then filters the resulting groups based on their computed aggregate:

```sql
SELECT region, SUM(quantity * unit_price) AS february_revenue
FROM orders
WHERE order_date >= '2024-02-01' AND order_date < '2024-03-01'
GROUP BY region
HAVING SUM(quantity * unit_price) > 300
ORDER BY region;
```

```
 region | february_revenue
--------+---------------------
 East   |             725.00
 North  |             470.00
 South  |             980.00
(3 rows)
```

Walking through this by the logical order above: `WHERE` first discards every row outside of February 2024 — only orders 8 through 14 survive, since every other order falls in January or March. `GROUP BY` then gathers the *remaining* seven rows by region: East (orders 9 and 12, totaling $725.00), North (orders 8 and 13, totaling $470.00), South (orders 10 and 14, totaling $980.00), and West (order 11 alone, totaling $108.00). Only then does `HAVING` discard any region whose February total doesn't exceed $300 — East, North, and South all clear that bar and are kept, while West's single $108.00 order falls short and is excluded. Notice that `WHERE` and `HAVING` are each doing a job the other could not: `WHERE` already removed eleven rows before grouping ever started, and `HAVING` then removed one more region *after* its total had been computed — a row-level exclusion and a group-level exclusion, working together in the same query.

## Internal Working (Preview)

The logical order above is not just a teaching convenience — it maps directly onto how the query planner treats each clause internally, and in one specific way that also matters for performance:

```
 Table's raw rows
        │
        ▼
   WHERE filter        ← runs FIRST, discarding non-matching rows early;
        │                 the query planner can often use an index here (Module 13)
        ▼
   GROUP BY bucketing   ← only the surviving rows are grouped — fewer rows
        │                  in means less work for this step
        ▼
   Aggregate computation ← SUM/COUNT/AVG/etc. computed per group
        │
        ▼
   HAVING filter        ← runs LAST, discarding entire groups whose
        │                  aggregate doesn't satisfy the condition
        ▼
   Remaining groups → SELECT projection → ORDER BY → LIMIT
```

This has a genuine efficiency consequence, not just a correctness one: because `WHERE` runs before grouping, every row it discards never has to be bucketed or aggregated at all — reducing the amount of data the (potentially expensive) grouping step has to process. `HAVING`, by contrast, can only discard a group *after* the full aggregate computation for that group has already been done. This is precisely why the best-practice advice (below) is to push as much filtering as possible into `WHERE`, reserving `HAVING` strictly for conditions that genuinely require an aggregate value.

## Real-World Analogy

Imagine a stadium with a security checkpoint and, separately, a stadium announcer. The **security checkpoint** (`WHERE`) checks each person individually, one at a time, *before* they're allowed inside and seated in their section — a person without a valid ticket never even gets counted as being in any section, because they never got past the gate. Once everyone who passed the checkpoint is seated in their assigned sections, the **stadium announcer** (`HAVING`) makes decisions based on facts about entire sections, not individuals — "we'll only announce sections with more than 500 attendees tonight." The announcer's rule is fundamentally about a group-level fact (a section's total headcount) that simply doesn't exist yet at the moment any single person is walking through the checkpoint — the headcount is only knowable after everyone admitted has already been seated and counted section by section.

## Why WHERE and HAVING Are Separate Clauses

The separation exists because the two clauses answer questions that are available at fundamentally different points in a query's logical execution. `WHERE` operates on facts already present in each raw row (Module 7 — Querying Basics), which are known before any grouping occurs. `HAVING` operates on facts that are *computed as a consequence of grouping* (Topic 2 of this module) — an aggregate value like a group's `SUM` or `COUNT` simply does not exist until the grouping and aggregation steps have already run. If SQL only had one filtering clause, it would either have to allow aggregate functions inside `WHERE` (which is logically incoherent, since groups don't exist yet at that stage) or force every row-level filter to be awkwardly re-expressed as a group-level one. Two clauses, each scoped to the stage of the pipeline where its condition actually makes sense, keeps both operations coherent and — as shown above — lets the query planner push the cheaper, earlier filter (`WHERE`) in front of the more expensive grouping work.

## Advantages

- **Correctness by construction** — because aggregate functions are disallowed in `WHERE`, PostgreSQL prevents you from ever writing a query that's logically incoherent (filtering on a value that doesn't exist yet), rather than silently producing a wrong result.
- **Performance** — filtering early with `WHERE` reduces the amount of data that ever reaches the (typically more expensive) grouping and aggregation stage.
- **Clear separation of row-level vs. group-level reasoning** — a reader of the query can immediately tell, just from which clause a condition sits in, whether it's about individual records or about a computed summary.

## Disadvantages / Limitations

- **Two clauses to remember instead of one** — beginners coming from the assumption that "filtering is filtering" have to learn that SQL genuinely distinguishes row-level and group-level filtering, which is an extra concept, not just extra syntax.
- **A condition that doesn't need an aggregate can still legally go in `HAVING`** (PostgreSQL allows it), which works but is wasteful — it filters after grouping/aggregation work has already been spent on rows that `WHERE` could have discarded earlier for free.
- **`HAVING` without `GROUP BY` is legal but easy to misread** — PostgreSQL treats the entire table as a single implicit group in that case (`HAVING COUNT(*) > 10` with no `GROUP BY` either returns the one aggregate row or nothing at all), which is a valid but unusual edge case worth being aware of rather than assuming is a mistake.

## Best Practices

- Put every condition that only needs a raw column's value in `WHERE`, even if the query also has a `GROUP BY`/`HAVING` — filter as early and as cheaply as possible.
- Reserve `HAVING` strictly for conditions that genuinely require an aggregate function's result (a `SUM`, `COUNT`, `AVG`, etc., computed per group).
- When a query needs both a row-level condition and a group-level condition (as in the February revenue example above), use both clauses together rather than trying to force one clause to do both jobs.
- When a `HAVING` condition's result surprises you, trace through the logical order by hand — compute what `WHERE` actually admits first, then what each group's aggregate value actually is — rather than assuming the threshold's effect at a glance, exactly as the worked example above demonstrates is necessary.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `WHERE SUM(column) > value` | Aggregate functions are not permitted in `WHERE`, because `WHERE` is evaluated before groups (and therefore aggregate values) exist. PostgreSQL raises an explicit error rather than attempting to evaluate it. |
| Using `HAVING` for a condition that doesn't involve any aggregate function (e.g., `HAVING region = 'East'`) | This is legal SQL and will run, but it's evaluated after grouping when it could have been evaluated as a row-level `WHERE` condition before grouping, discarding non-matching rows earlier and cheaper. |
| Assuming `HAVING` requires `GROUP BY` to be present | PostgreSQL allows `HAVING` with no `GROUP BY`, treating the entire result set as one implicit group — unusual, but valid, and worth recognizing rather than assuming is always an error. |
| Forgetting that a column filtered out by `WHERE` is unavailable to any later `GROUP BY`/`HAVING` step | Since `WHERE` runs first in the logical order, any row it excludes is permanently gone by the time grouping happens — there's no way for `HAVING` to "reconsider" a row `WHERE` already discarded. |

## Interview Questions

1. **Q: What is the fundamental difference between `WHERE` and `HAVING`?**
   A: `WHERE` filters individual rows before any grouping or aggregation occurs. `HAVING` filters entire groups after grouping and aggregation have already produced a computed value (like a `SUM` or `COUNT`) for each group. `WHERE` cannot reference an aggregate function's result because that result doesn't exist yet at the point `WHERE` is evaluated.

2. **Q: Describe SQL's logical processing order for a query with `GROUP BY` and `HAVING`.**
   A: `FROM` (identify source rows), `WHERE` (filter individual rows), `GROUP BY` (gather surviving rows into groups), `HAVING` (filter entire groups using aggregate values), `SELECT` (compute final output expressions), `ORDER BY` (sort), `LIMIT` (cut down row count). This is the order clauses are logically processed, which differs from the order they're textually written in a query.

3. **Q: Can `HAVING` be used without `GROUP BY`? What does it mean in that case?**
   A: Yes. Without `GROUP BY`, PostgreSQL treats the entire table as a single implicit group, so `HAVING` filters that one aggregate row — either it satisfies the condition and is returned, or it doesn't and the query returns zero rows.

4. **Q: Why is it more efficient to put a row-level condition in `WHERE` rather than `HAVING`, even though `HAVING` would technically also work?**
   A: `WHERE` discards non-matching rows before the (often expensive) grouping and aggregation work happens, so fewer rows need to be bucketed and summed. `HAVING` only runs after that work is already done, meaning a row-level condition placed there wastes computation on rows that could have been eliminated earlier for free.

## Summary

- `WHERE` filters individual rows before grouping and aggregation happen; `HAVING` filters entire groups after their aggregate values have already been computed.
- Aggregate functions cannot appear in `WHERE` because, at the point `WHERE` runs (logical step 2), no groups — and therefore no aggregate values — exist yet.
- SQL's logical processing order (`FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`) explains this distinction directly, rather than requiring it to be memorized as an arbitrary rule.
- `WHERE` and `HAVING` are commonly used together: `WHERE` narrows the raw rows first (cheaply), and `HAVING` then filters the resulting groups based on their aggregate values.
- Prefer `WHERE` for any condition that doesn't require an aggregate, even inside a grouped query, since it filters earlier and more efficiently than `HAVING` would for the same row-level condition.
