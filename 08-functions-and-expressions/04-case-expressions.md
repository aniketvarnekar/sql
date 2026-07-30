# CASE Expressions

## Learning Objectives

By the end of this section you should be able to:
- Distinguish simple `CASE` (equality against a single expression) from searched `CASE` (arbitrary boolean conditions)
- Use `CASE` inside a `SELECT` list to produce a conditional, derived column
- Use `CASE` inside `ORDER BY` and `WHERE` to drive custom sorting and conditional filtering
- Explain why `CASE` is generally preferable to ad hoc boolean-arithmetic tricks for the same logic

## Prerequisites

- [Date and Time Functions](03-date-and-time-functions.md) — not a strict dependency, but this topic assumes the same fluency with multi-clause queries built up across this module.
- Module 7 — Querying Basics, specifically comparison and boolean operators (`=`, `<`, `AND`, `OR`) — `CASE`'s conditions are built entirely from those operators, just packaged into a branching structure.

## Motivation

So far, every function in this module transforms a value using a fixed rule — `UPPER()` always uppercases, `ROUND()` always rounds. Real reporting logic frequently needs something more: "show *this* label if the value is in *this* range, otherwise show *that* label." That's conditional branching, and until now nothing in this course has provided it inside a query. `CASE` is SQL's answer — an expression, usable anywhere a value is expected, that evaluates one of several branches based on conditions you define.

## Problem Statement

Consider a small `orders` table (a fresh, self-contained example for this topic):

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_name TEXT,
    status CHAR(1),         -- 'P' pending, 'S' shipped, 'D' delivered, 'C' cancelled
    total NUMERIC(10,2)
);

INSERT INTO orders (customer_name, status, total) VALUES
    ('Asha Verma',  'P',  42.50),
    ('Ben Okafor',  'S', 128.00),
    ('Chen Wei',    'D',  76.20),
    ('Deepak Rao',  'C',  15.00);
```

The `status` column stores a terse single-letter code — efficient to store, but meaningless to display to a customer or a manager reading a report. You also need to classify orders into "Small," "Medium," and "Large" based on `total`, and you need pending orders to sort to the top of a report regardless of their alphabetical customer name. None of `WHERE`, `ORDER BY`, or a plain function call alone can express "pick one of several outputs depending on a condition" — that's exactly the gap `CASE` fills.

## Concept

### Simple `CASE` — Matching a Single Expression Against Fixed Values

Simple `CASE` evaluates one expression once, then compares it for **equality** against a list of possible values.

```sql
SELECT customer_name, status,
       CASE status
           WHEN 'P' THEN 'Pending'
           WHEN 'S' THEN 'Shipped'
           WHEN 'D' THEN 'Delivered'
           WHEN 'C' THEN 'Cancelled'
           ELSE 'Unknown'
       END AS status_label
FROM orders;
```

```
 customer_name | status | status_label
----------------+--------+---------------
 Asha Verma    | P      | Pending
 Ben Okafor    | S      | Shipped
 Chen Wei      | D      | Delivered
 Deepak Rao    | C      | Cancelled
(4 rows)
```

Simple `CASE` is concise, but limited: it can only test the *one* expression named right after `CASE` for equality against each `WHEN` value — it cannot express a range, a comparison, or a condition involving a different column.

### Searched `CASE` — Arbitrary Boolean Conditions

Searched `CASE` has no single "subject" expression — each `WHEN` is a full, independent boolean condition, evaluated in order, and the first one that's true wins.

```sql
SELECT customer_name, total,
       CASE
           WHEN total < 20  THEN 'Small'
           WHEN total < 100 THEN 'Medium'
           ELSE 'Large'
       END AS order_size
FROM orders;
```

```
 customer_name | total  | order_size
----------------+--------+-------------
 Asha Verma    |  42.50 | Medium
 Ben Okafor    | 128.00 | Large
 Chen Wei      |  76.20 | Medium
 Deepak Rao    |  15.00 | Small
(4 rows)
```

Note the ordering matters: because `WHEN total < 100` only gets evaluated if `total < 20` was already false, writing `'Medium'` as `total < 100` correctly captures "20 and up, but under 100" without needing to repeat the lower bound (`total >= 20 AND total < 100`) explicitly. `CASE` conditions are always checked top to bottom, and evaluation stops at the first match.

Searched `CASE` strictly generalizes simple `CASE` — anything simple `CASE` can express, searched `CASE` can too (just written as `WHEN status = 'P' THEN ...` instead of `WHEN 'P' THEN ...`), but not the reverse. Use simple `CASE` when you're purely matching one expression against exact values (it reads slightly cleaner); use searched `CASE` the moment you need a range, a comparison, or a condition spanning multiple columns.

### `CASE` Inside `ORDER BY`

Because `CASE` is just an expression, it can be placed inside `ORDER BY` to define a custom sort order that has nothing to do with the natural ordering of any single column.

```sql
SELECT customer_name, status
FROM orders
ORDER BY CASE status
             WHEN 'P' THEN 1
             WHEN 'S' THEN 2
             WHEN 'D' THEN 3
             WHEN 'C' THEN 4
         END;
