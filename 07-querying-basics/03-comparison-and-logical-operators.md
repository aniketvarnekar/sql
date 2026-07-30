# Comparison and Logical Operators

## Learning Objectives

By the end of this section you should be able to:
- Use all six comparison operators (`=`, `<>`/`!=`, `<`, `>`, `<=`, `>=`) inside a `WHERE` clause
- Combine multiple conditions with `AND`, `OR`, and `NOT`
- State the precedence order between `NOT`, `AND`, and `OR`, and know when parentheses are required to get the result you actually intend
- Predict how a compound `WHERE` condition behaves when one of its columns is `NULL`

## Prerequisites

- [Filtering with WHERE](02-filtering-with-where.md) — you need `WHERE`'s per-row boolean evaluation established before combining multiple boolean expressions inside it.
- Module 03's [NULL and Three-Valued Logic](../03-data-types/05-null-and-three-valued-logic.md) — this topic assumes you already know `NULL` represents "unknown" and that SQL logic has three values (`TRUE`, `FALSE`, `UNKNOWN`), not two; here, that theory is applied directly to real `WHERE` conditions built from comparison and logical operators.

## Motivation

A single equality check (`category = 'Kitchen'`) answers only the simplest questions. Real filtering needs to combine several conditions — "Kitchen *or* Furniture, but under $50," "not discontinued *and* in stock" — and combining conditions correctly requires knowing exactly how `AND`, `OR`, and `NOT` interact, in what order they're evaluated, and how a `NULL` column quietly changes the outcome of a condition that looks, at a glance, like a simple yes-or-no test.

## Problem Statement

A colleague asks for "Kitchen or Furniture products under $50." You write what seems like the direct translation:

```sql
SELECT name, category, price
FROM products
WHERE category = 'Kitchen' OR category = 'Furniture' AND price < 50;
```

```
        name        |  category   | price
---------------------+-------------+--------
 Desk Lamp           | Furniture   |  34.95
 Coffee Mug          | Kitchen     |  11.25
 Electric Kettle     | Kitchen     |  45.00
 Blender_Pro 2000    | Kitchen     |  79.99
(4 rows)
```

Blender_Pro 2000 costs $79.99 — well over $50 — yet it's in the result. Every Kitchen product made it in regardless of price, which isn't what was asked for at all. This isn't a database bug; it's a direct consequence of operator precedence, and understanding exactly why is this topic's job.

## Concept

### The Comparison Operators

| Operator | Meaning | Example |
|---|---|---|
| `=` | Equal to | `category = 'Kitchen'` |
| `<>` | Not equal to (SQL standard) | `category <> 'Kitchen'` |
| `!=` | Not equal to (widely supported alias for `<>`, including in PostgreSQL) | `category != 'Kitchen'` |
| `<` | Less than | `price < 20` |
| `>` | Greater than | `price > 100` |
| `<=` | Less than or equal to | `price <= 20` |
| `>=` | Greater than or equal to | `price >= 100` |

`<>` is the SQL-standard spelling; `!=` is a widely supported alternative (PostgreSQL, MySQL, SQL Server, and others all accept it) that behaves identically. Either is correct — pick one convention for a codebase and stay consistent; this course uses `<>` in prose but shows `!=` here since you will see both in real code.

```sql
SELECT name, price FROM products WHERE price >= 45.00;
```

```
        name         | price
----------------------+--------
 Mechanical Keyboard  |  89.99
 Standing Desk        | 349.00
 Office Chair         | 199.50
 Electric Kettle      |  45.00
 Blender_Pro 2000     |  79.99
(5 rows)
```

### Combining Conditions: `AND`, `OR`, `NOT`

`AND` requires both sides to be `TRUE`; `OR` requires at least one side to be `TRUE`; `NOT` inverts a single condition.

```sql
SELECT name, category, price
FROM products
WHERE category = 'Kitchen' AND price > 40;
```

```
       name         | category | price
---------------------+----------+--------
 Electric Kettle     | Kitchen  |  45.00
 Blender_Pro 2000    | Kitchen  |  79.99
(2 rows)
```

