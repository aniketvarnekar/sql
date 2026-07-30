# ROW_NUMBER, RANK, and DENSE_RANK

## Learning Objectives

By the end of this section you should be able to:
- Explain what each of `ROW_NUMBER`, `RANK`, and `DENSE_RANK` computes
- Predict the exact output of all three functions on the same tied data, and explain why they differ
- Write a query that returns the top-N rows per group using a ranking window function
- Correctly filter on a ranking function's result by wrapping it in a subquery or CTE

## Prerequisites

- [The OVER Clause and PARTITION BY](01-the-over-clause-and-partition-by.md) — ranking functions are ordinary window functions; you need `OVER(...)` and `PARTITION BY` before their arguments make sense.
- Comfort with `ORDER BY` (Module 7 — Querying Basics) — every ranking function requires an `ORDER BY` inside its `OVER(...)` clause, since "rank" is meaningless without a defined order.

## Motivation

"Who's #1?" is one of the most common questions asked of any dataset — the top salesperson this month, the highest-scoring student, the most recent order per customer. Ranking rows *within* the entire table is easy with a plain `ORDER BY`. Ranking rows *within groups* — "who's #1 in each region," not just overall — is exactly the kind of question a window function was built to answer, and PostgreSQL gives you three closely related functions for it that differ in exactly one situation: when values tie.

## Problem Statement

Suppose you want to rank salespeople by their March 2024 sales figures, highest first. Two of your four salespeople — Asha and Ben — happened to sell the exact same amount that month: 11000 each. Now: is the answer "Chen is #1, Asha and Ben are tied for #2, and Deepa is #4" — or "Chen is #1, Asha and Ben are tied for #2, and Deepa is #3"? Both are defensible answers to "what rank is Deepa" depending on whether a tie "uses up" a rank position or not — and a plain sequential counter (assign 1, 2, 3, 4 with no regard for ties) gives yet a *third*, different answer. SQL provides a distinct function for each of these three interpretations, rather than forcing you to pick one behavior and work around it.

## Concept

### Setting Up the Tie

Using the `monthly_sales` table from the previous topic, filter to March 2024, where Asha and Ben both sold 11000:

```sql
SELECT salesperson, amount
FROM monthly_sales
WHERE month = '2024-03-01'
ORDER BY amount DESC;
```

```
 salesperson | amount
-------------+--------
 Chen        |  19000
 Asha        |  11000
 Ben         |  11000
 Deepa       |  10000
(4 rows)
```

Now rank these four rows using all three functions in the same query:

```sql
SELECT
    salesperson,
    amount,
    ROW_NUMBER() OVER (ORDER BY amount DESC, salesperson) AS row_number,
    RANK()       OVER (ORDER BY amount DESC, salesperson) AS rank,
    DENSE_RANK() OVER (ORDER BY amount DESC, salesperson) AS dense_rank
FROM monthly_sales
WHERE month = '2024-03-01'
ORDER BY amount DESC, salesperson;
```

```
 salesperson | amount | row_number | rank | dense_rank
-------------+--------+------------+------+------------
 Chen        |  19000 |          1 |    1 |          1
 Asha        |  11000 |          2 |    2 |          2
 Ben         |  11000 |          3 |    2 |          2
 Deepa       |  10000 |          4 |    4 |          3
(4 rows)
```

This single result set shows exactly how the three functions diverge, and only when a tie is present.

### `ROW_NUMBER()` — Always Unique, Never Ties

`ROW_NUMBER()` assigns a strictly sequential integer — 1, 2, 3, 4... — to every row in the window, in `ORDER BY` order, with **no regard for ties**. Even though Asha and Ben both sold 11000, `ROW_NUMBER` gives them different numbers (2 and 3), broken arbitrarily unless you add a deterministic tiebreaker column to `ORDER BY` (which the example above does, using `salesperson` as a secondary sort key — without it, PostgreSQL is free to give Asha and Ben either 2-then-3 or 3-then-2, and which one you get is not guaranteed to be stable across runs).

- **Never repeats a number.**
- **Always produces exactly 1, 2, 3, ..., N** with no gaps, for N rows in the window.

### `RANK()` — Ties Share a Rank, Then Skip

`RANK()` gives tied rows the *same* rank number, but the *next* distinct value's rank **skips ahead** by however many rows tied for the position before it. Here, Asha and Ben both tie for rank 2 — and Deepa, the next row down, gets rank **4**, not 3, because two rows (Asha and Ben) already occupied positions 2 and 3.

- **Ties get identical ranks.**
- **Gaps appear after a tie**, sized exactly to the number of tied rows.

### `DENSE_RANK()` — Ties Share a Rank, No Gaps

`DENSE_RANK()` also gives tied rows the same rank — but the next distinct value's rank always increments by exactly 1, regardless of how many rows tied before it. Deepa gets **dense_rank 3** here (not 4), because only two *distinct* amount values (19000, then 11000) preceded her 10000.

- **Ties get identical ranks**, exactly like `RANK()`.
- **No gaps ever** — the highest dense rank in a window always equals the number of *distinct* values, not the number of rows.

