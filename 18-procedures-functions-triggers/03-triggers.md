# Triggers

## Learning Objectives

By the end of this section you should be able to:
- Write a PL/pgSQL trigger function that returns `TRIGGER`, and attach it to a table with `CREATE TRIGGER`
- Explain the difference between `BEFORE` and `AFTER` timing, and between row-level (`FOR EACH ROW`) and statement-level (`FOR EACH STATEMENT`) firing
- Use the special `NEW` and `OLD` record variables correctly for `INSERT`, `UPDATE`, and `DELETE` events
- Build a working audit-log trigger that records every change to a table automatically
- Identify why triggers are a common source of hidden, hard-to-debug side effects

## Prerequisites

- [Stored Procedures](02-stored-procedures.md) — trigger functions are written in the same PL/pgSQL block structure introduced across Topics 1 and 2; this topic assumes that syntax (`DECLARE`, `IF`, `RETURN`) is already familiar.
- [Module 6 — Modifying Data](../06-modifying-data/00-module-overview.md), specifically [INSERT](../06-modifying-data/01-insert.md) and [UPDATE](../06-modifying-data/02-update.md) — a trigger's entire purpose is to fire in response to these exact statements, so you need to already be comfortable with what each one does to a row before reasoning about when a trigger runs relative to it.
- [Module 5 — Constraints & Keys](../05-constraints-and-keys/00-module-overview.md) — constraints are the simpler, declarative alternative a trigger is sometimes reached for instead of; you need to know what a `CHECK` constraint or foreign key can already express to judge whether a trigger is solving a genuinely different problem.

## Motivation

Every mechanism covered so far in this module — functions, procedures — is something you call *explicitly*. But some rules need to hold no matter *how* a row gets changed: through your main application, through an ad hoc `psql` session, through a bulk data-loading script, through a colleague's one-off fix at 2 a.m. If a rule's enforcement depends on every single one of those code paths remembering to call the same function, it's not really a guarantee — it's a hope. A **trigger** is how the database attaches logic directly to a table itself, so that logic runs automatically, unconditionally, no matter which client or code path caused the change.

## Problem Statement

Suppose you need a complete, tamper-evident history of every change to an employee's salary — who changed it, when, and what the value was before and after — for compliance purposes. Relying on application code to remember to log this every time it updates `employees.salary` has an obvious hole: the very first time someone runs a manual `UPDATE employees SET salary = ... WHERE id = ...` directly against the database (to fix a one-off data-entry mistake, say), no audit row gets created, because there's no application code involved in that path at all. Any logging mechanism that lives *outside* the table can be bypassed by anything that talks to the table by a different route. What's needed is something wired directly into the table's own write path, so it's structurally impossible to change a row without it firing — which is exactly what a trigger provides.

## Concept

### The Audit-Log Schema

Continuing with the `employees` table from Module 6, add a table to hold the audit trail:

```sql
CREATE TABLE employees_audit (
    audit_id    SERIAL PRIMARY KEY,
    employee_id INTEGER NOT NULL,
    changed_at  TIMESTAMP NOT NULL DEFAULT now(),
    changed_by  TEXT NOT NULL DEFAULT current_user,
    operation   TEXT NOT NULL,
    old_salary  NUMERIC,
    new_salary  NUMERIC
);
```

### Step 1 — Write the Trigger Function

A trigger's logic lives in an ordinary-looking PL/pgSQL function, with one hard requirement: it must be declared `RETURNS TRIGGER`, and its body has access to two special record variables, `NEW` and `OLD`, that PostgreSQL populates automatically:

```sql
CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF (TG_OP = 'UPDATE' AND OLD.salary IS DISTINCT FROM NEW.salary) THEN
        INSERT INTO employees_audit (employee_id, operation, old_salary, new_salary)
        VALUES (NEW.id, 'UPDATE', OLD.salary, NEW.salary);
    ELSIF (TG_OP = 'INSERT') THEN
        INSERT INTO employees_audit (employee_id, operation, old_salary, new_salary)
        VALUES (NEW.id, 'INSERT', NULL, NEW.salary);
    ELSIF (TG_OP = 'DELETE') THEN
        INSERT INTO employees_audit (employee_id, operation, old_salary, new_salary)
        VALUES (OLD.id, 'DELETE', OLD.salary, NULL);
    END IF;

    RETURN NEW;
END;
$$;
```

