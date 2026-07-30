# Sorting with ORDER BY

## Learning Objectives

By the end of this section you should be able to:
- Sort query results by one or several columns, ascending or descending
- Explain why an explicit `ORDER BY` is the only reliable way to get a meaningful, repeatable row order at all
- Sort by column position as well as by name, and explain why one is more fragile than the other
- Control where `NULL` values sort using `NULLS FIRST`/`NULLS LAST`
- Sort by a computed expression that never appears in the `SELECT` list

## Prerequisites

- [SELECT and Projection](01-select-and-projection.md) — you need this module's logical processing order (`FROM → WHERE → SELECT → ORDER BY → LIMIT`) established there, since it explains a key asymmetry this topic relies on: `ORDER BY` runs *after* `SELECT`, unlike `WHERE`.
- [Filtering with WHERE](02-filtering-with-where.md) — sorting always applies to whatever rows survived filtering.

## Motivation

A result set with no defined order is only as useful as a stack of index cards dropped on the floor — everything you need might technically be there, but nobody can reliably say "give me the cheapest one" or "what came first" just by looking at whatever order it happened to land in. Nearly every real use of query results — a report, a leaderboard, a paginated list — depends on a specific, deliberate order, and `ORDER BY` is the only clause that guarantees one.

## Problem Statement

Run the same query against `products` twice in a row, with no `ORDER BY` at all:

```sql
SELECT name, price FROM products WHERE category = 'Kitchen';
```

Nothing in this statement says anything about order — and nothing guarantees the three Kitchen rows come back in the same sequence every time, on every version of PostgreSQL, on every run, especially once rows are updated, deleted, or the table grows large enough for the engine to choose a different execution strategy. A person building a report that needs "cheapest Kitchen item first, every time" cannot rely on this query at all as written — they need to say so explicitly.

## Concept

### Result Order Without `ORDER BY` Is Never Guaranteed

This is worth stating as plainly as possible: **a relation, in the relational model established in Module 2, is a set of tuples with no inherent order at all** — [Tables, Rows, and Columns](../02-relational-model/01-tables-rows-and-columns.md) covers this directly. Any order a query happens to return without `ORDER BY` is an accident of the current execution plan (how the engine happened to scan the table, in what sequence, this time), not a promise. It can, and does, change — after a bulk update, after the table grows enough to trigger a different scan strategy, after a routine maintenance operation, or simply between two runs of the identical query. If an order matters to you at all, `ORDER BY` is not optional politeness — it is the only thing standing between "reliable" and "coincidentally looked right during testing."

### Basic `ORDER BY`

```sql
SELECT name, price
FROM products
WHERE category = 'Kitchen'
ORDER BY price;
```

```
        name         | price
----------------------+--------
 Coffee Mug           |  11.25
 Electric Kettle      |  45.00
 Blender_Pro 2000     |  79.99
(3 rows)
```

Ascending order (`ASC`) is the default — lowest first for numbers, alphabetically first for text. Reverse it with `DESC`:

```sql
SELECT name, price
FROM products
WHERE category = 'Kitchen'
ORDER BY price DESC;
```

```
        name         | price
----------------------+--------
 Blender_Pro 2000     |  79.99
 Electric Kettle      |  45.00
 Coffee Mug           |  11.25
(3 rows)
```

### Sorting by Multiple Columns

Listing more than one column sorts primarily by the first, using each following column only to break ties among rows that are equal on everything before it:

```sql
SELECT name, category, price
FROM products
ORDER BY category ASC, price DESC;
```

```
         name           |    category     | price
-------------------------+-----------------+--------
 Mechanical Keyboard     | Electronics     |  89.99
 Wireless Mouse          | Electronics     |  24.99
 USB-C Cable             | Electronics     |   9.99
 Standing Desk           | Furniture       | 349.00
 Office Chair            | Furniture       | 199.50
 Desk Lamp               | Furniture       |  34.95
 100% Cotton Towel       | Home Goods      |  15.00
 Blender_Pro 2000        | Kitchen         |  79.99
 Electric Kettle         | Kitchen         |  45.00
 Coffee Mug              | Kitchen         |  11.25
 Stapler                 | Office Supplies |  12.00
 Whiteboard Marker Set   | Office Supplies |   8.75
 Notebook Pack of 5      | Office Supplies |   6.49
 Uncategorized Widget    |                 |   3.50
(14 rows)
```

