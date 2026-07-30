# Running Totals and Moving Averages

## Learning Objectives

By the end of this section you should be able to:
- Explain what a window **frame** is, and write an explicit `ROWS BETWEEN ... AND ...` clause
- Compute a running total using `UNBOUNDED PRECEDING` and `CURRENT ROW`
- Compute a bounded moving average using a fixed-size frame
- State PostgreSQL's default frame in both cases — when `ORDER BY` is present inside `OVER(...)`, and when it is absent — and explain why that default can silently change a query's result
- Explain the difference between a `ROWS` frame and a `RANGE` frame when tied values are present in the `ORDER BY` column

## Prerequisites

- [The OVER Clause and PARTITION BY](01-the-over-clause-and-partition-by.md) — this topic returns to, and fully explains, the "default frame" behavior only previewed there.
- [LAG and LEAD](04-lag-and-lead.md) — not strictly required, but this topic completes the same theme of "computing something across an ordered sequence of rows," using accumulation instead of positional lookup.

## Motivation

A running total — "how much has this salesperson sold, cumulatively, as of this month" — and a moving average — "what's the recent trend, smoothed over the last few periods" — are two of the most common calculations in any financial or performance report. Both require a window function to look at more than just the current row, but *less* than the whole partition: specifically, a *sliding slice* of nearby rows that changes as you move from row to row. That sliding slice is called a **frame**, and this topic is about controlling it precisely.

## Problem Statement

You want two new columns on the `monthly_sales` table: a running total of each salesperson's sales through the current month, and a 3-month moving average to smooth out any single unusually high or low month. A plain `SUM(amount) OVER (PARTITION BY salesperson)` (from Topic 1) gives the *same* grand total on every row — not a value that grows as the months progress. You need a way to tell PostgreSQL, precisely, "for this row, only sum the rows from the start of the partition through here" — and, separately, "for this row, only average the current row and the two before it." That precise control is exactly what a frame clause provides.

## Concept

### What a Frame Is

Every window function operates over a **frame** — the exact subset of the current partition's rows it reads, for a *given* row. A frame is always defined relative to the current row, and it can grow or shrink as you move row to row within the partition. The general syntax is:

```sql
<window_function> OVER (
    PARTITION BY <column(s)>
    ORDER BY <column(s)>
    <frame_mode> BETWEEN <frame_start> AND <frame_end>
)
```

`<frame_mode>` is either `ROWS` (count physical rows) or `RANGE` (count by logical value, aware of ties — covered later in this topic). `<frame_start>` and `<frame_end>` are typically one of:

| Frame boundary | Meaning |
|---|---|
| `UNBOUNDED PRECEDING` | All the way to the first row of the partition |
| `N PRECEDING` | N rows before the current row |
| `CURRENT ROW` | The current row itself |
| `N FOLLOWING` | N rows after the current row |
| `UNBOUNDED FOLLOWING` | All the way to the last row of the partition |

### Running Totals: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

A running total needs, for every row, the sum of every row from the very start of the partition through the current one — growing by exactly one row's worth each time you move forward:

```sql
SELECT
    salesperson,
    month,
    amount,
    SUM(amount) OVER (
        PARTITION BY salesperson
        ORDER BY month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM monthly_sales
ORDER BY salesperson, month;
```

```
 salesperson |   month    | amount | running_total
-------------+------------+--------+----------------
 Asha        | 2024-01-01 |  12000 |          12000
 Asha        | 2024-02-01 |  15000 |          27000
 Asha        | 2024-03-01 |  11000 |          38000
 Asha        | 2024-04-01 |  18000 |          56000
 Ben         | 2024-01-01 |   9000 |           9000
 Ben         | 2024-02-01 |   9500 |          18500
 Ben         | 2024-03-01 |  11000 |          29500
 Ben         | 2024-04-01 |  13000 |          42500
 Chen        | 2024-01-01 |  20000 |          20000
 Chen        | 2024-02-01 |  17000 |          37000
 Chen        | 2024-03-01 |  19000 |          56000
 Chen        | 2024-04-01 |  21000 |          77000
 Deepa       | 2024-01-01 |   8000 |           8000
 Deepa       | 2024-02-01 |   8000 |          16000
 Deepa       | 2024-03-01 |  10000 |          26000
 Deepa       | 2024-04-01 |   9000 |          35000
(16 rows)
```

Notice this is identical to the "preview" running total shown in Topic 1 — there, we relied on the *default* frame that kicks in once `ORDER BY` is present; here, the frame is written out explicitly. Both produce the same result on this data, but only one of them is unambiguous about what it's doing — more on that shortly.

