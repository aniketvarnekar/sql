# Subqueries in WHERE and FROM

## Learning Objectives

By the end of this section you should be able to:
- Use a subquery as a filter list with `IN`, and explain how it differs from the scalar subqueries in Topic 1
- Use a subquery as a **derived table** in a `FROM` clause, and query it exactly as if it were a real table
- State PostgreSQL's requirement that every derived table be aliased, and fix the error that results when it's missing
- Explain why `IN` requires the subquery to return exactly one column (though any number of rows)

## Prerequisites

- [Scalar Subqueries](01-scalar-subqueries.md) — this topic assumes you already understand what a subquery is and how it nests inside a larger statement; here, the subquery is deliberately **not** scalar (it returns a list of rows, or an entire table), so it's worth having Topic 1's single-value case fresh in mind as the contrast.

## Motivation

Not every question has an answer that collapses to one value. "Which orders were placed by a customer based in Austin?" doesn't need one number — it needs a whole list of qualifying `customer_id` values to check membership against. And "which customers spent more than $400 in total?" doesn't need a single value either — it needs an entire computed table (customer, total spent) that you then filter and read like any other table. This topic covers both of those shapes: a subquery used as a list, and a subquery used as a table.

## Problem Statement

Imagine you want every order placed by a customer located in Austin. You know how to filter `orders` by a literal `customer_id` (`WHERE customer_id = 1`), but "Austin" isn't a fact stored on `orders` at all — it's a fact about `customers`. You'd first have to run `SELECT customer_id FROM customers WHERE city = 'Austin';`, read off the resulting IDs by hand, and then hard-code them into a second query (`WHERE customer_id IN (1, 3)`) — fragile for the same reason a hand-typed average was fragile in Topic 1: the moment a new Austin customer signs up, your hand-typed list goes stale. Separately, imagine you want each customer's *total* amount spent, and then only the customers whose total exceeds $400 — but `SUM()` requires a `GROUP BY`, and you also want to filter and join on the *result* of that grouping, not on individual order rows. Both problems are solved by letting a subquery stand in for something bigger than a single value: a list, or a table.

## Concept

### Subqueries as a Filter List with `IN`

```sql
SELECT order_id, amount
FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE city = 'Austin'
)
ORDER BY order_id;
```

```
 order_id | amount
----------+--------
      101 | 250.00
      102 | 120.00
      104 | 500.00
      107 |  60.00
      109 |  60.00
(5 rows)
```

The inner query `SELECT customer_id FROM customers WHERE city = 'Austin'` returns a **list** of values — `1` (Ava Patel) and `3` (Chen Wu) — not a single value. `IN` doesn't require a scalar subquery; it requires a subquery that returns exactly **one column**, but any number of rows. The outer query then keeps every `orders` row whose `customer_id` appears anywhere in that list, exactly as if you had written `WHERE customer_id IN (1, 3)` by hand — except this list is computed live from `customers`, so it automatically includes any new Austin-based customer added later.

This is a genuinely useful pattern on its own, and it is also worth remembering as you read ahead: Topic 4 introduces `EXISTS`, which answers a very similar-sounding question — "does a related row exist?" — using a different mechanism that is often more efficient and, critically, safer around `NULL` values. Treat this `IN`-based version as the intuitive starting point; Topic 4 explains precisely when and why to prefer the alternative.

### Derived Tables: Using a Subquery in `FROM`

A subquery isn't limited to `WHERE` — it can also appear in a `FROM` clause, standing in for an entire table. A subquery used this way is called a **derived table** (sometimes an "inline view").

```sql
SELECT c.name, customer_totals.total_amount
FROM customers c
JOIN (
    SELECT customer_id, SUM(amount) AS total_amount
    FROM orders
    GROUP BY customer_id
) AS customer_totals
    ON c.customer_id = customer_totals.customer_id
WHERE customer_totals.total_amount > 400
ORDER BY customer_totals.total_amount DESC, c.name;
```

```
    name    | total_amount
------------+--------------
 Elin Kask  |       580.00
 Chen Wu    |       560.00
 Ava Patel  |       430.00
(3 rows)
```

Walking through this:
- The inner query `SELECT customer_id, SUM(amount) AS total_amount FROM orders GROUP BY customer_id` computes one row per customer who has placed at least one order, each with their total spend.
- That result — not a real, stored table — is given the name `customer_totals` and treated exactly like any other table for the rest of the statement: it's joined to `customers`, and its `total_amount` column is filtered in `WHERE` and used in `ORDER BY`.
- Ben Ortiz's total ($235.50) doesn't clear the $400 filter, so he's absent from the output. Dara Singh doesn't appear at all — not because she failed the `> 400` filter, but because she has **zero rows** in `orders` to begin with, so `GROUP BY customer_id` never produces a group for her in the first place, and the `JOIN` has nothing to match her against.

