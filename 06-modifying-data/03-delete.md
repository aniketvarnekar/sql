# DELETE

## Learning Objectives

By the end of this section you should be able to:
- Write a `DELETE` statement with a `WHERE` clause to remove specific rows
- Explain precisely why `DELETE` with no `WHERE` removes every row, and how to guard against triggering this by accident
- Use `DELETE ... USING` to remove rows from one table based on matching values in another
- Use `RETURNING` with `DELETE` to capture the rows that were removed
- Predict how a foreign key's `ON DELETE` action (from Module 5) changes what happens to related rows when you delete a referenced row

## Prerequisites

- [UPDATE](02-update.md) — the `WHERE`-clause dangers discussed there apply even more severely to `DELETE`, since removing a row is harder to walk back than overwriting one column.
- **Module 5 (Constraints & Keys)**, specifically foreign keys and their `ON DELETE` actions (`CASCADE`, `SET NULL`, `RESTRICT`, `NO ACTION`) — `DELETE` is where those actions actually take effect.

## Motivation

Rows don't just get added and edited — eventually, some need to be removed entirely: a cancelled order, a departing employee, test data left over from development, spam accounts. `DELETE` is the statement that removes rows outright, and it inherits every danger `UPDATE` has around a missing `WHERE` clause, plus a new dimension of danger: because tables are linked together (Module 5's foreign keys), deleting a row in one table can force PostgreSQL to also delete, block, or modify rows in a completely different table, depending on how that relationship was defined.

## Problem Statement

Suppose, alongside `employees`, you also track their orders:

```sql
CREATE TABLE orders (
    id           SERIAL PRIMARY KEY,
    employee_id  INTEGER REFERENCES employees(id),
    amount       NUMERIC NOT NULL CHECK (amount > 0),
    order_date   DATE NOT NULL DEFAULT CURRENT_DATE
);

INSERT INTO orders (employee_id, amount) VALUES
    (1, 4200),
    (1, 1800),
    (3, 950);
```

Now:
- Diego Marin (`id = 4`) leaves the company — his single row in `employees` should be removed.
- A batch of test rows accidentally inserted into `orders` needs to be cleared out entirely.
- Chen Wei's department is being shut down, and every employee in it should be removed — but you only know the department's *name*, not its `id`, so the condition lives in a *different* table.
- Someone needs to know exactly which orders were just deleted, for an audit log — without a separate query.
- Critically, when Asha Rao (`id = 1`, referenced by two rows in `orders`) is deleted, what happens to *her* orders? PostgreSQL needs a rule for that, and Module 5's foreign key `ON DELETE` actions are exactly that rule.

## Concept

### Anatomy of `DELETE`

```sql
DELETE FROM table_name
WHERE condition;
```

- `DELETE FROM table_name` — which table rows are removed from.
- `WHERE condition` — which rows qualify for removal. Exactly like `UPDATE`, this is optional syntactically — and exactly like `UPDATE`, omitting it is dangerous, only more so, because the result is rows *gone*, not just overwritten.

There is no column list and no `SET` clause — `DELETE` doesn't change values, it removes entire rows.

### `DELETE ... WHERE`

```sql
DELETE FROM employees
WHERE id = 4;
```

```
DELETE 1
```

PostgreSQL reports `DELETE 1`, confirming exactly one row was removed. Diego Marin's row no longer exists in `employees` at all — unlike an `UPDATE`, there is no "old version" of this row left behind to look at afterward.

You can delete based on any condition, just as with `SELECT` and `UPDATE`:

```sql
DELETE FROM orders
WHERE amount < 1000;
```

```
DELETE 1
```

This removes Chen Wei's 950 order, leaving Asha's two orders untouched.

### `DELETE` Without `WHERE` — Deletes Everything

This is valid, executable SQL:

```sql
DELETE FROM orders;
```

```
DELETE 2
```

Every remaining row in `orders` — both of Asha's — is gone. The table still exists (its structure, columns, and constraints are untouched — this is DML, not DDL), but it is now completely empty.

This is **the single most dangerous statement pattern in this module**. Consider what makes it so easy to trigger by accident:

- Typing `DELETE FROM employees;` and pressing Enter before finishing the `WHERE` clause you meant to add.
- A `WHERE` clause with a bug that always evaluates to true for every row (e.g., comparing a column to itself, or a logic error in an `OR` condition).
- Running a script against the wrong database entirely — the statement is correct, but the *target* was a mistake.

