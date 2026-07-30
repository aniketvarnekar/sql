# FULL OUTER JOIN and CROSS JOIN

## Learning Objectives

By the end of this section you should be able to:
- Write a `FULL OUTER JOIN` and predict which unmatched rows from both sides it preserves
- Explain `FULL OUTER JOIN` as a combination of `LEFT JOIN` and `RIGHT JOIN`
- Write a `CROSS JOIN` and explain what a Cartesian product is
- Identify at least one genuine, intentional use case for `CROSS JOIN`
- Recognize an accidental cross join caused by a forgotten join condition, and explain why it's dangerous

## Prerequisites

- **[LEFT and RIGHT OUTER JOIN](02-left-and-right-outer-join.md)** — `FULL OUTER JOIN` is directly defined in terms of combining what `LEFT JOIN` and `RIGHT JOIN` each preserve.

## Motivation

Topic 2 covered preserving unmatched rows from *one* side of a join. But sometimes you genuinely need both directions at once — "show me every customer and every order, matched where possible, but don't drop anything on either side" — for example, reconciling two systems where either side might have entries the other doesn't know about yet. Separately, this topic also covers `CROSS JOIN`, which has nothing to do with preserving unmatched rows at all — it's about deliberately generating every possible combination of two tables. The two are grouped in this topic because `CROSS JOIN` is also the accidental result of the single most dangerous, easy-to-make mistake in this entire module: forgetting a join condition.

## Problem Statement

Recall this module's data: Dara Singh (customer 4) has never placed an order, and order 105 is a guest checkout with no linked customer (`customer_id IS NULL`). Suppose a data-quality report needs to show *every* customer and *every* order side by side, flagging anything on either side that doesn't connect to the other — customers who've never ordered, and orders with no valid customer account. Neither `LEFT JOIN` nor `RIGHT JOIN` alone can show both kinds of gaps in a single query: `LEFT JOIN` would show Dara but hide order 105; `RIGHT JOIN` would show order 105 but hide Dara. You need something that preserves unmatched rows from *both* sides simultaneously.

## Concept

### Reusing the Running Schema

Same `customers` and `orders` tables and data as Topics 1 and 2:

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

### FULL OUTER JOIN

```sql
SELECT c.name, o.order_id, o.order_date
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
ORDER BY c.name NULLS LAST, o.order_id;
```

```
    name    | order_id | order_date
------------+----------+------------
 Ava Patel  |      101 | 2026-01-05
 Ava Patel  |      102 | 2026-02-10
 Ben Ortiz  |      103 | 2026-01-20
 Chen Wu    |      104 | 2026-03-01
 Dara Singh |          |
            |      105 | 2026-03-15
(6 rows)
```

This result has **6 rows** — the 4 genuinely matched pairs, plus Dara Singh (preserved with `NULL` order columns, exactly as `LEFT JOIN` would show her), plus order 105 (preserved with a `NULL` customer name, exactly as `RIGHT JOIN` would show it). `FULL OUTER JOIN` is precisely: **keep every matched pair, plus every unmatched row from the left table, plus every unmatched row from the right table** — nothing from either side is ever silently dropped.

You can think of `FULL OUTER JOIN`'s result as the combination of `LEFT JOIN`'s result and `RIGHT JOIN`'s result, with the genuinely matched rows counted only once (not duplicated):

```
   LEFT JOIN result           RIGHT JOIN result           FULL OUTER JOIN result
 (matched + unmatched      (matched + unmatched          (matched rows once, plus
  left rows)             +  right rows)              =    every unmatched row from
                                                            both sides)
```

Combined with the same "find rows with no match" idiom from Topic 2, you can isolate exactly the data-quality gaps from the Problem Statement:

```sql
SELECT c.name, o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id IS NULL OR o.order_id IS NULL;
```

```
    name    | order_id
------------+----------
 Dara Singh |
            |      105
(2 rows)
```

This single query surfaces both kinds of gap at once: a customer with no orders, and an order with no customer — something neither `LEFT JOIN` nor `RIGHT JOIN` alone could do in one pass.

### CROSS JOIN

`CROSS JOIN` is conceptually unrelated to preserving unmatched rows — it has no `ON` condition at all, because it isn't matching anything. Instead, it produces every possible combination of a row from the first table with a row from the second: the **Cartesian product**.

