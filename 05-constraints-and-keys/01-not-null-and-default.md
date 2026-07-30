# NOT NULL and DEFAULT

## Learning Objectives

By the end of this section you should be able to:
- Declare a column `NOT NULL` and explain exactly what guarantee that adds
- Give a column a `DEFAULT` value and explain precisely when that default is applied
- Explain how `NOT NULL` and `DEFAULT` interact on the same column
- Add a `NOT NULL` constraint to a column on a table that already has rows, and handle any existing `NULL`s that would otherwise block it

## Prerequisites

- Module 4's "Creating Tables" and "Altering Tables" topics — you need to already be comfortable writing a `CREATE TABLE` statement and using `ALTER TABLE` to modify an existing one, since every example below is one or the other.
- Module 3's coverage of `NULL` semantics — this entire topic is about controlling when a column is and isn't allowed to hold `NULL`, so you need to already understand that `NULL` means "no value recorded," not zero, not an empty string, and not "unknown but present."

## Motivation

Not every column in a table should be optional. A customer record without a name is useless; an order without a status is ambiguous. Left unrestricted, a table will happily store rows with gaping holes in exactly the columns that matter most, and every single query, report, and downstream calculation built on top of that table now has to defensively check for missing data everywhere, forever. `NOT NULL` and `DEFAULT` are the two simplest, cheapest tools a relational database gives you to stop that problem before a single row is ever written.

## Problem Statement

Suppose you're designing a `customers` table for a simple e-commerce system. Without any constraints, this is completely legal:

```sql
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    TEXT,
    email        TEXT,
    loyalty_tier TEXT
);

INSERT INTO customers (full_name, email) VALUES (NULL, NULL);
```

That `INSERT` succeeds. You now have a customer with no name and no email — a row that is, for almost every practical purpose, garbage, sitting in your table exactly as "validly" as a complete row. Every report that displays customer names, every email campaign that sends to `email`, and every query that assumes `loyalty_tier` holds something meaningful now has to add defensive `NULL`-handling logic, and it has to do so *everywhere*, because the database itself never refused the bad row in the first place. The fix is to make the database refuse to store what should never be missing, and to auto-fill what has an obvious sensible default — rather than relying on every piece of application code that touches this table to remember to check.

## Concept

### `NOT NULL` — Making a Column Mandatory

Adding `NOT NULL` after a column's type tells PostgreSQL to reject any `INSERT` or `UPDATE` that would leave that column as `NULL` for a row.

```sql
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    TEXT NOT NULL,
    email        TEXT NOT NULL,
    loyalty_tier TEXT
);
```

Now the earlier bad insert is rejected outright:

```sql
INSERT INTO customers (full_name, email) VALUES (NULL, NULL);
```

```
ERROR:  null value in column "full_name" of relation "customers" violates not-null constraint
DETAIL:  Failing row contains (1, null, null, null).
```

Note `email` is also `NOT NULL` here, but PostgreSQL reports only the *first* violated constraint it encounters while checking the row — the row is still invalid for two reasons, but you only see one error at a time. `loyalty_tier` was left without `NOT NULL`, so it's genuinely optional:

```sql
INSERT INTO customers (full_name, email) VALUES ('Asha Rao', 'asha@example.com');

SELECT customer_id, full_name, email, loyalty_tier FROM customers;
```

```
 customer_id |  full_name  |        email        | loyalty_tier
-------------+-------------+----------------------+--------------
           1 | Asha Rao    | asha@example.com     |
(1 row)
```

`loyalty_tier` is blank (displayed as empty by `psql`, representing `NULL`) — that's allowed, because the column has no `NOT NULL` constraint.

### `DEFAULT` — Filling In a Value Automatically

`DEFAULT` supplies a value PostgreSQL uses automatically whenever an `INSERT` doesn't explicitly specify that column.

```sql
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    TEXT NOT NULL,
    email        TEXT NOT NULL,
    loyalty_tier TEXT NOT NULL DEFAULT 'standard',
    created_at   TIMESTAMP NOT NULL DEFAULT now()
);

INSERT INTO customers (full_name, email) VALUES ('Ben Ochieng', 'ben@example.com');

SELECT customer_id, full_name, loyalty_tier, created_at FROM customers;
```

```
 customer_id |  full_name  | loyalty_tier |         created_at
-------------+-------------+--------------+----------------------------
           1 | Ben Ochieng | standard     | 2026-07-31 09:14:02.113932
(1 row)
```