Categories are grouped alphabetically (each column can have its own direction — here `category` is ascending while `price`, within each category, is descending), and `Uncategorized Widget`'s `NULL` category sorts last, which the next section explains.

### Sorting by Column Position vs. Column Name

`ORDER BY` also accepts a plain number, referring to the `SELECT` list's position rather than a column name:

```sql
SELECT name, category, price
FROM products
ORDER BY 2, 3 DESC;
```

This sorts by the 2nd `SELECT`-list column (`category`) then the 3rd (`price`, descending) — producing an identical result to the named version above. It works, but it's fragile: if the `SELECT` list is later reordered, or a column inserted or removed from it, the meaning of `ORDER BY 2, 3` silently changes to refer to whatever now occupies those positions, without any error. Naming columns explicitly in `ORDER BY` doesn't have this problem — the sort key stays correct regardless of how the `SELECT` list is edited later.

### `NULLS FIRST` / `NULLS LAST`

A `NULL` has no natural position in an ordering — it isn't "less than" or "greater than" any real value — so PostgreSQL has to pick a convention. By default, PostgreSQL sorts `NULL` as if it were **larger than every other value**: it sorts last in ascending order, and first in descending order. This default is exactly why `Uncategorized Widget` (whose `category` is `NULL`) landed at the very end of the `ORDER BY category ASC` result above.

You can override this explicitly:

```sql
SELECT name, discontinued_on
FROM products
ORDER BY discontinued_on, id;
```

```
          name           | discontinued_on
--------------------------+-------------------
 Electric Kettle          | 2023-06-15
 Desk Lamp                | 2024-11-01
 Wireless Mouse           |
 USB-C Cable              |
 Mechanical Keyboard      |
 Standing Desk            |
 Office Chair             |
 Notebook Pack of 5       |
 Stapler                  |
 Whiteboard Marker Set    |
 Coffee Mug               |
 Blender_Pro 2000         |
 100% Cotton Towel        |
 Uncategorized Widget     |
(14 rows)
```

```sql
SELECT name, discontinued_on
FROM products
ORDER BY discontinued_on NULLS FIRST, id;
```

```
          name           | discontinued_on
--------------------------+-------------------
 Wireless Mouse           |
 USB-C Cable              |
 Mechanical Keyboard      |
 Standing Desk            |
 Office Chair             |
 Notebook Pack of 5       |
 Stapler                  |
 Whiteboard Marker Set    |
 Coffee Mug               |
 Blender_Pro 2000         |
 100% Cotton Towel        |
 Uncategorized Widget     |
 Electric Kettle          | 2023-06-15
 Desk Lamp                | 2024-11-01
(14 rows)
```

The 12 still-active products (`discontinued_on IS NULL`) now come first, with the two actually-discontinued products pushed to the end — useful, for instance, for a report meant to surface "products still needing attention (no discontinuation recorded)" ahead of ones already resolved. (Both examples add `id` as a secondary sort key purely to make row order among ties fully deterministic and easy to follow — a good habit whenever a query's primary sort key alone doesn't uniquely determine order.)

### Sorting by an Expression Not in the `SELECT` List

The sort key doesn't have to be a projected column at all — any valid expression is allowed, computed per row purely for ordering purposes:

```sql
SELECT name, price
FROM products
ORDER BY (discontinued_on IS NOT NULL), id;
```

```
          name           | price
--------------------------+--------
 Wireless Mouse           |  24.99
 USB-C Cable              |   9.99
 Mechanical Keyboard      |  89.99
 Standing Desk            | 349.00
 Office Chair             | 199.50
 Notebook Pack of 5       |   6.49
 Stapler                  |  12.00
 Whiteboard Marker Set    |   8.75
 Coffee Mug               |  11.25
 Blender_Pro 2000         |  79.99
 100% Cotton Towel        |  15.00
 Uncategorized Widget     |   3.50
 Desk Lamp                |  34.95
 Electric Kettle          |  45.00
(14 rows)
```

