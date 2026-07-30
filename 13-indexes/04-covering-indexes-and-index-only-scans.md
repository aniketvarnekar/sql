# Covering Indexes and Index-Only Scans

## Learning Objectives

By the end of this section you should be able to:
- Define a covering index as one that contains every column a specific query needs
- Explain why a covering index lets PostgreSQL skip the table (heap) entirely for that query
- Use PostgreSQL's `INCLUDE` clause to add non-key columns to an index
- Explain the difference between an Index Scan and an Index Only Scan, and why the latter can be dramatically faster
- Recognize the trade-offs of widening an index purely to make it covering

## Prerequisites

- [Clustered vs. Non-Clustered Indexes](03-clustered-vs-non-clustered-indexes.md) — you need to understand that every PostgreSQL index lookup normally requires an extra hop from the index into the table's heap, because that exact extra hop is what this topic shows you how to sometimes eliminate.
- [B-Tree and Composite Indexes](02-b-tree-and-composite-indexes.md) — you need to understand composite indexes and the leftmost-prefix rule, since the `INCLUDE` clause introduced here is best understood as an extension of a composite index's idea.

## Motivation

Topic 3 established that a normal PostgreSQL index lookup is a two-step process: find the entry in the index, then fetch the matching row from the heap. For a query that only touches a handful of matching rows, that extra heap hop is a minor cost. But for a query that matches a large number of rows — a report scanning tens of thousands of orders, say — that second hop, repeated once per matching row, can dominate the query's total cost, especially if those rows are scattered across many different physical disk locations. If there were a way to avoid that second hop altogether, a large class of read-heavy queries could become dramatically faster. There is such a way, and it's a natural extension of the composite index idea from Topic 2.

## Problem Statement

Consider a composite index built to speed up a common support-dashboard query:

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

Now a report runs frequently:

```sql
SELECT status, total_amount
FROM orders
WHERE customer_id = 42 AND status = 'shipped';
```

The index fully satisfies the `WHERE` clause (a complete leftmost-prefix match, per Topic 2) — it can find exactly the matching rows' locations extremely quickly. But look closely at the `SELECT` list: it asks for `total_amount`, and `total_amount` is not part of this index at all. So for every matching row the index identifies, PostgreSQL must still perform the extra hop into the heap (Topic 3) just to retrieve the `total_amount` value the query actually asked for. The index did excellent work narrowing down *which* rows match — but the query still can't avoid touching the table itself, because one requested column lives only there.

## Concept

### Covering Indexes

An index is called a **covering index** for a specific query when it contains *every* column that query needs — every column in the `WHERE` clause, and every column in the `SELECT` list (and any other clause referencing a column, such as `ORDER BY`). When an index covers a query completely, PostgreSQL can answer the entire query using only the index — the heap never needs to be consulted at all (with one caveat covered below).

### The `INCLUDE` Clause

PostgreSQL lets you extend an index with additional columns that are stored in the index purely as extra payload — not as part of the sorted search key, but simply carried alongside it so a covering query can read their values directly from the index:

```sql
CREATE INDEX idx_orders_customer_status_covering
    ON orders (customer_id, status)
    INCLUDE (total_amount);
```

| Piece | Meaning |
|---|---|
| `(customer_id, status)` | The index's actual **key columns** — sorted, leftmost-prefix-searchable, exactly as in Topic 2. |
| `INCLUDE (total_amount)` | `total_amount` is stored in the index's leaf entries too, but it is **not** part of the sort order and is not usable for the leftmost-prefix rule — you cannot search or filter efficiently on an `INCLUDE`d column the way you can on a key column. |

With this index in place, the earlier query now has everything it needs sitting inside the index itself:

```sql
SELECT status, total_amount
FROM orders
WHERE customer_id = 42 AND status = 'shipped';
```

```
  status  | total_amount
----------+--------------
 shipped  |       142.50
(1 row)
```

`customer_id` and `status` are used to find the matching entry (key columns); `status` and `total_amount` — everything the `SELECT` list asks for — are readable directly from that same entry, with no need to fetch anything from the heap at all.

### Why Distinguish Key Columns from `INCLUDE`d Columns?

It might seem simpler to just make every useful column a key column instead of bothering with `INCLUDE`. There are real reasons to keep the distinction:

- **`INCLUDE`d columns don't bloat the sorted structure's comparisons.** The B-tree only has to compare key columns to navigate and maintain sort order; extra payload columns add storage but not comparison overhead.
- **`INCLUDE`d columns are excluded from uniqueness checks.** If this index also enforced a `UNIQUE` constraint, only the key columns would be checked for uniqueness — `INCLUDE`d columns can repeat freely without violating it. This matters when you want a unique constraint on `(customer_id, status)` alone while still wanting `total_amount` available for a covering index — declaring `total_amount` as a key column would incorrectly fold it into the uniqueness check.
- **It keeps the index's real search semantics honest.** Anyone reading `(customer_id, status) INCLUDE (total_amount)` can immediately tell which columns actually participate in searching and sorting versus which are just carried along as payload — an index with five key columns invites (incorrect) assumptions that any of the five can be searched via the leftmost-prefix rule.

