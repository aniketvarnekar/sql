# Altering Tables

## Learning Objectives

By the end of this section you should be able to:
- Add, drop, and rename columns on an existing table using `ALTER TABLE`
- Change a column's data type, and explain why doing so can be dangerous or costly on a large, live table
- Rename an entire table
- Add and drop constraints via `ALTER TABLE` at a conceptual level, and know where their full treatment lives

## Prerequisites

- [Creating Tables](02-creating-tables.md) — you need an existing table to alter; this topic continues directly from the `products` table designed there.

## Motivation

No table survives contact with real requirements unchanged. New features need new columns; a column you thought would always be short text turns out to need more room; a table's name stops fitting what it actually represents. `ALTER TABLE` is how a table's structure evolves *after* it already holds real data — and unlike a fresh `CREATE TABLE`, every alteration here has to reckon with rows that already exist, which is precisely what makes this topic more delicate than the last one.

## Problem Statement

Suppose the `products` table from the previous topic has been live for months, with thousands of real rows in it. Now the requirements change: marketing wants a `description` column that doesn't exist yet; someone realizes `quantity_in_stock` should really have been called `stock_count`; and a column that was declared as a small integer now needs to hold larger numbers than originally expected. You cannot simply run `CREATE TABLE` again — the table, and its data, already exist. You need a way to reshape a table *in place*, without losing what's already stored in it.

## Concept

### Adding a Column

```sql
ALTER TABLE products
    ADD COLUMN description TEXT;
```

Every existing row immediately gets this new column, filled with `NULL` (the absence of a value — a concept fully covered in Module 3) unless you specify a default:

```sql
ALTER TABLE products
    ADD COLUMN is_featured BOOLEAN DEFAULT false;
```

Here, every existing row's `is_featured` is set to `false` immediately, and every future `INSERT` that doesn't mention `is_featured` will also default to `false`.

You can guard against re-running this against a column that already exists:

```sql
ALTER TABLE products
    ADD COLUMN IF NOT EXISTS description TEXT;
```

### Dropping a Column

```sql
ALTER TABLE products
    DROP COLUMN description;
```

This is destructive and immediate: every value ever stored in that column, for every row, is gone the moment this statement completes. There is no `UNDROP`. As with adding, you can guard against the column already being absent:

```sql
ALTER TABLE products
    DROP COLUMN IF EXISTS description;
```

### Renaming a Column

```sql
ALTER TABLE products
    RENAME COLUMN quantity_in_stock TO stock_count;
```

Note the slightly different keyword shape here — `RENAME COLUMN ... TO ...`, not `ALTER COLUMN`. Renaming a column changes only its name in the catalog; every value already stored under the old name is preserved exactly, now accessible under the new name. Any existing query, view, or application code that referred to the old column name (`quantity_in_stock`) will now fail with an "column does not exist" error until updated to the new name — renaming is a structural change with ripple effects on everything that reads the table.

### Changing a Column's Data Type

```sql
ALTER TABLE products
    ALTER COLUMN price TYPE NUMERIC(12, 2);
```

For some type changes, PostgreSQL needs to know exactly *how* to convert every existing value from the old type to the new one — this is done with a `USING` clause:

```sql
ALTER TABLE products
    ALTER COLUMN stock_count TYPE TEXT
    USING stock_count::TEXT;
```

`stock_count::TEXT` is PostgreSQL's cast syntax (converting a value from one type to another, covered in Module 3 and again in Module 8) — here, converting each existing integer value into its text representation. If PostgreSQL cannot figure out an unambiguous, safe conversion on its own, it requires you to supply this `USING` expression explicitly; some type changes (like widening `NUMERIC(10,2)` to `NUMERIC(12,2)`, or `INTEGER` to `BIGINT`) are unambiguous enough that PostgreSQL performs them without you needing to write `USING` at all.

### Why Changing a Column's Type Can Be Dangerous on Large Tables

This deserves special attention because it is one of the most common ways a well-intentioned schema change causes a real production outage.

When you change a column's type, PostgreSQL must guarantee that every existing value in that column is valid under the new type. Depending on the specific old type and new type involved, PostgreSQL handles this in one of two very different ways:

| Kind of change | What PostgreSQL does | Cost on a large table |
|---|---|---|
| A change PostgreSQL can prove is always safe and representation-compatible (e.g., `VARCHAR(50)` to `VARCHAR(100)`, or `NUMERIC(10,2)` to `NUMERIC(12,2)`) | Only the catalog metadata is updated — the on-disk bytes for existing rows don't need to change at all. | Effectively instant, regardless of table size. |
| A change involving an actual conversion of stored values (e.g., `TEXT` to `INTEGER`, `INTEGER` to `TEXT`, shrinking a type, changing numeric precision in a way that could lose data) | PostgreSQL must rewrite the *entire table*, row by row, converting every single existing value to the new type. | Proportional to the table's total size — this can take anywhere from seconds to hours on a table with millions of rows. |

The second case is the dangerous one, for two compounding reasons:

- **Time.** Rewriting every row of a huge table is not instantaneous — on a busy production table, this could take a long time, during which the table is essentially unusable for its normal purpose.
- **Locking.** While PostgreSQL rewrites the table, it holds an `ACCESS EXCLUSIVE` lock on it — the strongest lock PostgreSQL has, which blocks *every* other statement (even plain `SELECT`s) from that table until the alteration finishes (locking is covered in full in Module 14). In practice, this means a type change that takes ten minutes on a large table can make an entire application feature (or the whole application) appear frozen or unresponsive for those ten minutes.

This is why, in real production systems, a type change on a large, actively used table is treated as a carefully planned operation — often done during low-traffic maintenance windows, or via more advanced techniques (adding a new column with the new type, backfilling it gradually, then swapping it in) that avoid a single long-held lock. Those advanced migration techniques are outside this module's scope, but understanding *why* they exist — to avoid exactly the locking cost described here — is the key takeaway.

### Renaming a Table

```sql
ALTER TABLE products
    RENAME TO items;
```

Like renaming a column, this only changes the catalog entry — no rows are touched or rewritten — but every existing reference to the old table name (queries, views, application code) breaks until updated.

### Adding and Dropping Constraints (Preview)

`ALTER TABLE` is also how you attach a rule to an already-existing table, or remove one:

```sql
ALTER TABLE products
    ADD CONSTRAINT positive_price CHECK (price > 0);

ALTER TABLE products
    DROP CONSTRAINT positive_price;
```

At this stage, all you need is the shape of the statement — `ADD CONSTRAINT name ...` and `DROP CONSTRAINT name`. What `CHECK` means, how PostgreSQL names constraints automatically if you don't, and the full family of constraint types (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `NOT NULL`) are the entire subject of Module 5. One practical note worth knowing now: adding a constraint to a table that already has data forces PostgreSQL to check every existing row against the new rule immediately — if even one existing row violates it, the `ALTER TABLE ADD CONSTRAINT` statement fails outright and the constraint is not added.

### Combining Multiple Alterations

Multiple changes can be bundled into a single `ALTER TABLE` statement, comma-separated:

```sql
ALTER TABLE products
    ADD COLUMN description TEXT,
    ALTER COLUMN price TYPE NUMERIC(12, 2);
```

This runs both changes as a single atomic operation (an all-or-nothing unit, previewed as TCL behavior in Module 1 and covered fully in Module 14) — either both succeed, or if either fails, neither is applied.

## Internal Working (Preview)

```
ALTER TABLE products ALTER COLUMN stock_count TYPE TEXT USING stock_count::TEXT;
        │
        ▼
 PostgreSQL checks: is this a metadata-only change, or does it require rewriting stored values?
        │
   ┌────┴─────┐
   ▼          ▼
metadata-only   full table rewrite required
(instant,       │
brief lock)     ▼
           PostgreSQL acquires an ACCESS EXCLUSIVE lock,
           reads every existing row, applies the USING
           expression to convert the value, and writes
           the entire table out again in the new format
                │
                ▼
           Lock released once the rewrite completes
```

This is also why `ADD COLUMN` with a plain `DEFAULT` on a modern PostgreSQL version (11+) is cheap even on huge tables for most simple default values: PostgreSQL is able to record the default in the catalog and apply it lazily as rows are read, rather than rewriting the entire table immediately — an important optimization added specifically because early PostgreSQL versions *did* rewrite the whole table for this case.

## Real-World Analogy

Altering a table that already holds data is like renovating an office building while employees still work in it, rather than while it's an empty shell. Adding a new empty shelf (`ADD COLUMN`) or renaming a room's sign (`RENAME COLUMN`/`RENAME TO`) can usually be done in seconds with almost no disruption. But converting every filing cabinet in the building from one format to an incompatible one (a real type change requiring a rewrite) means physically touching every single existing folder to reformat it — and while that's happening, you often have to lock the whole floor so nobody can walk in and grab a folder mid-renovation. The bigger the building (table), the longer that lockdown lasts.

## Why `ALTER TABLE` Was Designed This Way

`ALTER TABLE` exists because the relational model's requirement that "every row conforms to the same structure" (Module 2) doesn't mean that structure must be permanently frozen — it means that at every moment in time, every row must conform to *whatever the current structure is*. When you alter a table, PostgreSQL's job is to get every existing row into a state consistent with the new structure before allowing any further access, which is exactly why type changes that require reinterpreting stored bytes must rewrite the table and lock it in the meantime — there is no safe way to have some rows still "in the old format" while queries assume the new one. This is the same principle from Module 1 — the DBMS enforcing structural rules centrally — applied to structure that changes after the fact rather than only at creation time.

## Advantages

- **Structure can evolve without losing data** — adding, renaming, or retyping a column preserves every existing value (except an explicit `DROP COLUMN`, which is deliberately destructive).
- **Metadata-only changes are cheap** — many common alterations (renaming, widening a numeric type, many `ADD COLUMN` cases) complete near-instantly regardless of table size, because PostgreSQL is smart about when a full rewrite is actually necessary.
- **Multiple changes can be bundled atomically** — a single `ALTER TABLE` with several clauses either fully succeeds or fully fails together, avoiding a half-migrated table.

