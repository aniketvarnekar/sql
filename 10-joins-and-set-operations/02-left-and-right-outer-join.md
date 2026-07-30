# LEFT and RIGHT OUTER JOIN

## Learning Objectives

By the end of this section you should be able to:
- Explain what "outer" means in an outer join, in contrast to `INNER JOIN`
- Write a `LEFT JOIN` and predict exactly which rows and `NULL`s it produces
- Explain why `RIGHT JOIN` is the mirror image of `LEFT JOIN`, and why `LEFT JOIN` is used almost exclusively in practice
- Write the "find rows with no match" pattern using `LEFT JOIN` plus a `NULL` check
- Explain why a filter on the right-hand table must go in `ON`, not `WHERE`, to preserve outer join behavior

## Prerequisites

- **[INNER JOIN](01-inner-join.md)** — this topic directly builds on `INNER JOIN`'s behavior; every outer join in this topic is defined as "`INNER JOIN`'s result, plus some additional unmatched rows kept."

## Motivation

Topic 1 ended by pointing out `INNER JOIN`'s biggest limitation: it silently drops any row that doesn't have a match on the other side. Sometimes that's exactly what you want. But very often it isn't — "list every customer, and their most recent order if they have one" is a completely reasonable, common business question, and it explicitly requires customers *without* any order to still appear in the result. `LEFT` and `RIGHT OUTER JOIN` exist to answer exactly that kind of question: keep every row from one specified side, whether or not it found a match, and fill in the gaps with `NULL` where no match exists.

## Problem Statement

Recall this module's customers: Ava Patel, Ben Ortiz, Chen Wu, and Dara Singh — but Dara has never placed an order. If you ask "list every customer along with their orders" using `INNER JOIN`, Dara vanishes from the result entirely, because `INNER JOIN` only keeps matched pairs:

```sql
SELECT c.name, o.order_id, o.order_date
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
ORDER BY c.name;
```

```
   name    | order_id | order_date
-----------+----------+------------
 Ava Patel |      101 | 2026-01-05
 Ava Patel |      102 | 2026-02-10
 Ben Ortiz |      103 | 2026-01-20
 Chen Wu   |      104 | 2026-03-01
(4 rows)
```

Dara Singh is nowhere to be seen — not because of a bug, but because `INNER JOIN` is doing exactly what it's defined to do. If the actual business question was "show me *every* customer, including ones who haven't ordered yet, so I know who to send a welcome discount to," this result is silently wrong for that purpose. You need a join that keeps Dara in the output even though she has no match in `orders`.

## Concept

### Reusing the Running Schema

This topic uses the exact same `customers` and `orders` tables and data from Topic 1 — Dara Singh (customer 4, no orders) and order 105 (a guest order, `customer_id IS NULL`) are both present, and both matter here.

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT NOT NULL,
    city        TEXT
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date  DATE NOT NULL,
    status      TEXT NOT NULL
);

INSERT INTO customers (name, email, city) VALUES
    ('Ava Patel',  'ava@example.com',  'Austin'),
    ('Ben Ortiz',  'ben@example.com',  'Denver'),
    ('Chen Wu',    'chen@example.com', 'Austin'),
    ('Dara Singh', 'dara@example.com', 'Seattle');

INSERT INTO orders (customer_id, order_date, status) VALUES
    (1, '2026-01-05', 'shipped'),
    (1, '2026-02-10', 'shipped'),
    (2, '2026-01-20', 'cancelled'),
    (3, '2026-03-01', 'shipped'),
    (NULL, '2026-03-15', 'shipped');
```

### LEFT JOIN

```sql
SELECT c.name, o.order_id, o.order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
ORDER BY c.name, o.order_id;
```

```
    name    | order_id | order_date
------------+----------+------------
 Ava Patel  |      101 | 2026-01-05
 Ava Patel  |      102 | 2026-02-10
 Ben Ortiz  |      103 | 2026-01-20
 Chen Wu    |      104 | 2026-03-01
 Dara Singh |          |
