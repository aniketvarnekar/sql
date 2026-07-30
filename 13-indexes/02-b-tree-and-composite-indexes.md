# B-Tree and Composite Indexes

## Learning Objectives

By the end of this section you should be able to:
- Describe, at a conceptual level, how a B-tree is structured (balanced, sorted, logarithmic lookup) and sketch a simple diagram of one
- Explain why the B-tree is PostgreSQL's default index type
- Create a composite (multi-column) index with correct PostgreSQL syntax
- State the leftmost-prefix rule and predict, for a given query, whether a specific composite index can be used
- Explain why column order inside a composite index definition is a meaningful design decision, not an arbitrary one

## Prerequisites

- [What Is an Index?](01-what-is-an-index.md) — this topic assumes you already understand the general idea of an index as a sorted structure pointing back to table rows, and the storage/write cost trade-off it introduces. This topic goes one level deeper: *what specific structure* PostgreSQL uses, and how that structure behaves once more than one column is involved.

## Motivation

Topic 1 established that an index is a "sorted structure," but glossed over exactly *how* something can be sorted and still be searched quickly, and said nothing about what happens once you want to index more than one column at a time — which is extremely common in real schemas (finding "this customer's shipped orders" needs both `customer_id` and `status` at once). Understanding the actual structure PostgreSQL uses, and the very specific rule that governs multi-column indexes, is what separates "I added an index" from "I added an index that the planner can actually use for my query."

## Problem Statement

A sorted list sounds like it should be trivially fast to search — and it is, *if* you search it correctly. Consider a sorted list of a million `customer_id` values. Naively scanning from the front until you find (or pass) the value you want is still, in the worst case, a full linear scan — no better than an unsorted table. What actually makes a sorted structure fast is the ability to repeatedly cut the remaining search space in half (or into many pieces at once), the way you'd look up a word in a physical dictionary: you don't start at "A" and read forward — you open to roughly the right spot, see you've overshot or undershot, and narrow in. A plain sorted list on disk doesn't give you that "jump to roughly the right spot" ability efficiently. PostgreSQL needs a structure that is sorted *and* organized so that narrowing in on any value takes very few steps, no matter how large the table grows. That structure is the B-tree.

## Concept

### What a B-Tree Actually Looks Like

A B-tree ("balanced tree") is a tree-shaped structure with three properties that matter enormously for indexing:

1. **Sorted** — at every level of the tree, values are kept in order.
2. **Balanced** — every leaf (the bottom-most nodes, where the actual pointers to table rows live) sits at exactly the same depth from the root. No value is "harder to find" than any other purely because of where it happens to sit in the tree.
3. **Shallow relative to its size** — because each node in the tree can hold many entries (not just one or two), the tree's height grows very slowly as the number of indexed values grows. Doubling the number of rows in a table does *not* double the number of steps needed to find a value — it typically adds, at most, one more level to the tree.

A simplified three-level B-tree indexing `customer_id` values might look like this:

```
                         ┌─────────────┐
                         │  Root node  │
                         │  [ 40 | 80 ]│
                         └──────┬──────┘
                ┌───────────────┼───────────────┐
                ▼                ▼               ▼
        ┌───────────┐     ┌───────────┐   ┌───────────┐
        │ [10 | 25] │     │ [55 | 70] │   │ [90 |110] │
        └─────┬─────┘     └─────┬─────┘   └─────┬─────┘
              ▼                 ▼               ▼
     ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
     │ 7 → row         │ │ 42 → row        │ │ 91 → row        │
     │ 10 → row        │ │ 55 → row        │ │ 95 → row        │
     │ 25 → row        │ │ 70 → row        │ │ 110 → row       │
     └─────────────────┘ └─────────────────┘ └─────────────────┘
        leaf nodes           leaf nodes           leaf nodes
```

To find `customer_id = 42`: start at the root, see that 42 falls between 40 and 80, follow the middle branch; at that internal node, see that 42 falls between 25 and 55 (there's a range implied by 25's sibling ordering) and follow down to the leaf holding 42; read the pointer stored right next to it straight to the matching row. Three quick comparisons found one value out of eleven shown here — and critically, the *same three-or-so comparisons* would find a value even if this tree indexed ten million rows instead of eleven, because each additional level of the tree lets it distinguish between exponentially more values. This "logarithmic" growth (formally `O(log n)`) is the mathematical reason a B-tree lookup barely slows down as a table grows from thousands to billions of rows, in sharp contrast to the full table scan's `O(n)` from Topic 1.

