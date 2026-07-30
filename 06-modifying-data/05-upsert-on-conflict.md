# UPSERT with ON CONFLICT

## Learning Objectives

By the end of this section you should be able to:
- Explain the "insert, or update if it already exists" problem and why naive application-level solutions to it are unsafe
- Write `INSERT ... ON CONFLICT DO NOTHING` to silently skip a conflicting insert
- Write `INSERT ... ON CONFLICT (...) DO UPDATE SET ...` to update the existing row instead
- Use the special `EXCLUDED` pseudo-table to reference the values that were about to be inserted, from inside the `DO UPDATE` clause
- Explain why `ON CONFLICT` avoids race conditions that a "check first, then insert" approach in application code cannot avoid

## Prerequisites

- [INSERT](01-insert.md) — `ON CONFLICT` is a clause attached directly to `INSERT`; you need to be comfortable with ordinary `INSERT` syntax first.
- **Module 5 (Constraints & Keys)**, specifically `UNIQUE` constraints and primary keys — a "conflict," in this topic's sense, specifically means a `UNIQUE`/primary key violation, so you need to know what those constraints guarantee.
- [UPDATE](02-update.md) — the `DO UPDATE` form of `ON CONFLICT` behaves like an `UPDATE`'s `SET` clause, applied conditionally.

## Motivation

An extremely common real-world need doesn't fit neatly into "insert a new row" or "update an existing row" alone — it's "insert this row, *unless one like it already exists*, in which case update it instead." Think of syncing inventory counts from a warehouse feed, recording a user's latest login timestamp, or upserting a row keyed by an external system's ID. Handling this correctly — especially when multiple processes might be doing it *at the same time* — is surprisingly easy to get wrong if you reach for the obvious "check first, then decide" approach. PostgreSQL's `ON CONFLICT` clause solves this cleanly, in a single atomic statement.

## Problem Statement

Suppose you track warehouse stock in an `inventory` table, keyed by a product's SKU (stock-keeping unit, a unique product code):

```sql
CREATE TABLE inventory (
    sku             TEXT PRIMARY KEY,
    quantity        INTEGER NOT NULL DEFAULT 0,
    last_restocked  DATE
);

INSERT INTO inventory (sku, quantity, last_restocked) VALUES
    ('WIDGET-100', 50, '2026-07-20'),
    ('GADGET-200', 30, '2026-07-22');
```

A nightly feed from the warehouse reports current stock for a list of SKUs — some already exist in `inventory`, some are brand new. For each SKU, you want to: insert it if it's new, or update its `quantity` and `last_restocked` if it already exists.

The naive approach, in application logic, looks like this (described in prose, not SQL, since this is exactly the anti-pattern being avoided):

1. `SELECT * FROM inventory WHERE sku = 'WIDGET-100';`
2. If no row came back, run an `INSERT`.
3. If a row came back, run an `UPDATE`.

