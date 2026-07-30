# Filtering with WHERE

## Learning Objectives

By the end of this section you should be able to:
- Write a `WHERE` clause that restricts a query to a specific subset of rows
- Explain that `WHERE` evaluates a boolean expression once per row, keeping only rows where it's `TRUE`
- Clearly distinguish *filtering* (choosing rows) from *projecting* (choosing columns)
- Recognize that `WHERE` can reference any column of a row, regardless of whether that column is ever projected in the `SELECT` list

## Prerequisites

- [SELECT and Projection](01-select-and-projection.md) — you need projection (choosing columns) clearly established before contrasting it with filtering (choosing rows); this topic also assumes the `products` table set up there.

## Motivation

A table's entire contents are rarely what you actually want. A product catalog might hold thousands of rows, but a given question — "what's out of stock?", "what's in the Kitchen category?", "what costs more than $50?" — only cares about a small, specific subset of them. Without a way to express *which rows qualify*, every query would have to return everything and force something else (a spreadsheet, an application, a human scrolling) to do the filtering after the fact. `WHERE` lets you push that decision into the database itself, stated declaratively, once, as part of the query.

## Problem Statement

Continuing with the `products` table from Topic 1, a colleague wants two very ordinary things:

1. "Show me everything in the Electronics category."
2. "Show me everything priced under $20."

Without `WHERE`, the only tool available is `SELECT`'s column list — which has no concept of rows at all, only columns. `SELECT name, price FROM products` returns all 14 rows no matter what columns you name; there is no way to narrow *which rows* appear using the `SELECT` list alone. A separate mechanism is needed to answer "which rows" — that mechanism is `WHERE`.

## Concept

### The Shape of a `WHERE` Clause

`WHERE` sits between `FROM` and any `ORDER BY`, and holds a single boolean expression — one that evaluates, independently, for every row the `FROM` clause produces:

```sql
SELECT name, price
FROM products
WHERE category = 'Electronics';
```

```
        name         | price
----------------------+-------
 Wireless Mouse       | 24.99
 USB-C Cable          |  9.99
 Mechanical Keyboard  | 89.99
(3 rows)
```

```sql
SELECT name, price
FROM products
WHERE price < 20;
```

```
          name          | price
-------------------------+-------
 USB-C Cable             |  9.99
 Notebook Pack of 5      |  6.49
 Stapler                 | 12.00
 Whiteboard Marker Set   |  8.75
 Coffee Mug              | 11.25
 100% Cotton Towel       | 15.00
 Uncategorized Widget    |  3.50
(7 rows)
```

For every one of the 14 rows in `products`, PostgreSQL asks the exact same question — "does this row's `price` satisfy `price < 20`?" — and keeps only the rows that answer yes. Nothing about the rows themselves changes; rows that don't qualify are simply excluded from the result entirely.

### Filtering (Rows) vs. Projecting (Columns) — Two Independent Decisions

Topic 1 introduced *projection* as the `SELECT` list's job: choosing which columns appear. `WHERE` implements the other fundamental relational operation, *selection* — choosing which rows appear. These are genuinely independent decisions, and it's worth seeing that independence directly:

```sql
-- All columns, only some rows (filter, no projection)
SELECT * FROM products WHERE category = 'Kitchen';

-- Some columns, all rows (projection, no filter)
SELECT name, price FROM products;

-- Some columns, some rows (both)
SELECT name, price FROM products WHERE category = 'Kitchen';
```

A `WHERE` clause can reference *any* column of the row being tested, whether or not that column ever appears in the `SELECT` list:

```sql
SELECT name, price
FROM products
WHERE stock_quantity = 0;
```

```
     name      | price
---------------+--------
 Office Chair  | 199.50
 Electric Kettle | 45.00
(2 rows)
```

`stock_quantity` is used entirely for filtering here — deciding which rows qualify — but never appears in the output. This is only possible because, per the logical processing order introduced in Topic 1, `WHERE` operates on the *full* row (every column, as stored) before projection ever throws anything away. If filtering happened *after* projection, this query wouldn't work at all, because `stock_quantity` would already be gone by the time `WHERE` ran.

### Combining Conditions (A Preview)