Neither `loyalty_tier` nor `created_at` was mentioned in the `INSERT`, yet both were filled in — `loyalty_tier` with the literal `'standard'`, and `created_at` with the result of calling the function `now()` **at the moment the row is inserted**. A `DEFAULT` can be any constant, or any expression/function PostgreSQL can evaluate, including functions like `now()` or `CURRENT_DATE` that produce a different value on every call.

A default is only used when the column is *omitted* from the `INSERT`'s column list — explicitly providing `NULL` still fails if the column is also `NOT NULL`, and explicitly providing any other value overrides the default entirely:

```sql
INSERT INTO customers (full_name, email, loyalty_tier)
VALUES ('Chen Wei', 'chen@example.com', 'gold');

INSERT INTO customers (full_name, email, loyalty_tier)
VALUES ('Dara Singh', 'dara@example.com', NULL);
```

```
ERROR:  null value in column "loyalty_tier" of relation "customers" violates not-null constraint
```

The first insert succeeds and stores `'gold'`, ignoring the default entirely, because a value was explicitly supplied. The second insert *explicitly* asked for `NULL`, which is not the same thing as omitting the column — `DEFAULT` only ever engages when a column is left out of the statement, so an explicit `NULL` against a `NOT NULL` column still fails.

### How `NOT NULL` and `DEFAULT` Interact

These two clauses are independent but frequently paired, and the combination is extremely common in real schemas:

| Combination | Behavior |
|---|---|
| `NOT NULL` alone | Column must always be given an explicit, non-`NULL` value on every `INSERT`. |
| `DEFAULT` alone | Column may be omitted (falls back to the default) or explicitly set to `NULL` (both are legal — `NULL` is still an allowed value here). |
| `NOT NULL DEFAULT ...` | Column may be omitted (falls back to the default), but can never be explicitly set to `NULL`. This is the most common pairing for "this must always have a sensible value, and here's what it should be when the caller doesn't care." |
| Neither | Column is fully optional; omitting it, or setting it explicitly, both leave/set it to `NULL`. |

`orders.status` is a good real-world case for `NOT NULL DEFAULT`:

```sql
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    status       TEXT NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(10, 2) NOT NULL
);

INSERT INTO orders (customer_id, total_amount) VALUES (1, 149.99);

SELECT order_id, customer_id, order_date, status, total_amount FROM orders;
```

```
 order_id | customer_id | order_date | status  | total_amount
----------+-------------+------------+---------+--------------
        1 |           1 | 2026-07-31 | pending |       149.99
(1 row)
```

Every new order automatically starts as `'pending'`, dated today, without the application ever having to think about it — but `status` can never silently become `NULL`, because it's also `NOT NULL`.

### Adding `NOT NULL` to a Table That Already Has Rows

The examples so far declare `NOT NULL` at `CREATE TABLE` time, when the table is empty and there's nothing to violate yet. In a live system, tables almost always already have data by the time you realize a column should have been mandatory. Attempting to add the constraint directly often fails:

```sql
-- Suppose loyalty_tier already has some existing NULL rows.
ALTER TABLE customers
    ALTER COLUMN loyalty_tier SET NOT NULL;
```

```
ERROR:  column "loyalty_tier" of relation "customers" contains null values
```

PostgreSQL checks *every existing row* before allowing the constraint, and refuses if even one row would violate it. This is the correct, safe behavior — the alternative (silently allowing the constraint while leaving pre-existing `NULL`s in place) would make `NOT NULL` a lie. There are two honest ways forward:

**Option 1 — Backfill the existing `NULL`s, then add the constraint:**

```sql
UPDATE customers
SET loyalty_tier = 'standard'
WHERE loyalty_tier IS NULL;

ALTER TABLE customers
    ALTER COLUMN loyalty_tier SET NOT NULL;
```

**Option 2 — Add a `DEFAULT` alongside the backfill**, so future rows never fall into the same trap:

```sql
ALTER TABLE customers
    ALTER COLUMN loyalty_tier SET DEFAULT 'standard';

UPDATE customers
SET loyalty_tier = 'standard'
WHERE loyalty_tier IS NULL;

ALTER TABLE customers
    ALTER COLUMN loyalty_tier SET NOT NULL;
```

