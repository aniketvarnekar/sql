# Correlated Subqueries

## Learning Objectives

By the end of this section you should be able to:
- Define a correlated subquery and identify the specific outer-query column reference that makes it correlated
- Explain why a correlated subquery's answer can differ per outer row, in contrast to the non-correlated subqueries from Topics 1–2
- Work through a complete example of a correlated subquery, row by row, and predict its output
- Rewrite a correlated subquery as an equivalent join, and explain (at a conceptual level, with a forward pointer to Module 20) why the join version can be faster

## Prerequisites

- [Scalar Subqueries](01-scalar-subqueries.md) — this topic's examples are themselves scalar (one value per outer row), so you need to already be comfortable with the single-value form.
- [Subqueries in WHERE and FROM](02-subqueries-in-where-and-from.md) — the join-rewrite in this topic reuses the derived-table technique from Topic 2 directly.
- **Module 10 (Joins & Set Operations)** — rewriting a correlated subquery as a join assumes you can already write an `INNER JOIN` comfortably.

## Motivation

Every subquery so far in this module has computed one answer and reused that same answer for every row (or every candidate) it was compared against — the global average order amount was `184.55` no matter which order you were looking at. But plenty of real questions need a *different* comparison value depending on which row you're currently examining: "is this order bigger than *this customer's own* average?" is a fundamentally different, per-row question than "is this order bigger than the average across everyone." Correlated subqueries are how SQL expresses "the comparison value depends on the row I'm currently looking at."

## Problem Statement

Topic 1 answered "which orders are above the average order amount" using one fixed number, `184.55`, for every order. But consider a subtly different, equally realistic question: "which orders are unusually large *for that particular customer*?" Ava Patel's orders average $143.33; Chen Wu's average $280.00. An order of $160 is unremarkable for Chen (below his average) but would be Ava's largest order by far (above her average). A single global threshold can't distinguish these two customers' very different normal spending levels — you need the threshold itself to change depending on whose order you're currently evaluating. That's a correlated subquery's entire reason to exist.

## Concept

### Definition: What Makes a Subquery "Correlated"

> A **correlated subquery** is a subquery that references a column from the *outer* query. Because its result can depend on the outer row currently being evaluated, it is conceptually re-evaluated once for every row the outer query considers — in contrast to a **non-correlated subquery**, which contains no reference to the outer query and can be evaluated just once, independent of any particular row (exactly like every subquery in Topics 1–2).

The tell-tale sign of correlation is a column from the outer query's table appearing inside the subquery's own `WHERE` clause.

### Worked Example: Orders Above Each Customer's Own Average

```sql
SELECT o.order_id, o.customer_id, o.amount
FROM orders o
WHERE o.amount > (
    SELECT AVG(o2.amount)
    FROM orders o2
    WHERE o2.customer_id = o.customer_id
)
ORDER BY o.customer_id, o.order_id;
```

```
 order_id | customer_id | amount
----------+-------------+--------
      101 |           1 | 250.00
      106 |           2 | 160.00
      104 |           3 | 500.00
      108 |           5 | 430.00
(4 rows)
```

The reference `o2.customer_id = o.customer_id` is what makes this subquery correlated: `o.customer_id` comes from the *outer* query's current row, not from anywhere inside the subquery itself. Walking through it row by row:

| Outer row (`o`) | Correlated subquery computes | Comparison | Included? |
|---|---|---|---|
| Order 101, Ava, $250.00 | AVG of Ava's orders (250, 120, 60) = 143.33 | 250.00 > 143.33 | Yes |
| Order 102, Ava, $120.00 | 143.33 | 120.00 > 143.33 | No |
| Order 109, Ava, $60.00 | 143.33 | 60.00 > 143.33 | No |
| Order 103, Ben, $75.50 | AVG of Ben's orders (75.50, 160) = 117.75 | 75.50 > 117.75 | No |
| Order 106, Ben, $160.00 | 117.75 | 160.00 > 117.75 | Yes |
| Order 104, Chen, $500.00 | AVG of Chen's orders (500, 60) = 280.00 | 500.00 > 280.00 | Yes |
| Order 107, Chen, $60.00 | 280.00 | 60.00 > 280.00 | No |
| Order 108, Elin, $430.00 | AVG of Elin's orders (430, 150) = 290.00 | 430.00 > 290.00 | Yes |
| Order 110, Elin, $150.00 | 290.00 | 150.00 > 290.00 | No |
| Order 105, `NULL` customer, $40.00 | `o2.customer_id = NULL` matches nothing (`NULL` never equals `NULL`) → `AVG` of zero rows = `NULL` | `40.00 > NULL` is `UNKNOWN` | No |