### Moving Averages: A Bounded Frame

A moving average needs a frame that *doesn't* grow indefinitely — instead, it slides, always covering the current row plus a fixed number of rows before it. Here's a 3-month moving average (the current month plus the 2 preceding it):

```sql
SELECT
    salesperson,
    month,
    amount,
    ROUND(AVG(amount) OVER (
        PARTITION BY salesperson
        ORDER BY month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_3mo
FROM monthly_sales
ORDER BY salesperson, month;
```

```
 salesperson |   month    | amount | moving_avg_3mo
-------------+------------+--------+-----------------
 Asha        | 2024-01-01 |  12000 |        12000.00
 Asha        | 2024-02-01 |  15000 |        13500.00
 Asha        | 2024-03-01 |  11000 |        12666.67
 Asha        | 2024-04-01 |  18000 |        14666.67
 Ben         | 2024-01-01 |   9000 |         9000.00
 Ben         | 2024-02-01 |   9500 |         9250.00
 Ben         | 2024-03-01 |  11000 |         9833.33
 Ben         | 2024-04-01 |  13000 |        11166.67
 Chen        | 2024-01-01 |  20000 |        20000.00
 Chen        | 2024-02-01 |  17000 |        18500.00
 Chen        | 2024-03-01 |  19000 |        18666.67
 Chen        | 2024-04-01 |  21000 |        19000.00
 Deepa       | 2024-01-01 |   8000 |         8000.00
 Deepa       | 2024-02-01 |   8000 |         8000.00
 Deepa       | 2024-03-01 |  10000 |         8666.67
 Deepa       | 2024-04-01 |   9000 |         9000.00
(16 rows)
```

Walk through Asha's row for April: the frame is `2 PRECEDING AND CURRENT ROW`, so it covers February (15000), March (11000), and April (18000) — averaging to `(15000 + 11000 + 18000) / 3 = 14666.67`. January has dropped out of the window entirely by April, which is exactly what makes this a *moving* average rather than a running one. Notice, too, that January and February for every salesperson show a smaller average than 3 months' worth — the frame simply uses however many preceding rows actually exist (there's no error or `NULL` for having "fewer than 2 preceding rows"; PostgreSQL just uses what's available).

### The Default Frame — And Why It's Worth Knowing Precisely

PostgreSQL applies a default frame whenever you don't write one explicitly, and the default **depends on whether `ORDER BY` is present** inside `OVER(...)`:

| `OVER(...)` contains | Default frame |
|---|---|
| No `ORDER BY` | `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` (the entire partition) |
| `ORDER BY` present, no frame clause given | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (a running slice) |

This explains, precisely, the behavior first flagged in Topic 1: `SUM(amount) OVER (PARTITION BY salesperson)` (no `ORDER BY`) gives every row in a partition the *same* full-partition total, because its default frame is the whole partition. The moment you add `ORDER BY month` to that same call — `SUM(amount) OVER (PARTITION BY salesperson ORDER BY month)` — the default frame silently switches to a running slice, and the exact same `SUM` call turns into a running total. Nothing about the function or the partition changed; only the frame default did, triggered purely by the presence of `ORDER BY`.

### `ROWS` vs. `RANGE` — Why It Matters When Values Tie

The default frame uses `RANGE`, not `ROWS` — and these two frame modes behave identically *only* when the `ORDER BY` column has no ties. When ties exist, they diverge in an important way: `RANGE` treats all rows with an equal `ORDER BY` value as a single **peer group**, and gives every row in that peer group the *same* frame result; `ROWS` counts physical row positions, so tied rows can get *different* results depending on their arbitrary position in storage or sort order.

Recall that in the East region, Asha and Ben both sold exactly 11000 in March. Compare `ROWS` and `RANGE` cumulative sums over all East region rows, ordered by `amount`:

```sql
SELECT
    salesperson,
    amount,
    SUM(amount) OVER (
        ORDER BY amount, salesperson
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS rows_cumsum,
    SUM(amount) OVER (
        ORDER BY amount
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS range_cumsum
FROM monthly_sales
WHERE region = 'East'
ORDER BY amount, salesperson;
```

```
 salesperson | amount | rows_cumsum | range_cumsum
-------------+--------+-------------+---------------
 Ben         |   9000 |        9000 |          9000
 Ben         |   9500 |       18500 |         18500
 Asha        |  11000 |       29500 |         40500
 Ben         |  11000 |       40500 |         40500
 Asha        |  12000 |       52500 |         52500
 Ben         |  13000 |       65500 |         65500
 Asha        |  15000 |       80500 |         80500
 Asha        |  18000 |       98500 |         98500
(8 rows)
```