Note that `SET DEFAULT` and `SET NOT NULL` are two entirely separate `ALTER COLUMN` clauses — adding a default does **not** retroactively fill in existing rows; it only affects future `INSERT`s that omit the column. The backfill `UPDATE` is always a separate, deliberate step.

`DROP NOT NULL` reverses the constraint just as easily:

```sql
ALTER TABLE customers
    ALTER COLUMN loyalty_tier DROP NOT NULL;
```

## Internal Working (Preview)

- **`NOT NULL`** is checked at the row level, on every `INSERT` and every `UPDATE` that touches the column, before the row is written to disk. Internally, every stored row (a "tuple") carries a small bitmap indicating which columns are `NULL` for that row; `NOT NULL` simply forbids the corresponding bit from ever being set for that column. It is one of the cheapest constraints PostgreSQL enforces — no lookups, no scanning other rows, just a check against the row currently being written.
- **`DEFAULT`** is stored as an expression in PostgreSQL's system catalog (specifically `pg_attrdef`), attached to the column. When an `INSERT` omits a column, the planner substitutes the catalog's stored default expression in place of a value before the row is constructed — for a literal like `'standard'` this is instant; for a function call like `now()`, the function is evaluated fresh at insert time, which is exactly why two different rows inserted seconds apart get two slightly different `created_at` values.
- Adding `SET NOT NULL` to an already-populated table requires PostgreSQL to scan the existing table once to verify no violating row exists — this is why it can be a relatively slow operation on a very large table, unlike adding the constraint at `CREATE TABLE` time on an empty table, which is instant.

```
INSERT omits column
        │
        ▼
Look up column's DEFAULT in catalog (pg_attrdef), if any
        │
        ▼
Evaluate the default expression now (literal or function call)
        │
        ▼
Row assembled with that value
        │
        ▼
NOT NULL check: is this column's final value NULL?
        │
   ┌────┴────┐
  yes        no
   │          │
 REJECT     WRITE ROW
```

## Real-World Analogy

Think of a government form with two kinds of fields. Some fields are marked with a red asterisk — "Full Legal Name," "Date of Birth" — and the clerk at the counter will hand the form straight back to you if you leave one blank; that's `NOT NULL`. Other fields already have something pre-printed in light gray ("Country: United States") that you can leave as-is or cross out and change; that's `DEFAULT`. A field can be both: pre-filled *and* marked mandatory, meaning you're free to accept the pre-filled value, but you're not allowed to erase it and hand in a blank.

## Why NOT NULL and DEFAULT Were Designed This Way

The relational model (introduced in Module 2) treats every column as holding a value describing some fact about a row — and `NULL` exists specifically to represent the honest absence of a fact, not a placeholder for "zero" or "empty text." Given that, a database needs *some* way to say "this fact is not optional for this kind of entity" — a customer must have a name; an order must have a status. `NOT NULL` is the most fundamental integrity rule a relational database offers precisely because it operationalizes that requirement at the one place guaranteed to see every write: the database engine itself, rather than trusting every application, script, and future developer to remember to check. `DEFAULT` exists as a companion because forcing a value to always be present is only user-friendly if the database can also supply a sensible one automatically — otherwise every `INSERT` statement across your entire codebase would need to be rewritten the moment a new mandatory column is added.

## Advantages

- **Centralized guarantee** — the rule "this column is never missing" is enforced once, by the database, for every application, script, or person that ever writes to the table — not re-implemented and potentially forgotten in a dozen places.
- **Fails fast and loudly** — a rejected `INSERT` surfaces a data problem immediately, at the moment it would occur, rather than allowing bad data to sit silently in the table until a downstream report or calculation breaks unexpectedly.
- **`DEFAULT` reduces boilerplate** — application code and ad hoc scripts don't need to know or supply routine values (like a creation timestamp or a starting status) on every insert.
- **Self-documenting schema** — reading a `CREATE TABLE` statement with `NOT NULL` and `DEFAULT` clauses tells you, at a glance, which columns are mandatory and what "typical" values look like, without reading any application code.

## Disadvantages / Limitations

- **Adding `NOT NULL` to a populated table isn't free** — you must first resolve every existing `NULL`, which sometimes requires a business decision (what *should* an old, missing value become?) rather than a purely mechanical fix.
- **A `DEFAULT` can mask a genuine data-entry omission** — if an application forgets to supply a value it should have supplied, a default silently fills the gap instead of surfacing the bug; defaults are best reserved for values that are genuinely optional or truly have a sensible fallback, not as a workaround for sloppy application code.
- **Neither constraint validates the *content* of a value** — `NOT NULL` only forbids absence; a `NOT NULL` `email` column happily accepts `'not an email'`. Validating the *shape* of a value beyond presence requires `CHECK` constraints (Topic 5 of this module).