```sql
CREATE TABLE shipping_methods (
    method_name TEXT PRIMARY KEY,
    base_cost   NUMERIC NOT NULL
);

INSERT INTO shipping_methods (method_name, base_cost) VALUES
    ('Standard', 5.00),
    ('Express',  15.00);

SELECT c.name, s.method_name, s.base_cost
FROM customers c
CROSS JOIN shipping_methods s
ORDER BY c.name, s.method_name;
```

```
    name    | method_name | base_cost
------------+-------------+-----------
 Ava Patel  | Express     |     15.00
 Ava Patel  | Standard    |      5.00
 Ben Ortiz  | Express     |     15.00
 Ben Ortiz  | Standard    |      5.00
 Chen Wu    | Express     |     15.00
 Chen Wu    | Standard    |      5.00
 Dara Singh | Express     |     15.00
 Dara Singh | Standard    |      5.00
(8 rows)
```

`customers` has 4 rows, `shipping_methods` has 2 rows, and the result has exactly **4 × 2 = 8 rows** — every possible pairing. Nothing was "matched"; every customer is simply paired with every shipping method.

### When a Cartesian Product Is Actually Useful

`CROSS JOIN` looks alarming at first (an exploding row count) but it has genuine, intentional uses whenever the actual goal is to **generate every combination** of two independent sets, rather than to relate existing data:

- **Seeding a rates or eligibility table** — the example above is realistic: a real e-commerce system needs to know the shipping cost for every customer/method combination (perhaps before applying customer-specific discounts), and generating that combination table by hand for every customer would be tedious and error-prone as customers are added.
- **Generating a calendar or date scaffold** — combining a table of `dates` with a table of `store_locations` to produce one row per location per day, as a base to left-join actual sales data onto (so days/locations with zero sales still appear as zero, not missing).
- **Building all possible test or configuration combinations** — pairing a table of product `sizes` with a table of `colors` to enumerate every SKU variant that could exist.

In every genuine use case, the defining trait is the same: you *want* every combination, because the two tables represent independent dimensions being combined on purpose, not two things being related through a shared key.

### The Danger of an Accidental Cross Join

The far more common way `CROSS JOIN` shows up in real queries is by accident — specifically, by forgetting a join condition. Recall from Topic 1 that the old comma-style join produces a full Cartesian product first, then filters it down with `WHERE`:

```sql
-- Forgot to add the WHERE condition relating the two tables
SELECT c.name, o.order_id
FROM customers c, orders o;
```

```
    name    | order_id
------------+----------
 Ava Patel  |      101
 Ava Patel  |      102
 Ava Patel  |      103
 Ava Patel  |      104
 Ava Patel  |      105
 Ben Ortiz  |      101
 Ben Ortiz  |      102
   ...
(20 rows)
```

With 4 customers and 5 orders, this produces **4 × 5 = 20 rows** — every customer paired with every order, including completely nonsensical pairings like "Ava Patel, order 103" even though order 103 actually belongs to Ben Ortiz. This is exactly what happens when you write explicit `JOIN` syntax but forget the `ON` clause, or write `INNER JOIN table_b` and accidentally omit the matching condition needed to relate it correctly — PostgreSQL requires *some* `ON` for explicit `JOIN` syntax, but a mistakenly trivial or incomplete one (for example, `ON true`, or a condition that matches far more broadly than intended) can produce the same explosive, wrong result while still being syntactically valid.

This is dangerous specifically because:
- **It's syntactically valid SQL** — the database will not raise an error; it will happily execute the query and return a large, wrong result set that looks like data, not like a failure.
- **The row count explosion isn't always obvious** — with only 4 and 5 rows, 20 rows is noticeably large, but with two tables of 10,000 rows each, the "correct" join might return roughly 10,000 rows while the accidental cross join returns 100,000,000 — plausible enough to not be instantly recognized as wrong, especially if the query is only glanced at rather than checked carefully, and severe enough to seriously slow down or crash a production database.
- **It silently corrupts any aggregate built on top of it** — a `SUM` or `COUNT` computed over an accidental cross join's result will be wildly, confidently wrong, with no error to alert you.

## Internal Working (Preview)

For `FULL OUTER JOIN`, PostgreSQL's execution engine typically implements it as a variant of a hash join or merge join that tracks, for rows on *both* sides, whether each one found at least one match during the join — any row still unmatched once the join completes is emitted once with `NULL`s filling the other side, for whichever side it came from.