Unlike a `SELECT` with no `WHERE` (harmless — it just displays every row so you can look at it), a `DELETE` with no `WHERE` **permanently removes every row**, and — outside of an open transaction (Module 14) or a restored backup — there is no way to get that data back. This is precisely why the very first mention of this danger appeared all the way back in [Your First Query](../01-introduction/05-your-first-query.md)'s Common Mistakes table, and why it's worth repeating here in full depth.

**How to prevent this mistake:**
- Write and run the `WHERE` clause as a `SELECT` first, and only convert it to `DELETE` once you've visually confirmed it selects exactly the rows you intend to remove.
- Wrap the statement in an explicit transaction (`BEGIN` ... `DELETE` ... `ROLLBACK` if the reported row count looks wrong, `COMMIT` if it looks right) — Module 14 covers this pattern in depth, but it is worth adopting immediately.
- Many `psql` configurations and GUI tools support an "autocommit off" mode specifically so a destructive statement doesn't take effect until you explicitly confirm it.
- Some teams restrict `DELETE` (and `UPDATE`) without a `WHERE` clause via database-level permissions or code review policy, precisely because the syntax alone provides no protection.

### `DELETE ... USING` — Deleting Based on Another Table

Just as `UPDATE ... FROM` lets an update reference a second table, `DELETE ... USING` lets a delete's `WHERE` clause reference a second table, letting you delete rows based on a condition that lives elsewhere:

```sql
DELETE FROM table_name
USING other_table
WHERE table_name.matching_column = other_table.matching_column
  AND other_table.some_condition;
```

For example, to remove every employee in the `Support` department, when you only know the department by name:

```sql
DELETE FROM employees
USING departments
WHERE employees.department_id = departments.id
  AND departments.name = 'Support';
```

```
DELETE 1
```

Conceptually, PostgreSQL matches each row of `employees` against `departments` using the join-like condition in `WHERE`, and deletes every `employees` row that finds a match satisfying the full condition (including `departments.name = 'Support'`) — `departments` itself is untouched; only rows in the table named after `DELETE FROM` are ever removed.

### `RETURNING` with `DELETE`

Exactly as with `INSERT` and `UPDATE`, `DELETE` supports `RETURNING` to capture the rows that were removed, in the same statement:

```sql
DELETE FROM orders
WHERE order_date < '2025-01-01'
RETURNING id, employee_id, amount;
```

```
 id | employee_id | amount
----+-------------+--------
  8 |           2 |   500
  9 |           3 |   300
(2 rows)
```

This is especially valuable for `DELETE`, since there's no other way to see what was removed after the fact — the rows are gone from the table immediately, so `RETURNING` is your only chance to capture their values (for logging, auditing, or moving them elsewhere) as part of the same operation.

### Interaction with Foreign Key `ON DELETE` Actions

Module 5 introduced foreign key `ON DELETE` actions, which control what happens to *referencing* rows when a referenced row is deleted. `DELETE` is the statement where those actions actually activate. Recall the four options:

| `ON DELETE` action | Behavior when the referenced row is deleted |
|---|---|
| `RESTRICT` / `NO ACTION` (default if omitted) | The `DELETE` on the referenced row is itself rejected if any referencing row still exists. |
| `CASCADE` | Every referencing row is automatically deleted along with the referenced row. |
| `SET NULL` | Every referencing row's foreign key column is set to `NULL`; the referencing rows themselves survive. |
| `SET DEFAULT` | Every referencing row's foreign key column is set to that column's declared default value. |

Suppose `orders.employee_id` was declared with no `ON DELETE` action at all (the default, equivalent to `RESTRICT`):

```sql
CREATE TABLE orders (
    id           SERIAL PRIMARY KEY,
    employee_id  INTEGER REFERENCES employees(id),
    amount       NUMERIC NOT NULL CHECK (amount > 0),
    order_date   DATE NOT NULL DEFAULT CURRENT_DATE
);
```

Trying to delete Asha (`id = 1`), who still has orders referencing her:

```sql
DELETE FROM employees WHERE id = 1;
```

```
ERROR:  update or delete on table "employees" violates foreign key constraint "orders_employee_id_fkey" on table "orders"
DETAIL:  Key (id)=(1) is still referenced from table "orders".
```

The delete is rejected outright — PostgreSQL refuses to leave `orders` rows pointing at an `employee_id` that no longer exists. Now compare what happens with each alternative `ON DELETE` action declared on the foreign key instead:

```sql
-- CASCADE: deleting the employee also deletes their orders
employee_id INTEGER REFERENCES employees(id) ON DELETE CASCADE

DELETE FROM employees WHERE id = 1;
-- DELETE 1   (employees)  -- and both of Asha's orders are silently deleted too
```