(5 rows)
```

`customers` is written on the **left** side of `LEFT JOIN` (the table named in `FROM`), and `orders` is on the **right** side (the table named after `LEFT JOIN`). The rule is: **every row from the left table appears in the output at least once, whether or not it has a match.** When Dara has no matching row in `orders`, PostgreSQL still includes her row from `customers`, and fills every column that would have come from `orders` (`order_id`, `order_date`) with `NULL`. This is precisely why `LEFT JOIN` is sometimes called `LEFT OUTER JOIN` — the word `OUTER` is optional in PostgreSQL and every major database; `LEFT JOIN` and `LEFT OUTER JOIN` are exactly the same thing.

Contrast this directly with `INNER JOIN`'s result from the Problem Statement: identical 4 matched rows, plus one additional row (Dara, with `NULL`s) that `INNER JOIN` would have dropped. This is the pattern to internalize for every outer join in this module: **an outer join's result always contains everything `INNER JOIN` would have produced, plus some extra unmatched rows with `NULL`s filled in.**

### RIGHT JOIN

`RIGHT JOIN` is the exact mirror image: it keeps every row from the **right** table instead, filling in `NULL`s for the left side where there's no match.

```sql
SELECT c.name, o.order_id, o.order_date
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id
ORDER BY o.order_id;
```

```
   name    | order_id | order_date
-----------+----------+------------
 Ava Patel |      101 | 2026-01-05
 Ava Patel |      102 | 2026-02-10
 Ben Ortiz |      103 | 2026-01-20
 Chen Wu   |      104 | 2026-03-01
           |      105 | 2026-03-15
(5 rows)
```

Here it's order 105 (the guest checkout with no `customer_id`) that gets preserved with a `NULL` customer name, while Dara Singh — who has no orders — is dropped this time, because `RIGHT JOIN` only guarantees every row from `orders` (the right table) is kept, not every row from `customers`.

### Why LEFT JOIN and RIGHT JOIN Are Mirror Images

Every `RIGHT JOIN` can be rewritten as an equivalent `LEFT JOIN` simply by swapping which table is named first:

```sql
-- Identical result to the RIGHT JOIN above
SELECT c.name, o.order_id, o.order_date
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id
ORDER BY o.order_id;
```

This produces exactly the same result set as the `RIGHT JOIN` version. `LEFT` and `RIGHT` are not two independently different operations — they are the *same* operation, "preserve every row from one designated side," and the keyword just tells you which side, relative to how the two table names are written.

### Why LEFT JOIN Dominates in Practice

Given that `RIGHT JOIN` is always rewritable as a `LEFT JOIN`, most SQL style guides — and most real codebases — use `LEFT JOIN` almost exclusively and essentially never write `RIGHT JOIN`, for a few concrete, practical reasons:

- **One mental model instead of two.** If you only ever reach for `LEFT JOIN`, you never have to stop and figure out "wait, is it the left or right table I want preserved here?" for two different keywords — you just always put the table you want fully preserved first, in `FROM`.
- **Readability follows top-to-bottom reading order.** In a query chaining several joins (Topic 5), a chain of `LEFT JOIN`s reads naturally top-to-bottom: "start from customers, then bring in orders if they exist, then bring in order items if they exist." Mixing in a `RIGHT JOIN` partway through a long chain forces the reader to mentally reorder which table is actually being preserved at that point.
- **It's purely convention, not a technical limitation.** `RIGHT JOIN` is fully supported and behaves exactly as documented in PostgreSQL — nothing is wrong with using it — but because it's rare in practice, a `RIGHT JOIN` appearing in code tends to make a reader pause and double-check its direction, whereas a `LEFT JOIN` doesn't. Consistency with the convention your codebase already uses is itself valuable.

### The "Find Rows With No Match" Pattern

One of the most common practical uses of `LEFT JOIN` is finding rows in one table that have *no* corresponding row in another — for example, "which customers have never placed an order?" This is done by combining `LEFT JOIN` with a `WHERE` check for `NULL` on a column that could only be `NULL` if there was no match:

```sql
SELECT c.name, c.email
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

```
    name    |      email
------------+------------------
 Dara Singh | dara@example.com
(1 row)
```

Here's why this works: `o.order_id` is `orders`' primary key, so it can never legitimately be `NULL` in an actual matched row — the *only* way `o.order_id` shows up as `NULL` in this result is if `LEFT JOIN` had no match to fill it with, meaning that customer has zero orders. This pattern — `LEFT JOIN` plus `WHERE <right_table>.<col> IS NULL` — is extremely common in real reporting and data-quality queries ("find products never ordered," "find employees with no assigned manager," "find accounts with no linked payment method") and is worth memorizing as a named idiom, not re-deriving from scratch every time.

### Why the Filter's Location Matters: ON vs. WHERE

A subtle but important trap with outer joins: where you put an additional filter condition changes the result.

```sql
-- Filter in ON: still an outer join — Dara is preserved
SELECT c.name, o.order_id, o.status
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id AND o.status = 'shipped'
ORDER BY c.name;
```

```
    name    | order_id | status
------------+----------+---------
 Ava Patel  |      101 | shipped
 Ava Patel  |      102 | shipped
 Ben Ortiz  |          |
 Chen Wu    |      104 | shipped
 Dara Singh |          |
(5 rows)
```