### Side-by-Side Summary

| Function | Ties get same value? | Skips ranks after a tie? | Highest value equals |
|---|---|---|---|
| `ROW_NUMBER()` | No — always unique | N/A (never ties) | Number of rows in the window |
| `RANK()` | Yes | Yes — by the tie's size | Can exceed the number of distinct values |
| `DENSE_RANK()` | Yes | No | Exactly the number of distinct values |

### The Top-N-Per-Group Pattern

The single most common practical use of a ranking window function is "give me the top N rows per group" — for example, each salesperson's two best months:

```sql
WITH ranked_months AS (
    SELECT
        salesperson,
        month,
        amount,
        ROW_NUMBER() OVER (PARTITION BY salesperson ORDER BY amount DESC) AS rn
    FROM monthly_sales
)
SELECT salesperson, month, amount
FROM ranked_months
WHERE rn <= 2
ORDER BY salesperson, rn;
```

```
 salesperson |   month    | amount
-------------+------------+--------
 Asha        | 2024-04-01 |  18000
 Asha        | 2024-02-01 |  15000
 Ben         | 2024-04-01 |  13000
 Ben         | 2024-03-01 |  11000
 Chen        | 2024-04-01 |  21000
 Chen        | 2024-01-01 |  20000
 Deepa       | 2024-03-01 |  10000
 Deepa       | 2024-04-01 |   9000
(8 rows)
```

Notice `PARTITION BY salesperson` here — the row number resets to 1 at the start of each salesperson's group, so "top 2" means top 2 *within that salesperson*, not top 2 overall. Also notice the `WITH ranked_months AS (...)` — a **common table expression**, previewed here and covered fully in Module 17 (CTEs and Recursion) — is what makes filtering on `rn` possible at all.

### Why You Can't Just Write `WHERE rn <= 2`

You might expect to write:

```sql
SELECT
    salesperson, month, amount,
    ROW_NUMBER() OVER (PARTITION BY salesperson ORDER BY amount DESC) AS rn
FROM monthly_sales
WHERE rn <= 2;   -- ERROR
```

This fails, because — as established in the previous topic — window functions are evaluated *after* `WHERE`. At the point `WHERE` runs, `rn` doesn't exist yet as a usable value, so PostgreSQL raises an error (`column "rn" does not exist`). The fix is always the same shape: compute the ranking window function in a subquery or CTE, then filter against its already-materialized result column in an outer query, exactly as the top-N example above does.

## Internal Working (Deep Dive)

For all three ranking functions, PostgreSQL performs the same first two steps described in the previous topic (partition, then sort within each partition by the `OVER(...)`'s `ORDER BY`), and then walks each partition's sorted rows in order, tracking two pieces of state:

```
   position := 0        (row count seen so far, this partition)
   rank_counter := 0     (last assigned rank/dense_rank value)
   previous_value := <none>

   for each row in partition, in ORDER BY order:
       position := position + 1

       ROW_NUMBER  := position                      (always increments)

       if current value != previous_value:
           RANK       := position                    (jumps to current position)
           DENSE_RANK := DENSE_RANK of previous row + 1   (always +1)
       else:
           RANK       := RANK of previous row         (repeat)
           DENSE_RANK := DENSE_RANK of previous row    (repeat)

       previous_value := current value
```

This is exactly why `RANK` "jumps" after a tie (it uses raw row `position`, which has already advanced past the tied rows) while `DENSE_RANK` tracks a separate counter that only advances on genuinely new values.

## Real-World Analogy

Think of three different ways a race's results could be posted:

- **`ROW_NUMBER`** is like assigning bib-based finish order when two runners cross the line at the exact same instant — someone has to be listed first on the sheet, even if the stopwatch says it's a dead heat. Every position on the sheet is unique, tie or not.
- **`RANK`** is a race's *official* medal ranking: if two runners tie for gold, both get "1st place," and the next runner is announced as coming in **3rd**, not 2nd — because two people already stood on the position ahead of them.
- **`DENSE_RANK`** is more like ranking runners by "which finish time bracket" they landed in: if two people tie for the fastest bracket, the next distinct bracket is still just the *second* bracket down — brackets don't skip numbers just because more than one runner landed in the top one.

## Why These Three Functions Were Designed This Way

A single "ranking" function couldn't satisfy every real use case, because "what happens on a tie" is a genuine, unavoidable design fork with three legitimate answers — and each answer is correct for different questions. "Give me exactly N distinct rows, no matter what" wants `ROW_NUMBER`. "Give me the *official* competitive standings, where ties matter" wants `RANK`. "Give me which of N distinct value-tiers each row falls into" wants `DENSE_RANK`. Rather than picking one behavior and forcing everyone to work around it with extra logic, the SQL standard (which introduced all three together, alongside window functions generally, in SQL:1999) provides all three as first-class functions, keeping each query's intent explicit and declarative rather than hiding a tie-breaking decision inside custom logic.

## Advantages

- **Precise, unambiguous semantics.** Each function has one well-defined behavior on ties — no need to hand-roll tie-breaking logic.
- **Directly solves "top N per group."** Combined with `PARTITION BY` and a wrapping subquery/CTE, this is a compact, declarative replacement for what would otherwise require a correlated subquery per group.
- **Composable with other window functions.** You can compute `ROW_NUMBER`, `RANK`, and other window functions in the same `SELECT`, over different or matching windows, as shown above.

## Disadvantages / Limitations

- **Cannot be filtered directly in `WHERE`/`HAVING`.** As shown above, this always requires an extra subquery or CTE layer — a genuine, unavoidable syntactic cost.
- **`ROW_NUMBER` without a fully deterministic `ORDER BY` gives unstable tie-breaking.** If your `ORDER BY` doesn't uniquely determine row order (as with Asha/Ben's tied 11000 without a secondary sort key), which row gets which number among ties is not guaranteed to be consistent across runs — often a source of confusing, hard-to-reproduce bugs.
- **`RANK`'s gaps can surprise consumers of a report** who expect ranks to run 1, 2, 3, 4 without skips — worth being deliberate about which of the three functions a stakeholder actually wants before shipping a report.