### Index Only Scans

When the query planner recognizes that an index fully covers a query, it chooses a plan called an **Index Only Scan** rather than an ordinary **Index Scan**. The distinction (which Topic 5 will show you how to see directly in `EXPLAIN` output) is:

| Plan type | What it does |
|---|---|
| **Index Scan** | Search the index to find matching entries, then fetch each matching row from the heap to retrieve any column not present in the index. |
| **Index Only Scan** | Search the index to find matching entries, and read every requested column's value directly from the index itself — no heap fetch needed. |

An important, honest caveat: even an Index Only Scan can still sometimes need to peek at the heap. PostgreSQL's MVCC model (Module 14 covers this in depth) means a row's visibility to a given transaction isn't recorded in the index — it's tracked via the heap. PostgreSQL maintains a **visibility map** that records which heap pages are known to contain only rows visible to everyone, letting it skip the heap safely for those pages. If a table's visibility map is out of date (typically because recently changed pages haven't been processed by `VACUUM` yet), an otherwise-eligible Index Only Scan may still need to check the heap for those specific pages. In practice, a table that is `VACUUM`ed regularly (PostgreSQL does this automatically via autovacuum in most default configurations) gets the full benefit of Index Only Scans consistently.

## Internal Working (Preview)

```
Ordinary Index Scan for SELECT status, total_amount ... WHERE customer_id = 42 AND status = 'shipped':

  1. Search index (customer_id, status) → find matching entry
  2. Entry has TID pointing into heap
  3. Fetch heap page at that TID  ← extra I/O, repeated per matching row
  4. Read total_amount from the fetched heap row
  5. Return (status, total_amount)

Index Only Scan, same query, using (customer_id, status) INCLUDE (total_amount):

  1. Search index → find matching entry
  2. status and total_amount are BOTH already sitting in this entry
  3. (Check visibility map — if page is all-visible, skip the heap entirely)
  4. Return (status, total_amount) straight from the index entry
```

For a query matching one row, skipping step 3 saves a single heap fetch — small in absolute terms. For a report matching fifty thousand rows scattered across the heap, skipping fifty thousand separate heap fetches is often the difference between a query that takes seconds and one that takes milliseconds — which is exactly why covering indexes matter most for read-heavy, high-row-count queries rather than simple single-row lookups.

## Real-World Analogy

