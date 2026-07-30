# What Is an Index?

## Learning Objectives

By the end of this section you should be able to:
- Explain why checking every row of a table (a full table scan) becomes a serious performance problem as a table grows
- Define an index as a separate, ordered data structure that maps column values back to the location of the rows that contain them
- Write `CREATE INDEX` and `DROP INDEX` statements with correct PostgreSQL syntax
- Explain why an index is not "free" — the storage and write-speed cost it imposes on every table it's attached to
- Decide, for a given column, whether adding an index is likely to help or is a waste

## Prerequisites

- [Module 5 — Constraints & Keys](../05-constraints-and-keys/00-module-overview.md) — you need to already understand what a primary key and a `UNIQUE` constraint guarantee (uniqueness, non-duplication), because this topic reveals that PostgreSQL enforces both of those guarantees using exactly the mechanism this topic introduces.
- [Your First Query](../01-introduction/05-your-first-query.md) — you need to be comfortable with `SELECT ... WHERE ... ORDER BY`, since every example in this topic is about making that exact kind of query fast.
- Module 7 (Querying Basics) and Module 10 (Joins & Set Operations): this topic assumes you already know that `WHERE` filters rows, `ORDER BY` sorts them, and a join matches rows between two tables — the entire point of an index is to make those three operations fast.

## Motivation

Every query you have written so far has almost certainly returned instantly. That is not because SQL is inherently fast — it's because every table you've practiced against has had a handful of rows. A database that has to check ten rows to answer `WHERE customer_id = 42` will do it in microseconds no matter how it's built. The same query against a table with fifty million rows, checking every single one, is a fundamentally different problem. Indexes are the single most consequential performance tool in a relational database, and understanding them is the difference between writing SQL that merely works and writing SQL that keeps working as your data grows by orders of magnitude.

## Problem Statement

Imagine an `orders` table for an online store, tracking every order ever placed:

```sql
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL,
    status       TEXT NOT NULL,
    total_amount NUMERIC(10, 2) NOT NULL
);
```

On day one, this table has 500 rows. A customer support agent runs:

```sql
SELECT order_id, order_date, status, total_amount
FROM orders
WHERE customer_id = 42
ORDER BY order_date DESC;
```

PostgreSQL has no way to know, in advance, which of the 500 rows belong to customer 42 — nothing about the table's physical storage groups rows by `customer_id`. So it does the only thing it can: it reads every row, checks whether `customer_id` equals 42, and keeps the ones that match. This is called a **full table scan** (PostgreSQL calls it a **Sequential Scan**, or **Seq Scan** — you'll see this term constantly once Topic 5 introduces `EXPLAIN`). With 500 rows, this finishes before you'd notice any delay at all.

Three years later, the store has processed 8 million orders. The exact same query now has to read and check 8 million rows to find the same customer's handful of orders — even though it's still just looking for the same tiny fraction of the table. The work the database does grows directly in proportion to the table's size, regardless of how few rows actually match. This is the core problem: **a full table scan's cost is linear in the size of the table** (computer scientists call this `O(n)` — the cost grows in direct, one-to-one proportion with the row count `n`). Scanned against a 500-row table, `O(n)` is invisible. Scanned against an 8-million-row table, it can mean multi-second (or worse) delays on a query a user expects to feel instant.

## Concept

### The Core Idea: A Separate, Ordered Lookup Structure

An **index** is a separate data structure, stored alongside a table, that holds a sorted copy of one or more of that table's columns, along with a pointer back to the exact physical location of the row each value came from. Instead of scanning the whole table to find `customer_id = 42`, PostgreSQL can instead:

1. Look inside the small, sorted index structure for the value `42` — because the index is sorted, this is a fast, targeted lookup (Topic 2 explains exactly why it's fast).
2. Follow the pointer(s) stored next to that value straight to the row(s) in the table that actually contain it.

Crucially, the table itself is untouched by this process except for the handful of rows the index says to fetch. The index does the hard work of narrowing millions of rows down to a handful; the table only has to hand over those few rows.

### Creating an Index

The basic syntax:

```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

Breaking this down:

| Piece | Meaning |
|---|---|
| `CREATE INDEX idx_orders_customer_id` | Create a new index, given the name `idx_orders_customer_id` (a naming convention, not a required format — PostgreSQL doesn't care what you call it, but `idx_<table>_<column(s)>` is a widely used, self-documenting pattern). |
| `ON orders (customer_id)` | Build the index against the `orders` table, indexing the `customer_id` column. |

After this statement runs, PostgreSQL builds the index by reading the entire table once, extracting every `customer_id` value along with a pointer to its row, and organizing that into a sorted structure (Topic 2 covers exactly what that structure looks like — a B-tree, PostgreSQL's default). From this point on, PostgreSQL's query planner has a *choice* for any query filtering on `customer_id`: scan the whole table, or use this index. As you'll see in Topic 5, the planner makes that choice automatically, based on cost estimates — you never explicitly tell a query "use this index."

Re-running the earlier query after this index exists:

```sql
SELECT order_id, order_date, status, total_amount
FROM orders
WHERE customer_id = 42
ORDER BY order_date DESC;
```

```
 order_id | order_date | status    | total_amount
----------+------------+-----------+--------------
     8842 | 2026-06-14 | delivered |       142.50
     6215 | 2025-11-02 | delivered |        58.00
     1190 | 2024-03-27 | delivered |        99.99
(3 rows)
```

The *result* is identical to what a full scan would have returned — an index never changes what a query returns, only how fast the database gets there. That guarantee matters: adding or removing an index is purely a performance decision, never a correctness decision.

### Dropping an Index

Indexes can be removed just as easily as they're created:

```sql
DROP INDEX idx_orders_customer_id;
```

This deletes the index structure entirely. Any query that relied on it for speed silently falls back to scanning the table — nothing breaks, but things that were fast may become slow again.

### Indexes Aren't Free

It's tempting, once you see how much an index can speed up a `SELECT`, to reach for "just index everything." This is wrong, and understanding why is as important as understanding why indexes help. Every index imposes two real, ongoing costs:

**1. Storage.** An index is not a lightweight annotation — it is a genuine copy of (at least) the indexed column's data, reorganized into its own structure, stored on disk right alongside the table. A B-tree index on a large text column, on a table with tens of millions of rows, can easily be tens or hundreds of megabytes in size, sometimes rivaling the size of the table itself. Ten indexes on the same table means ten separate structures, all consuming storage, all needing to be backed up, all needing to be loaded into memory to be useful.

**2. Write speed.** This is the less obvious but often more important cost. Every time a row is inserted, updated (on the indexed column), or deleted, PostgreSQL cannot just update the table — it must also update *every index* that includes the affected column, keeping each one correctly sorted and consistent. A table with no indexes has exactly one thing to update on `INSERT`: the table itself. A table with six indexes has to update the table *and* six separate sorted structures on every single `INSERT`. This directly slows down every `INSERT`, `UPDATE`, and `DELETE` (Module 6 covered these statements; this is the reason a heavily-indexed table can feel noticeably slower to write to).

This is the fundamental trade-off of indexing: **you are trading write speed and storage for read speed.** That trade is very often worth it — most real applications read far more than they write — but it is never automatic, and it is why "add an index to every column, just in case" is bad advice.

### When to Add an Index

An index earns its cost when the column it's built on is genuinely used to narrow down or order rows, frequently, on a table large enough for the difference to matter:

- **Columns frequently used in `WHERE`** — e.g., `customer_id` in the example above, or a `status` column filtered on constantly by a dashboard.
- **Columns frequently used in `JOIN` conditions** — the column on the "many" side of a foreign key relationship (Module 10 covers joins in depth) is one of the single most common and valuable places to add an index, since joins repeatedly look up matching rows by that column.
- **Columns frequently used in `ORDER BY`** — because a B-tree index stores values in sorted order already (Topic 2), sorting by an indexed column can sometimes be nearly free, since the database can walk the index in order instead of sorting the result set after the fact.

### When *Not* to Add an Index

- **Small tables.** If a table has a few hundred rows, a full scan finishes in a fraction of a millisecond regardless. The planner will frequently ignore a small table's index entirely and scan anyway, correctly judging that the overhead of consulting the index isn't worth it for so little data. Adding an index here buys you nothing and still costs storage and write overhead.
- **Columns rarely referenced in a query.** An index that's never used by any real query is pure cost with no corresponding benefit — it still slows down every write and still consumes storage.
- **Columns with very low selectivity.** *Selectivity* describes how well a value narrows down the rows that match it — a highly selective column (like `customer_id`, or `order_id`) has many distinct values, so filtering on one value eliminates almost all other rows. A column like a `boolean` (`is_deleted`, `is_active`) has only two possible values, so filtering on one of them still matches roughly half the table (or some other large, non-trivial fraction). An index on a low-selectivity column often doesn't help the planner at all: if half the table matches, it's frequently cheaper to just scan the whole table directly than to hop through an index one matching row at a time, and PostgreSQL's planner will usually make exactly that choice, leaving the index paying its storage/write cost while never actually being used for that query.

## Internal Working (Preview)

At a conceptual level, adding an index changes what the database has available when it plans a query, without changing the table itself at all:

```
Table "orders" (the heap — physical rows, no guaranteed order)
┌───────────┬─────────────┬────────────┬───────────┬──────────────┐
│ order_id  │ customer_id │ order_date │ status    │ total_amount │
├───────────┼─────────────┼────────────┼───────────┼──────────────┤
│   1190    │     42      │ 2024-03-27 │ delivered │    99.99     │
│   4501    │     7       │ 2024-04-02 │ delivered │    45.00     │
│   6215    │     42      │ 2025-11-02 │ delivered │    58.00     │
│   8842    │     42      │ 2026-06-14 │ delivered │   142.50     │
│   ...     │    ...      │    ...     │   ...     │    ...       │
└───────────┴─────────────┴────────────┴───────────┴──────────────┘

Index "idx_orders_customer_id" (separate, sorted by customer_id)
┌─────────────┬──────────────────────┐
│ customer_id │ pointer to row       │
├─────────────┼──────────────────────┤
│      7      │  → row for order 4501│
│     42      │  → row for order 1190│
│     42      │  → row for order 6215│
│     42      │  → row for order 8842│
│    ...      │  → ...               │
└─────────────┴──────────────────────┘
```

To find `customer_id = 42`, PostgreSQL searches the small, sorted index (fast — Topic 2 explains why), finds all three matching entries sitting together (because the index is sorted by `customer_id`), and follows each pointer straight to its row in the table. The table itself was never scanned end to end — only the three rows that actually mattered were touched.

## Real-World Analogy

Think of a large printed reference book with no index at all — to find every mention of a specific topic, you would have to read the entire book, page by page, front to back. Now imagine the same book with an index at the back: a sorted list of topics, each with the exact page numbers where it appears. You look up the topic in the (short, alphabetically sorted) index, and go directly to the handful of pages it names — you never touch the pages that don't matter. The index at the back of the book is a smaller, separate, sorted structure that points back into the book's actual content; that is exactly what a database index is, applied to rows instead of pages.

## Why Indexes Are Designed This Way

SQL is a **declarative** language (established in Module 1): you write `WHERE customer_id = 42` describing *what* you want, not *how* to find it. The relational model, going back to its original formulation, deliberately separates the logical question ("which rows satisfy this condition?") from the physical mechanism used to answer it — a table's rows have no guaranteed physical order, and a query never has to know or care whether an index exists. This separation is precisely what lets a database administrator add, tune, or drop an index at any time, on a live system, without changing a single line of application SQL — the query's *meaning* stays identical; only the *speed* of answering it changes. Indexes exist as an optional, bolt-on acceleration structure specifically because the relational model keeps "what data satisfies this query" strictly separate from "how is that data physically found" — the index lives entirely in the second category.

## Advantages

- **Turns linear scans into targeted lookups** — for a highly selective condition, the difference between scanning millions of rows and following a handful of pointers is frequently the difference between a multi-second query and a sub-millisecond one.
- **Speeds up `WHERE`, `JOIN`, and `ORDER BY` simultaneously** — a single well-chosen index can help all three, since they all fundamentally rely on finding or ordering rows by a column's value.
- **Completely transparent to query results** — an index changes performance, never correctness; you can add and remove indexes freely without ever rewriting a query.
- **Enforces uniqueness efficiently** — PostgreSQL uses an index internally to enforce every `PRIMARY KEY` and `UNIQUE` constraint (Module 5), because checking "does this value already exist?" is exactly the fast lookup an index is built for.

## Disadvantages / Limitations

- **Storage cost** — every index is a genuine additional data structure occupying disk space, sometimes substantial for large tables or wide indexed columns.
- **Write-speed cost** — every `INSERT`, `UPDATE` (of an indexed column), and `DELETE` must additionally update every index touching that column, slowing down all write operations in proportion to how many indexes exist.
- **Diminishing and even negative returns on the wrong columns** — an index on a low-selectivity or rarely-queried column adds ongoing cost with little or no corresponding read benefit.
- **Not automatically used just because it exists** — the planner may still choose a full scan over a poorly-suited or low-selectivity index (Topic 5 shows exactly how to check what the planner actually chose).

## Best Practices

- Index the columns your real queries actually filter, join, or sort on — derive index decisions from your application's actual query patterns, not from guessing.
- Prefer indexing foreign key columns used in joins — this is one of the highest-value, most broadly applicable indexing decisions in ordinary schema design.
- Periodically review whether an index is still being used (PostgreSQL exposes usage statistics via its system catalogs) and drop indexes that never get used — an unused index is pure cost.
- Don't index a column purely because it appears in a `SELECT` list — indexes accelerate *finding* and *ordering* rows, not merely reading a column's value once a row is already located (Topic 4 revisits this distinction with covering indexes).
- Add indexes deliberately, one at a time, and measure — don't add a large batch of speculative indexes without confirming each one is pulling its weight.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Adding an index to every column "just in case" | Every index has a real, ongoing storage and write-speed cost; an index that's never used by a query is pure overhead with zero benefit. |
| Indexing a boolean or other very-low-selectivity column expecting a speedup | If a value matches a large fraction of the table, the planner usually finds a full scan cheaper than hopping through an index one row at a time — the index often won't even be used. |
| Assuming a newly created index is immediately used by every existing query | The planner decides whether to use an index based on cost estimates and up-to-date table statistics; a stale statistics snapshot or a small table can lead the planner to skip an index that would otherwise help. |
| Believing an index changes what a query returns | An index only ever changes *how fast* a correct result is produced — the result set from indexed and unindexed execution of the same query is always identical. |

## Interview Questions

1. **Q: In your own words, what is a database index, and why does it make lookups faster?**
   A: An index is a separate, ordered data structure built from one or more columns of a table, where each stored value is paired with a pointer back to the row it came from. Because the index is sorted and much smaller/more targeted than scanning the whole table, the database can locate matching values quickly and then follow the pointers directly to the relevant rows, avoiding the need to examine every row in the table.

2. **Q: Why shouldn't you add an index to every column in every table?**
   A: Every index costs storage (it's a genuine additional structure on disk) and slows down writes, because every `INSERT`, `UPDATE`, or `DELETE` that touches an indexed column must also update that index to keep it consistent. If a column isn't actually used to filter, join, or sort in real queries, an index on it provides no read benefit while still paying the full storage and write cost — a net loss.

3. **Q: A table has a `boolean` column called `is_premium_customer`, and roughly 40% of rows have it set to `true`. Would you index this column? Why or why not?**
   A: Generally no. A boolean column has very low selectivity — filtering on `true` still matches a large fraction (40%) of the table. For low-selectivity conditions like this, a full table scan is frequently cheaper than an index lookup that would still have to fetch a huge share of the table's rows one pointer at a time, so the planner is likely to ignore such an index anyway, leaving it as pure overhead.

4. **Q: Does creating an index change what a `SELECT` statement returns?**
   A: No. An index only ever affects how the database internally finds the rows that satisfy a query — the result set is always identical whether or not a relevant index exists. Indexes are purely a performance mechanism, never a correctness mechanism.

## Summary

- A **full table scan** examines every row and costs time in direct proportion to table size — fine on small tables, a serious bottleneck at scale.
- An **index** is a separate, sorted structure holding column values paired with pointers back to their rows, letting the database jump directly to matching rows instead of scanning everything.
- `CREATE INDEX idx_name ON table (column);` builds one; `DROP INDEX idx_name;` removes it — neither ever changes what a query returns, only how fast it runs.
- Indexes cost real storage and slow down every write that touches an indexed column — they are a deliberate trade-off of write speed for read speed, not a free performance upgrade.
- Add indexes to columns genuinely used in `WHERE`, `JOIN`, and `ORDER BY` on tables large enough to matter; avoid indexing small tables, rarely-queried columns, and low-selectivity columns like booleans.
