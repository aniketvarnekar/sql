# Clustered vs. Non-Clustered Indexes

## Learning Objectives

By the end of this section you should be able to:
- Define the general distinction between a clustered index (data physically stored in index order) and a non-clustered index (a separate structure holding pointers to data stored elsewhere)
- State, precisely, that PostgreSQL has no true, automatically-maintained clustered index — every PostgreSQL index is non-clustered
- Explain what the `CLUSTER` command does, and why it is a one-time physical reordering rather than a maintained property of the table
- Recognize this as a genuine point of divergence between relational database products, and know where to go for the full cross-database comparison

## Prerequisites

- [B-Tree and Composite Indexes](02-b-tree-and-composite-indexes.md) — you need to already understand that an index is a structure separate from the table, containing entries that point back to rows (Topic 1's diagram), before the clustered/non-clustered distinction — which is entirely about *how that pointer relationship works physically* — will make sense.

## Motivation

Once you understand that an index points back to a table's rows, a natural question follows: what does the table's own data actually look like on disk, and does it have any order of its own? The answer to that question differs meaningfully between relational database products, and getting it wrong — assuming PostgreSQL behaves like a system you've read about elsewhere — leads to real misunderstandings about performance and about what an index is actually doing.

## Problem Statement

Suppose the `orders` table from earlier topics has a B-tree index on `order_date`, and you run:

```sql
SELECT * FROM orders
WHERE order_date BETWEEN '2026-01-01' AND '2026-01-31'
ORDER BY order_date;
```

The index quickly identifies exactly which rows fall in that date range (Topic 2). But then a physical question arises: are those rows sitting *next to each other* on disk, so fetching them is one efficient, sequential read — or are they scattered across many different, unrelated disk locations, so fetching them means many small, separate reads, one per matching row? The answer depends entirely on whether the table's physical storage is organized around this index or not. This is exactly the clustered vs. non-clustered distinction, and it has a real, measurable performance impact on precisely this kind of range query.

## Concept

### The General Distinction

- A **clustered index** is one where the table's actual row data is physically stored on disk in the same order as the index itself. There is no meaningful separation between "the index" and "the table" — the index's leaf level effectively *is* the table, row data and all. Because there is only one possible physical ordering for a table's rows, a table can have **at most one** clustered index.
- A **non-clustered index** (also called a **secondary index**) is a separate structure entirely: it stores sorted key values alongside a pointer to wherever the corresponding row actually lives, and the table's own row data can sit in any physical order, unrelated to the index's sort order. A table can have as many non-clustered indexes as it needs, because none of them constrain where the actual row data lives.

The practical consequence: looking up a row through a non-clustered index requires two steps — first find the entry in the index, then follow its pointer to fetch the row from wherever it actually lives (a step sometimes called a "heap fetch," since the extra hop can mean reading from a different, non-adjacent disk location for every matching row). Looking up a row through a genuine clustered index requires effectively one step, because finding the position in the index *is* finding the row's actual data — there's no separate hop to a different location.

### PostgreSQL Has No True Clustered Index

This is the precise, factual claim this topic exists to make: **every index PostgreSQL creates is a non-clustered (secondary) index.** PostgreSQL stores a table's row data in a structure called the **heap** — an unordered (or at least not index-maintained) collection of rows, each identified internally by a physical location called a **TID** (tuple identifier, sometimes shown as `ctid`). Every single PostgreSQL index — whether it's the index behind a `PRIMARY KEY`, a `UNIQUE` constraint, or a plain `CREATE INDEX` — stores sorted key values paired with a TID pointing into the heap. There is no PostgreSQL index type where the table's row data itself lives inside the index's own sorted structure. This is true even for the primary key: contrary to a common assumption carried over from experience with other database products, declaring a `PRIMARY KEY` in PostgreSQL does **not** cause the table's rows to be physically stored in primary-key order.

```
PostgreSQL's actual model — always this shape, for every index:

Index (sorted, small)              Heap / table (unordered pile of rows)
┌─────────────┬─────────┐          ┌──────┬───────────────────────────┐
│ key value   │ TID     │  ─────►  │ TID  │ actual row data           │
├─────────────┼─────────┤          ├──────┼───────────────────────────┤
│     7       │ (0,3)   │          │(0,1) │ order_id 6215, ...        │
│    42       │ (0,1)   │          │(0,2) │ order_id 4501, ...        │
│    55       │ (2,7)   │          │(0,3) │ order_id 1190, ...        │
└─────────────┴─────────┘          │(2,7) │ order_id 8842, ...        │
                                    └──────┴───────────────────────────┘
                Every lookup is: search the index, then hop to the heap.
```

### The `CLUSTER` Command

PostgreSQL does provide a command called `CLUSTER`, and its name is a frequent source of confusion — it does **not** create or maintain a true clustered index. What it actually does is physically rewrite the table's heap, one time, reordering the rows on disk to match the order of a chosen existing index:

```sql
CLUSTER orders USING idx_orders_customer_id;
```

Immediately after this runs, the table's physical row order happens to match `idx_orders_customer_id`'s sort order — for a moment, range and grouped lookups by `customer_id` benefit from better data locality (matching rows are more likely to sit near each other on disk). But this ordering is **not maintained** going forward. Every subsequent `INSERT` appends new rows whever there's free space in the heap, and every `UPDATE` (under PostgreSQL's row-versioning model) can write a new row version to a completely different physical location — neither operation reshuffles existing rows to preserve the clustered order. Over time, as normal write activity continues, the table's physical order gradually drifts back away from the index's order, and the benefit erodes. To restore it, `CLUSTER` has to be run again, from scratch, as a deliberate, one-time maintenance operation — it is not something PostgreSQL keeps up to date automatically the way it keeps a B-tree's own sort order up to date.

