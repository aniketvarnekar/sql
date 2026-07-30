# TRUNCATE vs. DELETE

## Learning Objectives

By the end of this section you should be able to:
- Explain why `TRUNCATE` is dramatically faster than an unqualified `DELETE` for clearing an entire table
- State which of the two supports a `WHERE` clause, and why
- Explain the difference in how each interacts with row-level triggers
- Use `TRUNCATE ... RESTART IDENTITY` to reset a `SERIAL`/identity sequence back to its starting value
- Correctly describe both statements' transactional (rollback) behavior in PostgreSQL
- Choose the correct statement for a given "clear this data" scenario

## Prerequisites

- [DELETE](03-delete.md) — you need to understand `DELETE` (especially its no-`WHERE` behavior) before comparing it against an alternative.
- **Module 5 (Constraints & Keys)**, specifically `SERIAL`/auto-incrementing primary keys — `TRUNCATE`'s identity-reset behavior only makes sense once you know what a sequence is.
- [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md) — this topic revisits and fully explains the "TRUNCATE is classified as DDL, not DML" nuance first previewed there.

## Motivation

Both `DELETE FROM table;` (no `WHERE`) and `TRUNCATE TABLE table;` remove every row from a table, so at first glance they look interchangeable. They are not. Choosing between them matters a great deal in practice — for a small lookup table, the difference is irrelevant; for a table with tens of millions of rows that needs to be cleared nightly, choosing the wrong one can mean the difference between a query that finishes in milliseconds and one that takes minutes and generates a huge amount of unnecessary transaction log activity.

## Problem Statement

Suppose you run a nightly job that reloads a `daily_metrics` staging table from scratch — millions of rows in, all of them cleared out again before the next load. You could write:

```sql
DELETE FROM daily_metrics;
```

This works, but it removes each of those millions of rows one at a time, logging each individual removal for crash recovery purposes, which is slow and I/O-heavy for something that's conceptually just "empty the whole table." You could instead write:

```sql
TRUNCATE TABLE daily_metrics;
```

Which empties the table almost instantly by discarding its underlying storage pages wholesale rather than deleting rows one by one. But `TRUNCATE` isn't simply "a faster `DELETE`" — it has different rules around `WHERE`, different behavior with triggers, and different rules around foreign keys, all of which matter for choosing correctly.

## Concept

### Side-by-Side Comparison

| Aspect | `DELETE` (no `WHERE`) | `TRUNCATE` |
|---|---|---|
| **SQL category** | DML (Data Manipulation Language) | DDL (Data Definition Language) — see below for why |
| **Supports `WHERE`** | Yes — can remove a subset of rows | No — always removes *all* rows in the table; there is no partial `TRUNCATE` |
| **Speed on large tables** | Slower — removes and logs each row individually | Much faster — deallocates the table's storage pages directly, without processing rows one by one |
| **Row-level triggers** | Fire once per deleted row (`FOR EACH ROW` triggers, Module 18) | Do **not** fire by default (only statement-level `AFTER TRUNCATE` triggers can respond, and only if explicitly defined) |
| **Resets `SERIAL`/identity sequences** | No — the sequence keeps counting from wherever it left off | Yes, if you add `RESTART IDENTITY` — resets the sequence back to its starting value |
| **Transactional / rollback-safe in PostgreSQL** | Yes — can be rolled back like any other statement inside a transaction | Yes, in PostgreSQL specifically — also fully rollback-safe inside a transaction (this is a PostgreSQL strength; not every database makes `TRUNCATE` transactional) |
| **Interacts with foreign keys referencing this table** | Enforces `ON DELETE` actions row by row, exactly as covered in Topic 3 | By default, refuses to truncate a table that is referenced by a foreign key from another table with existing rows, unless you add `CASCADE` (which also truncates the referencing tables) |
| **Returns affected row count / supports `RETURNING`** | Yes — reports `DELETE n` and supports `RETURNING` | No — reports only `TRUNCATE TABLE`, with no row count and no `RETURNING` support |

### Why `TRUNCATE` Is Faster

A `DELETE` with no `WHERE` still processes the statement as a *data manipulation* operation: PostgreSQL visits each row, marks it as removed (via MVCC, previewed in Topic 2), checks any relevant foreign key constraints per row, and writes enough information to the write-ahead log for crash recovery and to let other transactions see a consistent view of the data (Module 14). All of that work scales with the number of rows.

`TRUNCATE`, by contrast, works at the *storage* level: it deallocates the disk pages that make up the table's data outright — closer to "detach and discard the file" than "erase every entry inside the file one by one." This is why it is classified as **DDL**, not DML, despite the fact that its visible effect (an empty table) looks identical to a `DELETE`'s — the *implementation* is structural (affecting how storage is organized) rather than row-by-row data manipulation. This is exactly the nuance flagged back in Module 1's [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md).