## Disadvantages / Limitations

- **Some type changes are expensive and disruptive at scale** — a full-table rewrite under an `ACCESS EXCLUSIVE` lock can make a large, busy table unavailable for the duration, as detailed above.
- **Renames break every existing reference** — renaming a column or table doesn't automatically update queries, views, or application code that referenced the old name; those must be found and updated manually.
- **`DROP COLUMN` is immediately destructive with no undo** — there is no built-in "restore this column" after the fact; the only recovery is a prior backup (backups are outside this course's DDL focus but worth knowing about operationally).

## Best Practices

- Before running a type change on a table you suspect is large or actively used in production, find out its size and traffic first, and consider whether it warrants a maintenance window or a more gradual migration strategy rather than a single blocking `ALTER TABLE`.
- Prefer additive changes (`ADD COLUMN`) over destructive ones (`DROP COLUMN`, risky retyping) when you're not fully certain a change is final — it's much easier to later drop an unused column than to recover data you already deleted.
- After any rename (column or table), search your queries, views, and application code for the old name before considering the migration complete.
- Bundle related alterations into a single `ALTER TABLE` statement when they logically belong together, so they succeed or fail as one unit rather than leaving the table in a partially-migrated state if a later statement in a sequence fails.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Running a type change on a large production table without considering its size or the lock it will hold. | A full-table-rewrite type change takes an `ACCESS EXCLUSIVE` lock, blocking all reads and writes for the duration — on a large table this can cause a real, visible outage rather than a quick, invisible schema tweak. |
| Assuming `RENAME COLUMN` also updates every view, query, and piece of application code that used the old name. | It only updates the catalog entry for that one table — anything else referencing the old name will break with a "column does not exist" error until it's manually updated. |
| Running `ALTER TABLE ... DROP COLUMN` without being certain the data is no longer needed. | The drop is immediate and irreversible within the database — there's no `UNDROP`; recovering that data afterward requires a backup taken before the drop. |
| Adding a `CHECK` constraint to an existing table without checking whether current data already violates it. | `ADD CONSTRAINT` validates every existing row immediately; if any row violates the new rule, the entire statement fails and no constraint is added — worth checking with a `SELECT` first on a large table to avoid a surprise failure. |

## Interview Questions

1. **Q: Why can changing a column's data type on a large table be dangerous in production?**
   A: Some type changes require PostgreSQL to rewrite every existing row to convert its stored value into the new type. This rewrite takes time proportional to the table's size and requires PostgreSQL to hold an `ACCESS EXCLUSIVE` lock for the duration, blocking all other reads and writes on that table — on a large, busy table this can cause a significant, visible outage, whereas metadata-only changes (like widening a `VARCHAR` length) are effectively instant.

2. **Q: What is the difference between `ALTER COLUMN ... RENAME` and `ALTER COLUMN ... TYPE`?**
   A: Renaming (`RENAME COLUMN old TO new`) only changes the column's name in the catalog — no stored values are touched, and it's always fast. Changing type (`ALTER COLUMN col TYPE newtype`) may require converting every existing stored value into the new type, which can require a full, potentially slow and lock-heavy table rewrite depending on the specific type change.

3. **Q: What happens to existing rows when you `ADD COLUMN` with a `DEFAULT` value versus without one?**
   A: Without a default, every existing row gets the new column set to `NULL`. With a `DEFAULT`, every existing row is populated with that default value immediately (modern PostgreSQL versions do this efficiently without necessarily rewriting the whole table for simple defaults), and any future `INSERT` that omits the column also receives that default.

4. **Q: What happens if you try to `ADD CONSTRAINT` a `CHECK` rule to a table that already contains data violating that rule?**
   A: The `ALTER TABLE ADD CONSTRAINT` statement fails entirely and the constraint is not added — PostgreSQL validates every existing row against a new constraint at the moment it's added, and even a single violating row is enough to reject the whole operation.

## Summary

- `ALTER TABLE` reshapes an existing table's structure in place: `ADD COLUMN`, `DROP COLUMN`, `RENAME COLUMN ... TO ...`, `ALTER COLUMN ... TYPE ...`, and `RENAME TO` (for the table itself) cover the most common changes.
- Some changes are metadata-only and effectively instant regardless of table size; others require PostgreSQL to rewrite every row and hold a blocking `ACCESS EXCLUSIVE` lock for the duration — knowing which kind you're about to run is essential before touching a large production table.
- `DROP COLUMN` is immediately destructive with no undo; renaming a column or table breaks any existing reference to the old name until it's updated everywhere.
- `ADD CONSTRAINT` / `DROP CONSTRAINT` let you attach or remove rules on an existing table, but adding one validates every current row against it immediately — the full behavior of constraints themselves is Module 5's subject.
- Next, Topic 4 covers the more drastic operations of removing a table entirely, or wiping all its rows while keeping its structure, and the important safety distinctions between them.