```sql
-- Filter in WHERE: effectively cancels the outer join for this condition
SELECT c.name, o.order_id, o.status
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.status = 'shipped'
ORDER BY c.name;
```

```
   name    | order_id | status
-----------+----------+---------
 Ava Patel |      101 | shipped
 Ava Patel |      102 | shipped
 Chen Wu   |      104 | shipped
(3 rows)
```

In the first version, `o.status = 'shipped'` is part of the *join condition* — it only affects which `orders` rows are eligible to match, but every customer is still preserved (Ben Ortiz appears with `NULL`s because his only order was cancelled, not shipped). In the second version, `WHERE o.status = 'shipped'` is applied *after* the join completes, and it discards any row where that condition is false — including the `NULL`-filled rows for customers with no shipped order, since `NULL = 'shipped'` is never true. The net effect: the second query silently behaves almost like an `INNER JOIN` for that condition, defeating the entire purpose of using `LEFT JOIN`. **Rule of thumb: conditions about the right-hand (preserved-if-matched) table's own data belong in `ON`; conditions about the left-hand (always-kept) table belong in `WHERE`.**

## Internal Working (Preview)

Conceptually, PostgreSQL builds a `LEFT JOIN` in two logical steps:

```
 Step 1: Compute the INNER JOIN result
         (only genuinely matched pairs)
                    │
                    ▼
 Step 2: For every row in the LEFT table that
         contributed no matched pair in Step 1,
         add it back once, with NULL in every
         column that would have come from the
         right table
                    │
                    ▼
        Final LEFT JOIN result set
```

This is a conceptual model, not literally how the execution engine computes it — PostgreSQL's planner has dedicated outer-join algorithms (hash join and merge join variants that natively track "did this row find a match yet") that produce the same guaranteed result far more efficiently than literally computing an inner join first and then a second pass. The important guarantee for you as a query writer is the *result*, not the mechanism: every left-side row appears at least once, period.

## Real-World Analogy

Think of `LEFT JOIN` like a school attendance roster combined with a permission-slip signup sheet for a field trip. The roster (the left table) lists every student in the class — you print every single name, no matter what. Next to each name, you write in which permission slip they signed up with, if any. A student who never turned one in still appears on the printed roster, just with a blank next to their name — you don't erase them from the class list just because they didn't sign up. `RIGHT JOIN` would be the same idea starting from the signup sheet instead: every signed permission slip gets listed, and if a slip somehow doesn't match any known student (a smudged name, a transfer student), it still shows up with a blank where the roster info would go.

## Why Outer Joins Were Designed This Way

`INNER JOIN` (Topic 1) maps cleanly onto strict relational algebra's natural join, but real-world questions frequently need to express "show me everything on this side, whether or not it's related to anything yet" — a customer who just signed up, a product that hasn't sold, an employee not yet assigned a manager. Without a dedicated outer join, answering that question would require running two separate queries (one for matched rows, one for unmatched rows) and manually stitching the results together yourself — exactly the kind of low-level, imperative bookkeeping SQL's declarative design (Module 1) exists to eliminate. `LEFT`/`RIGHT JOIN` let you describe "preserve this side no matter what" directly, in one declarative statement, and leave the DBMS to work out how to compute that guarantee efficiently.

## Advantages

- **No data loss for the preserved side** — every row from the designated table is guaranteed to appear, which is exactly right for reports that must account for "everyone," not just "everyone with existing activity."
- **Directly supports the extremely common "find what's missing" question** — the `LEFT JOIN` + `IS NULL` pattern answers "who/what has zero related records" cleanly and efficiently, using an index on the join column when one exists.
- **Composable with further filtering** — as shown above, you can still narrow down which right-side rows are eligible to match (via `ON`) without losing the outer join's core guarantee.

## Disadvantages / Limitations

- **Easy to accidentally cancel the outer-join behavior** — as shown above, putting a right-side filter in `WHERE` instead of `ON` silently turns a `LEFT JOIN` back into something that behaves like an `INNER JOIN` for that condition, and PostgreSQL gives no warning that this happened.
- **`NULL`s ripple into every downstream computation** — any column pulled from the non-preserved side will be `NULL` for unmatched rows, and any expression or aggregate built from that column needs to account for that (Module 3's `NULL` semantics, and Module 9's aggregate functions, both matter here — `COUNT(o.order_id)` behaves very differently from `COUNT(*)` when `order_id` can be `NULL`).
- **`RIGHT JOIN` is rarely used, which can itself cause confusion** — because it's uncommon, a `RIGHT JOIN` appearing in a codebase (or in someone else's example) tends to require extra care to read correctly, precisely because readers are less practiced at it.