## Best Practices

- Default to `NOT NULL` for every column unless you can name a concrete, legitimate reason a value might be genuinely unknown — it's far easier to relax a constraint later (`DROP NOT NULL`) than to discover years of accumulated `NULL`s you now have to clean up.
- Use `DEFAULT` for values that have an obvious, safe starting point (`status DEFAULT 'pending'`, `created_at DEFAULT now()`) but not for values where a wrong guess would be actively misleading (never default a `country` column to some arbitrary country just to satisfy `NOT NULL`).
- When adding `NOT NULL` to a live, populated table, always run the backfill `UPDATE` as a deliberate, reviewed step — never assume "no rows will be affected" without checking with `SELECT COUNT(*) FROM table WHERE column IS NULL;` first.
- Pair `NOT NULL` with `DEFAULT` whenever a column should never be blank but also has an obvious typical value — this is the most common and most useful combination in real schemas.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Believing `DEFAULT` retroactively fills in existing `NULL` rows | `DEFAULT` only ever applies to future `INSERT`s that omit the column; it has no effect on rows already in the table. Existing `NULL`s must be fixed with an explicit `UPDATE`. |
| Explicitly inserting `NULL` and expecting the column's `DEFAULT` to kick in instead | A default only applies when a column is *omitted* from the `INSERT`. Explicitly writing `NULL` is a deliberate value, not an omission, and still violates `NOT NULL` if present. |
| Adding `NOT NULL` directly to a populated table without checking for existing `NULL`s first | PostgreSQL scans the whole table and rejects the `ALTER TABLE` outright if even one row would violate the new constraint — always check and backfill first. |
| Using a `DEFAULT` to paper over a column that should really be required input from the caller | Defaults are for genuinely sensible fallbacks, not a way to avoid fixing application code that forgets to supply a value it should always know. |

## Interview Questions

1. **Q: What is the difference between a column being `NULL` and a column having a `DEFAULT` of `NULL`?**
   A: There's no real distinction in outcome — a column with no `DEFAULT` at all simply stores `NULL` when omitted from an `INSERT`, which is functionally identical to declaring `DEFAULT NULL`. The meaningful distinction is between a column having *some* non-`NULL` default (which fills in automatically when omitted) versus having none (which leaves the column as `NULL` when omitted), and separately, whether `NOT NULL` is present to forbid `NULL` outright regardless of how it would otherwise arise.

2. **Q: You need to add `NOT NULL` to a column on a table with 5 million existing rows, some of which have `NULL` in that column. What's the correct sequence of steps?**
   A: First identify and resolve the existing `NULL`s with a deliberate `UPDATE` (backfilling them with a real or sensible placeholder value based on business rules), optionally add a `DEFAULT` so future inserts don't reintroduce the problem, and only then run `ALTER TABLE ... ALTER COLUMN ... SET NOT NULL`, which PostgreSQL will now accept because no row violates it.

3. **Q: If a column is declared `DEFAULT 'standard'` but not `NOT NULL`, can a row still end up with `NULL` in that column?**
   A: Yes. `DEFAULT` only supplies a value when a column is omitted from an `INSERT`. Without `NOT NULL`, nothing stops a caller from explicitly writing `NULL` for that column, which overrides the default entirely and stores an actual `NULL`.

## Summary

- `NOT NULL` forbids a column from ever being `NULL`, enforced by the database on every `INSERT` and `UPDATE`, rather than trusting application code to check.
- `DEFAULT` supplies a value automatically whenever a column is *omitted* from an `INSERT` — it never applies to a column explicitly set to `NULL`, and it never retroactively affects existing rows.
- `NOT NULL DEFAULT ...` together is the most common real-world pairing: a column that must always hold something, with a sensible built-in fallback when the caller doesn't specify one.
- Adding `NOT NULL` to a populated table requires resolving any existing `NULL`s first — PostgreSQL will scan the table and reject the constraint otherwise.
- Neither constraint validates *what* the value is, only whether it's present — validating content requires `CHECK` constraints, covered later in this module.