`discontinued_on` never appears in the `SELECT` list — only `name` and `price` are projected — yet the boolean expression `discontinued_on IS NOT NULL` (`FALSE` for active products, `TRUE` for discontinued ones) is a completely valid sort key, since PostgreSQL sorts booleans `FALSE` before `TRUE`. This works because, per the logical processing order, `ORDER BY` runs *after* `WHERE` has already selected the full candidate rows — it isn't limited to only the columns the `SELECT` list chose to keep.

### A Key Asymmetry: `ORDER BY` *Can* Reference a `SELECT`-List Alias

[SELECT and Projection](01-select-and-projection.md) showed that a `WHERE` clause cannot reference an alias defined in the `SELECT` list, because `WHERE` runs before `SELECT` is ever computed. `ORDER BY` is different — it runs *after* the `SELECT` list, so it *can* reference a `SELECT`-list alias directly:

```sql
SELECT name, price * 1.08 AS price_with_tax
FROM products
WHERE category = 'Kitchen'
ORDER BY price_with_tax DESC;
```

```
        name         | price_with_tax
----------------------+-----------------
 Blender_Pro 2000     |         86.3892
 Electric Kettle      |         48.6000
 Coffee Mug           |         12.1500
(3 rows)
```

This single asymmetry — `WHERE` can't see `SELECT` aliases, `ORDER BY` can — is a direct, practical consequence of the logical order `FROM → WHERE → SELECT → ORDER BY → LIMIT` that this module has been building toward since Topic 1.

## Internal Working (Preview)

Updating this module's running pipeline diagram to include sorting:

```
 FROM      → identify source rows
 WHERE     → keep only qualifying rows           (WHERE cannot see SELECT's aliases)
 SELECT    → compute requested columns/expressions (projection)
 ORDER BY  → sort the projected rows              (CAN see SELECT's aliases — it runs after them)
 LIMIT     → cap the row count                    (covered next topic)
```

Sorting itself requires the engine to compare rows against each other, not just test each row independently the way `WHERE` does — which means, absent a supporting structure, the database generally has to gather all qualifying rows before it can produce a fully sorted order (you can't know a row is "first" until you've seen every candidate). If a **B-tree index** (Module 13) already exists on the sort column, PostgreSQL can sometimes read rows directly in that index's order and skip a separate sort step entirely — this is one of the most common, high-value uses of an index in practice, and Module 20 (Performance Tuning) covers recognizing it in an execution plan.

## Real-World Analogy

Think of a postal sorting facility. Letters arrive in whatever order trucks happened to drop them off — an order with no meaning at all. Sorting by ZIP code (a primary key), then by street name within a ZIP code (a secondary, tie-breaking key), imposes a deliberate, purposeful order on top of that arbitrary arrival sequence — and nothing about the letters changes in the process, only the order they're handed out for delivery. Assuming letters would naturally "come out sorted" just because that's how you needed them, without ever actually running them through the sorting machine, is exactly the mistake of assuming query results are ordered without an explicit `ORDER BY`.

## Why ORDER BY Was Designed This Way

