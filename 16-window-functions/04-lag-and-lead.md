# LAG and LEAD

## Learning Objectives

By the end of this section you should be able to:
- Use `LAG` to read a previous row's value directly, within an ordered window, without a self-join
- Use `LEAD` to read the next row's value in the same way
- Correctly supply the offset and default-value arguments to both functions
- Write a query that compares each row to the prior period's value (e.g., month-over-month change)

## Prerequisites

- [The OVER Clause and PARTITION BY](01-the-over-clause-and-partition-by.md) — `LAG`/`LEAD` are window functions that require `PARTITION BY` and, critically, `ORDER BY` inside `OVER(...)` to have any defined meaning.
- Comfort with `NULL` handling in expressions (Module 3 — Data Types; Module 8 — Functions and Expressions) — the first row of any partition has no "previous" row, and understanding how that absence is represented (and how to override it) is central to this topic.

## Motivation

"How does this month compare to last month?" is one of the most common questions in any time-series report, and answering it row-by-row used to require joining a table to a shifted copy of itself — matching each row to "the row one period earlier" using date arithmetic in the join condition. It's fiddly, easy to get subtly wrong (off-by-one-period errors are extremely common), and computationally wasteful for something conceptually this simple. `LAG` and `LEAD` let you reach directly to a neighboring row's value, declared as part of the same flat query, with no join at all.

## Problem Statement

Take Deepa's four months of sales: 8000, 8000, 10000, 9000. You want, next to every month's amount, the *previous* month's amount, and the change between them — so a report reader can immediately see "sales went up 2000 this month" instead of having to mentally subtract themselves. Without a dedicated function for this, you'd have to self-join `monthly_sales` to itself, matching each row to the row exactly one month earlier for the same salesperson — an extra join, an extra date-arithmetic condition, and a real chance of subtly mismatching rows if the underlying data ever has a gap (a missing month).

## Concept

### `LAG` — Reading the Previous Row

`LAG(expression, offset, default)` returns the value of `expression` from the row `offset` positions *before* the current row, within the current window (partition + order). If there's no such row (e.g., you're on the first row of the partition), it returns `default` — or `NULL` if no default is given.

```sql
SELECT
    salesperson,
    month,
    amount,
    LAG(amount) OVER (PARTITION BY salesperson ORDER BY month) AS prev_month_amount
FROM monthly_sales
ORDER BY salesperson, month;
```

```
 salesperson |   month    | amount | prev_month_amount
-------------+------------+--------+---------------------
 Asha        | 2024-01-01 |  12000 |
 Asha        | 2024-02-01 |  15000 |              12000
 Asha        | 2024-03-01 |  11000 |              15000
 Asha        | 2024-04-01 |  18000 |              11000
 Ben         | 2024-01-01 |   9000 |
 Ben         | 2024-02-01 |   9500 |               9000
 Ben         | 2024-03-01 |  11000 |               9500
 Ben         | 2024-04-01 |  13000 |              11000
 Chen        | 2024-01-01 |  20000 |
 Chen        | 2024-02-01 |  17000 |              20000
 Chen        | 2024-03-01 |  19000 |              17000
 Chen        | 2024-04-01 |  21000 |              19000
 Deepa       | 2024-01-01 |   8000 |
 Deepa       | 2024-02-01 |   8000 |               8000
 Deepa       | 2024-03-01 |  10000 |               8000
 Deepa       | 2024-04-01 |   9000 |              10000
(16 rows)
```

Each salesperson's January row has a blank (`NULL`) `prev_month_amount` — there is no month before January in this dataset, so `LAG` correctly has nothing to return. `PARTITION BY salesperson` guarantees that the "previous row" for Ben's January never accidentally reaches back into Asha's December — each salesperson's sequence is entirely self-contained.

### Computing an Actual Month-over-Month Change

`LAG`'s real power shows up combined with ordinary arithmetic:

```sql
SELECT
    salesperson,
    month,
    amount,
    LAG(amount) OVER (PARTITION BY salesperson ORDER BY month) AS prev_month_amount,
    amount - LAG(amount) OVER (PARTITION BY salesperson ORDER BY month) AS month_over_month_change
FROM monthly_sales
ORDER BY salesperson, month;
```