```sql
SELECT name, category
FROM products
WHERE category = 'Electronics' OR category = 'Home Goods';
```

```
        name         |  category
----------------------+-------------
 Wireless Mouse       | Electronics
 USB-C Cable          | Electronics
 Mechanical Keyboard  | Electronics
 100% Cotton Towel    | Home Goods
(4 rows)
```

```sql
SELECT name, category
FROM products
WHERE NOT (category = 'Electronics');
```

```
         name           |    category
-------------------------+-----------------
 Standing Desk           | Furniture
 Office Chair            | Furniture
 Desk Lamp               | Furniture
 Notebook Pack of 5      | Office Supplies
 Stapler                 | Office Supplies
 Whiteboard Marker Set   | Office Supplies
 Coffee Mug              | Kitchen
 Electric Kettle         | Kitchen
 Blender_Pro 2000        | Kitchen
 100% Cotton Towel       | Home Goods
(10 rows)
```

Notice `Uncategorized Widget` (whose `category` is `NULL`) doesn't appear here either — `NOT (category = 'Electronics')` evaluates to `UNKNOWN` for it, not `TRUE`, for reasons this topic returns to below.

### Operator Precedence — Why the Problem Statement's Query Went Wrong

When `AND` and `OR` appear in the same expression without parentheses, SQL doesn't evaluate them left to right — it applies a fixed precedence, identical in spirit to arithmetic's "multiplication before addition." From highest to lowest:

```
1. NOT   (evaluated first)
2. AND
3. OR    (evaluated last)
```

`AND` binds more tightly than `OR` — exactly like `*` binds more tightly than `+` in `2 + 3 * 4` (which is `14`, not `20`). The Problem Statement's query,

```sql
WHERE category = 'Kitchen' OR category = 'Furniture' AND price < 50
```

is therefore actually grouped like this:

```sql
WHERE category = 'Kitchen' OR (category = 'Furniture' AND price < 50)
```

Read that way, the bug is obvious: *any* Kitchen product satisfies the whole condition through the first `OR` branch alone, entirely independent of price — the `price < 50` check only ever applies to Furniture. To get "(Kitchen or Furniture) and under $50," the intended grouping must be written explicitly:

```sql
SELECT name, category, price
FROM products
WHERE (category = 'Kitchen' OR category = 'Furniture') AND price < 50;
```

```
       name        |  category  | price
--------------------+------------+--------
 Desk Lamp          | Furniture  |  34.95
 Coffee Mug         | Kitchen    |  11.25
 Electric Kettle    | Kitchen    |  45.00
(3 rows)
```

Blender_Pro 2000 ($79.99) is correctly excluded now. The parentheses didn't change what SQL is *capable* of expressing — the original query was always going to be grouped `OR (AND)` by precedence — they changed which grouping actually got expressed, to match what was actually meant.

### `NULL` and Three-Valued Logic Inside Compound Conditions

Module 3's [NULL and Three-Valued Logic](../03-data-types/05-null-and-three-valued-logic.md) established the full truth tables for `AND`, `OR`, and `NOT` extended to a third value, `UNKNOWN` — worth a compact recap here, since it governs exactly how compound `WHERE` conditions behave once any column involved can be `NULL`:

| | Key rule |
|---|---|
| `AND` | `FALSE AND anything` is always `FALSE`; `TRUE AND UNKNOWN` is `UNKNOWN` |
| `OR` | `TRUE OR anything` is always `TRUE`; `FALSE OR UNKNOWN` is `UNKNOWN` |
| `NOT` | `NOT UNKNOWN` is `UNKNOWN` (never flips to a definite answer) |

`WHERE` keeps a row only when the whole condition resolves to a definite `TRUE` — `FALSE` and `UNKNOWN` are both excluded, indistinguishably. Watch this play out directly against `Uncategorized Widget`, whose `category` is `NULL`:

```sql
SELECT name, category
FROM products
WHERE category <> 'Kitchen';
```

