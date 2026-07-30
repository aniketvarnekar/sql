# NTILE and Distribution Functions

## Learning Objectives

By the end of this section you should be able to:
- Use `NTILE(N)` to divide a window's rows into N roughly equal-sized buckets
- Explain why `NTILE` can split tied values across different buckets, unlike the ranking functions from the previous topic
- Use `PERCENT_RANK` and `CUME_DIST` to express a row's relative standing within its window as a fraction
- Choose correctly between `NTILE`, `PERCENT_RANK`, and `CUME_DIST` depending on whether you need discrete buckets or a continuous relative position

## Prerequisites

- [ROW_NUMBER, RANK, and DENSE_RANK](02-row-number-rank-and-dense-rank.md) — `NTILE`, `PERCENT_RANK`, and `CUME_DIST` are all ordering-based window functions built on the same `OVER (ORDER BY ...)` foundation, and it helps to already know how ties are handled by the ranking functions before seeing how differently `NTILE` treats them.
- [The OVER Clause and PARTITION BY](01-the-over-clause-and-partition-by.md) — for `PARTITION BY`, used here exactly as before.

## Motivation

"Rank" answers "where does this row stand, exactly, position by position." Often, though, you don't want that much precision — you want to know which *segment* a row falls into: the top 25%, the bottom 25%, or somewhere in the middle. Performance reviews sort employees into quartiles. Pricing tiers group customers by spend brackets. Standardized test results are reported as percentiles. All of these are a different kind of question than "what's the exact rank" — they're "which slice of the distribution does this fall in" — and PostgreSQL has a dedicated family of functions for exactly that.

## Problem Statement

Suppose you want to sort every sale record in your `monthly_sales` table into performance quartiles — the bottom 25% of sale amounts, the next 25%, and so on up to the top 25% — so you can flag which individual sales fell in the weakest bracket. `RANK` and `DENSE_RANK` give you an exact position or tier count, but neither one divides your data into a *fixed number* of equally sized groups by request — you'd have to manually compute cutoff values yourself and then classify each row against them. `NTILE` does exactly this division for you, declaratively, in one window function call.

## Concept

### `NTILE(N)` — Dividing Rows into N Buckets

`NTILE(N)` assigns every row in the window a bucket number from 1 to N, distributing rows as evenly as possible across the buckets, in `ORDER BY` order. Applied to all 16 rows of `monthly_sales`, split into 4 buckets (quartiles) by amount:

```sql
SELECT
    salesperson,
    month,
    amount,
    NTILE(4) OVER (ORDER BY amount, salesperson) AS quartile
FROM monthly_sales
ORDER BY quartile, amount, salesperson;
```

```
 salesperson |   month    | amount | quartile
-------------+------------+--------+----------
 Deepa       | 2024-01-01 |   8000 |        1
 Deepa       | 2024-02-01 |   8000 |        1
 Ben         | 2024-01-01 |   9000 |        1
 Deepa       | 2024-04-01 |   9000 |        1
 Ben         | 2024-02-01 |   9500 |        2
 Deepa       | 2024-03-01 |  10000 |        2
 Asha        | 2024-03-01 |  11000 |        2
 Ben         | 2024-03-01 |  11000 |        2
 Asha        | 2024-01-01 |  12000 |        3
 Ben         | 2024-04-01 |  13000 |        3
 Asha        | 2024-02-01 |  15000 |        3
 Chen        | 2024-02-01 |  17000 |        3
 Asha        | 2024-04-01 |  18000 |        4
 Chen        | 2024-03-01 |  19000 |        4
 Chen        | 2024-01-01 |  20000 |        4
 Chen        | 2024-04-01 |  21000 |        4
(16 rows)
```

Sixteen rows divide evenly into four buckets of exactly four rows each. Bucket 1 is the weakest quartile of individual sale amounts (8000–9000); bucket 4 is the strongest (18000–21000).

Notice the `ORDER BY amount, salesperson` inside `OVER(...)` — the secondary `salesperson` column is there for the same reason it mattered with `ROW_NUMBER` in the previous topic: `amount` alone doesn't uniquely order every row (several ties exist), and without a deterministic tiebreaker, PostgreSQL is free to place tied rows in either adjacent bucket inconsistently across runs.

### When Rows Don't Divide Evenly