This looks reasonable — until two processes run it concurrently for the same SKU. Both might run step 1 at nearly the same instant, both see "no row exists," and both then attempt an `INSERT` in step 2. The second `INSERT` fails with a `UNIQUE`/primary key violation (Module 5, and Topic 1 of this module), because the first process's insert already committed in the tiny window between the two processes' checks. This is a classic **race condition** (formally addressed by Module 14's discussion of concurrency and isolation) — and application code trying to patch around it (catching the error and retrying as an `UPDATE`) is fragile, easy to get subtly wrong, and adds complexity that shouldn't be necessary for what is conceptually a simple, single operation.

## Concept

### The `ON CONFLICT` Clause

PostgreSQL lets you attach an `ON CONFLICT` clause directly to an `INSERT` statement, telling the database exactly what to do *if* the row you're trying to insert collides with an existing `UNIQUE` or primary key value — all within one atomic statement, with no separate check beforehand:

```sql
INSERT INTO table_name (columns...)
VALUES (values...)
ON CONFLICT (conflict_target)
DO NOTHING | DO UPDATE SET ...;
```

- **`conflict_target`** — the column (or columns) whose `UNIQUE`/primary key constraint might be violated. This tells PostgreSQL exactly which constraint you mean, in case a table has more than one.
- **`DO NOTHING`** — if a conflict occurs, silently skip this row; no error, no change to the existing row.
- **`DO UPDATE SET ...`** — if a conflict occurs, update the existing row instead, using a `SET` clause just like `UPDATE`'s.

### `ON CONFLICT DO NOTHING`

Use this when a conflicting row should simply be left alone — the insert attempt is silently ignored, with no error and no modification to the row that was already there:

```sql
INSERT INTO inventory (sku, quantity, last_restocked)
VALUES ('WIDGET-100', 999, '2026-07-31')
ON CONFLICT (sku) DO NOTHING;
```

```
INSERT 0 0
```

PostgreSQL reports `INSERT 0 0` — zero rows inserted — because `WIDGET-100` already existed, and `DO NOTHING` means exactly that: nothing happened, and critically, **no error was raised**, unlike a plain `INSERT` hitting the same conflict (Topic 1). The existing row's `quantity` (still 50) is completely untouched:

```sql
SELECT * FROM inventory WHERE sku = 'WIDGET-100';
```

```
    sku     | quantity | last_restocked
------------+----------+----------------
 WIDGET-100 |       50 | 2026-07-20
(1 row)
```

This is ideal for "insert if new, otherwise leave it exactly as it is" scenarios — for example, recording a user's very first login without ever overwriting it on subsequent logins.

### `ON CONFLICT (...) DO UPDATE SET ...` — True Upsert

This is the full "insert, or update if it exists" behavior, commonly called an **upsert** (a portmanteau of "update" and "insert"):

```sql
INSERT INTO inventory (sku, quantity, last_restocked)
VALUES ('WIDGET-100', 999, '2026-07-31')
ON CONFLICT (sku)
DO UPDATE SET quantity = 999,
              last_restocked = '2026-07-31';
```

```
INSERT 0 1
```

PostgreSQL reports `INSERT 0 1` — a slightly unusual-looking but meaningful count: zero rows were freshly inserted, one row was affected via the conflict path. Querying confirms the existing row was updated in place, with its `sku` (the conflict target) unchanged:

```
    sku     | quantity | last_restocked
------------+----------+----------------
 WIDGET-100 |      999 | 2026-07-31
(1 row)
```

### The `EXCLUDED` Pseudo-Table

Hard-coding the same literal values twice (once in `VALUES`, again in `DO UPDATE SET`) is repetitive and error-prone — especially once the values being inserted are the result of a computation rather than a literal. PostgreSQL provides a special pseudo-table named `EXCLUDED` inside the `DO UPDATE` clause, representing the row that *would have been inserted* had there been no conflict — letting you reference those values directly instead of retyping them:

```sql
INSERT INTO inventory (sku, quantity, last_restocked)
VALUES ('WIDGET-100', 999, '2026-07-31')
ON CONFLICT (sku)
DO UPDATE SET quantity = EXCLUDED.quantity,
              last_restocked = EXCLUDED.last_restocked;
```

This produces exactly the same result as the previous example, but now the values are only written once (inside `VALUES`), and `DO UPDATE SET` simply references what would have been inserted via `EXCLUDED.column_name`. This becomes especially powerful when the update should be a *combination* of the new and existing values, not simply an overwrite — for example, adding newly-arrived stock to whatever quantity is already on hand, rather than replacing it:

```sql
INSERT INTO inventory (sku, quantity, last_restocked)
VALUES ('WIDGET-100', 25, '2026-08-01')
ON CONFLICT (sku)
DO UPDATE SET quantity = inventory.quantity + EXCLUDED.quantity,
              last_restocked = EXCLUDED.last_restocked;
```

```
INSERT 0 1
```

```sql
SELECT * FROM inventory WHERE sku = 'WIDGET-100';
```

```
    sku     | quantity | last_restocked
------------+----------+----------------
 WIDGET-100 |     1024 | 2026-08-01
(1 row)
```

Here `inventory.quantity` refers to the row's **existing** value (999 from before) and `EXCLUDED.quantity` refers to the **incoming** value (25) from this statement's `VALUES` — the two are added together (999 + 25 = 1024) in a single atomic step. This pattern — accumulating a delta onto an existing total — is exactly the kind of operation that is unsafe to express as a separate `SELECT` followed by an `UPDATE` in application code, for the race-condition reasons explored below.

### Handling a Brand-New Row in the Same Statement

Crucially, the *same* statement handles both cases — new and existing — without you needing to know in advance which one applies:

```sql
INSERT INTO inventory (sku, quantity, last_restocked)
VALUES ('THINGAMAJIG-300', 75, '2026-08-01')
ON CONFLICT (sku)
DO UPDATE SET quantity = inventory.quantity + EXCLUDED.quantity,
              last_restocked = EXCLUDED.last_restocked;
```

```
INSERT 1 0
```

Since `THINGAMAJIG-300` didn't previously exist, no conflict occurred at all — PostgreSQL reports `INSERT 1 0` (one row freshly inserted, zero via the conflict path), and the `DO UPDATE` clause is simply never invoked for this row. This is the entire point: one statement, correctly handling "new" and "already exists" as two branches of the same atomic operation, decided by the database itself rather than by your application checking beforehand.

### Adding a Conditional Guard with `WHERE`

`DO UPDATE` can itself carry a `WHERE` clause, letting you update only under an additional condition — for example, only overwrite `last_restocked` if the incoming date is actually newer than what's on file:

```sql
INSERT INTO inventory (sku, quantity, last_restocked)
VALUES ('WIDGET-100', 10, '2026-07-15')
ON CONFLICT (sku)
DO UPDATE SET quantity = inventory.quantity + EXCLUDED.quantity,
              last_restocked = EXCLUDED.last_restocked
WHERE EXCLUDED.last_restocked > inventory.last_restocked;
```

If the incoming `last_restocked` (2026-07-15) is *older* than the existing one (2026-08-01, from the previous example), this `WHERE` condition is false, and the row is left entirely unchanged — even the `quantity` addition is skipped, because the entire `DO UPDATE` action is conditional on this `WHERE`.

## Internal Working (Preview)

```
 INSERT ... ON CONFLICT (target) DO NOTHING | DO UPDATE ...
       │
       ▼
 Attempt to insert the row, as an ordinary INSERT would
       │
       ▼
 Does it violate the UNIQUE/PRIMARY KEY constraint named by (target)?
       │
       ├─ NO  → row inserted normally, exactly like a plain INSERT
       │
       └─ YES → instead of raising an error and aborting:
                  │
                  ├─ DO NOTHING  → discard the attempted row, no error, no change
                  │
                  └─ DO UPDATE   → run the DO UPDATE's SET clause against the
                                    EXISTING row, with EXCLUDED bound to the
                                    values that were about to be inserted,
                                    optionally filtered by DO UPDATE's own WHERE
       │
       ▼
 The ENTIRE statement — the conflict check and the resulting action — happens
 as one atomic operation, with no window for another transaction to interleave
 between "check" and "act"
```

This atomicity is the crucial internal detail. PostgreSQL doesn't implement `ON CONFLICT` as "run a `SELECT`, then decide" internally either — the conflict detection and the resulting `INSERT`-or-`UPDATE` action happen as a single, indivisible operation at the storage-engine level, using the same unique index (Module 13) that enforces the constraint in the first place. Two concurrent `ON CONFLICT` upserts targeting the same key are automatically serialized by the database (one proceeds, the other briefly waits its turn) rather than both racing to check-then-act independently.

## Real-World Analogy

Think of a hotel front desk handling a returning guest's room key card. A naive, unsafe process would be: the clerk looks up whether this guest already has an active key on file, and if not, issues a new one — but if two clerks at two different desks are checking the same guest's name at nearly the same instant, both might see "no key on file" and both attempt to issue a brand-new key, causing a conflict at the key-card system's back end. `ON CONFLICT` is like a single, disciplined key-card machine that the clerk hands the guest's name to directly: the machine itself checks whether a key already exists for that name and either issues a fresh one or reprograms the existing one to the requested room, as one indivisible action at the machine's software level — no two clerks can ever be caught making the same "no key exists yet" assumption at the same time, because the decision and the action are the same atomic step, not two separate steps a human clerk performs based on a possibly-outdated observation.

## Why ON CONFLICT Was Designed This Way

The insert-or-update problem is fundamentally a **concurrency** problem as much as a data-shape problem: any implementation that separates "check whether a conflicting row exists" from "act on that finding" as two distinct steps has an unavoidable gap between them, during which another transaction can change the answer to the check you already made (this general class of bug is explored fully in Module 14's discussion of race conditions and isolation levels). `ON CONFLICT` exists specifically to collapse "check" and "act" into a single atomic statement handled entirely inside the database engine, using the same underlying mechanism (a unique index lookup) that already enforces the constraint being checked — rather than asking every application developer to correctly reason about race conditions themselves, every single time this pattern comes up. This is a direct extension of SQL's declarative philosophy (Module 1): you describe the desired *outcome* ("this row should end up existing with these values, whether or not it already did") and let the database guarantee that outcome correctly and atomically, rather than you writing out the imperative "check, then branch" logic yourself.

## Advantages

- **Eliminates the check-then-insert race condition entirely** — the conflict check and the resulting action happen as one atomic database operation, so no window exists for another process to interleave.
- **A single statement replaces multi-step application logic** — no need for a `SELECT`, a conditional branch, and a separate `INSERT`/`UPDATE`, all coordinated correctly by hand.
- **`EXCLUDED` lets updates reference incoming values directly** — enabling powerful patterns like accumulating a delta onto an existing value (`inventory.quantity + EXCLUDED.quantity`) in one step.
- **A conditional `WHERE` on `DO UPDATE`** lets you express nuanced business rules (e.g., "only overwrite if the incoming data is newer") without extra round trips.

## Disadvantages / Limitations

- **Requires a `UNIQUE` or primary key constraint to exist on the conflict target** — `ON CONFLICT` cannot detect "conflicts" based on arbitrary business logic that isn't backed by an actual constraint; if two rows should be considered "the same" by some rule that isn't enforced as `UNIQUE`, this clause has nothing to key off of.
- **`ON CONFLICT` syntax is PostgreSQL-specific** — other databases express the same upsert idea with different syntax (e.g., a `MERGE` statement, or `INSERT ... ON DUPLICATE KEY UPDATE` in some other systems); portable code targeting multiple vendors needs to account for this (Module 22).
- **Only one conflict target can be handled per clause** — if a table has multiple separate `UNIQUE` constraints that could each independently conflict, you generally need to pick the specific one this statement is meant to resolve, or restructure the approach.

## Best Practices

- **Always specify the conflict target explicitly** (`ON CONFLICT (sku)`), even if the table has only one `UNIQUE` constraint — it documents your intent and avoids ambiguity if another constraint is added later.
- **Prefer `ON CONFLICT` over "check, then insert" application logic** for any insert-or-update scenario under concurrent access — this is the single most important takeaway of this topic.
- **Use `EXCLUDED` instead of retyping literal values** in `DO UPDATE SET`, both to avoid duplication and because it correctly generalizes to computed/derived incoming values.
- **Add a `WHERE` clause to `DO UPDATE` when "always overwrite" isn't actually the right behavior** — e.g., guard against an out-of-order or stale update overwriting newer data with older data.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Implementing "insert or update" as a `SELECT` followed by a conditional `INSERT`/`UPDATE` in application code | Introduces a race condition under concurrent access — two processes can both see "no existing row" and both attempt a conflicting insert; `ON CONFLICT` avoids this entirely by making the check-and-act one atomic operation. |
| Forgetting to name the conflict target (`ON CONFLICT` with no column list) when a `DO UPDATE` is present | `ON CONFLICT DO UPDATE` requires specifying which constraint the conflict refers to (via a column list or constraint name) — omitting it is a syntax error in that form (it's only optional when paired with `DO NOTHING`, meaning "any conflict at all"). |
| Retyping literal values in `DO UPDATE SET` instead of using `EXCLUDED` | Duplicates the same values in two places, and breaks the moment the incoming values are computed rather than literal — `EXCLUDED.column` always refers to the row that was about to be inserted. |
| Assuming `DO UPDATE SET quantity = EXCLUDED.quantity` overwrites, while intending to add the incoming amount to the existing amount | `EXCLUDED.quantity` alone is the new value; to accumulate rather than overwrite, you need `inventory.quantity + EXCLUDED.quantity`, explicitly referencing both the existing and incoming values. |

## Interview Questions

1. **Q: What problem does `INSERT ... ON CONFLICT` solve, and why isn't a `SELECT`-then-`INSERT`/`UPDATE` in application code an adequate solution?**
   A: It solves the "insert this row, or update it if it already exists" (upsert) problem. A `SELECT`-then-branch approach in application code has an unavoidable gap between checking whether a row exists and acting on that finding — under concurrent access, two processes can both observe "no row exists" and both attempt a conflicting insert, causing an error or inconsistent data. `ON CONFLICT` performs the check and the resulting action as a single atomic database operation, eliminating that gap.

2. **Q: What is `EXCLUDED`, and what does it represent?**
   A: `EXCLUDED` is a special pseudo-table available inside an `ON CONFLICT ... DO UPDATE` clause, representing the row that was about to be inserted (i.e., the values from the `INSERT`'s `VALUES`/`SELECT`) before the conflict was detected. It lets `DO UPDATE SET` reference those incoming values directly, e.g. `SET quantity = EXCLUDED.quantity`, instead of retyping literals.

3. **Q: What is the difference between `ON CONFLICT DO NOTHING` and `ON CONFLICT DO UPDATE`?**
   A: `DO NOTHING` silently discards the conflicting insert attempt, leaving the existing row completely untouched and raising no error. `DO UPDATE` instead modifies the existing row according to a `SET` clause (which can reference both the existing row's columns and the incoming `EXCLUDED` values), effectively performing a true insert-or-update (upsert).

4. **Q: How would you write an upsert that adds newly-arrived stock to an existing inventory count, rather than overwriting it, using `ON CONFLICT`?**
   A: `INSERT INTO inventory (sku, quantity) VALUES ('WIDGET-100', 25) ON CONFLICT (sku) DO UPDATE SET quantity = inventory.quantity + EXCLUDED.quantity;` — `inventory.quantity` is the row's existing value, `EXCLUDED.quantity` is the newly-incoming value, and the two are summed atomically as part of the same statement.

## Summary

- The insert-or-update ("upsert") problem arises whenever you need to add a row if it's new, or update it if a matching row already exists.
- A "check first, then insert or update" approach in application code has a race-condition gap under concurrent access — two processes can both see "no row exists" and collide.
- `INSERT ... ON CONFLICT (target) DO NOTHING` silently skips an insert that would violate a `UNIQUE`/primary key constraint, with no error and no change to the existing row.
- `INSERT ... ON CONFLICT (target) DO UPDATE SET ...` performs a true upsert — updating the existing row instead of failing — as a single atomic statement.
- The `EXCLUDED` pseudo-table, available inside `DO UPDATE`, refers to the values that were about to be inserted, letting you combine incoming and existing values (e.g., accumulating a delta) without retyping literals.
- `ON CONFLICT` avoids race conditions specifically because the conflict check and the resulting action happen as one indivisible database operation, using the same unique index that enforces the underlying constraint.
- Next, the [Module Summary](06-module-summary.md) consolidates everything this module covered — `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, and `ON CONFLICT` — into one recap.