```

```
 customer_name | status
----------------+--------
 Asha Verma    | P
 Ben Okafor    | S
 Chen Wei      | D
 Deepak Rao    | C
(4 rows)
```

Here, sorting by `status` alone would put `'C'` (Cancelled) before `'D'` (Delivered) alphabetically — not the priority a real operations report wants. Mapping each status to an explicit priority number via `CASE` lets you sort by *business meaning* instead of alphabetical accident.

### `CASE` Inside `WHERE`

`CASE` can also appear inside a `WHERE` condition, useful when the filtering logic itself needs to branch depending on another column's value.

```sql
SELECT customer_name, status, total
FROM orders
WHERE CASE
          WHEN status = 'C' THEN FALSE
          ELSE total > 50
      END;
```

```
 customer_name | status | total
----------------+--------+--------
 Ben Okafor    | S      | 128.00
 Chen Wei      | D      |  76.20
(2 rows)
```

This reads as: "exclude cancelled orders outright; for everything else, only keep orders over 50." Asha's order (42.50, pending) is excluded because it fails the `total > 50` check; Deepak's is excluded outright for being cancelled, regardless of its total. `CASE` in `WHERE` is less common than in `SELECT`/`ORDER BY` — most conditional filtering can be expressed more directly with `AND`/`OR` — but it becomes genuinely useful once the filtering rule itself needs to differ depending on another column, as shown here.

### `CASE` vs. a Series of Boolean Tricks

Because PostgreSQL allows casting a boolean to an integer, it's possible to replace some simple `CASE` logic with boolean arithmetic:

```sql
-- Using CASE:
SELECT SUM(CASE WHEN status = 'D' THEN 1 ELSE 0 END) AS delivered_count FROM orders;

-- Using a boolean-to-integer cast instead:
SELECT SUM((status = 'D')::INT) AS delivered_count FROM orders;
```

Both return the same result here (`1`), but the two approaches are not equally good choices:

| | `CASE` | Boolean-to-integer trick |
|---|---|---|
| Readability | Explicit, self-documenting — reads as plain conditional logic | Requires knowing that `TRUE`/`FALSE` can be cast to `1`/`0`, which isn't obvious to every reader |
| Portability | Standard SQL, works essentially identically across databases | Casting a boolean directly to an integer is not universally supported the same way across every database vendor (Module 22 covers these gaps) |
| Flexibility | Naturally extends to more than two outcomes, non-numeric outputs, ranges | Only works for two-outcome, numeric scenarios |

`CASE` is almost always the better default: it is more explicit, generalizes to more than a simple true/false split, and is standard SQL rather than leaning on a vendor-specific casting convenience.

## Internal Working (Preview)

`CASE` is evaluated as a genuine expression during row processing — for each row, the database engine walks the `WHEN` conditions **in the order they're written** and stops at the first one that evaluates true, returning that branch's result (or the `ELSE` value, or `NULL` if no `ELSE` is given and no condition matched). This top-to-bottom, stop-at-first-match evaluation is guaranteed by the SQL standard — it is not something the query optimizer is free to reorder, which matters because later branches may deliberately assume earlier ones already failed (as seen in the "Medium" branch above, which implicitly relies on "Small" having already been ruled out).

```
 For each row:
   WHEN condition 1 true?  ── yes ──▶ return result 1, stop
        │ no
        ▼
   WHEN condition 2 true?  ── yes ──▶ return result 2, stop
        │ no
        ▼
      ... continue ...
        │ none matched
        ▼
   ELSE result (or NULL if no ELSE)