| Piece | Meaning |
|---|---|
| `RETURNS TRIGGER` | The required return type for any function used as a trigger — it doesn't return an ordinary value; it returns a row telling PostgreSQL how to proceed (see below). |
| `NEW` | A record holding the row's values *after* the change — available for `INSERT` and `UPDATE`, not for `DELETE` (there is no "after" row once a row is deleted). |
| `OLD` | A record holding the row's values *before* the change — available for `UPDATE` and `DELETE`, not for `INSERT` (there is no "before" row for a brand-new one). |
| `TG_OP` | A special variable PostgreSQL provides inside every trigger function, holding the text `'INSERT'`, `'UPDATE'`, or `'DELETE'` — useful when one function is shared across multiple event types, as here. |
| `IS DISTINCT FROM` | A `NULL`-safe inequality comparison — ordinary `<>` would silently fail to flag a change if either side were `NULL`; `IS DISTINCT FROM` treats `NULL` as a comparable value instead of propagating `NULL` the way most operators do. |
| `RETURN NEW;` | For a row-level `BEFORE` or `AFTER` trigger on `INSERT`/`UPDATE`, returning `NEW` tells PostgreSQL to proceed with that row's values. Returning `NULL` from a `BEFORE` trigger instead cancels the operation for that row entirely (used for validation, not needed here). |

### Step 2 — Attach the Trigger with `CREATE TRIGGER`

```sql
CREATE TRIGGER trg_employees_audit
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
EXECUTE FUNCTION log_salary_change();
```

| Piece | Meaning |
|---|---|
| `CREATE TRIGGER trg_employees_audit` | Names the trigger — trigger names must be unique per table, and are worth naming descriptively (see Best Practices). |
| `AFTER INSERT OR UPDATE OR DELETE ON employees` | The **timing** (`AFTER`, as opposed to `BEFORE`) and the **events** this trigger fires for, on the `employees` table. A single trigger definition can cover multiple event types, as here, with `TG_OP` used inside the function body to tell them apart. |
| `FOR EACH ROW` | Fires once **per affected row** — for a multi-row `UPDATE` affecting five rows, this trigger fires five times, once per row, each time with that row's own `NEW`/`OLD` values. |
| `EXECUTE FUNCTION log_salary_change();` | The trigger function to run — must already exist and must return `TRIGGER`. |

### Seeing It Work

```sql
UPDATE employees SET salary = 105000 WHERE name = 'Asha Rao';
```

```
UPDATE 1
```

```sql
SELECT employee_id, operation, old_salary, new_salary, changed_at
FROM employees_audit
ORDER BY audit_id DESC
LIMIT 1;
```

```
 employee_id | operation | old_salary | new_salary |         changed_at
-------------+-----------+------------+------------+----------------------------
           1 | UPDATE    |      95000 |     105000 | 2026-07-31 09:14:02.918273
(1 row)
```

No application code inserted that audit row — it happened purely as a structural consequence of the `UPDATE` statement itself, regardless of what issued it.

### `BEFORE` vs. `AFTER`

| Timing | Runs... | Can it change the row being written? | Typical use |
|---|---|---|---|
| `BEFORE` | Immediately before the row is actually written | Yes — modifying and returning `NEW` changes what actually gets stored; returning `NULL` cancels the write for that row entirely | Validation, normalization (e.g., trimming whitespace, forcing lowercase on an email column), defaulting a column based on other columns |
| `AFTER` | Immediately after the row has been written | No — the row is already final; `NEW`/`OLD` here are for *reading*, not altering, what happened | Auditing, cascading updates to other tables, notifying/queuing side effects |

A `BEFORE` validation example, showing a trigger cancelling an insert outright:

```sql
CREATE OR REPLACE FUNCTION reject_negative_salary()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.salary < 0 THEN
        RAISE EXCEPTION 'Salary cannot be negative, got %', NEW.salary;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_reject_negative_salary
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION reject_negative_salary();
```

This is deliberately a slightly artificial example — a `CHECK (salary >= 0)` constraint (Module 5) expresses the exact same rule far more simply, with no procedural code at all. It's shown here to make the mechanical point that `BEFORE` triggers can reject a write; [When to Use Each](04-when-to-use-each.md) covers in depth why a constraint, not a trigger, is almost always the right choice for a rule this simple.

### Row-Level vs. Statement-Level

`FOR EACH ROW` fires the trigger function once per affected row. `FOR EACH STATEMENT` fires it exactly once per statement, regardless of how many rows it touched — and critically, a statement-level trigger's function has **no access to `NEW` or `OLD`**, since there's no single row to attach them to.

```sql
CREATE OR REPLACE FUNCTION log_bulk_update()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'An UPDATE statement just ran against employees';
    RETURN NULL; -- ignored for statement-level triggers, but a value must be returned
END;
$$;

CREATE TRIGGER trg_bulk_update_notice
AFTER UPDATE ON employees
FOR EACH STATEMENT
EXECUTE FUNCTION log_bulk_update();
```