```sql
-- Slow for a large table: processes and logs every row individually
DELETE FROM daily_metrics;

-- Fast regardless of table size: discards storage pages wholesale
TRUNCATE TABLE daily_metrics;
```

### `WHERE` Support — `DELETE` Only

`TRUNCATE` has no concept of a condition — it always removes every row, with no exceptions:

```sql
TRUNCATE TABLE daily_metrics WHERE metric_date < '2026-01-01';
```

```
ERROR:  syntax error at or near "WHERE"
```

If you need to remove only *some* rows, `DELETE` is your only option between the two — this is often the deciding factor on its own: `TRUNCATE` is exclusively an "empty the whole table" tool.

### Triggers — `DELETE` Fires Them, `TRUNCATE` Does Not (by Default)

Row-level triggers (covered fully in Module 18 — Procedures, Functions & Triggers) are pieces of logic that automatically run in response to a data change on each affected row — for example, logging every deletion to an audit table. `DELETE` fires a `FOR EACH ROW` trigger once per row removed, because it genuinely processes each row individually. `TRUNCATE` does not fire row-level triggers at all, because it never processes individual rows — only statement-level `TRUNCATE` triggers (a distinct, less commonly used trigger type) can respond to it, and only if one was explicitly created for that purpose. If your table relies on row-level triggers for something important (audit logging, cascading business logic), reaching for `TRUNCATE` instead of `DELETE` will silently skip that logic entirely.

### Resetting Identity/`SERIAL` Sequences

A `SERIAL` column (Module 4/5) is backed by a **sequence** — an independent counter object that hands out the next integer every time a row is inserted, and does not know or care whether rows using earlier values still exist. Deleting rows does not reset this counter:

```sql
CREATE TABLE daily_metrics (
    id          SERIAL PRIMARY KEY,
    metric_date DATE NOT NULL,
    value       NUMERIC
);

INSERT INTO daily_metrics (metric_date, value) VALUES
    ('2026-07-29', 10.5),
    ('2026-07-30', 11.2);

DELETE FROM daily_metrics;

INSERT INTO daily_metrics (metric_date, value) VALUES ('2026-07-31', 9.8);

SELECT * FROM daily_metrics;
```

```
 id | metric_date | value
----+-------------+-------
  3 | 2026-07-31  |   9.8
(1 row)
```

Even though the table was completely emptied, the new row got `id = 3`, not `id = 1` — the sequence remembers it already handed out 1 and 2, regardless of whether those rows still exist. `TRUNCATE` offers an explicit option to reset this:

```sql
TRUNCATE TABLE daily_metrics RESTART IDENTITY;

INSERT INTO daily_metrics (metric_date, value) VALUES ('2026-07-31', 9.8);

SELECT * FROM daily_metrics;
```

```
 id | metric_date | value
----+-------------+-------
  1 | 2026-07-31  |   9.8
(1 row)
```

`RESTART IDENTITY` resets the underlying sequence back to its original starting value (1, by default), so the next insert gets `id = 1` again. Without this keyword, `TRUNCATE TABLE daily_metrics;` alone empties the table but leaves the sequence exactly where it was — `RESTART IDENTITY` must be requested explicitly (the opposite, `CONTINUE IDENTITY`, is the default and can also be written explicitly for clarity).

### `CASCADE` with `TRUNCATE`

If another table has a foreign key referencing the table you're truncating, PostgreSQL refuses by default, for the same referential-integrity reason `DELETE` refuses to remove a still-referenced row (Topic 3):

```sql
TRUNCATE TABLE employees;
```

```
ERROR:  cannot truncate a table referenced in a foreign key constraint
DETAIL:  Table "orders" references "employees".
HINT:  Truncate table "orders" too, or use TRUNCATE ... CASCADE.
```

Adding `CASCADE` truncates both the target table and every table that references it:

```sql
TRUNCATE TABLE employees CASCADE;
```

```
NOTICE:  truncate cascades to table "orders"
TRUNCATE TABLE
```

Note that `TRUNCATE ... CASCADE` here means "also truncate dependent tables" — a different (though conceptually related) use of the word `CASCADE` from a foreign key's `ON DELETE CASCADE` action (Topic 3); don't confuse the two, since one is a per-row delete-time behavior and the other is a truncate-time, whole-table behavior.

### Transactional Rollback Behavior in PostgreSQL

A distinctive PostgreSQL strength: **both** `DELETE` and `TRUNCATE` are fully transactional in PostgreSQL — either can be rolled back if issued inside a transaction that hasn't been committed yet (Module 14 covers transactions in full):

