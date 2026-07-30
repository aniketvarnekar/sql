# Aggregate Functions

## Learning Objectives

By the end of this section you should be able to:
- Explain what an aggregate function is and how it collapses many rows into a single value
- Use `COUNT(*)`, `COUNT(column)`, `SUM`, `AVG`, `MIN`, and `MAX` correctly
- State precisely how each aggregate function treats `NULL` values, and why `COUNT(*)` and `COUNT(column)` can return different numbers
- Use `COUNT(DISTINCT ...)` to count unique values rather than all matching rows

## Prerequisites

This is the first topic in Module 09, so there is no prior topic in this module to depend on. Beyond that, this topic assumes **Module 7 — Querying Basics** (writing a plain `SELECT ... FROM ... WHERE ...` query) and **Module 8 — Functions and Expressions** (writing an arithmetic expression like `quantity * unit_price`, which several examples below aggregate over).

## Motivation

Almost nobody who runs a business asks "show me every single order." They ask "how many orders did we get this month?", "what's our total revenue?", "what's our average order size?" Every one of those questions has the same shape: take a large number of rows and reduce them to one number. Writing that reduction by hand — pulling every row back to your application and looping over it to add things up — is slow, wastes network bandwidth, and throws away work the database is specifically built to do efficiently in one pass over the data. Aggregate functions are SQL's built-in answer to "give me one summary number computed from many rows."

## Problem Statement

Suppose you're asked: "How many orders have we received, and what's the total revenue across all of them?" With only the tools from Modules 7 and 8, your only option is to run `SELECT * FROM orders;`, pull back every row, and manually count them and add up the revenue yourself outside the database — in a spreadsheet, in your head, or in application code. That's already annoying with 16 rows. It becomes completely impractical with 16 million rows: you'd be dragging an enormous amount of data across the network just to compute a single number the database could have computed internally, without ever sending you a single row of detail. Aggregate functions let you ask for the *summary* directly, and let the database compute it internally, close to the data, without shipping every row to you first.

## Concept

### The Running Example: an `orders` Table

Every topic in this module reuses the same table, so results from one topic can be checked against the results of another. Create it and populate it now:

```sql
CREATE TABLE orders (
    order_id       SERIAL PRIMARY KEY,
    customer       TEXT NOT NULL,
    region         TEXT NOT NULL,
    category       TEXT NOT NULL,
    quantity       INTEGER NOT NULL,
    unit_price     NUMERIC(10,2) NOT NULL,
    discount_code  TEXT,
    order_date     DATE NOT NULL
);

INSERT INTO orders (customer, region, category, quantity, unit_price, discount_code, order_date) VALUES
    ('Priya',  'East',  'Electronics',      2, 250.00, 'SPRING10', '2024-01-05'),
    ('Marcus', 'West',  'Furniture',        1, 400.00, NULL,       '2024-01-08'),
    ('Priya',  'East',  'Office Supplies', 10,  12.50, NULL,       '2024-01-12'),
    ('Elena',  'North', 'Electronics',      1, 800.00, NULL,       '2024-01-15'),
    ('Diego',  'South', 'Furniture',        2, 300.00, 'WELCOME5', '2024-01-19'),
    ('Marcus', 'West',  'Electronics',      3, 150.00, NULL,       '2024-01-22'),
    ('Sofia',  'East',  'Electronics',      1, 999.00, NULL,       '2024-01-27'),
    ('Elena',  'North', 'Office Supplies',  8,  15.00, 'SPRING10', '2024-02-02'),
    ('Priya',  'East',  'Furniture',        1, 450.00, 'WELCOME5', '2024-02-06'),
    ('Diego',  'South', 'Electronics',      4, 220.00, 'WELCOME5', '2024-02-11'),
    ('Marcus', 'West',  'Office Supplies',  6,  18.00, NULL,       '2024-02-14'),
    ('Sofia',  'East',  'Furniture',        2, 275.00, NULL,       '2024-02-19'),
    ('Elena',  'North', 'Furniture',        1, 350.00, NULL,       '2024-02-23'),
    ('Diego',  'South', 'Office Supplies',  5,  20.00, 'SPRING10', '2024-02-27'),
    ('Priya',  'East',  'Electronics',      1, 999.00, NULL,       '2024-03-03'),
    ('Marcus', 'West',  'Electronics',      2, 150.00, 'WELCOME5', '2024-03-08');
```