Statement-level triggers are useful when you care that *something happened* (e.g., "invalidate a cache after any bulk update") rather than caring about the specifics of every individual row that changed.

## Internal Working (Preview)

For a row-level trigger, this is the firing sequence for a single `UPDATE` statement affecting several rows:

```
 UPDATE employees SET salary = ... WHERE department_id = 1;
                     │
                     ▼
      BEFORE STATEMENT triggers fire once (if any exist)
                     │
                     ▼
   For each matching row:
       ┌─────────────────────────────────────┐
       │ BEFORE ROW triggers fire             │
       │   (can modify NEW, or cancel the row)│
       │              │                        │
       │              ▼                        │
       │   Row is actually written             │
       │              │                        │
       │              ▼                        │
       │ AFTER ROW triggers fire                │
       │   (NEW/OLD reflect the final row)     │
       └─────────────────────────────────────┘
                     │
                     ▼
       AFTER STATEMENT triggers fire once (if any exist)
```

If multiple triggers of the same timing and event exist on the same table, PostgreSQL fires them in alphabetical order by trigger name — which is precisely why descriptive, consistently prefixed trigger names (see Best Practices) matter beyond mere readability; the name also controls execution order when order matters. A `BEFORE` trigger that returns `NULL` stops the operation for that row entirely — no later `BEFORE` trigger, no actual write, and no `AFTER` trigger runs for that row, since as far as the database is concerned that row's change never happened.

## Real-World Analogy

A trigger is like a security camera and alarm system wired directly into a building's doors, not a logbook a security guard has to remember to fill out. It doesn't matter whether an employee, a visitor, or a maintenance worker opens the door — the system fires the same way every single time, because it's physically attached to the door itself, not to any particular person's memory or diligence. A logging process that depends on every person who might open a door remembering to write it down is exactly the fragile, bypassable arrangement the Problem Statement described — and exactly what wiring the alarm directly into the door eliminates.

## Why Triggers Were Designed This Way

Module 5 established that constraints let a table enforce simple, self-contained rules about its own rows — "this value can't be null," "this value must be positive." But some rules genuinely can't be expressed as a constraint: they involve *history* (what was the value before this change?), *other tables* (log this change into a separate audit table), or *side effects* (recompute a cached total elsewhere) rather than a single row judging only its own column values in isolation. Triggers exist to cover exactly that gap — they're the mechanism for "when this table changes, automatically also do this," attached at the same structural layer as a constraint (the table itself, inside the DBMS), so the same guarantee constraints give you — this rule cannot be bypassed no matter which client touches the table — extends to logic more complex than a constraint's single-row, single-table expression can capture.

## Advantages

- **Impossible to bypass accidentally.** Because a trigger is attached to the table itself, not to any particular application code path, it fires identically regardless of what issued the write — solving exactly the audit-logging problem from the Problem Statement.
- **Centralizes cross-cutting logic.** Cache invalidation, audit trails, and denormalized-column maintenance can be defined once, at the table, instead of duplicated across every place that writes to it.
- **Can enforce validation logic too complex for a `CHECK` constraint.** A `BEFORE` trigger can reference other tables or prior row state (via `OLD`) in ways a single-row `CHECK` constraint fundamentally cannot.

## Disadvantages / Limitations

- **Hidden side effects.** A plain-looking `UPDATE employees SET salary = ...` can silently also write to an audit table, recompute other data, or reject the statement outright — none of which is visible by reading the `UPDATE` statement itself. This is the single most-cited real-world complaint about triggers, and it is a genuine, not exaggerated, cost.
- **Hard to debug and trace.** When something in a table unexpectedly changes, or a statement unexpectedly fails, a developer unfamiliar with the schema may not think to check for a trigger at all — the failure or side effect appears to come from "nowhere" relative to the statement they actually ran.
- **Performance is easy to misjudge.** A row-level trigger runs once *per row* — on a bulk `UPDATE` touching a million rows, a trigger with even a small amount of per-row work multiplies that cost a million times over, in a way that's invisible just from looking at the original statement.
- **Can cascade unpredictably.** A trigger on table A that writes to table B, which itself has a trigger that writes to table C, creates a chain of side effects that can be very difficult to reason about, and in pathological cases can even recurse back onto the original table.
- **Ordering across multiple triggers is a subtle, easy-to-forget rule.** Same-timing, same-event triggers on one table fire in alphabetical order by name — a fact that's easy to be unaware of until it causes a real bug.

## Best Practices