```sql
-- SET NULL: deleting the employee leaves their orders, with employee_id now NULL
employee_id INTEGER REFERENCES employees(id) ON DELETE SET NULL

DELETE FROM employees WHERE id = 1;
-- DELETE 1   (employees)  -- Asha's two orders survive, but employee_id becomes NULL on both
```

This is why choosing the right `ON DELETE` action at table-design time (Module 5) is not a minor detail — it determines whether a single `DELETE` on a "parent" row silently ripples out and deletes data in other tables (`CASCADE`), leaves orphaned-but-intact records (`SET NULL`), or simply refuses to happen at all until you deal with the dependent rows yourself (`RESTRICT`, the safest default).

## Internal Working (Preview)

```
 DELETE statement
       │
       ▼
 Identify candidate rows: evaluate WHERE (and USING's join condition, if present)
       │
       ▼
 For each candidate row, BEFORE removing it:
       │
       ├─▶ Check whether any OTHER table's foreign key references this row
       │      ├─ If a referencing row exists AND the action is RESTRICT/NO ACTION → abort the WHOLE statement
       │      ├─ If the action is CASCADE → recursively delete the referencing row(s) first
       │      └─ If the action is SET NULL / SET DEFAULT → update the referencing row(s)' FK column first
       │
       ▼
 If all checks pass → mark the candidate row(s) as removed (again, via MVCC —
 not an immediate physical erase, so a concurrent transaction can still see the
 pre-delete state until your transaction commits; Module 14 covers this)
       │
       ▼
 Return RETURNING output (if any)
```

An important nuance: `CASCADE` deletes can themselves cascade further, if the referencing table is in turn referenced by yet another table with its own `ON DELETE CASCADE` — a single `DELETE` on one "root" row can ripple through several tables. This is powerful, but it's also why `CASCADE` should be chosen deliberately (Module 5) rather than by default "just in case."

## Real-World Analogy

Think of `DELETE` like a records office permanently shredding a specific file. `WHERE` is the instruction for which files to pull before shredding — no instruction means "shred every file in the entire cabinet," which is exactly as catastrophic as it sounds, and just as irreversible as a real paper shredder (there's no "un-shred" once it's done, unless you kept a backup copy elsewhere). Before shredding a file, the clerk (the DBMS) first checks a cross-reference index: does any *other* file in the building point to this one? If the office's policy for that kind of file is "never shred it while something still references it" (`RESTRICT`), the clerk refuses and hands you back an explanation. If the policy is "shred anything that depends on it too" (`CASCADE`), the clerk goes and shreds those dependent files as well, without asking again. If the policy is "just blank out the reference on dependent files" (`SET NULL`), the clerk leaves those files in the cabinet but crosses out the field that used to point to the shredded file.

## Why DELETE Was Designed This Way

`DELETE`'s lack of a mandatory `WHERE` mirrors `UPDATE`'s design (Topic 2) for the same reason: `SELECT`, `UPDATE`, and `DELETE` share one uniform rule about `WHERE` — no condition means every row qualifies — rather than each statement inventing its own special-case behavior. The foreign key interaction is a direct consequence of the relational model's core idea (Module 2) that related facts live in separate tables, linked by key values, instead of one denormalized table holding everything — the moment data is split across tables, deleting a row in one table raises an unavoidable question ("what about the rows elsewhere that pointed at it?"), and `ON DELETE` actions (Module 5) are SQL's explicit, declared answer to that question, checked and enforced automatically rather than left to inconsistent application-level logic scattered across every place a delete might happen.

## Advantages

- **Precise removal** — `DELETE ... WHERE` removes exactly the rows matching a condition, leaving everything else in the table untouched.
- **Referential integrity is enforced automatically** — you cannot accidentally leave "dangling" foreign key references behind; the `ON DELETE` action you declared in Module 5 governs the outcome consistently, every time, for every caller.
- **`DELETE ... USING` avoids fetch-then-delete round trips** — a condition living in another table can drive the deletion directly, in one statement, inside the database engine.
- **`RETURNING` captures otherwise-lost data** — since deleted rows are gone from the table immediately, `RETURNING` is the only way to see their final values as part of the same operation (e.g., for an audit log).

## Disadvantages / Limitations

- **Irreversible outside a transaction** — once committed, a `DELETE` cannot be undone by any SQL statement; recovery requires either an open, uncommitted transaction (Module 14) or restoring from a backup.
- **`WHERE` being optional is exactly what makes the "delete everything" mistake possible** — the same flexibility that supports a legitimate "clear this whole table" case is what makes an accidentally-omitted `WHERE` catastrophic.
- **Row-by-row deletion can be slow for very large tables** — `DELETE` (unlike `TRUNCATE`, Topic 4) removes and logs rows individually, which becomes a meaningful cost difference when clearing millions of rows at once.
- **`CASCADE` can produce surprising, wide-reaching effects** — a single `DELETE` on one row can silently remove many rows across several tables if `ON DELETE CASCADE` chains were set up without careful thought.