For `CROSS JOIN`, there is no matching logic to run at all — the engine simply emits one output row for every combination of an input row from each side, which is why it has no `ON` clause: there is nothing to evaluate a condition against. This also means a `CROSS JOIN`'s cost is entirely predictable and multiplicative: joining an `N`-row table with an `M`-row table via `CROSS JOIN` always produces exactly `N × M` rows, with no filtering step to reduce that number — which is precisely why an *accidental* cross join is so much more punishing at scale than an accidental inner join gone slightly wrong.

## Real-World Analogy

`FULL OUTER JOIN` is like reconciling two independent guest lists for a merged event — one list from the venue's booking system, one from the caterer's headcount system. You lay both lists side by side and match names that appear on both. Anyone on the venue's list but missing from the caterer's list still gets shown (with a blank on the caterer side) so you know to tell the caterer about them; anyone on the caterer's list but missing from the venue's list is shown the same way in reverse. Nobody from either list is silently dropped just because they didn't appear on both.

`CROSS JOIN` is like a tailor's fabric-and-pattern combination chart: if you have 4 fabrics and 3 patterns, the chart deliberately lists all 12 fabric/pattern combinations, because the whole point is to enumerate every option available to a customer — not to "match" a fabric to "its" pattern, since there's no such existing relationship to match on in the first place.

## Why These Were Designed This Way

`FULL OUTER JOIN` exists because relational data reconciliation is a genuinely common need — two systems, two data sources, or two points in time, each with entries the other doesn't yet have — and expressing "keep everything from both, matched where possible" as a single declarative join is far simpler and less error-prone than manually running a `LEFT JOIN`, a `RIGHT JOIN`, and stitching the two result sets together yourself (which, notably, is close to what Topic 6's `UNION` could do, but with far more manual bookkeeping). `CROSS JOIN` exists because "every combination of two sets" is itself a legitimate, common data-generation need, distinct from relating existing data — and giving it its own explicit keyword (rather than only being reachable "by accident" via a forgotten condition) lets a reader immediately tell, from the SQL itself, whether a Cartesian product was *intentional* or a bug. A `CROSS JOIN` in code signals "yes, I meant to do this"; a comma-join with a missing `WHERE`, or an `INNER JOIN` quietly missing its condition, gives the reader no such signal at all — which is exactly why this course recommends explicit `JOIN` syntax throughout.

## Advantages

- **`FULL OUTER JOIN` guarantees zero silent data loss from either side** — ideal for reconciliation and data-quality auditing where losing track of an unmatched record on *either* side would be a real problem.
- **`CROSS JOIN` cleanly and efficiently generates combinations** — for genuine combination-generation needs, it's far simpler and more maintainable than manually writing out or scripting every pairing.
- **Both make their intent explicit in the SQL itself** — a reader sees `FULL OUTER JOIN` and immediately knows "gaps on either side are preserved here"; a reader sees `CROSS JOIN` and immediately knows "this is intentionally every combination," rather than having to guess whether a large result set was a mistake.

## Disadvantages / Limitations

- **`FULL OUTER JOIN` results contain `NULL`s from both directions**, meaning downstream logic (filters, aggregates, display code) has to handle "missing on the left" and "missing on the right" as two distinct cases, roughly doubling the `NULL`-handling complexity compared to a one-sided outer join.
- **`CROSS JOIN`'s row count grows multiplicatively, not additively** — joining three tables of 1,000 rows each via `CROSS JOIN` produces a billion rows, so it must be used deliberately and only against tables you know are appropriately small, or combined with filtering afterward.
- **An accidental Cartesian product is one of the easiest, most damaging mistakes to make in SQL** — because it's syntactically valid and returns a plausible-looking (if wrong or oversized) result rather than an error, it can go unnoticed until it causes a performance incident or a visibly wrong report.

## Best Practices