`discount_code` is deliberately nullable — not every order used one — because it's the cleanest column in this table to demonstrate `NULL` handling with. Confirm the data with `SELECT * FROM orders ORDER BY order_id;`:

```
 order_id | customer | region | category         | quantity | unit_price | discount_code | order_date
----------+----------+--------+------------------+----------+------------+----------------+------------
        1 | Priya    | East   | Electronics      |        2 |     250.00 | SPRING10       | 2024-01-05
        2 | Marcus   | West   | Furniture        |        1 |     400.00 |                | 2024-01-08
        3 | Priya    | East   | Office Supplies  |       10 |      12.50 |                | 2024-01-12
        4 | Elena    | North  | Electronics      |        1 |     800.00 |                | 2024-01-15
        5 | Diego    | South  | Furniture        |        2 |     300.00 | WELCOME5       | 2024-01-19
        6 | Marcus   | West   | Electronics      |        3 |     150.00 |                | 2024-01-22
        7 | Sofia    | East   | Electronics      |        1 |     999.00 |                | 2024-01-27
        8 | Elena    | North  | Office Supplies  |        8 |      15.00 | SPRING10       | 2024-02-02
        9 | Priya    | East   | Furniture        |        1 |     450.00 | WELCOME5       | 2024-02-06
       10 | Diego    | South  | Electronics      |        4 |     220.00 | WELCOME5       | 2024-02-11
       11 | Marcus   | West   | Office Supplies  |        6 |      18.00 |                | 2024-02-14
       12 | Sofia    | East   | Furniture        |        2 |     275.00 |                | 2024-02-19
       13 | Elena    | North  | Furniture        |        1 |     350.00 |                | 2024-02-23
       14 | Diego    | South  | Office Supplies  |        5 |      20.00 | SPRING10       | 2024-02-27
       15 | Priya    | East   | Electronics      |        1 |     999.00 |                | 2024-03-03
       16 | Marcus   | West   | Electronics      |        2 |     150.00 | WELCOME5       | 2024-03-08
(16 rows)
```

### What an Aggregate Function Is

> An **aggregate function** takes a *set of rows* as input and returns a *single value* computed from them — it collapses many rows down to one.

This is fundamentally different from every function covered in Module 8. A function like `UPPER(name)` or `ROUND(price, 2)` operates **row by row** — it takes one row's value in, produces one value out, and the number of rows in your result set is unchanged. An aggregate function operates **across rows** — it takes many rows in, and produces exactly one value out for the whole set (or, as you'll see in Topic 2, one value per group of rows).

### `COUNT(*)` — Counting Rows

`COUNT(*)` counts the number of rows, full stop — it does not look at the contents of any column at all:

```sql
SELECT COUNT(*) AS total_orders FROM orders;
```

```
 total_orders
--------------
           16
(1 row)
```

Notice the shape of this result: sixteen rows of raw data went in, and exactly **one row** came out. That collapsing — many rows in, one row out — is the defining behavior of every aggregate function used without `GROUP BY` (Topic 2 shows how `GROUP BY` changes this to "one row per group" instead of "one row total").

### `COUNT(column)` — Counting Non-`NULL` Values

`COUNT(column)` looks different from `COUNT(*)` in one crucial way: it only counts rows where that specific column is **not** `NULL`.

```sql
SELECT
    COUNT(*)            AS total_orders,
    COUNT(discount_code) AS orders_with_discount
FROM orders;
```

```
 total_orders | orders_with_discount
--------------+-----------------------
           16 |                     7
(1 row)
```