```
         name           |    category
-------------------------+-----------------
 Wireless Mouse          | Electronics
 USB-C Cable             | Electronics
 Mechanical Keyboard     | Electronics
 Standing Desk           | Furniture
 Office Chair            | Furniture
 Desk Lamp               | Furniture
 Notebook Pack of 5      | Office Supplies
 Stapler                 | Office Supplies
 Whiteboard Marker Set   | Office Supplies
 100% Cotton Towel       | Home Goods
(10 rows)
```

Ten rows — every non-Kitchen product *except* `Uncategorized Widget`, even though "Uncategorized Widget" is obviously not literally "Kitchen." Its `category` is `NULL`, so `NULL <> 'Kitchen'` evaluates to `UNKNOWN`, and `WHERE` silently drops it, exactly the same mechanism as Module 3's `department <> 'Sales'` example. This isn't specific to a single condition, either — it propagates through compound expressions the same way: any `AND`/`OR` chain that touches a `NULL` column can quietly resolve to `UNKNOWN` for that row rather than the `TRUE` or `FALSE` you might expect at a glance, which is exactly why `IS NULL`/`IS NOT NULL` (this module's next-but-one topic) exist as a dedicated, NULL-safe tool rather than relying on `=`/`<>`.

## Internal Working (Preview)

For a compound `WHERE` condition, the engine builds and evaluates an expression tree, respecting precedence, per row:

```
        OR
       /  \
  category=    AND
  'Kitchen'   /    \
        category=   price<50
        'Furniture'
```

For each row, the tree is evaluated bottom-up: the leaves (individual comparisons) resolve first, each to `TRUE`, `FALSE`, or `UNKNOWN`; then each `AND`/`OR` node combines its two children per the three-valued truth tables, until a single result reaches the root. Only rows where that root result is exactly `TRUE` survive into the next stage of the query. Parentheses in your SQL text directly control the *shape* of this tree — they don't change how any individual `AND`/`OR` node behaves, only which operands get grouped under which node.

## Real-World Analogy

Think of a venue's entry policy: "Staff, or VIP guests with a reservation." Read without deliberate grouping, that sentence is genuinely ambiguous between "(Staff or VIP), each additionally needing a reservation" and "Staff (no reservation needed), or (VIP with a reservation)" — and the two policies let in very different sets of people. Precedence rules are what a *language* uses to resolve that ambiguity by default (SQL always resolves it as the second reading, since `AND` binds tighter than `OR`); parentheses are how a person removes all doubt about which policy they actually meant, exactly as a venue's printed rules would spell out "(Staff) or (VIP AND has a reservation)" explicitly rather than trusting everyone to parse the sentence identically.

## Why These Operators Were Designed This Way

SQL's `AND`/`OR`/`NOT` precedence mirrors the same convention as arithmetic's `*`/`+` precedence and most general-purpose programming languages' boolean operators, specifically so the rule is learnable once and reused everywhere rather than being an arbitrary, SQL-specific quirk. Three-valued logic, meanwhile, is not an operator-level design choice at all — it's a direct, necessary consequence of allowing `NULL` (Codd's Rule 3, covered in [The Relational Model and Codd's Rules](../02-relational-model/03-the-relational-model-and-codds-rules.md)) to exist as a column state: once a value can be genuinely unknown, comparisons involving it cannot honestly collapse to only `TRUE` or `FALSE`, and `AND`/`OR`/`NOT` have to be defined consistently over that third possibility for `WHERE`'s filtering behavior to remain logically coherent at all.

## Advantages

- **Precedence rules are consistent with general mathematical and programming convention** — `AND` before `OR` mirrors `*` before `+`, minimizing what has to be memorized as SQL-specific.
- **Explicit parentheses give complete, unambiguous control** — any grouping you can imagine is expressible exactly, regardless of what the default precedence would otherwise produce.
- **Three-valued logic keeps every expression's meaning well-defined** — even an `UNKNOWN` result is a precise, principled outcome, not an undefined or arbitrary one.

## Disadvantages / Limitations