Because a relation is formally a set with no inherent order (Module 2), *something* has to exist to impose a meaningful sequence on results when one is needed — and SQL's design keeps that entirely separate from, and logically after, the decisions about which rows qualify (`WHERE`) and which columns to show (`SELECT`). This separation is deliberate: it lets the database engine store and retrieve rows in whatever physical order is most efficient internally, entirely free of any obligation to also make that the "correct" logical order — the physical/logical independence traced back to Codd's model in [The Relational Model and Codd's Rules](../02-relational-model/03-the-relational-model-and-codds-rules.md). `ORDER BY` running after `SELECT` (rather than before, or simultaneously) is also what allows it to reference computed aliases directly, since by the time sorting happens, those computed values already exist.

## Advantages

- **Flexible, multi-key sorting with independent directions per column** — a single clause expresses arbitrarily nuanced tie-breaking logic.
- **Can sort by data that isn't even part of the output** — as the boolean-expression example demonstrated, the sort key and the projected columns are entirely independent choices.
- **Explicit control over `NULL` placement** — `NULLS FIRST`/`NULLS LAST` resolves an otherwise ambiguous case deliberately, rather than leaving it to an implicit default a reader has to look up.

## Disadvantages / Limitations

- **Sorting isn't free** — without a supporting index, the engine generally must gather every qualifying row before it can produce a correctly ordered result, which costs more as row counts grow (Module 13, Module 20).
- **Sorting by position (`ORDER BY 2`) is fragile** — editing the `SELECT` list later silently changes what position 2 refers to, with no error to flag the change.
- **Without `ORDER BY` at all, row order is entirely unspecified** — not just "unsorted," but genuinely subject to change between runs, which is easy to overlook until it causes a real, hard-to-reproduce bug.

## Best Practices

- Name columns explicitly in `ORDER BY` rather than using their position — it survives edits to the `SELECT` list safely.
- Be explicit with `NULLS FIRST`/`NULLS LAST` whenever the default (`NULL`s sort as the largest value) isn't obviously what a report or feature actually needs.
- Never assume any output order without an explicit `ORDER BY`, even if a query has "always looked sorted" in testing — that appearance is coincidental, not guaranteed.
- Add a deterministic tie-breaking column (often a primary key) to any `ORDER BY` where the primary sort key alone could leave ties in an arbitrary order, especially before combining sorting with `LIMIT` (next topic).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a table "naturally" returns rows in insertion order without `ORDER BY` | Row order without an explicit `ORDER BY` is never guaranteed — it reflects the current execution plan, not a promise, and can change at any time. |
| Using `ORDER BY 1, 2` and then later reordering the `SELECT` list | The sort now silently refers to different columns than intended, with no error raised — always prefer explicit column names. |
| Forgetting `NULL`'s default sort placement and being surprised where `NULL` rows land | PostgreSQL sorts `NULL` as the largest value by default (last in `ASC`, first in `DESC`) — use `NULLS FIRST`/`NULLS LAST` explicitly whenever that default isn't what's needed. |
| Assuming `WHERE`'s alias restriction also applies to `ORDER BY` | `ORDER BY` runs after the `SELECT` list is computed and can reference its aliases directly — only `WHERE` (which runs before `SELECT`) cannot. |

## Interview Questions

1. **Q: Is result set order guaranteed without an explicit `ORDER BY`?**
   A: No. A relation is formally an unordered set of tuples; any apparent order without `ORDER BY` is an accidental byproduct of the current execution plan, not a guarantee, and can change between runs or as the table changes.

2. **Q: Can `ORDER BY` reference a column alias defined in the `SELECT` list? Why does this differ from `WHERE`?**
   A: Yes — `ORDER BY` runs after the `SELECT` list is evaluated (per the logical processing order `FROM → WHERE → SELECT → ORDER BY`), so the alias already exists by the time sorting happens. `WHERE`, by contrast, runs before `SELECT`, so it cannot reference a `SELECT`-list alias.

3. **Q: What determines where `NULL` values sort by default in PostgreSQL, and how would you override it?**
   A: PostgreSQL treats `NULL` as larger than any real value by default, so `NULL`s sort last in ascending order and first in descending order. `NULLS FIRST`/`NULLS LAST` after the sort expression overrides this explicitly, regardless of `ASC`/`DESC`.

4. **Q: What's a risk of using `ORDER BY 2` instead of `ORDER BY column_name`?**
   A: If the `SELECT` list is later edited — a column added, removed, or reordered — `ORDER BY 2` silently starts referring to whatever column now occupies that position, with no error, potentially breaking the intended sort order without any visible signal.

## Summary

- `ORDER BY` sorts the final result; `ASC` (default) or `DESC` can be set independently per sort column, and multiple columns break ties in the order listed.
- Sorting by column name is more robust than sorting by position, which silently breaks if the `SELECT` list is later edited.
- `NULL` sorts as the largest value by default in PostgreSQL; `NULLS FIRST`/`NULLS LAST` overrides this explicitly.
- A sort key can be any valid expression, including one that never appears in the `SELECT` list at all.
- `ORDER BY` runs after the `SELECT` list in SQL's logical processing order, which is exactly why it — unlike `WHERE` — can reference a `SELECT`-list alias.
- Without an explicit `ORDER BY`, row order is never guaranteed — the next topic, [LIMIT, OFFSET, and DISTINCT](07-limiting-and-distinct.md), depends on this fact directly.
