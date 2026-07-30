# Dropping and Truncating Tables

## Learning Objectives

By the end of this section you should be able to:
- Remove a table entirely with `DROP TABLE`, including the `IF EXISTS` safeguard
- Explain the difference between `CASCADE` and `RESTRICT` when other database objects depend on a table, and know which one PostgreSQL uses by default
- Explain, at a structural level, how `TRUNCATE` differs from `DROP TABLE` — what each does to the table itself and what each does to its rows

## Prerequisites

- [Altering Tables](03-altering-tables.md) — you should already be comfortable with how `ALTER TABLE` reshapes a table in place before learning the more drastic operations that remove it (or its contents) entirely.

## Motivation

Not every structural change is additive. Sometimes a table has genuinely outlived its purpose — an experiment that didn't pan out, a table replaced by a better design — and needs to be removed completely. Other times, you don't want to remove the table's *structure* at all, only wipe out its *contents* and start fresh. These are two different operations with two very different blast radii, and confusing them (or misunderstanding what happens when other objects depend on the table you're removing) is a well-known source of real production incidents.

## Problem Statement

Suppose the `products` table from earlier topics in this module has an early sibling — an `experimental_products` table someone created months ago to try out an idea, now completely unused and forgotten. You want it gone entirely: no structure, no rows, nothing left behind. Separately, imagine a `staging_products` table used purely as a temporary scratch area before nightly data loads — you want to empty it completely every night, but you want its column structure to remain exactly as-is, ready for tomorrow's load, without having to run `CREATE TABLE` all over again. These are two distinct needs, and SQL provides two distinct tools for them.

## Concept

### Dropping a Table

```sql
DROP TABLE experimental_products;
```

This permanently removes the table's structure *and* every row it contained. There is no `UNDROP` — recovering from this requires a backup taken beforehand. If the table might not exist and you don't want an error in that case (useful in idempotent setup/teardown scripts, exactly as with `DROP DATABASE` in Topic 1):

```sql
DROP TABLE IF EXISTS experimental_products;
```

Without `IF EXISTS`, dropping a table that isn't there raises:

```
ERROR:  table "experimental_products" does not exist
```

### `CASCADE` vs. `RESTRICT`

Tables are rarely isolated islands. Other database objects can depend on a table — most commonly, a **foreign key** in another table pointing at it (foreign keys are fully covered in Module 5, but the dependency concept matters here regardless), or a **view** built on top of it (Module 12). What should happen if you try to drop a table that something else depends on?

PostgreSQL gives you an explicit choice, via two keywords you can append to `DROP TABLE`:

| Keyword | Behavior |
|---|---|
| `RESTRICT` | Refuse to drop the table if anything else depends on it. This is PostgreSQL's **default** behavior — if you write plain `DROP TABLE name;` with neither keyword, it behaves exactly as if you'd written `RESTRICT`. |
| `CASCADE` | Drop the table, and automatically drop every dependent object along with it (dependent foreign key constraints, views built on it, and so on). |

For example, suppose a view called `low_stock_products` was built on top of `products` (views are covered fully in Module 12, but the dependency they create matters here):

```sql
DROP TABLE products;
```