`total_orders` is 16 because `COUNT(*)` counts every row regardless of content. `orders_with_discount` is 7 because only 7 of the 16 rows have a non-`NULL` `discount_code` — the other 9 rows placed no discount code, and `COUNT(discount_code)` silently skips those. This is the single most common source of confusion around `COUNT`: **`COUNT(*)` counts rows; `COUNT(column)` counts values present in that column.** They only agree when the column is `NOT NULL` for every row.

### `SUM`, `AVG`, `MIN`, `MAX`

These four aggregate functions all operate on a numeric (or, for `MIN`/`MAX`, any orderable) expression:

```sql
SELECT
    SUM(quantity * unit_price)          AS total_revenue,
    ROUND(AVG(quantity * unit_price), 2) AS average_order_value,
    MIN(quantity * unit_price)          AS smallest_order,
    MAX(quantity * unit_price)          AS largest_order
FROM orders;
```

```
 total_revenue | average_order_value | smallest_order | largest_order
---------------+----------------------+-----------------+----------------
       7731.00 |               483.19 |          100.00 |         999.00
(1 row)
```

A few things worth noticing:

- None of these are raw columns — `quantity * unit_price` is an **expression** (Module 8), computed per row first, and only then fed into the aggregate. Aggregate functions happily accept any expression that produces a value per row, not just a bare column name.
- `AVG` is wrapped in `ROUND(..., 2)`. Left unrounded, `AVG` on a `NUMERIC` column in PostgreSQL can return a value with many more decimal places than is useful to display (e.g. `483.187500000000000000`) — rounding for display is a habit worth forming from your very first `AVG`.
- `MIN` and `MAX` aren't limited to numbers — they work on dates, text (alphabetical order), and any other type PostgreSQL knows how to compare. `MIN(order_date)` would return the earliest date in the table; `MAX(customer)` would return the alphabetically last customer name.

### `NULL` Handling: the One Rule to Memorize

> Every aggregate function except `COUNT(*)` **ignores `NULL` values entirely** — as if the rows containing them were never there for that computation.

This matters most for `AVG`, because it's easy to assume `NULL` is silently treated as `0`. It is not:

```sql
SELECT
    AVG(quantity)                              AS avg_quantity,
    COUNT(*)                                   AS row_count,
    COUNT(quantity)                            AS non_null_quantity_count
FROM orders;
```

Every `quantity` value in this table happens to be filled in, so this particular query wouldn't show a difference — but imagine a table where 4 out of 16 orders had a `NULL` quantity (perhaps a quantity wasn't recorded yet). `AVG(quantity)` would divide the sum of the 12 known quantities by **12**, not by 16 — the 4 `NULL` rows are excluded from both the numerator and the denominator, not counted as zero. If you actually want `NULL` treated as `0` for an average, you must say so explicitly, typically with `COALESCE(quantity, 0)` (Module 8) inside the aggregate: `AVG(COALESCE(quantity, 0))`.

### `COUNT(DISTINCT ...)` — Counting Unique Values

Plain `COUNT(column)` counts every non-`NULL` occurrence, including repeats. `COUNT(DISTINCT column)` counts only the **distinct** non-`NULL` values:

```sql
SELECT
    COUNT(customer)          AS total_customer_mentions,
    COUNT(DISTINCT customer) AS distinct_customers,
    COUNT(DISTINCT region)   AS distinct_regions,
    COUNT(DISTINCT discount_code) AS distinct_discount_codes
FROM orders;
```

```
 total_customer_mentions | distinct_customers | distinct_regions | distinct_discount_codes
--------------------------+----------------------+-------------------+---------------------------
                       16 |                    5 |                4 |                        2
(1 row)
```

`total_customer_mentions` is 16 because every row has a customer name. `distinct_customers` is 5 because, despite 16 orders, there are only five unique customers placing them (Priya, Marcus, Elena, Diego, Sofia) — several customers ordered more than once. `distinct_discount_codes` is 2, counting only `SPRING10` and `WELCOME5` once each, and — consistent with the `NULL`-ignoring rule above — not counting the 9 `NULL` discount codes as a "third distinct value" at all.

### Combining Multiple Aggregates in One `SELECT`