## Best Practices

- Always add enough columns to `ORDER BY` inside a ranking function's `OVER(...)` to make the order fully deterministic, especially when using `ROW_NUMBER` — ties on your primary sort column will otherwise be broken arbitrarily.
- Default to `DENSE_RANK` when you want a "tier" or "bracket" number that stakeholders will read as a count of distinct groups; default to `RANK` when you specifically need competition-style standings where ties visibly cost the next position its expected number.
- Use `ROW_NUMBER` (not `RANK` or `DENSE_RANK`) whenever you need *exactly* N rows per group, such as top-N-per-group queries — `RANK` can return more than N rows if there's a tie at the Nth position (e.g., `WHERE rank <= 2` could return three rows if two rows tie for rank 2).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `RANK()` for a "give me exactly the top 2 rows per group" query | If there's a tie at the cutoff rank, `RANK` will return more than 2 rows for that group (both tied rows share the rank, and both pass `<= 2`) — use `ROW_NUMBER` when the row count must be exact. |
| Assuming `RANK()` and `DENSE_RANK()` always produce identical output | They only diverge in the presence of ties — on data with no ties at all, both produce the exact same sequence as `ROW_NUMBER`, which is why the difference is easy to miss until real tied data shows up in production. |
| Omitting `ORDER BY` from a ranking function's `OVER(...)` clause | All three ranking functions require an `ORDER BY` inside `OVER(...)` — without one, PostgreSQL raises an error, since "rank" is undefined without a specified order. |
| Trying to filter a ranking function's output with `WHERE` in the same `SELECT` it's computed in | Window functions are evaluated after `WHERE` — the ranking column must be computed in a subquery/CTE first, then filtered in an outer query. |

## Interview Questions

1. **Q: Given two rows tied for 2nd place, what would `RANK()` and `DENSE_RANK()` assign to the row that comes next?**
   A: `RANK()` gives both tied rows a rank of 2, then assigns the next row rank 4 (skipping 3, since two rows already occupied positions 2 and 3). `DENSE_RANK()` also gives both tied rows a rank of 2, but assigns the next row rank 3 (no gap), since only two distinct values have appeared so far.

2. **Q: Why does `ROW_NUMBER()` never produce duplicate values, even with tied data?**
   A: `ROW_NUMBER()` assigns a strictly sequential position (1, 2, 3, ...) based purely on row order within the window, ignoring whether the `ORDER BY` values themselves are equal — every row gets a distinct number, with ties among the underlying data broken arbitrarily (or deterministically, if a secondary `ORDER BY` column is specified).

3. **Q: How would you write a query returning each department's top 3 highest-paid employees?**
   A: Compute `ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)` inside a CTE or subquery, aliasing it (e.g., `rn`), then in an outer query filter `WHERE rn <= 3`. `ROW_NUMBER` is preferred over `RANK` here so that exactly 3 rows are returned per department even if salaries tie.

4. **Q: Why can't a window function's result be referenced directly in the same query's `WHERE` clause?**
   A: SQL's logical processing order evaluates `WHERE` before window functions are computed, so the window function's output doesn't exist yet at that stage. It must first be computed and materialized (in a subquery or CTE), and only then can an outer query's `WHERE` filter on it.

## Summary

- `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` all assign a rank position based on an `ORDER BY` inside `OVER(...)`, and produce identical output when no ties exist — they only diverge on tied values.
- `ROW_NUMBER()` never ties — every row gets a unique, sequential number.
- `RANK()` gives tied rows equal ranks, then skips ahead by the size of the tie for the next distinct value.
- `DENSE_RANK()` gives tied rows equal ranks, with no gaps — its maximum value equals the count of distinct values, not the count of rows.
- The "top N per group" pattern combines `PARTITION BY` with one of these functions inside a subquery/CTE, then filters on the resulting rank column in an outer query — `ROW_NUMBER` is the right choice when the row count per group must be exact.
- The next topic, `NTILE` and distribution functions, builds on this same ranking machinery to bucket rows into groups (like quartiles) and measure relative standing.