Notice order 106 (Ben's $160.00 order) is on this list — even though, back in Topic 1, that same order did **not** clear the *global* average of $184.55. This is the entire point: $160 is unremarkable globally, but it is Ben's biggest order relative to his own typical spending. A non-correlated, global-average query and this correlated, per-customer query genuinely disagree on order 106 — they're answering two different questions that happen to sound similar in English. Also notice order 105 is correctly excluded: because its `customer_id` is `NULL`, and `NULL` is never equal to anything (not even another `NULL`), the correlated subquery finds zero matching rows for it, `AVG` of zero rows is `NULL`, and comparing `40.00 > NULL` yields `UNKNOWN` rather than `TRUE` — so the row is dropped. This three-valued-logic behavior around `NULL` reappears, with much higher stakes, in Topic 4.

### Contrast: Non-Correlated vs. Correlated

| | Non-correlated (Topics 1–2) | Correlated (this topic) |
|---|---|---|
| References the outer query? | No | Yes — an outer column appears inside the subquery |
| Conceptually evaluated | Once, independent of any specific row | Once per outer row, using that row's own values |
| Example | `WHERE amount > (SELECT AVG(amount) FROM orders)` | `WHERE amount > (SELECT AVG(amount) FROM orders o2 WHERE o2.customer_id = o.customer_id)` |
| Answer changes based on which row you're looking at? | No — the same fixed value every time | Yes — a different value per customer |

### Rewriting as a Join

The correlated subquery above can be rewritten using the derived-table technique from Topic 2, computing every customer's average once and joining to it:

```sql
SELECT o.order_id, o.customer_id, o.amount
FROM orders o
JOIN (
    SELECT customer_id, AVG(amount) AS avg_amount
    FROM orders
    GROUP BY customer_id
) AS customer_avg
    ON customer_avg.customer_id = o.customer_id
WHERE o.amount > customer_avg.avg_amount
ORDER BY o.customer_id, o.order_id;
```

```
 order_id | customer_id | amount
----------+-------------+--------
      101 |           1 | 250.00
      106 |           2 | 160.00
      104 |           3 | 500.00
      108 |           5 | 430.00
(4 rows)
```

Identical result — but shaped completely differently under the hood. The derived table `customer_avg` computes **every** customer's average in a single `GROUP BY` pass, once, and the `JOIN` then looks each order up against its own customer's precomputed average. Order 105's `NULL` `customer_id` is excluded the same way [INNER JOIN](../10-joins-and-set-operations/01-inner-join.md) always excludes it — `NULL` never matches `NULL` in a join condition either, so it never finds a row in `customer_avg` to join against.

## Internal Working (Preview or Deep Dive)

Conceptually, the correlated version executes as a **nested loop**: for every row the outer query considers, the outer row's value is "bound" into the inner query, the inner query runs using that specific value, and the comparison is checked — then the process repeats for the next outer row.

```
 for each row in orders (outer):
     bind o.customer_id into the inner query
     run: SELECT AVG(amount) FROM orders WHERE customer_id = <bound value>
     compare o.amount > (that result)
     keep or discard the outer row
 (repeat for the next outer row, from scratch)
```

The join-based rewrite instead computes every group's average **once**, in a single pass over `orders` (the `GROUP BY`), and then performs one pass matching orders to their precomputed group average — no re-running of an inner query per outer row at all. This is why correlated subqueries are a frequent flag in performance discussions (Module 20): naively, a correlated subquery evaluated as a literal nested loop costs work roughly proportional to *(outer rows) × (inner query cost)*, whereas the equivalent grouped join computes the same information in roughly one pass over each table. In practice, PostgreSQL's planner is often smart enough to automatically transform a correlated subquery into something resembling the join-based plan internally — so the two are not always as different in actual performance as the naive nested-loop picture above suggests. Whether that transformation happens for a given query, and how to check, is exactly what Module 20's treatment of `EXPLAIN` and execution plans is for.

## Real-World Analogy

A correlated subquery is like a teacher asking, for every single student, "is your score above *your own class's* average?" — to answer that, the teacher has to know which class each student is in before computing anything, and a student in a different class gets compared against a different number. A join-based rewrite is like the teacher first computing each class's average once on the board, labeled by class, and then walking down the full roster checking each student against their own class's already-computed number — the same final judgments, reached by doing the shared work (computing each class's average) exactly once instead of re-deriving it for every single student.

## Why Correlated Subqueries Were Designed This Way

SQL's logical row-by-row evaluation model (first introduced conceptually in Module 01 — [Your First Query](../01-introduction/05-your-first-query.md), and formalized fully in Module 7) treats a `WHERE` condition as something evaluated per candidate row. A correlated subquery is simply what falls out naturally when you allow that per-row condition to itself contain an arbitrary nested query, with the outer row's own column values available as parameters to that nested query. It's a direct, minimal extension of "a `WHERE` clause is evaluated per row" — nothing new was invented; correlated subqueries are what you get when a subquery is allowed to see the row it's being evaluated against, which is a natural consequence of SQL's declarative, per-row filtering semantics rather than a special-cased feature bolted on top of it.

## Advantages

- **Expresses "relative to my own group" naturally.** "Above my own customer's average" reads directly as English when written as a correlated subquery, without first requiring you to build and name a separate grouped result.
- **No separate named step required for a one-off comparison.** For a single, simple correlated condition, this can be more concise than first constructing a derived table purely to join against it.
- **General-purpose.** A correlated subquery's condition can be arbitrarily complex — not limited to the kind of aggregate-per-group logic a join-based rewrite handles cleanly, unlike some alternative rewrites that only work for specific shapes of question.

## Disadvantages / Limitations

- **Potential performance cost at scale.** As described above, a literal per-row re-evaluation is far more expensive than computing shared group information once — a real concern once tables grow large, covered in depth (with actual measurement tools) in Module 20.
- **Can be evaluated once per output row even in a `SELECT` list**, not just in `WHERE` — a correlated subquery placed in the `SELECT` list is a common, easy-to-miss source of a query that scales badly as the number of output rows grows.
- **Harder to read than the join-based rewrite for some readers**, once the correlation reference is buried in a deeply nested condition — the join version makes the "one average per customer" step visible and named, rather than implicit inside a nested `WHERE`.

## Best Practices

- Whenever you notice you've written a correlated subquery whose job is "compute a per-group aggregate and compare to it," pause and consider the join-based rewrite from this topic — it's frequently both clearer and (potentially) faster on larger tables.
- Reserve correlated subqueries for cases that are genuinely awkward to express as a join — for example, an `EXISTS` check (Topic 4) or a condition that doesn't reduce cleanly to "join to a precomputed aggregate."
- On any correlated subquery you're unsure about, use `EXPLAIN` (Module 20) once your table sizes are realistic — don't assume "correlated" automatically means "slow"; verify it for your actual data and PostgreSQL version.
- Always double- and triple-check which column belongs to which alias in a correlated subquery (`o2.customer_id = o.customer_id` above) — a typo that references the wrong alias is a common, silent source of incorrect results rather than an error, since both aliases are often valid column names.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Forgetting to alias the outer and inner instances of the same table differently | Without distinct aliases (`o` and `o2` above), PostgreSQL cannot tell which `customer_id` reference belongs to the outer row and which belongs to the inner subquery, and raises an ambiguous-reference error. |
| Assuming a correlated subquery in a `SELECT` list runs only once total | It can be conceptually (and sometimes literally) re-evaluated once per row returned by the outer query — a correlated subquery is not automatically as cheap as the non-correlated ones from Topics 1–2. |
| Confusing a correlated subquery's `NULL`-handling with a bug | As shown with order 105 above, a correlated subquery correctly excludes rows whose correlated column is `NULL`, because `NULL = NULL` is never `TRUE` — this is correct three-valued-logic behavior, not an error to "fix." |
| Assuming every correlated subquery can be trivially rewritten as a join | Some correlated conditions (particularly existence checks, Topic 4) rewrite cleanly; others involve logic that doesn't reduce neatly to a single `GROUP BY` and join, and are more naturally left as a correlated subquery. |

## Interview Questions

1. **Q: What makes a subquery "correlated," and how does that differ from the subqueries covered earlier in this module?**
   A: A correlated subquery references a column from the outer query inside its own condition, so its result can differ depending on which outer row is currently being evaluated. Non-correlated subqueries (Topics 1–2) contain no such reference and can be evaluated once, independent of any specific outer row.

2. **Q: Give an example of a question that requires a correlated subquery rather than a plain aggregate subquery.**
   A: "Which orders are above that specific customer's own average order amount?" requires comparing each order to a threshold that depends on which customer placed it — a single global average (a non-correlated subquery) cannot express a per-customer threshold.

3. **Q: Why can correlated subqueries be slower than an equivalent join?**
   A: Naively, a correlated subquery re-runs its inner query once per outer row considered, costing roughly (outer rows) × (inner query cost). An equivalent join-based rewrite typically computes any needed aggregate once, in a single pass, then joins — avoiding repeated re-computation. Modern query planners can sometimes optimize a correlated subquery into a similarly efficient plan automatically, which is why actually checking with `EXPLAIN` (Module 20) matters more than assuming.

4. **Q: Is every correlated subquery rewritable as a join?**
   A: Many are, especially ones that reduce to "compare against a per-group aggregate," as shown in this topic's worked example. Others — particularly ones only checking for the existence of a matching row without needing its value — are more naturally expressed with `EXISTS`, covered in Topic 4, which is technically a specific, common form of correlated subquery.

## Summary

- A **correlated subquery** references a column from the outer query, so it can produce a different result per outer row — unlike the **non-correlated** subqueries in Topics 1–2, which compute one fixed value independent of any row.
- The worked example — orders above each customer's own average — shows a case where the correlated (per-customer) answer genuinely differs from the non-correlated (global average) answer for the same underlying data.
- `NULL` correlated values (like order 105's `NULL` `customer_id`) correctly produce no match and no matching aggregate, correctly excluding that row rather than signaling an error.
- Many correlated subqueries can be rewritten as a join to a derived table that precomputes the needed aggregate once — often clearer and, on large tables, often faster, though the precise performance comparison requires checking with `EXPLAIN` (Module 20).
- Reserve correlated subqueries for logic that doesn't reduce cleanly to a join, and default to a join-based rewrite once you notice the correlated condition is really just "compare to a per-group aggregate."