A `WHERE` clause isn't limited to one comparison — conditions can be combined with `AND`, `OR`, and `NOT`:

```sql
SELECT name, price
FROM products
WHERE category = 'Kitchen' AND price > 40;
```

```
        name       | price
--------------------+-------
 Electric Kettle    | 45.00
 Blender_Pro 2000   | 79.99
(2 rows)
```

The full set of comparison operators, how `AND`/`OR`/`NOT` combine, and the precedence rules that govern mixed conditions are this module's next topic, [Comparison and Logical Operators](03-comparison-and-logical-operators.md) — this topic deliberately stays focused on what `WHERE` fundamentally *does* (test a boolean expression per row) before expanding what that expression can contain.

### `WHERE` Cannot Reference a `SELECT`-List Alias

One direct, practical consequence of `WHERE` running before `SELECT` (Topic 1's logical processing order): an alias defined in the `SELECT` list is not visible inside that same query's `WHERE` clause.

```sql
SELECT price * 1.08 AS price_with_tax
FROM products
WHERE price_with_tax > 50;
```

```
ERROR:  column "price_with_tax" does not exist
LINE 3: WHERE price_with_tax > 50;
              ^
HINT:  Perhaps you meant to reference the column "products.price".
```

`price_with_tax` doesn't exist yet at the point `WHERE` is evaluated — it's computed afterward, per surviving row, by the `SELECT` list. To filter on a computed value, repeat the expression itself in `WHERE`:

```sql
SELECT price * 1.08 AS price_with_tax
FROM products
WHERE price * 1.08 > 50;
```

```
 price_with_tax
-----------------
         48.6000
         86.3892
        377.5200
        215.4600
```

(This returns Electric Kettle, Blender_Pro 2000, Standing Desk, and Office Chair — every row where `price * 1.08` exceeds 50.)

## Internal Working (Preview)

Conceptually, for a table scan without any supporting index, PostgreSQL evaluates `WHERE`'s boolean expression row by row:

```
 for each row in the table's storage:
       │
       ▼
 evaluate WHERE's expression against this row's column values
       │
       ├─ TRUE  → row passes through to the next stage (projection, sorting, ...)
       └─ FALSE or UNKNOWN → row is discarded, never appears in the output
```

This "test every row" approach (a *sequential scan*) is correct but not necessarily fast — for a table of a few thousand rows it's instant, but for a table of tens of millions of rows, testing every single row for every single query becomes expensive. Module 13 (Indexes) covers how the database can build a structure that lets it jump directly to qualifying rows for certain kinds of `WHERE` conditions, skipping the need to test every row at all — but the *logical meaning* of `WHERE` (keep only rows where the condition is `TRUE`) is identical whether or not an index is involved; only the *speed* of getting there differs.

## Real-World Analogy

Think of `WHERE` like a kitchen colander used while boiling pasta. Every single piece of pasta and every drop of water goes through the same colander, and the colander applies one uniform physical test — "is this bigger than a hole?" — independently to everything poured through it. Water passes through; pasta doesn't. Nothing about the pasta or the water was altered by the colander; some of it was simply kept, and the rest let go. What you do with the pasta afterward — how you plate it, which specific pieces of it you photograph for a menu — is a separate decision, made after straining, exactly as projection (the `SELECT` list) is a separate, later decision from filtering (`WHERE`).

## Why WHERE Was Designed This Way

SQL is declarative: you state the *condition* a row must satisfy, not a step-by-step procedure for testing rows one at a time. `WHERE` is the clearest expression of that idea in the whole language — a single boolean expression describes *what qualifies*, and the database is entirely responsible for figuring out *how* to find the qualifying rows, whether that means scanning everything or using an index to skip straight to them. This mirrors exactly the physical/logical independence Module 2 traces back to Codd's original 1970 proposal: your `WHERE` clause's meaning never changes as the table grows from 10 rows to 10 million, or as the database's internal strategy for finding matching rows evolves — only the performance characteristics do.

## Advantages

- **Expresses intent, not mechanism** — a `WHERE` clause states exactly what a qualifying row looks like, without requiring you to write a manual loop or comparison procedure.
- **Uniform across `SELECT`, `UPDATE`, and `DELETE`** — the same `WHERE` syntax and semantics used here to filter what a query *returns* is used, unchanged, to decide what an `UPDATE` or `DELETE` *affects* (covered in Module 6).
- **Composable with projection** — because filtering and projection are independent operations, you can freely mix "which rows" and "which columns" without either affecting the other's logic.

## Disadvantages / Limitations

- **`WHERE` cannot filter on the result of an aggregate function** (like a computed total or average across a group of rows) — that requires the `HAVING` clause, covered in Module 9 (Aggregation), because aggregates aren't computed until after rows have already been grouped, which happens later than `WHERE` in the logical order.
- **A `WHERE` clause without any supporting index can be genuinely slow on very large tables**, since every row potentially needs to be individually tested — Module 13 (Indexes) and Module 20 (Performance Tuning) cover how to avoid this at scale.
- **A `WHERE` clause can reference any column that exists in `FROM`'s underlying rows, but never a `SELECT`-list alias** — a specific, easy-to-forget consequence of the logical order, illustrated above.

## Best Practices

- Filter as precisely as the question actually requires — a `WHERE` clause that's too loose returns rows a caller then has to filter again elsewhere, defeating the purpose of pushing the decision into the database.
- Don't repeat a `WHERE` clause's job in application code "just in case" — if the condition genuinely defines which rows are wanted, express it once, in the query, and trust the result.
- When you need to filter on a computed value, either repeat the expression in `WHERE` (as shown above) or, once later modules introduce them, use a Common Table Expression (Module 17) to name the computation once and filter on that name afterward.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Trying to filter on an aggregate (e.g., a category's average price) using `WHERE` | `WHERE` operates on individual rows before any grouping/aggregation happens; filtering on an aggregate result requires `HAVING` (Module 9), which runs after grouping. |
| Referencing a `SELECT`-list alias inside the same query's `WHERE` clause | `WHERE` is evaluated before the `SELECT` list, per the logical processing order — the alias doesn't exist yet; repeat the underlying expression instead. |
| Assuming `WHERE` and `SELECT`'s column list do the same kind of job | `WHERE` decides *which rows* survive (selection); the `SELECT` list decides *which columns* appear (projection) — they operate on entirely different axes of the result and can be combined freely. |
| Believing rows are permanently altered or removed from the table by a `SELECT ... WHERE` | A `SELECT`, even with a very restrictive `WHERE`, only reads and returns data — nothing in the underlying table changes (Module 1's DQL vs. DML distinction). |

## Interview Questions

1. **Q: What does the `WHERE` clause conceptually do to each row in a table?**
   A: It evaluates a boolean expression against every row produced by `FROM`, independently, and keeps only the rows for which that expression is `TRUE` — rows evaluating to `FALSE` (or `UNKNOWN`, in the presence of `NULL`) are excluded from the result.

2. **Q: What's the difference between filtering and projecting?**
   A: Filtering (`WHERE`) chooses which *rows* survive into the result, based on a boolean condition. Projecting (the `SELECT` list) chooses which *columns* (or computed expressions) appear for each surviving row. They're independent, composable operations acting on different axes of the data.

3. **Q: Why can't a `WHERE` clause reference a column alias defined in that same query's `SELECT` list?**
   A: Because of SQL's logical processing order, `WHERE` is evaluated before the `SELECT` list is computed — the alias simply doesn't exist as a named value at the point rows are being filtered. You need to repeat the underlying expression in `WHERE` instead of the alias.

## Summary

- `WHERE` sits between `FROM` and `ORDER BY` and evaluates a single boolean expression once per row, keeping only rows where it's `TRUE`.
- Filtering (`WHERE`, choosing rows) and projecting (the `SELECT` list, choosing columns) are independent relational operations that can be freely combined.
- A `WHERE` clause can reference any column of the underlying row, even one that never appears in the `SELECT` list — because filtering runs on the full row, before projection discards anything.
- A `WHERE` clause cannot reference a `SELECT`-list alias, because `WHERE` is logically evaluated before the `SELECT` list computes that alias.
- The exact rules for combining multiple conditions inside `WHERE` — comparison operators, `AND`/`OR`/`NOT`, and precedence — are the subject of the next topic.