The leaf nodes are also linked to each other in sorted order (not shown above for simplicity), which is what lets PostgreSQL efficiently answer *range* queries too — `WHERE customer_id BETWEEN 40 AND 90` or `ORDER BY customer_id` can walk sideways across connected leaves in sorted order, rather than needing a fresh top-to-bottom search for every value in the range.

### Why B-Tree Is PostgreSQL's Default

PostgreSQL supports several index types (hash, GIN, GiST, BRIN, and others, each suited to specific specialized data — full-text search, geometric data, and so on), but the B-tree (`btree`) is the default for `CREATE INDEX`, because it is the only one that handles the entire common set of comparisons well:

```sql
-- All of these can use a plain B-tree index on total_amount:
SELECT * FROM orders WHERE total_amount = 99.99;
SELECT * FROM orders WHERE total_amount > 100;
SELECT * FROM orders WHERE total_amount BETWEEN 50 AND 150;
SELECT * FROM orders ORDER BY total_amount;
```

Equality, ranges (`<`, `>`, `<=`, `>=`, `BETWEEN`), and sorting are, by a wide margin, the most common operations in real-world queries — and a B-tree's sorted, linked-leaf structure supports all of them well out of the box. Because `btree` is the default, you don't need to specify it explicitly:

```sql
CREATE INDEX idx_orders_total_amount ON orders (total_amount);
-- exactly equivalent to:
CREATE INDEX idx_orders_total_amount ON orders USING btree (total_amount);
```

### Composite (Multi-Column) Indexes

A composite index (also called a multi-column or compound index) indexes more than one column together, in a single structure, rather than building a separate index per column:

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

Conceptually, this builds one sorted structure where rows are ordered *first* by `customer_id`, and *then*, within each `customer_id`, ordered by `status`:

```
customer_id | status     | → row
------------|------------|--------
      7     | delivered  | → ...
      7     | shipped    | → ...
     42     | cancelled  | → ...
     42     | delivered  | → ...
     42     | delivered  | → ...
     42     | shipped    | → ...
     55     | delivered  | → ...
```

Notice the sort order: every `customer_id = 42` row sits together, and *within* that group, rows are further sorted by `status`. This structure is exactly what makes the leftmost-prefix rule below true.

### The Leftmost-Prefix Rule

This is the single most important, and most commonly misunderstood, fact about composite indexes: **a composite index can be used to satisfy a query only if the query's conditions form an unbroken prefix of the index's column list, starting from the leftmost column.**

Given `idx_orders_customer_status ON orders (customer_id, status)`:

| Query condition | Can this index help? | Why |
|---|---|---|
| `WHERE customer_id = 42` | Yes | Uses just the leftmost column — a valid prefix. |
| `WHERE customer_id = 42 AND status = 'shipped'` | Yes, fully | Uses both columns in order — the complete prefix. |
| `WHERE status = 'shipped'` | No (not via this index) | Skips the leftmost column entirely — `status` values are only sorted *within* each `customer_id` group, so there's no single sorted run of all `status` values to search. |
| `WHERE customer_id = 42 AND total_amount > 100` | Partially | Can use the index to narrow to `customer_id = 42` (leftmost prefix satisfied), but must then check `total_amount` by examining each matching row directly, since `total_amount` isn't in this index at all. |

The reason is directly visible in the diagram above: the structure is sorted by `customer_id` first. All the `status = 'shipped'` rows are scattered across different `customer_id` groups rather than sitting together anywhere in the structure — there is no shortcut to find "all shipped rows, regardless of customer" using this particular index, so PostgreSQL would have to fall back to scanning it entirely (no better than scanning the table) or ignore it and scan the table directly.

### Why Column Order in the Definition Matters

Because of the leftmost-prefix rule, `(customer_id, status)` and `(status, customer_id)` are **not interchangeable** — they are two different structures, useful for two different sets of queries:

```sql
-- Good for: WHERE customer_id = ...  and  WHERE customer_id = ... AND status = ...
CREATE INDEX idx_a ON orders (customer_id, status);

-- Good for: WHERE status = ...  and  WHERE status = ... AND customer_id = ...
CREATE INDEX idx_b ON orders (status, customer_id);
```