Once you have a derived table, you can do anything to it that you could do to a real table: filter it, join it to other tables, aggregate it further, or select specific columns from it. This is the core idea to internalize: **a derived table lets you build multi-step logic — aggregate, then filter, then join — inside a single statement**, without needing a permanent, named object like a view (Module 12) to hold the intermediate result.

### The Alias Requirement

PostgreSQL requires every derived table to have an alias. Omit it, and the query fails outright:

```sql
SELECT *
FROM (
    SELECT customer_id, SUM(amount) AS total_amount
    FROM orders
    GROUP BY customer_id
);
```

```
ERROR:  subquery in FROM must have an alias
LINE 1: SELECT * FROM (SELECT customer_id, SUM(amount) AS total_am...
                      ^
HINT:  For example, FROM (SELECT ...) [AS] foo.
```

The fix is exactly what the earlier example already did — add `AS customer_totals` (or any name) right after the closing parenthesis:

```sql
SELECT *
FROM (
    SELECT customer_id, SUM(amount) AS total_amount
    FROM orders
    GROUP BY customer_id
) AS customer_totals;
```

The `AS` keyword itself is optional in PostgreSQL (`) customer_totals` alone would also work), but the alias is not — some name is mandatory. This is a stricter rule than ordinary table aliasing, where naming a real table (`FROM orders o`) is only a readability convenience, never a requirement.

## Internal Working (Preview)

When a derived table appears in `FROM`, PostgreSQL's planner has a choice about how to actually execute it, and this is worth previewing even before Module 20 covers it properly:

```
 Simple derived table (e.g., a plain SELECT with a WHERE)
        │
        ▼
 Planner often "flattens" it — folds the inner query's logic
 directly into the outer query's plan, as if you'd written one
 query all along (no separate intermediate result materializes)

 Complex derived table (e.g., one containing GROUP BY, as above)
        │
        ▼
 Planner more often computes it as a genuinely separate step —
 the grouped/aggregated result is produced first, then joined
```

This means a derived table is not always literally "computed once into a temporary table and then queried" under the hood — for simple cases, the optimizer can merge (flatten) the subquery's logic directly into the surrounding query, producing a single efficient plan with no real intermediate table ever materializing. Whether flattening happens depends on what the derived table contains (aggregation, `LIMIT`, `DISTINCT`, and window functions all tend to block flattening, since they must be computed as one coherent step). You can always see exactly what the planner chose to do with `EXPLAIN`, covered fully in Module 20.

## Real-World Analogy

A derived table is like a scratch calculation you do on a whiteboard in the middle of a meeting: you compute an intermediate result (say, a running subtotal per department), write it on the board under a label, and then everyone in the room refers to "the subtotal board" for the rest of the discussion — pointing at rows in it, comparing numbers on it — without needing to recompute it each time. The label is not optional: if you erased the label and just left an unlabeled grid of numbers on the whiteboard, nobody could later say "look at the subtotal board" — there would be no name to refer to. That's exactly why PostgreSQL insists on an alias: an unnamed derived table has no way to be referenced by anything else in the query.

## Why Subqueries in WHERE and FROM Were Designed This Way

The relational model (Module 02) treats the result of *any* query as itself a relation (a table) — rows and named columns — regardless of whether that relation happens to be permanently stored on disk or freshly computed. `IN` and derived tables are simply SQL exposing that theoretical closure property directly: since a query's result is a relation, it's entirely consistent that a relation can be *used* anywhere another relation could be used — including inside another query's `FROM` or `WHERE` clause. The mandatory alias requirement follows from the same idea taken to its logical conclusion: every relation in a query needs a name so it can be unambiguously referred to elsewhere in that same statement, and a derived table is no exception just because it's temporary and unstored.

## Advantages

- **Solves multi-step problems in one statement.** Aggregate, then filter, then join — all without a separate query, a temporary table, or a permanently stored view.
- **Always reflects live data.** Like every subquery in this module, an `IN`-list or a derived table is recomputed every time the outer query runs, so it never goes stale the way a hand-typed list of IDs would.
- **Composable like any other table.** Once aliased, a derived table can be joined, filtered, and further aggregated exactly like a real table — there's no special restricted syntax to learn for working with it afterward.