- Reach for `FULL OUTER JOIN` specifically for reconciliation-style questions ("what's on one side but not the other, in both directions") — for ordinary reporting where you only care about preserving one particular side, a plain `LEFT JOIN` is simpler and communicates intent more precisely.
- Whenever you intentionally write a `CROSS JOIN`, keep it against genuinely small, independent tables (configuration/lookup-style tables, not primary transactional tables), and add a comment explaining *why* a full combination is wanted, since it's an unusual enough pattern that future readers may otherwise assume it's a mistake.
- After writing any multi-table query, sanity-check the row count against what you'd expect (roughly the smaller table's row count for an inner/outer join on a foreign key) — a row count wildly larger than expected is the most reliable early warning sign of an accidental cross join.
- Always use explicit `JOIN` syntax with an explicit `ON` condition (Topic 1) specifically because it makes an unintentional Cartesian product far less likely than the old comma-style join, where a missing `WHERE` clause is silent and easy to overlook.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `FROM a, b` and forgetting the `WHERE` condition relating them | Silently produces a full Cartesian product with no error — every row of `a` paired with every row of `b` — which is almost never the intended result and can be severe at scale. |
| Using `FULL OUTER JOIN` when only one side's unmatched rows actually matter | Unnecessarily doubles the `NULL`-handling burden in the result compared to a plain `LEFT JOIN`, which is both simpler to write and clearer about intent when only one direction of "missing" matters. |
| Using `CROSS JOIN` against large, unrelated primary tables (e.g., all customers cross-joined with all orders) instead of small lookup tables | The row count explodes multiplicatively (rows in A × rows in B), which can produce an enormous, extremely slow, and essentially useless result for anything but genuinely small combination-generation tables. |
| Assuming a `CROSS JOIN` and an accidental comma-join-with-no-`WHERE` are different in effect | They produce the identical Cartesian product result — the only difference is that `CROSS JOIN` states the intent explicitly in the SQL, while the comma-join version gives no signal that a full combination (rather than a filtered join) was meant. |

## Interview Questions

1. **Q: How does `FULL OUTER JOIN` differ from `LEFT JOIN` and `RIGHT JOIN`?**
   A: `LEFT JOIN` preserves unmatched rows only from the left table; `RIGHT JOIN` preserves unmatched rows only from the right table; `FULL OUTER JOIN` preserves unmatched rows from both tables simultaneously, in addition to every genuinely matched pair, with `NULL`s filled in on whichever side didn't have a match for a given row.

2. **Q: What is a Cartesian product, and which join produces it?**
   A: A Cartesian product is every possible combination of a row from one table with a row from another — if table A has N rows and table B has M rows, the Cartesian product has N × M rows. `CROSS JOIN` produces this deliberately; it has no `ON` condition because it isn't matching rows on any relationship, it's enumerating every combination.

3. **Q: Give a legitimate, intentional use case for `CROSS JOIN`.**
   A: Generating a complete combination table where every pairing genuinely matters — for example, cross-joining a small table of customers with a small table of shipping methods to produce a base rate for every possible customer/method pair, or cross-joining a date range with a list of store locations to build a scaffold that ensures every location appears for every day, even days with zero activity.

4. **Q: How can a forgotten join condition accidentally turn an intended `INNER JOIN` into a Cartesian product, and why is that dangerous?**
   A: If a join condition is missing or too permissive (for example, using the old comma-style join without the matching `WHERE` clause, or an `ON` condition that doesn't actually relate the tables meaningfully), the database has no basis to exclude any pairing, so it returns every combination of rows from both tables. This is dangerous because it's syntactically valid SQL that executes without error and returns a plausible-looking but wrong (and potentially enormous) result set, rather than failing loudly — at scale, it can also severely degrade database performance.

## Summary

- `FULL OUTER JOIN` preserves every matched pair plus every unmatched row from *both* tables — conceptually the union of what `LEFT JOIN` and `RIGHT JOIN` would each preserve individually.
- `CROSS JOIN` produces a Cartesian product: every row from one table paired with every row from the other, with `N × M` total rows and no `ON` condition, because there's nothing being matched.
- Genuine `CROSS JOIN` use cases involve deliberately generating every combination of two independent sets — rate tables, scaffolding, or configuration enumeration — typically against small lookup-style tables.
- A forgotten or incomplete join condition silently produces the same Cartesian product by accident, and it is one of the most dangerous common SQL mistakes because it returns a plausible-looking, syntactically valid, but badly wrong result rather than an error.
- Always sanity-check a multi-table query's row count against expectations, and prefer explicit `JOIN...ON` syntax throughout to make accidental cross joins far less likely.