```
ERROR:  cannot drop table products because other objects depend on it
DETAIL:  view low_stock_products depends on table products
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

PostgreSQL refuses by default (`RESTRICT` behavior) and, helpfully, tells you exactly what's depending on the table. You have two honest choices from here: manually drop `low_stock_products` first, then drop `products` — or accept the automatic cascade:

```sql
DROP TABLE products CASCADE;
```

```
NOTICE:  drop cascades to view low_stock_products
DROP TABLE
```

`CASCADE` did what it advertised — it dropped `low_stock_products` along with `products`, and told you so via a `NOTICE`. This is powerful and convenient, and also exactly why it deserves caution: `CASCADE` can silently take down objects several layers deep (a view depending on another view depending on the table you're dropping, for instance) in a single statement, without asking for confirmation on each one individually.

### Preview: `DROP TABLE` vs. `TRUNCATE TABLE`

Both `DROP TABLE` and `TRUNCATE TABLE` are ways of getting rid of everything currently in a table, but they operate at fundamentally different levels:

```sql
DROP TABLE staging_products;
```
- Removes the table's **structure and its rows together**. After this, `staging_products` does not exist at all — no columns, no constraints, nothing. Any query referencing it fails with "relation does not exist."

```sql
TRUNCATE TABLE staging_products;
```
- Removes **only the rows**. The table itself — its columns, types, and constraints — remains exactly as it was, immediately ready to accept new rows via `INSERT`, with zero need to redefine anything.

```
sql_course=# SELECT count(*) FROM staging_products;
 count
-------
  8000
(1 row)

sql_course=# TRUNCATE TABLE staging_products;
TRUNCATE TABLE
sql_course=# SELECT count(*) FROM staging_products;
 count
-------
     0
(1 row)

sql_course=# \d staging_products
                    Table "public.staging_products"
   Column   |  Type   | Collation | Nullable | Default
------------+---------+-----------+----------+---------
 name       | text    |           |          |
 price      | numeric |           |          |
```

Notice the table's structure (shown by `\d`) is completely untouched after `TRUNCATE` — only the row count changed, from 8,000 to 0.

A useful way to hold the distinction in mind: `DROP TABLE` demolishes the shelf itself; `TRUNCATE TABLE` empties everything off the shelf while leaving the shelf standing exactly where it was, ready to be refilled.

This topic deliberately stops here with a *structural* contrast between `DROP` and `TRUNCATE`. `TRUNCATE`'s deeper comparison against `DELETE` (Module 6's `DELETE FROM table;` with no `WHERE`, which also removes all rows but works very differently internally, and has different implications for things like auto-incrementing `SERIAL` counters and transactional rollback) is the dedicated subject of Module 6 — that's where you'll learn exactly when to reach for one over the other in real data-modification workflows.

### `TRUNCATE` and Dependencies

`TRUNCATE` has its own version of the `CASCADE`/`RESTRICT` consideration, for a related but distinct reason: if another table has a foreign key pointing *into* the table you're truncating, PostgreSQL refuses by default, since truncating would instantly invalidate references those other rows depend on:

```sql
TRUNCATE TABLE products;
```

```
ERROR:  cannot truncate a table referenced in a foreign key constraint
DETAIL:  Table "order_items" references "products".
HINT:  Truncate table "order_items" too, or use TRUNCATE ... CASCADE.
```

Just as with `DROP TABLE`, appending `CASCADE` tells PostgreSQL to also truncate every table that references this one:

```sql
TRUNCATE TABLE products CASCADE;
```

Foreign keys themselves are fully covered in Module 5 — the point to take away here is that `CASCADE`/`RESTRICT` is a recurring PostgreSQL pattern for "what do I do about dependent objects," not something unique to `DROP TABLE` alone.

## Internal Working (Preview)

`DROP TABLE` and `TRUNCATE TABLE` differ sharply in what they actually do at the storage level:

```
DROP TABLE staging_products
        │
        ▼
 PostgreSQL removes the table's catalog entries (pg_class, pg_attribute)
 entirely, and deletes its underlying data files from disk.
 The table no longer exists in any form.


TRUNCATE TABLE staging_products
        │
        ▼
 PostgreSQL deallocates the table's existing data pages (marking
 the space as free) but keeps every catalog entry describing its
 columns, types, and constraints completely intact.
 The table still exists, now with zero rows.