A common, practical rule of thumb: put the column most often queried *alone*, or the column with the highest selectivity (Topic 1 defined selectivity), first — `customer_id`, with potentially millions of distinct values, is a far more useful leading column than `status`, which might only have five possible values (`pending`, `shipped`, `delivered`, `cancelled`, `returned`) and therefore narrows the search far less on its own. Designing a composite index means thinking concretely about which combinations of `WHERE` conditions your real queries actually use — not just listing "the columns this table has."

## Internal Working (Deep Dive)

Following a lookup for `customer_id = 42 AND status = 'shipped'` against `idx_orders_customer_status`:

```
1. Start at the root node of the B-tree.
2. Compare 42 against the root's separator values; follow the branch
   that could contain customer_id = 42.
3. Repeat at each internal level — each comparison eliminates a large
   fraction of the remaining tree (this is the O(log n) behavior).
4. Arrive at the leaf level. Because entries are sorted by
   (customer_id, status) together, all customer_id = 42 entries sit
   in a contiguous run, and within that run, they're further sorted
   by status — so status = 'shipped' within that run is also a
   quick, targeted lookup rather than a scan of every customer_id = 42 row.
5. Follow the pointer(s) at the matching leaf entries directly to the
   table rows.
```

Each internal node in a real PostgreSQL B-tree corresponds to a disk page (typically 8 KB) holding many separator entries — not just two or three as in the simplified diagrams above — which is exactly why the tree stays so shallow even for enormous tables: a single page can hold hundreds of entries, so each level of depth multiplies the number of distinguishable values by hundreds, not by two.

## Real-World Analogy

Think of a printed phone book sorted by last name, and then, within each last name, by first name — exactly like a composite index on `(last_name, first_name)`. If you know someone's last name, you can flip directly to roughly the right section instantly (leftmost prefix satisfied) — and if you also know their first name, you can narrow further within that last-name section just as quickly (full prefix satisfied). But if you only know someone's *first* name — "there's a Priya somewhere in this phone book, but I don't know her last name" — the sorting-by-last-name structure gives you no shortcut at all: Priyas are scattered across every letter of the alphabet, and you'd have to read the entire book to find them all. That's the leftmost-prefix rule, exactly: the phone book (and a composite index) is only a shortcut for the column(s) it was actually sorted by, starting from the first one.

## Why B-Trees and the Leftmost-Prefix Rule Exist This Way

A B-tree's balance and high fan-out (many entries per node) are a direct, deliberate answer to the mathematical problem in the Problem Statement: keep lookup cost growing logarithmically, not linearly, as data grows — a property that holds regardless of whether the table has a thousand rows or a billion. Composite indexes, and the leftmost-prefix rule that governs them, follow directly from choosing to store *one* physically sorted structure for multiple columns together rather than one structure per column: sorting necessarily has to pick a primary sort key first, then a secondary key within it, and so on — there is no way to sort by two columns "equally" at once. The leftmost-prefix rule is not an arbitrary limitation PostgreSQL imposes; it is the unavoidable, logical consequence of what "sorted by (A, B)" actually means as a physical ordering.

## Advantages

- **Logarithmic lookup cost** — a B-tree keeps searches fast even as a table grows from thousands to billions of rows, unlike the full table scan's linear cost from Topic 1.
- **Handles equality, ranges, and sorting in one structure** — the same index accelerates `=`, `<`, `>`, `BETWEEN`, and `ORDER BY`, covering the overwhelming majority of real-world query conditions.
- **One composite index can serve multiple related queries** — an index on `(customer_id, status)` speeds up both "find this customer's orders" and "find this customer's orders with this status," without needing two separate indexes.

## Disadvantages / Limitations

- **The leftmost-prefix rule is a hard constraint** — a composite index provides no shortcut at all for a query that only filters on a non-leading column, which can lead to confusing "why isn't my index being used?" situations if column order wasn't chosen deliberately.
- **A composite index is generally larger than a single-column index** — it stores more data per entry (both columns, for every row), and a B-tree indexing very wide or many columns can become a substantial structure in its own right.
- **Not every access pattern maps neatly onto one composite index** — a table queried in genuinely unrelated ways (sometimes by `customer_id`, sometimes independently by `status`, sometimes by neither together) may need more than one index, each paying its own storage/write cost (Topic 1).

