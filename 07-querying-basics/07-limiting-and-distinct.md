# LIMIT, OFFSET, and DISTINCT

## Learning Objectives

By the end of this section you should be able to:
- Use `LIMIT` and `OFFSET` to cap and paginate a result set
- Explain why `LIMIT` needs an accompanying `ORDER BY` to produce deterministic, repeatable results
- Explain the performance implications of a large `OFFSET`, at a high level
- Use `DISTINCT` to remove duplicate rows, and PostgreSQL's `DISTINCT ON` to keep one representative row per group
- State the complete logical processing order of a `SELECT` statement, start to finish

## Prerequisites

- [Sorting with ORDER BY](06-sorting-with-order-by.md) — `LIMIT`'s results are only meaningful once a deterministic order is established; this topic depends on that directly.

## Motivation

Real applications almost never want an entire result set dumped at once — a product listing page shows 20 items at a time, not 20,000; a "top 5" report wants exactly 5 rows, not everything sorted with the reader expected to stop reading manually. Just as often, a question only cares about *distinct* values ("what categories do we sell across?") rather than every individual row. `LIMIT`, `OFFSET`, and `DISTINCT` push exactly these two extremely common needs — capping/paginating, and deduplicating — into the database itself, rather than requiring every caller to fetch everything and post-process it.

## Problem Statement

A colleague building a paginated product listing page asks for "page 2, five products per page, cheapest first." Separately, another colleague, building a category filter dropdown for the same page, asks for "just the list of categories we actually sell — no repeats." Neither of these questions is answerable with anything covered so far in this module: `WHERE` narrows rows by condition, not by position or count, and nothing yet removes duplicate values from a projected column.

## Concept

### `LIMIT` — Capping the Row Count

```sql
SELECT name, price
FROM products
ORDER BY price ASC, id
LIMIT 5;
```

```
         name           | price
-------------------------+--------
 Uncategorized Widget    |   3.50
 Notebook Pack of 5      |   6.49
 Whiteboard Marker Set   |   8.75
 USB-C Cable             |   9.99
 Coffee Mug              |  11.25
(5 rows)
```

`LIMIT 5` returns at most the first 5 rows of whatever the rest of the query (crucially, including `ORDER BY`) would have produced. Without the `ORDER BY`, "the first 5 rows" has no defined meaning at all — the next section explains exactly why.

### `OFFSET` — Skipping Rows, for Pagination

`OFFSET n` skips the first `n` rows before `LIMIT` starts counting — combined, they implement pagination directly:

```sql
-- Page 1 (rows 1-5)
SELECT name, price FROM products ORDER BY price ASC, id LIMIT 5 OFFSET 0;
```

```
         name           | price
-------------------------+--------
 Uncategorized Widget    |   3.50
 Notebook Pack of 5      |   6.49
 Whiteboard Marker Set   |   8.75
 USB-C Cable             |   9.99
 Coffee Mug              |  11.25
(5 rows)
```

```sql
-- Page 2 (rows 6-10)
SELECT name, price FROM products ORDER BY price ASC, id LIMIT 5 OFFSET 5;
```

```
      name       | price
------------------+--------
 Stapler          |  12.00
 100% Cotton Towel |  15.00
 Wireless Mouse   |  24.99
 Desk Lamp        |  34.95
 Electric Kettle  |  45.00
(5 rows)
```

Page 2 picks up exactly where page 1 left off, because both queries share the identical `ORDER BY` — without that shared, deterministic order, there would be no guarantee the two pages don't overlap or skip a row between them.

### Why `LIMIT` Needs `ORDER BY` to Mean Anything Reliable

[Sorting with ORDER BY](06-sorting-with-order-by.md) established that row order is never guaranteed without an explicit `ORDER BY`. `LIMIT` without `ORDER BY` inherits that same problem directly: "the first 5 rows" of an unordered set is a meaningless, arbitrary selection that can change between runs of the identical query — after an update, after the table grows, or simply because the engine happened to choose a different scan strategy this time. A query like:

```sql
SELECT name, price FROM products LIMIT 5;
```

might return five perfectly reasonable-looking rows today and a *different* five tomorrow, with no error and no warning that anything changed — this is a correctness bug hiding behind output that looks superficially fine every time you happen to check it. **Never use `LIMIT` without a matching `ORDER BY` unless you genuinely don't care which rows come back at all** (a rare, specific case — like a quick sanity check that a table has *any* data in it).