Recall the index card catalog from earlier topics, sorted by call number, telling you which shelf and shelf-position a book sits at. An ordinary index card only tells you *where to walk* — you still have to physically go to the shelf and pull the book to find out anything else about it (its page count, its condition, whether it's checked out). Now imagine the card itself has been extended to also print the book's page count and current availability directly on the card. If all you needed to know was the page count and availability, you'd never have to walk to the shelf at all — you could answer the question standing at the card catalog. That's a covering index: the "extra" information printed on the card is exactly like an `INCLUDE`d column, and never having to walk to the shelf is exactly like an Index Only Scan.

## Why Covering Indexes and `INCLUDE` Were Designed This Way

The relational engine's fundamental cost driver, for any plan involving an index, is how many times it has to jump between the sorted index structure and the unordered heap (Topic 3) — each jump is a potential random-access disk or memory read, the single most expensive operation type in query execution. `INCLUDE` exists to let you deliberately eliminate that jump for specific, known, important queries, without forcing those extra columns into the index's sort key (where they'd add comparison overhead and incorrectly participate in any uniqueness enforcement the index provides). This is a direct, pragmatic response to the trade-off established back in Topic 3: since PostgreSQL's heap-plus-secondary-index model always costs an extra hop by default, `INCLUDE` gives you a targeted tool to buy that hop away, for exactly the queries where it's worth the extra index storage.

## Advantages

- **Eliminates the heap-fetch hop entirely for covered queries** — the single biggest cost reduction available for read-heavy queries that touch many rows.
- **Keeps sort-key semantics clean** — `INCLUDE`d columns add read-side benefit without adding search-key comparison cost or being (incorrectly) subject to a uniqueness check.
- **Composable with everything from Topic 2** — a covering index is still an ordinary composite B-tree for its key columns; the leftmost-prefix rule still applies identically to the key portion.

## Disadvantages / Limitations

- **Wider indexes cost more storage and more write overhead** — every `INCLUDE`d column is additional data stored in every index entry, and every `INSERT`/`UPDATE` touching any included column must still update the index (Topic 1's cost trade-off applies here just as much as to key columns).
- **Only benefits the specific queries it was designed for** — a covering index for one query's exact `SELECT` list provides no covering benefit to a differently-shaped query needing a column that wasn't included.
- **Doesn't eliminate the visibility-map caveat** — an Index Only Scan can still fall back to checking the heap on pages whose visibility map isn't up to date, so the benefit, while usually substantial and consistent in practice, isn't an absolute guarantee in every situation.

## Best Practices

- Reach for `INCLUDE` only for genuinely important, frequently-run, read-heavy queries where `EXPLAIN` (Topic 5) shows the heap-fetch step is a real, measurable cost — not as a default habit on every index.
- Keep `INCLUDE`d columns limited to what specific queries actually project (`SELECT`) — don't include "everything, just in case," which only adds storage and write cost without a corresponding query to benefit.
- Remember that `INCLUDE`d columns cannot be used for filtering via the leftmost-prefix rule — if a column needs to be searched on, it belongs as a key column, not as an `INCLUDE`d one.
- Let autovacuum run normally (or ensure regular `VACUUM`ing on write-heavy tables) so the visibility map stays current and Index Only Scans get their full benefit.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming any query touching an indexed column automatically gets an Index Only Scan | The index must contain *every* column the query needs (in the `SELECT` list and any other clause) — if even one requested column is missing from the index, PostgreSQL must fall back to an ordinary Index Scan with a heap fetch. |
| Treating `INCLUDE`d columns as searchable like key columns | `INCLUDE`d columns are payload only — they are not part of the sorted key and cannot be used to narrow a search via the leftmost-prefix rule. |
| Making every column of a table part of one giant covering index | This defeats the purpose of narrowing an index to a specific query's needs, ballooning storage and write cost while providing no benefit for queries that don't use all those columns. |
| Assuming an Index Only Scan never touches the heap under any circumstances | It can still check the heap for pages not yet marked all-visible in the visibility map — the benefit is usually large and consistent, but it isn't an absolute, unconditional guarantee. |

## Interview Questions

1. **Q: What makes an index a "covering index" for a particular query?**
   A: An index covers a query when it contains every column that query references — every column used in the `WHERE` clause (or other filtering/sorting clauses) and every column in the `SELECT` list. When an index fully covers a query, PostgreSQL can answer the query using only the index, without fetching anything from the underlying table.

2. **Q: What does PostgreSQL's `INCLUDE` clause do, and how is it different from adding a column as an additional key column?**
   A: `INCLUDE` adds a column to an index purely as extra stored payload, readable directly from the index, without making it part of the sorted search key. Unlike a true key column, an `INCLUDE`d column can't be used to narrow a search via the leftmost-prefix rule, and it is excluded from any uniqueness enforcement the index provides — it exists solely to make the index cover more queries without changing the index's search semantics.

3. **Q: Why can an Index Only Scan be dramatically faster than an ordinary Index Scan for a query matching many rows?**
   A: An ordinary Index Scan still has to fetch each matching row from the table's heap to retrieve columns not present in the index — a separate, potentially random-access read for every matching row. An Index Only Scan retrieves every needed column directly from the index itself, skipping the heap fetch entirely; for a query matching thousands of rows, eliminating thousands of separate heap fetches can turn a multi-second query into a near-instant one.

4. **Q: Can an Index Only Scan ever still touch the table, even when the index technically covers the query?**
   A: Yes. PostgreSQL tracks row visibility (which transactions are allowed to see a given row version, under MVCC) via the heap, not the index. It maintains a visibility map recording which heap pages are known to hold only universally-visible rows; for pages not yet marked that way (typically because they haven't been processed by `VACUUM` recently), an Index Only Scan must still check the heap for those specific rows, even though the index otherwise contains all the requested columns.

## Summary

- A **covering index** contains every column a specific query needs, letting the database answer the entire query from the index alone.
- PostgreSQL's **`INCLUDE`** clause adds non-key "payload" columns to an index — readable directly, but not searchable via the leftmost-prefix rule and excluded from uniqueness checks.
- An **Index Only Scan** skips the heap-fetch hop (Topic 3) entirely for covered queries, which can dramatically speed up read-heavy queries touching many rows.
- An Index Only Scan can still occasionally check the heap if a page's row visibility isn't yet recorded as up to date in PostgreSQL's visibility map — usually a minor caveat in a well-maintained (regularly vacuumed) database.
- Reach for `INCLUDE` deliberately, for specific important queries confirmed (via `EXPLAIN`, Topic 5) to benefit — not as a blanket habit, since wider indexes still cost storage and write overhead.