- **Precedence mistakes produce no error at all** — a misgrouped `AND`/`OR` expression is syntactically perfect, runs successfully, and simply returns the wrong rows, exactly as the Problem Statement demonstrated; there is no warning to catch it.
- **Three-valued logic is a genuine, recurring source of subtle bugs** — any condition touching a nullable column can quietly resolve to `UNKNOWN` rather than the `TRUE`/`FALSE` a reader assumes, and this compounds further once `NOT` and `IN`/`NOT IN` (this module's next topics) are involved.

## Best Practices

- **Parenthesize mixed `AND`/`OR` conditions explicitly, always** — even when the default precedence happens to already produce the grouping you want, explicit parentheses remove any doubt for a future reader (including yourself) and eliminate the entire class of bug the Problem Statement demonstrated.
- Prefer `<>` or `!=` consistently within one codebase rather than mixing both arbitrarily — either is correct, but consistency helps readability.
- Whenever a condition touches a column that can be `NULL`, explicitly decide and state whether `NULL` rows should be included (`OR column IS NULL`) or excluded, rather than assuming a comparison operator resolves that question for you.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing a mixed `AND`/`OR` condition without parentheses and assuming left-to-right evaluation | SQL applies fixed precedence (`NOT` > `AND` > `OR`), not left-to-right reading order — `A OR B AND C` is grouped as `A OR (B AND C)`, which is easy to misread. |
| Using `<>`/`!=` against a nullable column and expecting `NULL` rows to be included | A `NULL` column compared with `<>` evaluates to `UNKNOWN`, and `WHERE` excludes `UNKNOWN` results exactly like `FALSE` ones — the row silently disappears. |
| Assuming `!=` is nonstandard or unsupported | `!=` is a widely supported alias for `<>` in PostgreSQL and most other databases — both are equally valid; use whichever your team standardizes on. |
| Believing `NOT (condition)` always inverts a result to the "opposite" set of rows | `NOT UNKNOWN` is still `UNKNOWN`, not `TRUE` — a `NOT` applied to a condition touching a `NULL` column does not simply return "everything the original condition didn't match." |

## Interview Questions

1. **Q: What is the precedence order between `NOT`, `AND`, and `OR`, and why does it matter in practice?**
   A: `NOT` binds tightest, then `AND`, then `OR` — meaning `A OR B AND C` is evaluated as `A OR (B AND C)`, not left to right. It matters because a mixed condition written without parentheses can silently group in a way the author didn't intend, producing a query that runs without error but returns the wrong rows.

2. **Q: What does `TRUE AND NULL` (or `UNKNOWN`) evaluate to, and why?**
   A: `UNKNOWN`. `AND` can only resolve to a definite `FALSE` early if one side is already `FALSE`; if one side is `TRUE` and the other is genuinely unknown, the overall truth of the combination remains genuinely unknown too, so it can't collapse to a definite `TRUE` or `FALSE`.

3. **Q: Why might a `WHERE` clause using `<>` silently exclude rows you expect to see?**
   A: If the compared column is `NULL` for those rows, `<>` (or `=`) against it evaluates to `UNKNOWN`, not `TRUE` or `FALSE` — and `WHERE` only keeps rows where the condition is exactly `TRUE`, so rows with a `NULL` in that column are excluded exactly as if they'd failed the comparison outright.

## Summary

- The six comparison operators (`=`, `<>`/`!=`, `<`, `>`, `<=`, `>=`) and the three logical operators (`AND`, `OR`, `NOT`) are the building blocks of every `WHERE` condition beyond a single equality check.
- Precedence runs `NOT` > `AND` > `OR` — mixed conditions without explicit parentheses are grouped by this rule, not by reading order, and getting this wrong produces no error, only wrong results.
- Always parenthesize mixed `AND`/`OR` conditions explicitly — it costs nothing and eliminates an entire class of silent, hard-to-spot bugs.
- `NULL` extends `TRUE`/`FALSE` logic to a third value, `UNKNOWN`; `WHERE` excludes both `FALSE` and `UNKNOWN` rows identically, which is why comparisons against a nullable column can silently drop rows a reader expects to see.
- The next topic, [Pattern Matching with LIKE](04-pattern-matching-with-like.md), adds text-pattern comparison to this same `WHERE`-condition toolkit.