Look closely at the two tied 11000 rows. `rows_cumsum` gives Asha's row `29500` and Ben's row `40500` — two *different* values for two *identical* amounts, purely because `ROWS` counts them as separate physical positions. `range_cumsum` gives **both** tied rows `40500` — because `RANGE` treats "everything with amount ≤ 11000" as one peer group, and every row in that peer group sees the cumulative sum as of the *end* of the group, not partway through it. Whether you want the `ROWS` behavior or the `RANGE` behavior depends entirely on what the running total is supposed to mean when values tie — but you should never be *surprised* by which one you got, and relying on the implicit default is exactly how that surprise happens.

## Internal Working (Deep Dive)

For every row, once a partition's rows are sorted by `ORDER BY`, PostgreSQL determines that row's frame boundaries and evaluates the window function only over the rows falling inside them:

```
 Partition, sorted by ORDER BY:
   [ r1 ] [ r2 ] [ r3 ] [ r4 ] [ r5 ]
             │
             ▼  frame for r3 with ROWS BETWEEN 2 PRECEDING AND CURRENT ROW:
        ┌───────────────┐
        │ r1   r2   r3  │   ← only these three rows are summed/averaged for r3
        └───────────────┘

             ▼  frame for r3 with ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW:
   ┌─────────────────────┐
   │ r1   r2   r3        │   ← running total: everything from the start through r3
   └─────────────────────┘
```

For `ROWS`, this is a simple positional window that slides by exactly one row as you move to the next row — computationally cheap, since PostgreSQL can often maintain a running sum/count incrementally rather than re-summing the whole frame from scratch each time. For `RANGE`, the engine must additionally check each candidate row's `ORDER BY` value against the current row's value to determine peer-group membership, which is marginally more work and is precisely why tied values can end up sharing an identical frame result under `RANGE` but not under `ROWS`.

## Real-World Analogy

A running total is like a checkbook register: every new entry's balance is "everything before it, plus this one" — the running total only ever grows (or shrinks), never forgetting anything that came before.

A moving average is like a weather service reporting "the average high temperature over the last 7 days" each morning: yesterday's number is included, and the number from 8 days ago has just dropped out of the window — the reported average slides forward one day at a time, rather than accumulating every day since records began.

The `ROWS` vs. `RANGE` distinction is like the difference between "the 3 runners who crossed the finish line just before me" (a physical, position-based frame — `ROWS`) versus "everyone who finished within the same officially recorded time as me" (a value-based, tie-aware frame — `RANGE`) — two different, both legitimate, ways of answering "who's near me," which happen to give the same answer whenever nobody's times are tied.

## Why Frame Clauses Were Designed This Way

A single window function call needs to serve enormously different needs — a flat per-group total, a running total, and a bounded moving average are all, mechanically, "sum this expression over some subset of the partition" — the *only* thing that changes between them is which rows are included. Rather than inventing a separate function for each variant (a `RUNNING_SUM`, a `MOVING_AVG`, and so on), the SQL standard generalized the *subset itself* into an explicit, composable frame clause that any window function can be paired with. This keeps the feature declarative and orthogonal: you already know `SUM` and `AVG` from Module 9; the frame clause is a separate, reusable piece of syntax layered on top, rather than a proliferation of narrowly-named functions each hardcoding one specific frame shape.

## Advantages

- **One function, many frame shapes.** `SUM`, `AVG`, and every other aggregate function can be paired with any frame — flat total, running total, or moving window — without needing dedicated "running" or "moving" variants of each function.
- **Precise, explicit control.** Writing the frame out (`ROWS BETWEEN ... AND ...`) removes any ambiguity about exactly which rows contribute to each row's result — critical for financial and reporting calculations where "off by one row" is a real, consequential bug.
- **Efficient incremental computation.** For `ROWS`-based frames especially, the database can often compute a sliding sum/average incrementally rather than recomputing the entire frame's aggregate from scratch for every row.

## Disadvantages / Limitations

- **The default frame is easy to get wrong by omission.** As shown above, whether `ORDER BY` is present silently changes the default frame from "whole partition" to "running slice" — a query that looks like it should return a flat per-group total can quietly become a running total just because an `ORDER BY` was added for an unrelated reason (e.g., to make a tiebreaker deterministic).
- **`RANGE`'s peer-group behavior can be non-obvious to a report's readers.** Two rows with tied `ORDER BY` values showing an identical cumulative total, when they otherwise look like sequential entries, can look like a bug to someone unfamiliar with `RANGE` semantics — even though it's working exactly as specified.
- **Moving averages near partition boundaries use fewer rows than requested**, with no error or explicit flag — a 3-row moving average on the first row of a partition is really only averaging 1 row, which is correct behavior but easy to forget when interpreting results at the edges of a series.