```sql
BEGIN;

TRUNCATE TABLE daily_metrics;

SELECT COUNT(*) FROM daily_metrics;
-- count is 0 here, inside the transaction

ROLLBACK;

SELECT COUNT(*) FROM daily_metrics;
-- the rows are all back — the TRUNCATE never took effect
```

```
 count
-------
     2
(1 row)
```

This is worth calling out explicitly because it is *not* universally true across every database product — in some other systems, `TRUNCATE` implicitly commits immediately and cannot be rolled back, which is one of the sharper vendor differences covered in Module 22. In PostgreSQL, you can safely experiment with a `TRUNCATE` inside a `BEGIN`/`ROLLBACK` block with the same safety net as any other statement.

### When to Use Each

| Situation | Use |
|---|---|
| You need to remove only some rows | `DELETE ... WHERE` — `TRUNCATE` cannot filter |
| You need row-level triggers to fire for each removed row | `DELETE` — `TRUNCATE` skips them by default |
| You need to know exactly which/how many rows were removed, or need their values (`RETURNING`) | `DELETE` |
| You need to empty an entire large table as fast as possible, and don't need per-row triggers or `RETURNING` | `TRUNCATE` |
| You need the table's auto-increment counter to restart from 1 after clearing it | `TRUNCATE ... RESTART IDENTITY` |
| The table is referenced by foreign keys from other tables you also intend to clear | `TRUNCATE ... CASCADE` (deliberately, having confirmed which tables it will also empty) |

## Internal Working (Preview)

```
 DELETE FROM t;                          TRUNCATE TABLE t;
       │                                        │
       ▼                                        ▼
 Visit each row in t                    Deallocate t's storage pages directly
       │                                        │
       ├─ mark row removed (MVCC)               ├─ (no per-row visitation at all)
       ├─ check FK / ON DELETE actions           ├─ check no other table references t
       │    per row                              │    (unless CASCADE)
       ├─ fire row-level triggers per row         ├─ fire only TRUNCATE-specific
       │    (if any)                              │    triggers, if defined
       ├─ write WAL entries per row               ├─ write a small number of WAL entries
       │    (Module 14)                            │    describing the page deallocation
       ▼                                        ▼
 Cost scales with row count               Cost is close to constant, regardless
                                            of how many rows existed
```

The core reason for the speed difference is structural: `DELETE` is fundamentally a row-oriented operation (it must individually account for each row's MVCC visibility, its triggers, and its constraints), while `TRUNCATE` is a page-oriented, structural operation — it doesn't need to reason about individual rows at all, which is also exactly why it can't support `WHERE` (there's no per-row decision being made to selectively filter) and doesn't fire per-row triggers (there are no individual row-removal events to attach a trigger callback to).

## Real-World Analogy

Think of `DELETE` like carefully removing specific pages from a filing cabinet's folders one at a time, checking each page against a rulebook before pulling it, and writing a note in a log each time ("removed page 47 from folder 'Q3 metrics'"). Now think of `TRUNCATE` like instead removing the entire folder — drawer and all — and sliding a brand-new, identically-labeled empty folder into its place. The second approach is vastly faster (one action instead of thousands), but it can't selectively keep some pages and discard others ("no `WHERE`"), and nobody watching for "a page was removed" notifications (row-level triggers) will be notified, because no individual pages were ever handled — the whole folder was swapped out at once. `RESTART IDENTITY` is like also resetting a page-numbering stamp back to page 1, so the next page filed into the new folder starts fresh instead of continuing from where the old folder's numbering left off.

## Why TRUNCATE Exists Alongside DELETE

`TRUNCATE` exists because "empty this entire table, as fast as possible" is common enough (staging tables, temporary batch-processing tables, test data resets) to deserve a purpose-built, structural operation rather than forcing every full-table clear through `DELETE`'s row-by-row machinery. This is a direct performance/flexibility trade-off, consistent with the declarative philosophy from Module 1: SQL gives you two tools that produce a visually similar end state (an empty table) through fundamentally different internal mechanisms, and lets you choose based on what you actually need (selectivity and per-row behavior vs. raw speed) rather than forcing one mechanism to serve every case.

## Advantages

**`DELETE`:**
- Supports `WHERE` — can remove any subset of rows.
- Fires row-level triggers, so dependent logic (audit logging, cascading business rules) still runs.
- Supports `RETURNING` and reports an exact affected-row count.

**`TRUNCATE`:**
- Dramatically faster for clearing large or entire tables.
- Can reset a `SERIAL`/identity sequence back to its starting value with `RESTART IDENTITY`.
- Still fully transactional and rollback-safe in PostgreSQL.

## Disadvantages / Limitations