- Prefer a constraint (Module 5) over a trigger whenever the rule genuinely fits — single-row, single-table validation like "salary must be non-negative" belongs in a `CHECK` constraint, not a `BEFORE` trigger, precisely because a constraint is simpler, declarative, and doesn't hide procedural code behind an innocent-looking statement.
- Name triggers with a consistent, descriptive prefix (`trg_<table>_<purpose>`, as used throughout this topic) — both for readability and because trigger firing order for same-timing/same-event triggers depends on alphabetical name order.
- Keep trigger function bodies small and fast, especially for row-level triggers on tables with high write volume — every row-level trigger's cost is multiplied by every row a statement touches.
- Document a table's triggers prominently wherever the table itself is documented — triggers are invisible in application code, so the table's own documentation is often the only place a future reader will discover they exist at all.
- Avoid trigger chains that write back into the same table that fired them unless the recursive behavior is genuinely intentional and carefully bounded — accidental recursive triggers can cause runaway loops or very hard-to-diagnose bugs.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Forgetting a trigger function must return `TRIGGER`, not `VOID` or another ordinary type | `CREATE TRIGGER` requires a function declared `RETURNS TRIGGER` — using any other return type raises an error when you try to attach it. |
| Trying to read `OLD` inside an `INSERT` trigger, or `NEW` inside a `DELETE` trigger | `OLD` doesn't exist for `INSERT` (there is no prior row) and `NEW` doesn't exist for `DELETE` (there is no resulting row) — referencing the wrong one raises an error. |
| Using `<>` instead of `IS DISTINCT FROM` to detect a changed value | If either `OLD.salary` or `NEW.salary` were `NULL`, `<>` returns `NULL` (neither true nor false) rather than reliably detecting a change — silently skipping logging for that case. |
| Assuming a `BEFORE` trigger's modifications to `NEW` are visible to an `AFTER` trigger on the same event without re-reading | This is actually correct behavior worth double-checking, not assuming: an `AFTER` trigger sees the *final* `NEW` values, including any changes a `BEFORE` trigger made — but it's a common point of confusion when several triggers interact, worth verifying explicitly rather than guessing. |
| Writing a very expensive row-level trigger and only ever testing it against a handful of rows | A trigger's cost multiplies by every affected row; a trigger that feels instantaneous in a five-row test can meaningfully slow down a bulk operation touching hundreds of thousands of rows in production. |

## Interview Questions

1. **Q: What is the difference between `NEW` and `OLD` in a trigger function, and when is each available?**
   A: `NEW` holds the row's values after the change and is available for `INSERT` and `UPDATE`. `OLD` holds the row's values before the change and is available for `UPDATE` and `DELETE`. Neither exists for the event where it wouldn't make sense — no `OLD` for `INSERT` (nothing existed before), no `NEW` for `DELETE` (nothing exists after).

2. **Q: What's the practical difference between a `BEFORE` trigger and an `AFTER` trigger?**
   A: A `BEFORE` trigger runs before the row is written and can modify the row (by changing and returning `NEW`) or cancel the operation entirely (by returning `NULL`). An `AFTER` trigger runs once the row is already final and cannot alter what was written — it's used for reacting to a change (logging, cascading updates) rather than shaping it.

3. **Q: Why might you choose `FOR EACH STATEMENT` over `FOR EACH ROW`?**
   A: When the logic only cares that a statement happened at all — not the specifics of every individual row it touched — such as invalidating a cache once after a bulk update. A statement-level trigger fires exactly once per statement regardless of row count, and has no access to `NEW`/`OLD` since there's no single row to attach them to.

4. **Q: What is a real risk of relying heavily on triggers, beyond raw performance cost?**
   A: Hidden side effects and debuggability — a trigger makes an ordinary-looking `INSERT`/`UPDATE`/`DELETE` silently do more than the statement itself suggests, which can make bugs hard to trace (a developer reading only the application code may never discover the trigger exists) and can create difficult-to-follow cascades if triggers on different tables write to each other.

## Summary

- A trigger function must be written in a procedural language (PL/pgSQL in this course) and must return `TRIGGER`; it is attached to a table with `CREATE TRIGGER`, specifying timing (`BEFORE`/`AFTER`), events (`INSERT`/`UPDATE`/`DELETE`), and granularity (`FOR EACH ROW`/`FOR EACH STATEMENT`).
- `NEW` and `OLD` give a row-level trigger access to a row's values after and before the change, respectively — availability depends on the event.
- `BEFORE` triggers can modify or cancel a write before it happens; `AFTER` triggers react to a write that has already happened and cannot alter it.
- Triggers exist to guarantee logic runs no matter which client or code path caused a change — the same guarantee constraints (Module 5) give you, extended to logic too complex (cross-table, history-aware) for a constraint to express.
- Their biggest real cost is hidden side effects: an innocent-looking statement can silently do far more than it appears to, which makes triggers powerful but genuinely easy to overuse — a theme the next topic examines directly.