## Best Practices

- Default to `LEFT JOIN` and structure your `FROM` clause so the table you want fully preserved is always named first — avoid `RIGHT JOIN` in code you write, even though it's correct, simply for the consistency and readability benefits described above.
- When you need "find rows with no match," use the `LEFT JOIN` + `WHERE <right>.<col> IS NULL` idiom, and check the `NULL` on a column from the right-hand table that cannot legitimately be `NULL` in a real matched row (typically its primary key), not an arbitrary nullable column.
- Put conditions that should only restrict which right-side rows are *eligible to match* inside `ON`; put conditions that should restrict the final overall result (regardless of which side they reference) in `WHERE`, and always pause to check which one you actually mean when writing an outer join with additional filters.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Putting a right-table filter in `WHERE` on a `LEFT JOIN` and expecting unmatched left rows to still appear | `WHERE` runs after the join completes and evaluates against the (possibly `NULL`) right-side columns; a condition like `WHERE o.status = 'shipped'` is false for `NULL`, so it silently drops the very unmatched rows the `LEFT JOIN` was meant to preserve. |
| Checking `IS NULL` on a nullable column from the right table instead of its primary key, to find "no match" rows | If the checked column can legitimately be `NULL` even in a genuine matched row, you'll get false positives — always check a column guaranteed non-`NULL` in a real row, like the right table's primary key. |
| Using `RIGHT JOIN` out of habit from writing the tables in a particular order, instead of just reordering the `FROM` clause and using `LEFT JOIN` | Functionally harmless, but it breaks the "always preserve the first-named table" convention that makes long join chains (Topic 5) easy to read at a glance. |
| Assuming `LEFT JOIN` and `LEFT OUTER JOIN` are different | They are the exact same operation — `OUTER` is optional, purely stylistic, keyword syntax in PostgreSQL and every major SQL database. |

## Interview Questions

1. **Q: What is the difference between `INNER JOIN` and `LEFT JOIN`?**
   A: `INNER JOIN` includes only rows that have a match on both sides. `LEFT JOIN` includes everything `INNER JOIN` would, plus every unmatched row from the left (first-named) table, with `NULL` filled in for any column that would have come from the right table.

2. **Q: Why is `RIGHT JOIN` rarely seen in real codebases despite being fully valid SQL?**
   A: Any `RIGHT JOIN` can be rewritten as an equivalent `LEFT JOIN` by swapping which table is named first in the query. Since one form is enough to express both directions, most style conventions standardize on `LEFT JOIN` alone for a single, consistent mental model and more readable multi-join chains, rather than for any technical limitation of `RIGHT JOIN` itself.

3. **Q: How would you find all customers who have never placed an order, given `customers` and `orders` tables?**
   A: `LEFT JOIN customers` to `orders` on the customer key, then filter with `WHERE orders.order_id IS NULL` (checking the orders table's primary key, which can only be `NULL` in this result if a given customer had no matching order row at all).

4. **Q: If you write `LEFT JOIN orders o ON c.customer_id = o.customer_id WHERE o.status = 'shipped'`, does this still behave as a true left join?**
   A: Not for practical purposes — `WHERE` is evaluated after the join, and `o.status = 'shipped'` evaluates to false (not true) for the `NULL`-filled rows produced by unmatched customers, so those rows get filtered out. The result ends up including only customers with at least one shipped order, effectively behaving like an `INNER JOIN` with that extra condition — to keep true left-join behavior, `o.status = 'shipped'` should be moved into the `ON` clause.

## Summary

- `LEFT JOIN` preserves every row from the left (first-named) table, filling in `NULL` for any right-side column when there's no match; `RIGHT JOIN` does the identical thing for the right-named table instead.
- `RIGHT JOIN` is always rewritable as an equivalent `LEFT JOIN` by swapping table order — which is exactly why real-world SQL uses `LEFT JOIN` almost exclusively, by convention rather than necessity.
- The "find rows with no match" pattern — `LEFT JOIN` plus `WHERE <right_table>.<non-nullable column> IS NULL` — is one of the most useful and common outer join idioms in practice.
- A filter on the right-hand table's own data belongs in the `ON` clause if you want to preserve outer-join behavior; putting it in `WHERE` instead silently discards the unmatched rows you were trying to keep.
- Topic 3 extends this same idea one step further: preserving unmatched rows from *both* sides at once (`FULL OUTER JOIN`), plus the unrelated but easily confused `CROSS JOIN`.