`CLUSTER` also has real operational costs worth knowing before reaching for it: it rewrites the entire table, requires enough free disk space to hold a second copy of the table during the operation, and takes an `ACCESS EXCLUSIVE` lock, blocking essentially all other access to the table for its duration (transactions and locking are covered in full in Module 14). For these reasons, it is generally used sparingly, on tables with a genuinely dominant, stable access pattern, rather than as a routine maintenance task.

## Internal Working (Preview)

Every lookup through a PostgreSQL index, even the one behind a `PRIMARY KEY`, follows the same two-hop shape shown in the diagram above: search the index's B-tree for the key, retrieve the TID stored there, then fetch the actual row from the heap at that TID. This is precisely the limitation that motivates the next topic: if a query only needs columns that already happen to be stored *inside* the index itself, the second hop to the heap can sometimes be skipped entirely — that mechanism, called an index-only scan, is the subject of Topic 4.

## Real-World Analogy

A clustered index is like a dictionary: the entries themselves — the words and their definitions — are physically printed on the page in alphabetical order. There is no separate lookup step; opening to the right page *is* finding the content. A non-clustered index is like the index at the back of a large reference book (the analogy already used in Topic 1): the index's entries are sorted alphabetically, but the actual content they point to lives on pages organized by chapter, in whatever order the book's chapters happen to be in, completely unrelated to alphabetical order. Looking something up always takes two steps: find the term in the sorted index, then flip to the (unrelated-order) page it names. PostgreSQL always works the second way — even the "index" behind a primary key is the back-of-the-book kind, never the dictionary kind.

## Why PostgreSQL Was Designed This Way

This design is a direct consequence of how PostgreSQL implements concurrent updates. PostgreSQL uses a strategy called MVCC (multi-version concurrency control — covered in depth in Module 14, Transactions & Concurrency), where updating a row typically doesn't modify it in place; instead, a new version of the row is written, and the old version is retained until it's no longer needed by any in-progress transaction. Physically maintaining a true clustered order — where every row must stay in a precise sorted position on disk — is fundamentally in tension with this approach: constantly inserting new row versions in exactly the right sorted slot, under concurrent access from multiple transactions, is far more complex and costly to maintain automatically than simply appending new versions and letting a separate, purpose-built B-tree structure handle sorted lookups. PostgreSQL's designers chose the heap-plus-secondary-indexes model consistently, rather than making an exception for one privileged index — which is exactly why even the primary key's index behaves like every other index, and why `CLUSTER` is offered only as an explicit, on-demand physical reorganization rather than an ongoing guarantee.

This is also one of the clearest places where relational database products genuinely diverge from one another: some other database systems maintain a true, always-up-to-date clustered index (very often built automatically from the primary key), where the data heap and one particular index really are the same physical structure. PostgreSQL deliberately does not work this way. **Module 22 (SQL Across Databases) is dedicated to exactly this kind of cross-vendor divergence** — treat this topic as the PostgreSQL-precise version of the concept, and Module 22 as the place to see how other systems' choices compare.

## Advantages

- **Simplicity and consistency** — every PostgreSQL index, including the primary key's, works identically (heap plus pointer-based lookup), with no special-cased "the one clustered index" to reason about differently from the rest.
- **Multiple indexes are all equally "first-class"** — because no index is privileged as *the* clustered one, a table can have several indexes, each optimized for a different query pattern, without one of them monopolizing the table's physical layout.
- **Compatible with PostgreSQL's concurrency model** — the heap-plus-secondary-index approach avoids the overhead of constantly re-sorting physical storage under concurrent multi-version writes.

## Disadvantages / Limitations

- **Every index lookup costs an extra hop** — searching an index and then fetching the actual row from the heap is inherently a two-step process, compared to a true clustered index's single step, which can matter for range-heavy queries that would otherwise benefit from strong data locality (Topic 4's index-only scan is PostgreSQL's answer to *part* of this cost).
- **`CLUSTER` is a one-time, unmaintained operation** — unlike a genuinely maintained clustered index, its benefit degrades as ordinary write activity continues, and restoring it requires an expensive, table-locking rewrite rather than being automatic.
- **A common source of cross-system confusion** — engineers with experience in a database product that does maintain a true clustered index (often assumed by default around the primary key) can carry incorrect assumptions into PostgreSQL, expecting physical row order to track the primary key when it never does by default.