```
 salesperson |   month    | amount | prev_month_amount | month_over_month_change
-------------+------------+--------+--------------------+--------------------------
 Asha        | 2024-01-01 |  12000 |                    |
 Asha        | 2024-02-01 |  15000 |              12000 |                    3000
 Asha        | 2024-03-01 |  11000 |              15000 |                   -4000
 Asha        | 2024-04-01 |  18000 |              11000 |                    7000
 Ben         | 2024-01-01 |   9000 |                    |
 Ben         | 2024-02-01 |   9500 |               9000 |                     500
 Ben         | 2024-03-01 |  11000 |               9500 |                    1500
 Ben         | 2024-04-01 |  13000 |              11000 |                    2000
 Chen        | 2024-01-01 |  20000 |                    |
 Chen        | 2024-02-01 |  17000 |              20000 |                   -3000
 Chen        | 2024-03-01 |  19000 |              17000 |                    2000
 Chen        | 2024-04-01 |  21000 |              19000 |                    2000
 Deepa       | 2024-01-01 |   8000 |                    |
 Deepa       | 2024-02-01 |   8000 |               8000 |                       0
 Deepa       | 2024-04-01 |   9000 |              10000 |                   -1000
 Deepa       | 2024-03-01 |  10000 |               8000 |                    2000
(16 rows)
```

Deepa's February shows a `month_over_month_change` of exactly `0` — flat sales, no growth, no decline — read directly off the same row it describes, with no separate lookup step. Notice too that Asha's March (-4000) and Chen's February (-3000) show real declines, right alongside the growth months — the arithmetic works identically whether the change is positive, negative, or zero.

### `LEAD` — Reading the Next Row

`LEAD` is `LAG`'s mirror image: it reads a value from `offset` rows *ahead* of the current row, instead of behind it.

```sql
SELECT
    salesperson,
    month,
    amount,
    LEAD(amount) OVER (PARTITION BY salesperson ORDER BY month) AS next_month_amount
FROM monthly_sales
ORDER BY salesperson, month;
```

```
 salesperson |   month    | amount | next_month_amount
-------------+------------+--------+---------------------
 Asha        | 2024-01-01 |  12000 |              15000
 Asha        | 2024-02-01 |  15000 |              11000
 Asha        | 2024-03-01 |  11000 |              18000
 Asha        | 2024-04-01 |  18000 |
 Ben         | 2024-01-01 |   9000 |               9500
 Ben         | 2024-02-01 |   9500 |              11000
 Ben         | 2024-03-01 |  11000 |              13000
 Ben         | 2024-04-01 |  13000 |
 Chen        | 2024-01-01 |  20000 |              17000
 Chen        | 2024-02-01 |  17000 |              19000
 Chen        | 2024-03-01 |  19000 |              21000
 Chen        | 2024-04-01 |  21000 |
 Deepa       | 2024-01-01 |   8000 |               8000
 Deepa       | 2024-02-01 |   8000 |              10000
 Deepa       | 2024-03-01 |  10000 |               9000
 Deepa       | 2024-04-01 |   9000 |
(16 rows)
```

Here it's each salesperson's **April** (the last month in the dataset) that has a blank `next_month_amount` — there is no month after April for `LEAD` to reach forward to.

### The `offset` Argument

Both functions default to an `offset` of `1` (the immediately adjacent row) if not specified, but accept any positive integer to reach further back or forward:

```sql
SELECT
    salesperson,
    month,
    amount,
    LAG(amount, 2) OVER (PARTITION BY salesperson ORDER BY month) AS amount_two_months_ago
FROM monthly_sales
WHERE salesperson = 'Asha'
ORDER BY month;
```

```
 salesperson |   month    | amount | amount_two_months_ago
-------------+------------+--------+-------------------------
 Asha        | 2024-01-01 |  12000 |
 Asha        | 2024-02-01 |  15000 |
 Asha        | 2024-03-01 |  11000 |                  12000
 Asha        | 2024-04-01 |  18000 |                  15000
```

With `offset = 2`, both January and February return `NULL` — neither has a row two positions before it within the partition.

### The `default` Argument

The third argument replaces the `NULL` that would otherwise appear when there's no row to look at — most commonly used to substitute a sensible default like `0` for a "change from previous period" calculation on the very first row:

```sql
SELECT
    salesperson,
    month,
    amount,
    LAG(amount, 1, 0) OVER (PARTITION BY salesperson ORDER BY month) AS prev_month_amount_or_zero
FROM monthly_sales
WHERE salesperson = 'Ben'
ORDER BY month;
```

```
 salesperson |   month    | amount | prev_month_amount_or_zero
-------------+------------+--------+------------------------------
 Ben         | 2024-01-01 |   9000 |                           0
 Ben         | 2024-02-01 |   9500 |                        9000
 Ben         | 2024-03-01 |  11000 |                        9500
 Ben         | 2024-04-01 |  13000 |                       11000
```

January now shows `0` instead of a blank — useful specifically when the *next* calculation (like a percentage change) would otherwise have to special-case a `NULL` first row separately.

## Internal Working (Preview)

`LAG` and `LEAD` require both `PARTITION BY` (to know which rows are "neighbors" at all) and `ORDER BY` (to know which direction is "before" and which is "after") — without an `ORDER BY`, "the previous row" is meaningless, since PostgreSQL makes no guarantee about row order absent one. Internally, once a partition's rows are sorted by the `OVER(...)`'s `ORDER BY`, evaluating `LAG`/`LEAD` is a straightforward positional lookup:

```
  sorted rows within a partition:
     [ row 1 ]  [ row 2 ]  [ row 3 ]  [ row 4 ]
        ▲          ▲          ▲          ▲
        │          │          │          │
   LAG(x,1)    LAG(x,1)   LAG(x,1)   LAG(x,1)
   for row 2   for row 3  for row 4  for row 5 (doesn't exist)
   reads row 1 reads row 2 reads row 3   → NULL / default
```

Unlike `SUM`/`AVG`/ranking functions, `LAG`/`LEAD` don't aggregate or accumulate anything across a range — they perform a single, fixed positional offset lookup per row, which makes them computationally cheap relative to frame-based calculations (covered in the next topic).

## Real-World Analogy