## Best Practices

- Choose composite index column order based on your actual query patterns: put columns used in equality conditions (`=`) before columns used in range conditions (`>`, `<`, `BETWEEN`), and put the column most frequently queried alone first.
- Don't assume a composite index on `(a, b)` helps a query filtering only on `b` — check with `EXPLAIN` (Topic 5) rather than assuming.
- Avoid building a separate single-column index that's already a strict subset (a leftmost prefix) of an existing composite index — it's redundant, since the composite index already serves that exact case.
- When in doubt about column order, remember: the leading column should be the one that, on its own, narrows the data down the most (highest selectivity) or is filtered on most often independently of the others.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `(customer_id, status)` and `(status, customer_id)` are equivalent | They are two different physical sort orders; each only helps queries that filter starting from its own leftmost column. |
| Expecting a composite index to help a query that filters only on its second (or later) column | The leftmost-prefix rule means a query skipping the leading column(s) gets no benefit from that index at all. |
| Believing a bigger B-tree (more rows) means proportionally slower lookups | A B-tree's lookup cost grows logarithmically, not linearly — doubling the row count typically adds at most one extra level to the tree, not double the work. |
| Adding a wide composite index across every commonly-queried column "to be safe" | Wider composite indexes cost more storage and more write overhead per row (Topic 1), and columns beyond what the leftmost-prefix rule can actually use for a given query add cost without benefit for that query. |

## Interview Questions

1. **Q: Why is a B-tree, rather than a plain sorted array, used for database indexes?**
   A: A B-tree is balanced (every leaf is the same distance from the root) and each node holds many entries, so looking up a value takes a small, predictable number of steps that grows only logarithmically as the total number of indexed values grows. A plain sorted array on disk doesn't offer an efficient way to "jump" to roughly the right position without additional structure, and inserting into the middle of a sorted array is expensive at scale, whereas a B-tree is specifically designed to stay balanced and searchable as data is added.

2. **Q: You have a composite index on `(customer_id, order_date)`. Will a query with `WHERE order_date = '2026-01-01'` (with no condition on `customer_id`) use this index efficiently?**
   A: No. Because of the leftmost-prefix rule, this index is sorted first by `customer_id` and only secondarily by `order_date` within each `customer_id`. A query filtering on `order_date` alone skips the leading column, so matching rows for that date are scattered across every `customer_id` group rather than sitting together anywhere in the index — PostgreSQL cannot use this index to efficiently jump to just those rows.

3. **Q: When would you choose a composite index on `(a, b)` over two separate single-column indexes on `a` and on `b`?**
   A: When your real queries commonly filter on `a` alone, or on `a` and `b` together, a single composite index `(a, b)` serves both cases and is generally more storage- and write-efficient than maintaining two separate index structures. Two separate single-column indexes make more sense when queries need `a` and `b` independently of each other in ways a single leftmost-prefix-ordered structure can't serve well (e.g., frequent standalone filtering on `b` with no `a` condition at all).

4. **Q: Roughly how does lookup cost change if a B-tree-indexed table's row count grows from one million to one billion?**
   A: Very little in relative terms — because a B-tree's height grows logarithmically with the number of entries, and each node can hold many entries, going from a million to a billion rows typically adds only a couple of extra levels to the tree, not a thousand-fold increase in lookup steps. This is in sharp contrast to a full table scan, whose cost grows linearly (proportionally) with row count.

## Summary

- A **B-tree** is a balanced, sorted tree structure whose lookup cost grows logarithmically with data size, keeping searches fast even as a table scales from thousands to billions of rows.
- B-tree is PostgreSQL's **default** index type because it natively supports equality, range comparisons, and sorting — the most common query patterns in real SQL.
- A **composite index** indexes multiple columns together in one structure, physically sorted by the first column, then the second within it, and so on.
- The **leftmost-prefix rule**: a composite index only helps a query whose conditions form an unbroken prefix of the index's column list, starting from the leftmost column — skipping the leading column means the index provides no shortcut.
- **Column order in a composite index definition is a deliberate design decision** — choose the leading column based on what your real queries actually filter on most often or most selectively, not arbitrarily.