## Disadvantages / Limitations

- **Not reusable within the same query.** If you need the same derived table's result in two different places in one statement, you must either repeat the subquery (recomputing it, and risking the two copies silently drifting out of sync if edited separately) or switch to a CTE (Module 17), which can be referenced by name multiple times.
- **Readability drops with nesting.** A derived table containing another derived table, containing another subquery, quickly becomes hard for a human to trace top-to-bottom — a CTE (Module 17) that names each step in order is often clearer once you're more than one level deep.
- **`IN` with a large or unindexed subquery result can be costly.** If the subquery driving an `IN` list returns a very large number of rows, checking membership against all of them can be more expensive than an equivalent `EXISTS`-based query — the precise reasoning is covered in Topic 4.

## Best Practices

- Give every derived table a name that describes *what it computed*, not a generic placeholder — `customer_totals` reads far better than `t` or `sub` months later.
- Alias the derived table's computed columns too (`AS total_amount` above) — an unaliased expression column inside a derived table produces an unreadable auto-generated name when referenced from outside.
- Reach for `IN` with a subquery when you only need a simple list-membership check against one column; reach for a derived table when you need to filter or join against a *computed, multi-column* result (like a per-customer total).
- If a query accumulates more than one or two derived tables, or the same derived table's logic would be needed twice, switch to a CTE (Module 17) instead — the logic is equivalent, but a named, referenceable step is easier for the next reader (including future you) to follow.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Omitting the alias on a derived table | PostgreSQL requires it; the query fails outright with `subquery in FROM must have an alias`, unlike aliasing a real table, which is optional. |
| Writing `IN` with a subquery that returns more than one column | `IN (SELECT a, b FROM ...)` is only valid when compared against a matching row-value expression (e.g., `WHERE (x, y) IN (...)`) — a plain single-column `IN` requires the subquery to return exactly one column, or PostgreSQL raises a syntax/type error. |
| Assuming a derived table is always physically materialized before the rest of the query runs | For simple derived tables, the planner often flattens the subquery directly into the outer plan rather than computing it as a separate step first — whether this happens depends on what the subquery contains (Module 20 covers reading the actual plan). |
| Forgetting that columns from a derived table must be referenced through its alias, not the original table's name | Once a subquery is aliased as `customer_totals`, its columns are `customer_totals.total_amount`, not `orders.total_amount` — the derived table is a new, independent relation as far as the rest of the query is concerned. |

## Interview Questions

1. **Q: What is a derived table, and why must it be aliased in PostgreSQL?**
   A: A derived table is a subquery used in place of a table in a `FROM` clause — its result is treated exactly like a real table for the rest of the query. PostgreSQL requires an alias because every relation referenced in a query needs a name so other parts of the statement (its columns, joins to it) can unambiguously refer to it; omitting the alias raises `subquery in FROM must have an alias`.

2. **Q: What's the difference between a scalar subquery and a subquery used with `IN`?**
   A: A scalar subquery must return exactly one row and one column, and is used where SQL expects a single value. A subquery used with `IN` must return exactly one column but can return any number of rows — it's checked for list membership, not treated as a single value.

3. **Q: Why would you use a derived table instead of just writing a more complex single `WHERE`/`GROUP BY`?**
   A: A derived table lets you compute an intermediate result — like a per-customer aggregate total — and then filter, join, or further aggregate *that computed result* as if it were its own table. Some questions (e.g., "customers whose total spend exceeds $400") genuinely require filtering on a value that only exists after grouping, which a derived table expresses cleanly.

4. **Q: If you need the same derived table's logic twice in one query, what's a better alternative?**
   A: A common table expression (CTE), covered in Module 17. Unlike a derived table, a CTE is given a name once and can be referenced multiple times within the same statement without repeating (and risking a mismatch in) its logic.

## Summary

- A subquery used with `IN` returns a **list** — any number of rows, but exactly one column — and the outer query checks membership against that list.
- A **derived table** is a subquery used in `FROM`, standing in for an entire virtual table; once aliased, it can be joined, filtered, and aggregated exactly like a real table.
- PostgreSQL requires every derived table to have an alias — this is a hard syntax requirement, not a style preference, unlike aliasing a real table.
- Both patterns solve the same underlying problem as Topic 1's scalar subqueries — avoiding a fragile, hand-typed, two-step manual process — just for a list or a whole computed table instead of a single value.
- Whether a derived table is physically materialized or flattened directly into the outer query's plan is an optimizer decision, previewed here and covered fully with `EXPLAIN` in Module 20.