## Best Practices

- Never assume a PostgreSQL table's physical row order matches its primary key or any other index's order — always treat row order as unspecified unless you have just run `CLUSTER` and no writes have occurred since.
- Reach for `CLUSTER` only for tables with a strong, stable, and dominant access pattern (e.g., a mostly-append-only, rarely-updated reporting table repeatedly range-scanned by one particular column) — and plan for its exclusive lock and disk-space requirements rather than running it casually.
- If a workload is fundamentally range-scan-heavy and data locality matters a great deal, consider this a genuine architectural decision, and consult Module 22 for how the problem is handled differently across database products before assuming PostgreSQL's `CLUSTER` is a complete substitute.
- Don't reach for `CLUSTER` as a routine tuning step before first checking, with `EXPLAIN` (Topic 5), whether the actual bottleneck is really about physical row locality at all.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a PostgreSQL primary key automatically clusters the table's data physically | PostgreSQL's primary key is implemented as an ordinary, non-clustered B-tree index over the heap, exactly like any other index — declaring it does not reorder the table's physical storage at all. |
| Believing `CLUSTER` keeps the table permanently sorted going forward | `CLUSTER` performs a one-time physical rewrite; subsequent inserts and updates are not kept in that order, so the physical ordering gradually drifts and must be restored by running `CLUSTER` again. |
| Running `CLUSTER` on a busy, frequently-written production table without planning for its lock | `CLUSTER` takes an `ACCESS EXCLUSIVE` lock and rewrites the entire table, blocking normal access for its duration — treating it as a lightweight, routine operation can cause a real outage on a busy table. |
| Assuming this behavior is universal across all relational databases | Whether a database maintains a true clustered index (and whether one is created automatically) genuinely varies by product — this is exactly the kind of divergence Module 22 covers in depth. |

## Interview Questions

1. **Q: Does PostgreSQL have clustered indexes?**
   A: Not in the sense of an automatically-maintained structure where table data is physically kept in index order. Every PostgreSQL index — including the one behind a primary key — is a non-clustered (secondary) index: a separate sorted structure holding pointers (TIDs) back into the table's heap, which itself has no guaranteed physical order. PostgreSQL does provide a `CLUSTER` command that performs a one-time physical reorder to match a chosen index, but that ordering is not maintained afterward.

2. **Q: What's the practical difference between looking up a row via a true clustered index versus via a PostgreSQL-style non-clustered index?**
   A: A true clustered index requires one step — the row's data lives directly in the index at the position found. A PostgreSQL-style non-clustered index requires two steps: find the entry in the index, then follow its pointer to fetch the actual row from the heap, which is a real extra hop, though PostgreSQL's index-only scans can sometimes avoid the second hop when the index alone already contains everything the query needs.

3. **Q: If you run `CLUSTER orders USING idx_orders_customer_id` today, will the table still be physically ordered by `customer_id` a month from now, after normal application traffic has continued?**
   A: Not reliably. New rows inserted after the `CLUSTER` operation are simply appended wherever there's free space, and updated rows can be written to entirely new physical locations under PostgreSQL's row-versioning model — neither preserves the clustered order. The physical ordering established by `CLUSTER` gradually degrades with ongoing writes and would need to be restored by running `CLUSTER` again.

4. **Q: Why might PostgreSQL's designers have chosen not to maintain a true clustered index automatically, when some other database products do?**
   A: PostgreSQL uses multi-version concurrency control (MVCC), where updates typically create new row versions rather than modifying rows in place. Continuously re-sorting the physical heap to preserve a strict clustered order under this model, with concurrent transactions, is significantly more complex and costly than treating all indexes uniformly as separate structures pointing into an unordered heap — a design choice that trades the single-hop lookup benefit of a true clustered index for simpler, more uniform concurrency handling.

## Summary

- A **clustered index** physically stores table data in index order (at most one per table, since data can only be sorted one way); a **non-clustered (secondary) index** is a separate structure of pointers into data stored elsewhere (a table can have many).
- **PostgreSQL has no true, automatically-maintained clustered index** — every index, including the one behind a primary key, is non-clustered, pointing from a sorted B-tree into an unordered heap via a TID.
- The **`CLUSTER` command** performs a one-time physical reorder of the heap to match a chosen index's order — it is not maintained going forward, degrades with ongoing writes, and requires an exclusive table lock to run.
- This is a genuine point of divergence between relational database products — some do maintain a true, automatic clustered index (often tied to the primary key); PostgreSQL deliberately does not, for reasons rooted in its MVCC concurrency model — see **Module 22 (SQL Across Databases)** for the full cross-vendor comparison.
- This limitation — every index lookup costing an extra hop to the heap — is exactly what motivates the next topic, covering indexes and index-only scans.