- **`DELETE`** is slow at scale precisely because of the row-by-row work that makes it flexible — for a multi-million-row full-table clear, this cost is real and measurable.
- **`TRUNCATE`** cannot filter rows at all (all-or-nothing only), skips row-level triggers by default (which can silently break logic that depends on them), and refuses to run at all against a table referenced by foreign keys unless you add `CASCADE` and accept that it will also empty the referencing tables.
- Neither statement is a substitute for the other in every case — they solve overlapping but distinct problems, and misusing `TRUNCATE` where selective deletion or trigger behavior was actually required is a genuine correctness bug, not just a performance nuance.

## Best Practices

- **Default to `DELETE ... WHERE` for anything selective**; reach for `TRUNCATE` only when the intent is genuinely "empty this entire table."
- **Before truncating a table with `RESTART IDENTITY`, confirm nothing outside the database depends on previously-issued ids remaining "used up"** — resetting the sequence means the next inserted row can reuse an id that previously belonged to now-deleted data, which may or may not be safe depending on what else references those ids historically (e.g., external logs, archived records).
- **Check for triggers before choosing `TRUNCATE`** — if a table has row-level triggers your system relies on (e.g., audit logging), `TRUNCATE` will silently skip them; verify this is actually acceptable before using it.
- **Use `TRUNCATE ... CASCADE` deliberately, never habitually** — always know exactly which other tables will be emptied as a result before running it.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `TRUNCATE` is "just a faster `DELETE`" with no other differences | It skips row-level triggers by default, cannot filter with `WHERE`, and is classified as DDL rather than DML — the differences go well beyond speed. |
| Using `DELETE FROM table;` to clear a very large table nightly, and being surprised by how slow/log-heavy it is | For a genuine full-table clear with no need for per-row triggers, `TRUNCATE` accomplishes the same end state far more efficiently. |
| Forgetting that `TRUNCATE` alone does not reset a `SERIAL` sequence | Without `RESTART IDENTITY`, the sequence keeps counting from wherever it left off, exactly like a plain `DELETE` — the reset must be requested explicitly. |
| Assuming `TRUNCATE` is not transactional in PostgreSQL, based on experience with another database | PostgreSQL's `TRUNCATE` is fully transactional and can be rolled back — this is not guaranteed to be true in every database vendor (Module 22). |

## Interview Questions

1. **Q: What is the fundamental reason `TRUNCATE` is faster than `DELETE` for clearing an entire table?**
   A: `DELETE` is a row-oriented operation — it visits, checks constraints on, potentially fires triggers for, and logs the removal of each individual row, so its cost scales with row count. `TRUNCATE` is a page-oriented, structural operation — it deallocates the table's underlying storage pages directly without processing rows one by one, so its cost is close to constant regardless of how many rows existed.

2. **Q: Can `TRUNCATE` remove only rows matching a condition, like `DELETE ... WHERE` can?**
   A: No. `TRUNCATE` has no `WHERE` clause and always removes every row in the table — if you need to remove a subset of rows, `DELETE` is the only option of the two.

3. **Q: Does `TRUNCATE` fire the same triggers as `DELETE`?**
   A: No, not by default. `DELETE` fires row-level (`FOR EACH ROW`) triggers once per removed row. `TRUNCATE` does not process individual rows, so row-level triggers never fire; only a statement-level `TRUNCATE`-specific trigger, if explicitly defined, can respond to it.

4. **Q: How do you empty a table and make sure its `SERIAL` primary key starts back at 1 for the next insert?**
   A: `TRUNCATE TABLE table_name RESTART IDENTITY;` — plain `TRUNCATE` (or `DELETE`) empties the table but leaves the underlying sequence wherever it left off; `RESTART IDENTITY` explicitly resets that sequence back to its starting value.

## Summary

- `DELETE` (no `WHERE`) and `TRUNCATE` both can empty an entire table, but work completely differently internally: `DELETE` is row-by-row DML; `TRUNCATE` deallocates storage pages and is classified as DDL.
- Only `DELETE` supports a `WHERE` clause — `TRUNCATE` is always all-or-nothing.
- `DELETE` fires row-level triggers per removed row; `TRUNCATE` does not, by default.
- `TRUNCATE ... RESTART IDENTITY` resets a `SERIAL`/identity sequence back to its starting value; plain `DELETE` never touches the sequence at all.
- In PostgreSQL specifically, both statements are fully transactional and can be rolled back inside an open transaction — this is a notable PostgreSQL strength, not guaranteed across every database vendor.
- Use `DELETE` when you need selectivity, triggers, `RETURNING`, or an exact row count; use `TRUNCATE` when you need to empty an entire table as fast as possible.
- Next, Topic 5 covers `INSERT ... ON CONFLICT` — PostgreSQL's solution to the "insert this row, or update it if it already exists" problem.