Sixteen rows into four buckets divides perfectly, but real data rarely cooperates. If you instead asked for `NTILE(5)` on these same 16 rows (16 ÷ 5 = 3 remainder 1), PostgreSQL doesn't leave a leftover bucket — it distributes the remainder across the *earliest* buckets, one extra row at a time: the first `16 mod 5 = 1` bucket gets 4 rows, and the remaining 4 buckets get 3 rows each (4 + 3 + 3 + 3 + 3 = 16). This is a deliberate, documented rule — buckets never differ in size by more than one row — but it does mean you should not assume every `NTILE` bucket is exactly the same size unless your row count happens to divide evenly by N.

### `NTILE` Does Not Respect Ties

This is the single most important thing to understand about `NTILE`, and it's a genuine trap: `NTILE` buckets rows by **position** in the ordered sequence, not by **value**. Look closely at the result above: Ben's March 11000 landed in bucket 2, while Asha's March 11000 — the exact same amount — also landed in bucket 2 in this run, purely because the `salesperson` tiebreaker happened to keep them adjacent. Had the row directly before this tied pair been the boundary between buckets 1 and 2 instead, one 11000 row could easily have landed in a different bucket than another row with the identical amount. `NTILE` does not check whether tied rows deserve to be split — it only counts positions and cuts at fixed intervals. This is fundamentally different from `RANK`/`DENSE_RANK`, which always keep tied rows together with an identical rank. If your use case requires tied values to always land in the same bucket, `NTILE` is the wrong tool.

### `PERCENT_RANK` — Relative Standing as a Fraction, 0 to 1

`PERCENT_RANK` expresses a row's rank as a fraction between 0 and 1, computed as:

```
PERCENT_RANK = (rank - 1) / (total rows in window - 1)
```

where `rank` is the same tie-aware rank `RANK()` would produce. The lowest value in the window always gets `0`; the highest always gets `1`.

```sql
SELECT
    salesperson,
    month,
    amount,
    ROUND(PERCENT_RANK() OVER (ORDER BY amount)::numeric, 4) AS percent_rank,
    ROUND(CUME_DIST()    OVER (ORDER BY amount)::numeric, 4) AS cume_dist
FROM monthly_sales
ORDER BY amount, salesperson;
```

```
 salesperson |   month    | amount | percent_rank | cume_dist
-------------+------------+--------+--------------+-----------
 Deepa       | 2024-01-01 |   8000 |       0.0000 |    0.1250
 Deepa       | 2024-02-01 |   8000 |       0.0000 |    0.1250
 Ben         | 2024-01-01 |   9000 |       0.1333 |    0.2500
 Deepa       | 2024-04-01 |   9000 |       0.1333 |    0.2500
 Ben         | 2024-02-01 |   9500 |       0.2667 |    0.3125
 Deepa       | 2024-03-01 |  10000 |       0.3333 |    0.3750
 Asha        | 2024-03-01 |  11000 |       0.4000 |    0.5000
 Ben         | 2024-03-01 |  11000 |       0.4000 |    0.5000
 Asha        | 2024-01-01 |  12000 |       0.5333 |    0.5625
 Ben         | 2024-04-01 |  13000 |       0.6000 |    0.6250
 Asha        | 2024-02-01 |  15000 |       0.6667 |    0.6875
 Chen        | 2024-02-01 |  17000 |       0.7333 |    0.7500
 Asha        | 2024-04-01 |  18000 |       0.8000 |    0.8125
 Chen        | 2024-03-01 |  19000 |       0.8667 |    0.8750
 Chen        | 2024-01-01 |  20000 |       0.9333 |    0.9375
 Chen        | 2024-04-01 |  21000 |       1.0000 |    1.0000
(16 rows)
```

Both tied 8000 rows correctly share `percent_rank = 0.0000` (both are tied for the lowest rank), and both tied 11000 rows share `percent_rank = 0.4000` — **unlike `NTILE`, both `PERCENT_RANK` and `CUME_DIST` are peer-group aware**: tied values always receive identical results, because both functions are built directly on top of `RANK`-style tie handling, not on raw row position.

### `CUME_DIST` — Cumulative Distribution

`CUME_DIST` ("cumulative distribution") answers a subtly different question: "what fraction of all rows in the window have a value less than or equal to mine?"

```
CUME_DIST = (number of rows with value <= this row's value) / (total rows in window)
```

Both 8000 rows show `cume_dist = 0.1250` — 2 of the 16 rows have a value of 8000 or less, and 2/16 = 0.125. The highest value, 21000, always gets exactly `1.0000`, since every row's value is `<=` the maximum.

### `PERCENT_RANK` vs. `CUME_DIST` — the Difference