### The Performance Cost of a Large `OFFSET`

`OFFSET` is convenient, but it is not a magic teleport to "row 1,000,000" — conceptually, the database still generally has to produce (or at least count through) every row up to the offset before it can start returning the ones you actually asked for, then discard everything before the offset. This means `OFFSET 10 LIMIT 10` is cheap, but `OFFSET 1000000 LIMIT 10` against a large table can be dramatically more expensive, even though both requests return the same number of rows. This is a genuine, well-known real-world performance trap for "deep pagination" (a user clicking all the way to page 500 of search results) — Module 20 (Performance Tuning) covers the standard alternative in depth: **keyset (or "seek method") pagination**, which remembers the last row seen and continues from there (`WHERE price > <last seen price> ORDER BY price LIMIT 10`) instead of counting through everything with `OFFSET`. For now, the important takeaway is simply that `OFFSET`'s cost grows with how deep into the result set it skips — it is not free regardless of size.

### `DISTINCT` — Removing Duplicate Rows

`DISTINCT`, placed immediately after `SELECT`, removes duplicate rows from the final projected result — operating on the **entire projected row as a whole**, not any single column in isolation:

```sql
SELECT DISTINCT category
FROM products
ORDER BY category;
```

```
     category
------------------
 Electronics
 Furniture
 Home Goods
 Kitchen
 Office Supplies

(6 rows)
```