Think of reading a printed bank statement, where each line is one month's closing balance. To spot "did my balance go up or down this month," you naturally glance at the line *just above* the current one and compare — you don't recompute your balance from scratch, and you don't need to cross-reference a separate sheet. `LAG` is exactly that glance upward to the previous printed line; `LEAD` is glancing down at next month's line (if it's already been printed) to preview what's coming.

## Why LAG and LEAD Were Designed This Way

Before `LAG`/`LEAD` existed, comparing a row to its neighbor required a self-join: joining `monthly_sales` to a second copy of itself, on a condition like "same salesperson, and one month later" — expressed with explicit date arithmetic in the join condition. This works, but it's a join that doesn't represent any real relationship in the data (there's no foreign key connecting "this month's row" to "last month's row"); it's a workaround bent into the shape of a relationship purely to get positional access. `LAG`/`LEAD` give the query planner direct, positional access to a window's neighboring rows without needing to fabricate a join — a cleaner, declarative expression of "compare this row to its predecessor/successor," fully in keeping with the same "state what you want, not how to fetch it" philosophy from Module 01 that motivates every window function in this module.

## Advantages

- **No self-join required.** Row-to-neighbor comparisons collapse to a single `SELECT`, with no extra join condition to get subtly wrong.
- **Handles gaps and partition boundaries correctly.** `PARTITION BY` ensures Ben's January never accidentally "sees" Asha's December as its previous row; a naive self-join on date arithmetic alone is easy to get wrong at partition boundaries.
- **The `default` argument removes special-casing.** Providing a default value directly in the function call avoids a separate `COALESCE` or `CASE` step for handling the first/last row of a partition.

## Disadvantages / Limitations

- **Requires a meaningful `ORDER BY`.** If your data doesn't have a clear, well-defined sequence (e.g., no reliable date or sequence column), "previous row" is not a meaningful concept, and `LAG`/`LEAD` will happily return *some* row — just not necessarily the one you intended.
- **Fixed positional offset, not date-aware.** `LAG(amount, 1)` returns the previous *row*, not necessarily "one month ago" in the calendar sense — if a month is missing from the data entirely, `LAG` will silently skip over the gap and return the nearest actual previous row, which can be a subtle correctness issue for time-series data with missing periods.
- **Cannot look outside the current partition.** This is normally exactly what you want (as shown above), but it does mean a genuinely cross-partition "previous row" comparison isn't directly expressible with `LAG`/`LEAD` alone.

## Best Practices

- Always pair `LAG`/`LEAD` with an `ORDER BY` column that has one unambiguous meaning of "before" and "after" for your data — a date or sequence column, not something like `salesperson` alone.
- When your data can have missing periods (a month with no row at all), be explicit in your own mind (and documentation) that `LAG` returns the *previous existing row*, not necessarily "exactly one period back" — consider whether you need to generate a complete calendar sequence first if true period-over-period comparison (with gaps treated as zero, not skipped) is required.
- Use the `default` argument to avoid `NULL`-handling boilerplate downstream, especially when the very next calculation (a percentage change, a flag for "first appearance") would otherwise need extra `NULL` checks.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `LAG`/`LEAD` without `PARTITION BY` when the data genuinely has independent groups | Without `PARTITION BY`, the "previous row" for the first row of one salesperson's data could be the *last* row of a completely different salesperson, depending on sort order — always partition by the grouping column when one exists. |
| Assuming `LAG(amount, 1)` always means "exactly one calendar period ago" | It means "the previous *row* in this window's order" — if a period is missing from the underlying data, `LAG` silently returns the nearest earlier row that does exist, not `NULL` for the missing gap. |
| Forgetting that the first row of every partition will have `NULL` for `LAG` (and the last row `NULL` for `LEAD`), and not handling it downstream | Any arithmetic performed directly on a `LAG`/`LEAD` result (like subtraction) will itself become `NULL` for that row unless a `default` value is supplied or the `NULL` is explicitly handled. |
| Omitting `ORDER BY` inside `OVER(...)` entirely | `LAG`/`LEAD` require a defined sequence to know what "previous" and "next" mean — without `ORDER BY`, PostgreSQL raises an error, since there is no order to base the offset on. |

## Interview Questions

1. **Q: What problem do `LAG` and `LEAD` solve, and what was the common workaround before they existed?**
   A: They let a query directly access a previous or next row's value within an ordered window, without a separate join. Before they existed, the common workaround was a self-join — joining a table to a second copy of itself on a condition expressing "the row one period earlier/later" — which is more verbose and easier to get wrong at partition boundaries or data gaps.

2. **Q: What does the third argument to `LAG` do?**
   A: It supplies a default value to return when there is no row at the requested offset (for example, the first row of a partition, where `LAG(x, 1)` would otherwise return `NULL`) — commonly used to substitute `0` so that a downstream calculation (like a percentage change) doesn't need separate `NULL` handling.

3. **Q: Why must `LAG`/`LEAD` be used with an `ORDER BY` inside `OVER(...)`?**
   A: "Previous" and "next" only have meaning relative to a defined sequence. Without `ORDER BY`, there is no basis for determining which row comes before or after another, and PostgreSQL requires an explicit `ORDER BY` for these functions rather than guessing at an arbitrary order.

4. **Q: If a salesperson's data is missing March entirely, what will `LAG(amount, 1)` return for their April row?**
   A: It returns February's amount, not `NULL` and not a gap-aware "no data for March" indicator — `LAG` operates on row position within the window, not on calendar distance, so it silently skips over the missing month and reaches back to the nearest row that actually exists.

## Summary

- `LAG(expression, offset, default)` reads a value from a row `offset` positions before the current row in an ordered window; `LEAD` does the same, looking forward instead of backward.
- Both require `ORDER BY` inside `OVER(...)` to define what "before" and "after" mean, and should almost always be paired with `PARTITION BY` when the data has independent groups.
- The first row(s) of a partition have no valid `LAG` result (and the last row(s) have no valid `LEAD` result) unless a `default` value is supplied as the third argument.
- `LAG`/`LEAD` operate on row position within the window, not on calendar time — a missing period in the underlying data is silently skipped over, not treated as a gap.
- These functions eliminate the need for a self-join when comparing a row to its neighbor, keeping period-over-period comparisons declarative and free of join-condition bugs.
- The next topic builds on the same ordered-window foundation to compute running totals and moving averages, using explicit frame clauses.