| | `PERCENT_RANK` | `CUME_DIST` |
|---|---|---|
| Formula | `(rank - 1) / (rows - 1)` | `(rows with value <= this row) / (total rows)` |
| Range | 0 (lowest) to 1 (highest) | Always `> 0`, up to and including 1 (the maximum value always gets exactly 1) |
| Denominator | `rows - 1` | `rows` |
| Common use | "This row is at the Xth percentile of the distribution" | "X% of all rows are at or below this row's value" |

They answer closely related but distinct questions, and for data with no ties near the extremes, their values will be close but rarely identical (compare Deepa's 8000 rows: `percent_rank = 0.0000` vs. `cume_dist = 0.1250` — the lowest value always gets a `percent_rank` of exactly 0, but its `cume_dist` reflects that other rows also share that minimum value).

## Internal Working (Preview)

`NTILE(N)` works by first counting the total number of rows in the partition, dividing by N to get a base bucket size, and distributing any remainder one extra row at a time to the earliest buckets — then walking the sorted rows in order, incrementing to the next bucket number each time the current bucket's row quota is filled:

```
 total_rows := count of rows in partition
 base_size  := total_rows / N          (integer division)
 remainder  := total_rows mod N        (extra rows to distribute)

 bucket := 1
 rows_placed_in_bucket := 0
 rows_in_this_bucket := base_size + (1 if bucket <= remainder else 0)

 for each row, in ORDER BY order:
     assign row to `bucket`
     rows_placed_in_bucket += 1
     if rows_placed_in_bucket == rows_in_this_bucket:
         bucket += 1
         rows_placed_in_bucket := 0
         rows_in_this_bucket := base_size + (1 if bucket <= remainder else 0)
```

This confirms, mechanically, why `NTILE` ignores ties: the loop advances purely by **row count**, with no check on whether the current row's value matches the previous row's value.

`PERCENT_RANK` and `CUME_DIST`, by contrast, are computed directly from the same tie-aware `RANK`-style counters described in the previous topic's Internal Working section, which is why they never split tied rows the way `NTILE` can.

## Real-World Analogy

`NTILE` is like splitting a class of 16 students into 4 groups of 4 for a group project, purely by walking down an alphabetized or scored list and taking 4 names at a time — if two students tied for 4th and 5th place on the sorting criteria, one lands in group 1 and the other in group 2, purely because of where the cut happened to fall, not because they deserved different groups.

`PERCENT_RANK` and `CUME_DIST`, by contrast, are like reporting a student's percentile on a standardized test: two students with the identical score are always reported at the identical percentile — the report would look inconsistent (and wrong) if two students with the same test score were told they scored at different percentiles.

## Why These Functions Were Designed This Way

`NTILE` and the percentile-style functions solve genuinely different problems, which is why they're not merged into one function. `NTILE` exists for cases where a **fixed number of equally-sized groups** is the actual business requirement — a report that must show exactly 4 quartile groups, a pricing model with exactly 5 customer tiers — where the practical need for evenly-sized buckets outweighs the theoretical inconsistency of occasionally splitting a tie. `PERCENT_RANK` and `CUME_DIST` exist for cases where **consistency for identical values** matters more than fixed group sizes — statistical and percentile-style reporting, where two identical inputs producing two different outputs would be considered a correctness bug, not a rounding artifact. Offering both, rather than one "do everything" ranking-distribution function, keeps each function's behavior precise and predictable for the specific question it's meant to answer — consistent with the same declarative, precisely-defined philosophy behind `RANK` vs. `DENSE_RANK` in the previous topic.

## Advantages

- **`NTILE` gives you a guaranteed, fixed number of groups** with no manual cutoff-value computation — extremely useful for quartile/quintile/decile-style reporting.
- **`PERCENT_RANK`/`CUME_DIST` give a continuous, comparable relative-position value** that's independent of the total row count's exact number — a `percent_rank` of 0.75 always means "75% of the way up the distribution," whether the window has 16 rows or 16 million.
- **All three build directly on the same `OVER (ORDER BY ...)` foundation** already learned in the previous two topics — no new clause syntax to learn, just new functions.

## Disadvantages / Limitations

- **`NTILE` can split tied values across adjacent buckets**, which can look inconsistent or "wrong" to a report's audience if they notice two identical values in different quartiles — a real limitation to disclose to stakeholders when adopting `NTILE` for any bucketing where ties are common.
- **`NTILE` bucket sizes can differ by one row** whenever the row count doesn't divide evenly by N — not usually a problem, but worth knowing before assuming every bucket is identically sized.
- **`PERCENT_RANK` is undefined behavior-wise for a single-row window** (division by `rows - 1` would be division by zero) — PostgreSQL defines it as `0` for a window with exactly one row, a special case worth being aware of if your partitions can be very small.

## Best Practices

- Use `NTILE` when the requirement is genuinely "exactly N groups" (quartiles, quintiles, deciles) and you can tolerate ties occasionally splitting across a boundary.
- Use `PERCENT_RANK` or `CUME_DIST` when consistency for tied values matters more than fixed group sizes, or when you want a value that's comparable across differently-sized datasets.
- Always add a deterministic secondary `ORDER BY` column to `NTILE`'s window when your primary sort column has realistic ties — otherwise which row lands in which bucket, right at a boundary, is not guaranteed to be stable across query runs.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `NTILE(4)` always keeps identical values in the same bucket | `NTILE` buckets by row position, not by value — tied rows can and do land in different buckets depending purely on where the row-count boundary falls. |
| Assuming every `NTILE` bucket is exactly the same size | Buckets only divide perfectly when the row count is an exact multiple of N; otherwise, the remainder is distributed one row at a time to the earliest buckets, making some buckets one row larger than others. |
| Confusing `PERCENT_RANK`'s denominator with `CUME_DIST`'s | `PERCENT_RANK` divides by `(rows - 1)`, so the minimum value always yields exactly `0`; `CUME_DIST` divides by the full row count and never yields `0` — the minimum value's `cume_dist` reflects however many rows share that minimum, divided by the total. |
| Using `NTILE(100)` on a small dataset expecting genuine "percentiles" | With fewer than 100 rows, many buckets will be empty or contain fewer than one row's worth of granularity — `NTILE(100)` on 16 rows just assigns each row (and a few duplicated bucket numbers) across 16 of the 100 buckets, which is rarely what "percentile" reporting actually intends; `PERCENT_RANK`/`CUME_DIST` are almost always the better fit for true percentile-style reporting on small-to-medium datasets. |

## Interview Questions

1. **Q: What does `NTILE(4)` do, and how does it differ from `RANK()`?**
   A: `NTILE(4)` divides the rows in a window into 4 groups as evenly as possible, assigning each row a bucket number from 1 to 4, based on row position in `ORDER BY` order. `RANK()` assigns a competition-style rank, where tied rows always share the same rank — `NTILE`, by contrast, buckets purely by position and can split tied values into different buckets.

2. **Q: If two rows have the identical value and NTILE assigns them to different buckets, is that a bug?**
   A: No — that's expected, documented behavior. `NTILE` distributes rows evenly by position, not by value, and does not check whether adjacent tied rows "deserve" to be in the same bucket. If that guarantee is required, `PERCENT_RANK` or `CUME_DIST` should be used instead, since both are tie-aware.

3. **Q: What is the practical difference between `PERCENT_RANK` and `CUME_DIST`?**
   A: `PERCENT_RANK` computes `(rank - 1) / (total rows - 1)`, always giving the lowest value exactly 0 and the highest exactly 1. `CUME_DIST` computes the fraction of rows with a value less than or equal to the current row's value, out of the total row count — it's always greater than 0, and the highest value always gets exactly 1. They're closely related but use different denominators and answer subtly different framing questions ("relative rank position" vs. "cumulative proportion at or below").

4. **Q: When would you choose `NTILE` over `PERCENT_RANK` for a reporting requirement?**
   A: When the actual business requirement is a fixed number of equally-sized groups — for example, "assign every customer to exactly one of 5 spend tiers" — `NTILE(5)` directly produces that. `PERCENT_RANK` gives a continuous relative-position value instead, which would need additional manual bucketing logic to turn into discrete tiers.

## Summary

- `NTILE(N)` divides a window's rows into N roughly equal-sized buckets, numbered 1 through N, based on row position — not on value.
- `NTILE` can split tied values across adjacent buckets, since it counts positions rather than checking for ties — a key difference from `RANK`/`DENSE_RANK`.
- `PERCENT_RANK` expresses a row's relative rank as a fraction from 0 (lowest) to 1 (highest), using `(rank - 1) / (rows - 1)`.
- `CUME_DIST` expresses the fraction of rows at or below the current row's value, using `(rows <= current) / (total rows)` — always greater than 0, with the maximum value always equal to 1.
- Both `PERCENT_RANK` and `CUME_DIST`, unlike `NTILE`, are peer-group aware — tied values always receive identical results.
- The next topic moves from ordering-and-bucketing functions to `LAG` and `LEAD` — reaching directly across rows to compare a row against its neighbors.