Every example above already demonstrates this, but it's worth stating explicitly: you can compute as many aggregate values as you want in a single `SELECT` list, each over the same underlying rows, and they'll all collapse together into that same single output row. This is far more efficient than running five separate queries, since the database only needs to scan the table once to compute all five numbers simultaneously.

## Internal Working (Preview)

Conceptually, when PostgreSQL executes an aggregate query with no `GROUP BY`, it performs a single pass over every qualifying row (after `WHERE` has already filtered them, previewed in Topic 3), maintaining a small running accumulator per aggregate function:

```
 Row 1 ──▶ SUM accumulator += (row 1's expression value)
           COUNT accumulator += 1 (if not NULL, for COUNT(column))
           MIN/MAX accumulator updated if this row's value is more extreme
 Row 2 ──▶ ... same update, using Row 2's values ...
 Row 3 ──▶ ... and so on for every row ...
   │
   ▼
 After the last row: emit ONE row containing the final accumulator values
```

This is why aggregation is efficient even over huge tables: the database never needs to hold all the raw rows in memory at once to compute a `SUM` or `COUNT` — it only needs to remember a handful of running totals, updated incrementally as it streams through the rows once. `AVG` is typically implemented internally as exactly `SUM` and `COUNT` tracked together, divided at the very end — which is also why `AVG` inherits the same "`NULL`s are skipped, not zeroed" behavior as `SUM` and `COUNT`.

## Real-World Analogy

Think of an aggregate function like a cash register's end-of-day tally. The register doesn't hand the store manager a photocopy of every single receipt from the day so the manager can add them up by hand — it keeps a running total *as each sale happens* and, at closing time, simply reports one number: today's total revenue. `COUNT(*)` is "how many receipts were printed today" (every sale, no exceptions); `COUNT(discount_code)` is "how many of today's receipts had a coupon code applied" (skipping the ones that didn't use one) — the same drawer of receipts, counted two different, equally valid ways, depending on exactly what you're asking.

## Why Aggregate Functions Were Designed This Way

SQL is a declarative language (Module 1, [What Is SQL?](../01-introduction/02-what-is-sql.md)): you describe *what* result you want, not the step-by-step procedure to compute it. `SUM(quantity * unit_price)` is a direct expression of that philosophy — you're stating "I want the total," not writing a loop that initializes an accumulator to zero, iterates over rows, and adds to it. The database is free to compute that sum however is most efficient internally (a single sequential scan, a parallel scan across multiple CPU cores, or — once indexes are covered in Module 13 — sometimes even without touching the full row at all), and that freedom is only possible because you asked for the *result*, not the *procedure*. The `NULL`-skipping behavior follows directly from `NULL`'s meaning established in Module 3 — Data Types: `NULL` represents "unknown" or "absent," not zero, so folding an unknown value into a sum or average would silently fabricate data that was never actually recorded.

## Advantages

- **Massive reduction in data transferred** — the database returns one number (or a handful), not every underlying row, saving network bandwidth and client-side memory for large tables.
- **Single-pass efficiency** — one scan over the data computes as many aggregates as you ask for simultaneously, rather than one scan per aggregate.
- **Correct-by-default `NULL` semantics** — excluding unknown values from sums and averages (rather than silently treating them as zero) matches what "unknown" should mean, protecting you from quietly wrong reports.
- **Composable with everything else in `SELECT`** — aggregate expressions can use any function or arithmetic from Module 8 inside them, so you're never limited to aggregating a bare column.

## Disadvantages / Limitations

- **`NULL`-skipping can be misunderstood** — a beginner who expects `AVG` to treat missing values as zero will get a systematically wrong (typically inflated) average and may not notice, since the query runs without error.
- **`COUNT(*)` vs. `COUNT(column)` looks like a stylistic choice but isn't** — picking the wrong one silently changes what's being measured (all rows vs. rows with a value present), and both are syntactically valid, so there's no error to catch the mistake.
- **A single aggregate query gives you one number for the whole table** — to get a *breakdown* (revenue per region, count per customer) you need `GROUP BY`, covered next in Topic 2; aggregate functions alone can't express "one total per category."