```

## Real-World Analogy

A `CASE` expression works exactly like a flowchart or a decision tree used in a customer support script: "Is the customer asking about a refund? If yes, follow the refund process. If no, is it a technical issue? If yes, follow the technical process. Otherwise, escalate to a general agent." Each branch is checked in a fixed order, the first matching branch is followed, and everything else is short-circuited — nobody re-checks the refund condition after already going down the technical-issue path.

## Why CASE Was Designed This Way

SQL's `SELECT`, `WHERE`, and `ORDER BY` clauses are all built from expressions — column references, function calls, arithmetic, and comparisons — and `CASE` was designed to be just another *expression*, not a separate statement type, so that it can be dropped in anywhere an expression is already legal, exactly as shown across the `SELECT`/`WHERE`/`ORDER BY` examples above. This is consistent with SQL's declarative nature (Module 1): you describe *what value should appear given which condition*, and the engine evaluates that description per row, without you writing an explicit branching procedure the way you would in an imperative language.

## Advantages

- **A single, standard tool for all conditional-value needs** — `SELECT`, `WHERE`, and `ORDER BY` all support the exact same `CASE` syntax, so there's one mental model to learn rather than clause-specific conditional tricks.
- **More readable than boolean-arithmetic workarounds** — a `CASE` block reads close to plain English ("when this, then that"), self-documenting its own logic.
- **Generalizes cleanly beyond two outcomes** — unlike a boolean cast trick, `CASE` handles any number of branches with equal clarity.

## Disadvantages / Limitations

- **Can get visually long** for many branches, especially nested `CASE` expressions — deeply nested conditional logic is sometimes a sign the underlying data model itself should be reconsidered (e.g., a proper lookup/reference table, covered conceptually in Module 15's normalization discussion), rather than more `CASE` branches.
- **Order-dependence is a subtle risk** — because later branches can rely on earlier ones having failed, reordering `WHEN` clauses can silently change behavior, which is easy to overlook when someone edits a long `CASE` block later.

## Best Practices

- Prefer searched `CASE` over simple `CASE` unless you're doing pure equality matching on one expression — searched `CASE` is strictly more capable and rarely harder to read.
- Always include an explicit `ELSE` unless a `NULL` fallback result is genuinely the correct behavior — an omitted `ELSE` silently returns `NULL` for any unmatched row, which is easy to mistake for a bug later if it wasn't deliberate.
- When sorting with `CASE` in `ORDER BY`, keep the mapping (status code → sort priority) close to any other place the same codes are interpreted, so the two don't quietly drift out of sync.
- Reach for `CASE` over boolean-to-integer casting tricks by default — it's clearer and more portable.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Forgetting `ELSE`, then being surprised by unexpected `NULL`s | Without an `ELSE`, any row that matches no `WHEN` condition silently evaluates to `NULL` rather than raising an error — easy to miss until a downstream calculation (e.g., a `SUM`) is quietly wrong. |
| Writing overlapping `WHEN` conditions and assuming all matching branches apply | Only the **first** matching `WHEN` is used; if a row satisfies both the second and third condition, only the second one's result is returned — order matters. |
| Using simple `CASE` and trying to write a range condition in a `WHEN` | Simple `CASE` only supports equality against the single leading expression — a `WHEN` like `total < 100` is not valid there; that requires searched `CASE`. |
| Assuming a boolean-to-integer cast trick is more "performant" than `CASE` | In practice, the two compile to comparably efficient execution; the boolean-cast trick is not a meaningful performance optimization, and it costs readability and portability for no real benefit. |

## Interview Questions

1. **Q: What's the difference between simple `CASE` and searched `CASE`?**
   A: Simple `CASE` evaluates one expression once and compares it for equality against a list of `WHEN` values. Searched `CASE` has no single subject expression — each `WHEN` is its own independent boolean condition, which can express ranges, comparisons, or conditions spanning multiple columns. Searched `CASE` can express everything simple `CASE` can, plus more.

2. **Q: In a searched `CASE` with multiple `WHEN` clauses, what happens if a row matches more than one condition?**
   A: Only the first matching `WHEN`, in top-to-bottom order, is used — evaluation stops there. Later conditions, even if they'd also be true, are never reached for that row.

3. **Q: Can `CASE` be used inside an `ORDER BY` clause? Give a scenario where that's useful.**
   A: Yes — `CASE` is a valid expression anywhere an expression is legal, including `ORDER BY`. It's useful whenever the desired sort order doesn't match a column's natural ordering, such as mapping status codes to a custom business-priority order (e.g., pending orders should sort before shipped ones, even though that's not alphabetical or numerical order for the underlying codes).

4. **Q: Why is `CASE` generally preferred over a boolean-to-integer casting trick for conditional aggregation?**
   A: `CASE` is standard SQL, self-documenting, and generalizes naturally to more than two outcomes or non-numeric results. Boolean-to-integer casting relies on a vendor-specific convenience, is less immediately readable, and only works for strictly two-outcome numeric scenarios.

## Summary

- Simple `CASE` matches one expression against exact values; searched `CASE` evaluates independent boolean conditions and is strictly more flexible.
- `CASE` is an expression, not a statement — it can appear in `SELECT`, `WHERE`, `ORDER BY`, or anywhere else an expression is valid.
- Evaluation always proceeds top-to-bottom and stops at the first matching `WHEN` — order is significant, and later branches can safely assume earlier ones already failed.
- Always include an explicit `ELSE` unless an unmatched-row result of `NULL` is genuinely intended.
- Prefer `CASE` over boolean-to-integer casting tricks — it is clearer, more portable, and not meaningfully different in performance.