## Best Practices

- **Always run the intended `WHERE` clause as a `SELECT` first** to visually confirm the exact row set before converting it to a `DELETE` — this single habit prevents the vast majority of accidental mass deletions.
- **Wrap significant deletes in a transaction** and check the reported row count before committing (Module 14).
- **Know your foreign keys' `ON DELETE` actions before deleting a "parent" row** — check Module 5's constraint definitions (or query PostgreSQL's catalog) so a `DELETE` doesn't unexpectedly cascade or unexpectedly get rejected.
- **Prefer `RESTRICT`/`NO ACTION` as the default choice at table-design time** (Module 5) unless `CASCADE` or `SET NULL` is a deliberate, reviewed decision — an accidental `RESTRICT` rejection is recoverable (you just investigate and re-run); an accidental `CASCADE` deletion often is not.
- **Use `RETURNING` on deletes that matter** so you have a record of exactly what was removed, in case you need to reconstruct it later.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Running `DELETE FROM table;` with no `WHERE`, intending to remove only some rows | Removes every row in the table — always verify a `WHERE` clause is present (and correct) before running a `DELETE` against a real database. |
| Assuming a `DELETE` on a referenced row will just "quietly work" regardless of foreign keys | Without an explicit `ON DELETE CASCADE`/`SET NULL`, PostgreSQL rejects the delete outright (`RESTRICT`/`NO ACTION`, the default) if any row elsewhere still references it. |
| Not realizing `ON DELETE CASCADE` can delete far more data than expected | A cascade can ripple through multiple linked tables; always know your schema's cascade chains before deleting a row with many dependents. |
| Trying to query a deleted row's values after the fact, without having used `RETURNING` | Once a `DELETE` commits, the row's data is gone from the table; `RETURNING` is the only way to capture it in the same operation — there's no "undelete" query afterward. |

## Interview Questions

1. **Q: What happens if you run `DELETE FROM orders;` with no `WHERE` clause?**
   A: Every row in the `orders` table is removed — the table itself, its columns, and its constraints remain intact (this is DML, not DDL), but it ends up completely empty. This is irreversible outside of an open, uncommitted transaction or a restored backup.

2. **Q: You try to `DELETE` a row from a "parent" table, and PostgreSQL rejects it with a foreign key error. What's happening, and how do you fix it?**
   A: A foreign key elsewhere still references that row, and the foreign key's `ON DELETE` action is `RESTRICT` or `NO ACTION` (the default), which blocks deletion of a still-referenced row. To fix it, either delete/update the referencing rows first, or redefine the foreign key with `ON DELETE CASCADE` (to auto-delete referencing rows) or `ON DELETE SET NULL` (to null out the reference) if that behavior is actually desired.

3. **Q: How does `DELETE ... USING` differ from a plain `DELETE ... WHERE`?**
   A: `DELETE ... USING` lets the `WHERE` condition reference columns from a second table, effectively joining the target table against another table to decide which rows to delete — useful when the condition for deletion lives in a different table than the one being deleted from. A plain `DELETE ... WHERE` can only reference columns already in the target table (or a subquery).

4. **Q: Why is `RETURNING` particularly important for `DELETE`, compared to `SELECT`?**
   A: Because a `DELETE` removes rows immediately — once committed, there's no way to query those rows' values again. `RETURNING` captures the final values of deleted rows within the same statement, which is the only opportunity to see or log that data as part of the deletion itself.

## Summary

- `DELETE FROM table WHERE condition` removes rows matching the condition; unlike `UPDATE`, there's no `SET` clause — whole rows are removed, not individual column values changed.
- `DELETE` with no `WHERE` removes every row in the table — one of the most dangerous mistakes possible against a live database; always verify the `WHERE` clause (ideally by first running it as a `SELECT`) before executing.
- `DELETE ... USING` lets the deletion condition reference a second table, similar to how `UPDATE ... FROM` works.
- `RETURNING` captures the values of deleted rows within the same statement — the only way to see them once the delete commits.
- Foreign key `ON DELETE` actions from Module 5 (`RESTRICT`/`NO ACTION`, `CASCADE`, `SET NULL`, `SET DEFAULT`) determine what happens to referencing rows in other tables when a referenced row is deleted — and this is exactly where those actions take effect.
- Next, Topic 4 compares `DELETE` against `TRUNCATE`, PostgreSQL's other way to remove rows — and when each is the right tool.