```

This is precisely why Module 1's SQL-command-categorization topic classified `TRUNCATE` as **DDL** rather than DML, even though its visible effect ("remove all the data") sounds like something `DELETE` (DML) also does: internally, `TRUNCATE` behaves like a structural, page-deallocation operation rather than a row-by-row data change, which is also why it is typically dramatically faster than `DELETE FROM table;` on a large table — `DELETE` must individually process and log the removal of every row, while `TRUNCATE` simply deallocates entire pages at once. The full weight of that performance and behavioral difference is Module 6's focus.

## Real-World Analogy

Continuing the filing-cabinet analogy from earlier topics in this module: `DROP TABLE` is hauling the entire cabinet out of the building and destroying it — the drawers, the labels, everything, gone. `TRUNCATE TABLE` is instead pulling every single folder out of the cabinet's drawers and shredding them, while the cabinet itself — its drawers, its labels, its structure — stays exactly where it was, empty and ready to be refilled tomorrow. `CASCADE` versus `RESTRICT` is the difference between a moving crew that automatically also empties out anything stored *inside* that cabinet by other departments before hauling it away (`CASCADE`), versus a crew that stops and refuses the moment they notice someone else's things are inside (`RESTRICT`) — forcing you to explicitly decide what happens to those other things first.

## Why This Was Designed This Way

Giving `DROP` and `TRUNCATE` genuinely different scopes — one destroying structure and data together, one destroying only data — reflects that these solve two entirely different real needs: permanently retiring a table that should no longer exist at all, versus routinely resetting a table's contents while its role in the schema (and everything that depends on its structure, like application code expecting certain columns) remains unchanged. Making `RESTRICT` the default behavior for both, rather than `CASCADE`, follows the same conservative philosophy seen elsewhere in this course (e.g., PostgreSQL refusing to drop a database still in use, Topic 1) — a destructive operation with wide-reaching, hard-to-reverse consequences should require you to explicitly opt in to that wider blast radius, rather than accidentally triggering it by default.

## Advantages

- **`DROP TABLE` cleanly removes something no longer needed** — no leftover structure, no orphaned metadata cluttering the schema.
- **`TRUNCATE` is a fast, clean way to reset a table's contents** — ideal for staging tables, temporary working tables, or any table you deliberately want to empty and refill repeatedly without redefining its structure each time.
- **`CASCADE`/`RESTRICT` give you explicit, informed control** over exactly how far a destructive operation is allowed to reach, rather than either always failing on any dependency or always silently taking everything down with it.

## Disadvantages / Limitations

- **Both are irreversible within the database itself** — neither `DROP TABLE` nor `TRUNCATE TABLE` can be undone after the fact without a backup taken beforehand; there is no "recycle bin."
- **`CASCADE` can have a wider blast radius than expected** — on a schema with several layers of views or foreign keys built on top of each other, a single `CASCADE` can remove far more than the one object you intended to target, and it's easy to underestimate the full dependency chain on a complex schema.
- **`RESTRICT` (the default) can feel like friction** during legitimate cleanup of a genuinely obsolete table with several dependents, requiring you to either drop dependents individually or consciously choose `CASCADE` — this friction is intentional, not a flaw.

## Best Practices

- Always use `DROP TABLE IF EXISTS` and inspect the error/hint output carefully before ever reaching for `CASCADE` — read exactly what PostgreSQL says would be dropped, rather than adding `CASCADE` reflexively just to make an error go away.
- Prefer `TRUNCATE` over `DROP TABLE` when your actual intent is "empty this table, keep using it" — reaching for `DROP TABLE` followed by re-running `CREATE TABLE` to get the same structure back is unnecessary extra work and briefly leaves the table missing entirely for anything trying to use it in between.
- Before running `DROP TABLE ... CASCADE` or `TRUNCATE ... CASCADE` on a table you suspect has dependents, first query for those dependents deliberately (or simply attempt the plain, non-cascading version first and read the resulting error) so you know exactly what else will be affected before you commit to it.
- Take backups (an operational practice outside this module's DDL focus, but essential context) before any destructive DDL operation on data you cannot afford to lose, since neither `DROP` nor `TRUNCATE` offers an undo.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Reaching for `CASCADE` immediately after a plain `DROP TABLE` fails, without reading what it says would be dropped. | The error's `DETAIL`/`HINT` output tells you exactly what depends on the table — skipping straight to `CASCADE` risks dropping views or constraints you didn't realize existed or still needed. |
| Using `DROP TABLE` followed by `CREATE TABLE` just to empty a table you intend to keep using. | This briefly removes the table from existence entirely (breaking anything querying it in that window) and forces you to fully redeclare its structure — `TRUNCATE TABLE` achieves "empty it" directly, in one step, with the structure never disappearing. |
| Assuming `TRUNCATE` respects a `WHERE` clause like `DELETE` does. | `TRUNCATE` has no `WHERE` clause at all — it always removes every row in the table; if you need to remove only some rows, that's `DELETE FROM table WHERE ...`, covered in Module 6. |
| Forgetting that `TRUNCATE` can also be blocked, and also supports `CASCADE`, when other tables reference it via foreign keys. | `TRUNCATE` isn't automatically "safer" than `DROP` around dependencies — PostgreSQL applies the same `RESTRICT`-by-default, `CASCADE`-if-requested pattern to both. |

## Interview Questions

1. **Q: What is the structural difference between `DROP TABLE` and `TRUNCATE TABLE`?**
   A: `DROP TABLE` removes the table entirely — its structure (columns, types, constraints) and all its data — so the table no longer exists at all afterward. `TRUNCATE TABLE` removes only the rows; the table's structure remains completely intact and it's immediately ready to accept new data, with no need to recreate anything.

2. **Q: What is the difference between `CASCADE` and `RESTRICT` in a `DROP TABLE` statement, and which does PostgreSQL use by default?**
   A: `RESTRICT` refuses the drop if any other object (a view, a foreign key from another table) depends on the table being dropped — this is PostgreSQL's default when neither keyword is specified. `CASCADE` instead proceeds with the drop and automatically drops every dependent object along with it.

3. **Q: If a table you're trying to `DROP` has a view built on top of it, what happens by default, and how do you proceed?**
   A: By default (`RESTRICT` behavior), PostgreSQL refuses the drop and reports that the view depends on the table. You can either explicitly drop the view first and then the table, or add `CASCADE` to the `DROP TABLE` statement so PostgreSQL automatically drops the dependent view as part of the same operation.

4. **Q: Why is `TRUNCATE` generally much faster than deleting every row with `DELETE`, on a large table?**
   A: `TRUNCATE` works by deallocating the table's existing data pages wholesale, without processing or logging each row individually. `DELETE` (fully covered in Module 6) must examine, remove, and log each qualifying row one at a time, which scales with the number of rows removed rather than being a near-constant-time structural operation.

## Summary

- `DROP TABLE` (optionally `IF EXISTS`) permanently removes a table's structure and all its data together — the table ceases to exist entirely afterward, with no undo.
- `RESTRICT` (PostgreSQL's default) refuses to drop a table that other objects (views, foreign keys) depend on; `CASCADE` proceeds anyway and automatically drops those dependent objects too — read the error's detail before ever reaching for `CASCADE`.
- `TRUNCATE TABLE` removes only a table's rows, leaving its structure completely intact and immediately reusable — structurally very different from `DROP TABLE`, even though both can "empty" a table's visible contents.
- `TRUNCATE` supports its own `CASCADE`/`RESTRICT` behavior for tables referenced by foreign keys elsewhere, following the same pattern as `DROP TABLE`.
- The full comparison of `TRUNCATE` against `DELETE` (performance, transactional behavior, effect on `SERIAL` sequences) is deferred to Module 6 — this topic focused purely on `TRUNCATE` as a structural sibling to `DROP`, not as a data-manipulation tool.
- This closes out Module 04's topics — the next file, the Module Summary, consolidates everything covered across databases, table creation, alteration, and removal.