Six distinct values come back — the five real category names plus one blank row, because `DISTINCT` treats every `NULL` category as a single group of its own (`Uncategorized Widget`'s `NULL` collapses to one row, not zero and not many). If two or more *columns* are projected, `DISTINCT` considers the whole combination together:

```sql
SELECT DISTINCT category, (price >= 100) AS is_premium
FROM products
ORDER BY category, is_premium;
```

This returns one row per distinct *combination* of `category` and `is_premium` that actually occurs — not one row per distinct `category` and a separate one per distinct `is_premium` independently.

### `DISTINCT ON` — PostgreSQL's "One Row Per Group" Shortcut

A very common real need is subtly different from plain `DISTINCT`: not "remove exact duplicate rows," but "keep exactly one row per group, chosen by some rule" — for instance, "the most expensive product in each category." PostgreSQL provides `DISTINCT ON (expression)` for exactly this, and it is not part of standard SQL — a PostgreSQL-specific extension (Module 22 covers portability):

```sql
SELECT DISTINCT ON (category) category, name, price
FROM products
ORDER BY category, price DESC;
```

```
    category     |         name          | price
------------------+------------------------+--------
 Electronics      | Mechanical Keyboard    |  89.99
 Furniture        | Standing Desk          | 349.00
 Home Goods       | 100% Cotton Towel      |  15.00
 Kitchen          | Blender_Pro 2000       |  79.99
 Office Supplies  | Stapler                |  12.00
                  | Uncategorized Widget   |   3.50
(6 rows)
```

`DISTINCT ON (category)` groups rows by `category` and keeps only the **first** row PostgreSQL encounters within each group, according to the `ORDER BY` — which is exactly why the `ORDER BY` here starts with `category, price DESC`: sorting by `category` first groups them together, and `price DESC` within each group ensures the "first" row per group is the most expensive one. This is a strict requirement, not a suggestion: **`DISTINCT ON`'s expression(s) must appear as the leading expression(s) of `ORDER BY`**, or PostgreSQL raises an error:

```sql
SELECT DISTINCT ON (category) category, name, price
FROM products
ORDER BY price DESC;
```

```
ERROR:  SELECT DISTINCT ON expressions must match initial ORDER BY expressions
```

Without `category` leading the `ORDER BY`, PostgreSQL has no reliable way to determine which row is "first" within each `category` group, so it refuses to run the query at all rather than guess.

### The Complete Logical Processing Order of a SELECT

This module has been assembling this pipeline one stage at a time since Topic 1 — here it is complete, start to finish:

```
 FROM       → identify the source table(s) (and any joins, from Module 10)
     │
     ▼
 WHERE      → keep only rows where the condition is TRUE           (selection)
     │            cannot reference SELECT-list aliases
     ▼
 SELECT     → compute the requested columns/expressions             (projection)
     │            DISTINCT / DISTINCT ON dedupe here, on the projected rows
     ▼
 ORDER BY   → sort the resulting rows
     │            CAN reference SELECT-list aliases (unlike WHERE)
     ▼
 LIMIT/OFFSET → cap the row count and/or skip a leading portion
     │            meaningless/nondeterministic without ORDER BY above it
     ▼
 Final result set returned to the caller
```

Every topic in this module has been one stage of this exact pipeline: Topics 1–2 covered `FROM`/`WHERE`/`SELECT`'s core mechanics; Topics 3–5 covered everything that can go inside `WHERE`; Topic 6 covered `ORDER BY`; this topic closes with `LIMIT`/`OFFSET`/`DISTINCT`, the final stage. Two consequences worth restating because they trip people up in practice: `WHERE` can't see `SELECT`'s aliases (it runs first), while `ORDER BY` can (it runs after `SELECT`); and `LIMIT` only produces a meaningful, repeatable result when paired with an `ORDER BY` that fully determines row order.

## Internal Working (Preview)

For `DISTINCT`, the engine conceptually needs to compare every row against every other row to find duplicates — typically implemented internally as either a full sort (adjacent duplicate rows become trivial to spot once sorted) or a hash-based grouping, whichever the query planner judges cheaper for the specific data. For `LIMIT` combined with a supporting `ORDER BY` that an index already satisfies, PostgreSQL can sometimes stop scanning as soon as it has produced enough rows, without ever touching the rest of the table — a "top-N" optimization that only kicks in when the sort order lines up with an existing index (Module 13, Module 20). `OFFSET`, by contrast, generally has no equivalent shortcut: rows up to the offset still typically need to be produced (even if not returned to the caller) before the engine can start counting the ones you actually want.

## Real-World Analogy

`LIMIT`/`OFFSET` are like flipping to a specific page of an already-alphabetized phone book — "page 3" only means something consistent because the book is sorted; flipping to "page 3" of an unsorted, shuffled stack of index cards gives you an arbitrary handful that means nothing in particular and could be a completely different handful tomorrow. `DISTINCT` is like collapsing a mailing list down to one entry per household, when several family members were each separately listed with the exact same address. `DISTINCT ON` is like a company directory that lists only one representative photo per department — chosen by an explicit rule ("most senior," "first alphabetically") rather than arbitrarily, which is exactly why it demands the `ORDER BY` that defines "first" within each group.

## Why These Were Designed This Way

Pagination and deduplication are two of the most common real-world query needs, so SQL (and PostgreSQL specifically, for `DISTINCT ON`) provides direct, compact syntax for them rather than requiring every caller to fetch a full result set and post-process it in application code — consistent with the declarative philosophy established since Module 1: describe the shape of the result you want (a page of 10, a deduplicated list, one row per group), and let the database figure out how to produce it. `LIMIT`'s dependency on `ORDER BY` for meaningful results isn't a design flaw — it's a direct, unavoidable consequence of a relation being an unordered set (Module 2): "the first N rows" is simply not a well-defined question until an order has been imposed on the set. `DISTINCT ON`'s requirement that its expression lead `ORDER BY` exists for the same reason: PostgreSQL refuses to silently guess which row is "first" per group when the ordering doesn't actually establish that unambiguously.

## Advantages

- **`LIMIT`/`OFFSET` implement pagination and "top N" queries directly**, without pulling a full result set into an application just to trim it there.
- **`DISTINCT` removes duplicate rows concisely**, without a manual grouping step.
- **`DISTINCT ON` compactly solves the extremely common "one row per group, by some rule" problem** in a single, readable statement.

## Disadvantages / Limitations

- **`OFFSET` at scale is a genuine, well-documented performance trap** — its cost grows with how deep it skips, making naive deep pagination slow on large tables (Module 20 covers keyset pagination as the standard fix).
- **`LIMIT` without `ORDER BY` is nondeterministic** — not just unsorted, but genuinely subject to change between runs, which is a correctness risk, not merely a style concern.
- **`DISTINCT`/`DISTINCT ON` may require an internal sort or hash over the full result**, which isn't free on large result sets.
- **`DISTINCT ON` is PostgreSQL-specific** — code relying on it isn't portable to a database that doesn't support the same syntax (Module 22).

## Best Practices

- Never use `LIMIT` without a matching `ORDER BY` unless you genuinely don't care which specific rows come back.
- For deep pagination on large, performance-sensitive tables, be aware of `OFFSET`'s cost profile and consider keyset/cursor-based pagination (Module 20) instead of ever-increasing `OFFSET` values.
- Reach for `DISTINCT ON` deliberately, confirming you actually want "one row per group by this rule" — and always give it an `ORDER BY` that leads with the same expression(s), since PostgreSQL requires this and will error otherwise.
- Don't reach for `DISTINCT` reflexively to "clean up" a result set without understanding *why* duplicates are appearing in the first place — sometimes duplicate rows indicate a modeling or join issue (Module 10) worth fixing at the source instead.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `LIMIT` without `ORDER BY` and assuming the result is stable | Row order (and therefore which rows a `LIMIT` returns) is never guaranteed without an explicit `ORDER BY` — the same query can return different rows on different runs. |
| Assuming `DISTINCT` removes duplicates on a per-column basis | `DISTINCT` operates on the entire projected row as a combination — two rows are only removed as duplicates if every selected column matches between them. |
| Using `DISTINCT ON` with an `ORDER BY` that doesn't start with the same expression | PostgreSQL raises an explicit error (`SELECT DISTINCT ON expressions must match initial ORDER BY expressions`) — the ordering must define which row is "first" per group. |
| Assuming a large `OFFSET` is essentially free because it returns the same number of rows as a small one | `OFFSET`'s cost generally scales with how many rows it skips, not just how many it returns — deep pagination via `OFFSET` gets slower as the offset grows. |

## Interview Questions

1. **Q: Why is `ORDER BY` considered mandatory in practice whenever you use `LIMIT`?**
   A: Without an explicit `ORDER BY`, row order is entirely unspecified, so "the first N rows" `LIMIT` returns is arbitrary and can change between runs of the identical query. Pairing `LIMIT` with `ORDER BY` is what makes the result deterministic and repeatable.

2. **Q: What's the performance concern with using a large `OFFSET` for deep pagination, and what's an alternative approach?**
   A: Conceptually, the database generally still has to produce (or count through) all rows up to the offset before returning the requested rows, so cost grows with how deep the offset goes — making very deep pages slow on large tables. Keyset (seek-method) pagination — filtering with `WHERE` based on the last row seen instead of skipping with `OFFSET` — avoids this scaling problem (Module 20 covers it in depth).

3. **Q: What does `SELECT DISTINCT` operate on — an individual column or the whole projected row?**
   A: The whole projected row as a combination. Two rows are considered duplicates and collapsed into one only if every column in the `SELECT` list matches between them.

4. **Q: What does PostgreSQL's `DISTINCT ON` do, and what constraint does it place on `ORDER BY`?**
   A: It keeps exactly one row per distinct value (or combination) of its expression, choosing whichever row is "first" according to `ORDER BY`. PostgreSQL requires the `DISTINCT ON` expression(s) to be the leading expression(s) in `ORDER BY`, and raises an error otherwise, since it would otherwise have no defined way to determine which row counts as "first" within each group.

## Summary

- `LIMIT` caps the number of rows returned; `OFFSET` skips a leading portion first — together, they implement pagination.
- `LIMIT` only produces a deterministic, repeatable result when paired with an `ORDER BY` that fully determines row order; without one, the rows returned are not guaranteed to be the same across runs.
- A large `OFFSET` is not free — its cost generally grows with how many rows it skips, a genuine performance concern at scale (Module 20 covers the keyset-pagination alternative).
- `DISTINCT` removes duplicate rows based on the entire projected row; PostgreSQL's `DISTINCT ON` keeps one row per group according to a leading `ORDER BY` expression, and requires that expression to lead the `ORDER BY`.
- The complete logical processing order of a `SELECT` is `FROM → WHERE → SELECT → ORDER BY → LIMIT`/`OFFSET` — every topic in this module has been one stage of exactly this pipeline, and it's worth re-reading Topic 1's introduction of it now that every stage has been covered in full.
- This module's final topic, the [Module Summary](08-module-summary.md), consolidates everything covered across all seven topics.