## Best Practices

- Write `ROWS BETWEEN ... AND ...` explicitly whenever you want a running total or moving average — don't rely on the implicit default frame, even when it happens to produce the result you want today, because it silently depends on whether `ORDER BY` is present and can differ from `RANGE` the moment ties appear in your data.
- Reach for `RANGE` deliberately (not by accident via the default) only when you specifically want tied `ORDER BY` values to be treated as a single peer group with an identical result — otherwise prefer an explicit `ROWS` frame for predictable, position-based behavior.
- When computing a moving average near the start of a partition, remember the frame simply uses fewer rows rather than erroring — decide, based on your reporting requirements, whether an average based on fewer periods than intended should be flagged, suppressed, or left as-is.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `SUM(amount) OVER (PARTITION BY x)` and `SUM(amount) OVER (PARTITION BY x ORDER BY y)` always give the same result | The first uses the default full-partition frame (flat total repeated on every row); the second's default frame switches to a running slice purely because `ORDER BY` is present — these can produce very different numbers even though only an `ORDER BY` was added. |
| Assuming `ROWS` and `RANGE` are interchangeable | They only produce identical results when the `ORDER BY` column has no ties. With tied values, `RANGE` groups tied rows into one peer group with a shared result; `ROWS` treats each physical row separately, which can give tied rows different results. |
| Forgetting that `N PRECEDING` frames use fewer rows than requested near partition boundaries | A `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` frame on the second row of a partition only has 1 preceding row available, not 2 — it silently uses what exists rather than erroring or returning `NULL`. |
| Writing `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` when a running total "so far" was intended | This frame runs *forward* from the current row to the end of the partition — the reverse of a running total — producing a "remaining total" instead of a "cumulative-so-far total." Double-check the direction of `PRECEDING`/`FOLLOWING` against what the calculation is meant to represent. |

## Interview Questions

1. **Q: What is a window frame, and how does it differ from `PARTITION BY`?**
   A: `PARTITION BY` divides the result set into independent groups a window function operates within. A frame is a further, per-row subset *within* the current partition — the exact slice of rows (e.g., "from the start of the partition through the current row") that a specific row's calculation reads. `PARTITION BY` is fixed for every row in a group; the frame typically slides or grows as you move from row to row.

2. **Q: What is PostgreSQL's default frame when `ORDER BY` is present inside `OVER(...)` but no explicit frame clause is given?**
   A: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — a running slice from the start of the partition through the current row (or, with ties present, through the end of the current row's peer group).

3. **Q: How would you write a query for a trailing 7-row moving average of a daily metric?**
   A: `AVG(metric) OVER (ORDER BY date_column ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` — the frame covers the current row plus the 6 rows before it, for 7 rows total, and automatically uses fewer rows near the start of the data rather than erroring.

4. **Q: Two rows have an identical value in the `ORDER BY` column of a running-total window function. Under what frame mode would they get the same cumulative result, and under what mode might they differ?**
   A: Under `RANGE`, tied rows form a single peer group and receive the same cumulative result (computed through the end of that peer group). Under `ROWS`, tied rows are treated as distinct physical positions and can receive different cumulative results, since `ROWS` counts rows, not values.

## Summary

- A **frame** defines exactly which rows, within the current partition, a window function reads for a given row — expressed with `ROWS BETWEEN <start> AND <end>` (or `RANGE`).
- A running total uses `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`; a moving average uses a bounded frame like `ROWS BETWEEN N PRECEDING AND CURRENT ROW`.
- PostgreSQL's default frame is the entire partition when `OVER(...)` has no `ORDER BY`, and a running slice (`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`) when `ORDER BY` is present but no frame is written — a subtle, easy-to-miss behavior change triggered purely by adding `ORDER BY`.
- `ROWS` counts physical row positions; `RANGE` groups tied `ORDER BY` values into a shared peer group — the two only diverge when the `ORDER BY` column has ties, but when they do, the difference is significant.
- Writing frame clauses explicitly, rather than relying on the default, removes ambiguity and prevents an unrelated change (like adding `ORDER BY` for a tiebreaker) from silently altering a calculation's meaning.
- This closes out the module's core function set — the summary that follows consolidates every function covered and previews Module 17, which builds directly on the CTE pattern already used for top-N filtering in this module.