## Best Practices

- Default to `COUNT(*)` when you mean "how many rows," and reach for `COUNT(column)` only when you specifically mean "how many rows have a value in this column" — don't use them interchangeably out of habit.
- Always wrap `AVG` (and often `SUM`) in `ROUND(..., N)` for a sensible number of decimal places when the result will be displayed to a human.
- When a column can be `NULL` and you specifically want `NULL` treated as zero for a `SUM` or `AVG`, be explicit about it with `COALESCE(column, 0)` inside the aggregate — never rely on assumption.
- Use `COUNT(DISTINCT column)` whenever the question is "how many *different* values," not "how many rows" — conflating the two is a common source of inflated-looking counts.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `COUNT(*)` and `COUNT(some_column)` always return the same number | They only agree if `some_column` is `NOT NULL` on every row. Any `NULL` in that column makes `COUNT(column)` smaller than `COUNT(*)`. |
| Assuming `AVG(column)` treats `NULL` rows as `0` | `AVG` (like `SUM`, `MIN`, `MAX`, and `COUNT(column)`) simply skips `NULL` values — it does not include them in the count used as the denominator, so the average is computed only over the rows that actually have a value. |
| Writing `COUNT(DISTINCT customer, region)` expecting "distinct customer-region pairs" without checking the exact semantics | PostgreSQL does support multiple expressions inside `COUNT(DISTINCT ...)`, but not every database does, and the exact behavior (distinct combinations of the listed expressions) is worth confirming rather than assuming — see Module 22 (SQL Across Databases) for where such extensions diverge between vendors. |
| Trying to aggregate and see per-row detail in the same plain `SELECT`, with no `GROUP BY` | An aggregate function collapses the entire result down to one row; mixing an aggregate with a non-aggregated column (without `GROUP BY`) either errors or is meaningless — this exact rule is the subject of Topic 2. |

## Interview Questions

1. **Q: What is the difference between `COUNT(*)` and `COUNT(column_name)`?**
   A: `COUNT(*)` counts every row in the result set, regardless of content. `COUNT(column_name)` counts only the rows where that specific column is not `NULL`. They return the same number only if the column has no `NULL` values at all.

2. **Q: If a table has 100 rows and the `discount` column is `NULL` in 30 of them, what does `SELECT AVG(discount) FROM that_table;` divide by — 100 or 70?**
   A: 70. `AVG` ignores `NULL` values entirely; both the sum and the count used to compute the average are based only on the 70 rows where `discount` actually has a value. The 30 `NULL` rows are excluded, not treated as zero.

3. **Q: How would you count the number of distinct customers who have ever placed an order, from a table where each row is one order?**
   A: `SELECT COUNT(DISTINCT customer) FROM orders;` — this counts each unique, non-`NULL` customer value exactly once, regardless of how many orders that customer placed.

4. **Q: Can an aggregate function operate on an expression, like `quantity * unit_price`, rather than a single column?**
   A: Yes — an aggregate function accepts any expression that produces one value per row, including arithmetic on multiple columns, function calls, or `CASE` expressions. `SUM(quantity * unit_price)` computes the per-row line total first, then sums those computed values.

## Summary

- An aggregate function collapses many input rows into a single output value — `SUM`, `AVG`, `MIN`, `MAX`, and `COUNT` are the core set.
- `COUNT(*)` counts rows unconditionally; `COUNT(column)` counts only rows where that column is not `NULL` — these can and often do differ.
- Every aggregate function except `COUNT(*)` silently skips `NULL` values rather than treating them as zero, which directly follows from `NULL` meaning "unknown," not "zero."
- `COUNT(DISTINCT column)` counts unique non-`NULL` values rather than every occurrence.
- Aggregate functions can operate on any expression (arithmetic, function calls) that yields one value per row, not just a bare column.
- Without `GROUP BY`, an aggregate query always collapses the *entire* table into one output row — getting a per-category breakdown instead requires `GROUP BY`, covered next.
