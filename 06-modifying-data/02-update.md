# UPDATE

## Learning Objectives

By the end of this section you should be able to:
- Write an `UPDATE` statement that changes one or more columns using the `SET` clause
- Write a `WHERE` clause for `UPDATE` that references other columns in the same row, not just literal values
- Use `UPDATE ... FROM` to update rows in one table based on matching values in another table
- Explain precisely why omitting `WHERE` from an `UPDATE` is dangerous, and how to protect yourself from doing it by accident

## Prerequisites

- [INSERT](01-insert.md) — you need rows already in a table before there's anything to update, and this topic assumes you understand how constraints are checked on write, first explained there.
- **Module 5 (Constraints & Keys)** — every constraint checked on `INSERT` is checked again on `UPDATE`, since an update produces a new version of a row that must still satisfy every rule.

## Motivation

Data is rarely static once inserted — salaries get raised, order statuses change, email addresses get corrected, prices fluctuate. `UPDATE` is the statement that changes values in rows that already exist, without touching their identity (their primary key) or removing them. It's arguably the single most operationally dangerous DML statement in this module, because a small mistake in its `WHERE` clause doesn't just retrieve the wrong data (as a mistaken `SELECT` would) — it silently *overwrites* the wrong data, and that mistake persists until someone notices.

## Problem Statement

Continuing with the `employees` table from Topic 1:

```
 id |     name     | department_id | salary
----+--------------+---------------+--------
  1 | Asha Rao     |             1 |  95000
  2 | Ben Ochieng  |             1 |  88000
  3 | Chen Wei     |             2 |  76000
  4 | Diego Marin  |             3 |  61000
```

Suppose Asha gets a promotion and a raise to 105000. You could, in theory, `DELETE` her row and `INSERT` a new one with the same data plus the new salary — but that's wasteful, it would generate a *new* `id` (breaking anything that referenced her old `id`, like rows in an `orders` table), and it would needlessly re-check every constraint on a full new row instead of just the one column that actually changed. What you actually want is to modify the existing row in place, changing only the `salary` column, leaving her `id`, `name`, and every other column untouched. That's exactly what `UPDATE` does.

## Concept

### Anatomy of `UPDATE`

```sql
UPDATE table_name
SET column1 = value1,
    column2 = value2
WHERE condition;
```

- `UPDATE table_name` — which table's rows will be modified.
- `SET column = value, ...` — which column(s) change, and to what new value. This can be a literal, an expression involving *the row's own current values*, or the result of a subquery.
- `WHERE condition` — which rows are affected. **This is the single most important clause in this entire statement** — omit it, and every row in the table is updated.

### A Single-Column Update

```sql
UPDATE employees
SET salary = 105000
WHERE id = 1;
```

```
UPDATE 1
```

PostgreSQL reports `UPDATE 1`, confirming exactly one row was modified. Re-querying:

```
 id |     name     | department_id | salary
----+--------------+---------------+--------
  1 | Asha Rao     |             1 | 105000
  2 | Ben Ochieng  |             1 |  88000
  3 | Chen Wei     |             2 |  76000
  4 | Diego Marin  |             3 |  61000
```

Only Asha's row changed — her `id` and every other column are exactly as they were.

### Updating Multiple Columns at Once

Separate multiple `column = value` assignments with commas in a single `SET` clause — this updates all of them together, as one atomic change to the row:

```sql
UPDATE employees
SET salary = 90000,
    department_id = 2
WHERE id = 2;
```

```
UPDATE 1
```

Both `salary` and `department_id` change for Ben's row in a single statement — not two separate updates. This matters: if you instead ran two separate `UPDATE` statements, another concurrent query could theoretically read the row in between them, seeing a half-updated state (Module 14 covers this concurrency concern in depth).

### `SET` Expressions Referencing the Row's Own Current Value

The right-hand side of a `SET` assignment can reference the row's *current* value of any column, not just a brand-new literal — extremely useful for relative changes like "give everyone a 10% raise":

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 1;
```

```
UPDATE 2
```

PostgreSQL reads each matching row's current `salary`, computes `salary * 1.10` using *that row's* value, and writes the result back — every matching row gets its own 10% increase, computed independently, not all set to one shared value.

### `WHERE` Referencing Other Columns

The `WHERE` clause isn't limited to comparing a column to a literal — it can compare one column in the row to another column in the same row, which is common for conditional business rules:

```sql
UPDATE employees
SET salary = salary * 1.05
WHERE salary < 80000
  AND department_id = department_id;   -- illustrative: compares a column to itself, always true
```

A more realistic example — give a catch-up raise only to employees who are paid less than a colleague-independent threshold *and* were hired before a cutoff date:

```sql
UPDATE employees
SET salary = 85000
WHERE salary < 85000
  AND hire_date < '2024-01-01';
```

Here `WHERE` combines a comparison against a literal (`hire_date < '2024-01-01'`) with a comparison involving the column being updated itself (`salary < 85000`) — PostgreSQL evaluates the `WHERE` condition against each row's *current, pre-update* values to decide whether that row qualifies at all, before computing any new value.

### `UPDATE ... FROM` — Updating Based on Another Table

Often the new value you want to `SET` doesn't come from the row itself or a literal — it comes from a *different table*. PostgreSQL's `UPDATE ... FROM` extension (not standard SQL, but supported by PostgreSQL and several other databases) lets you reference a second table directly inside the `UPDATE`:

```sql
UPDATE table_name
SET column1 = other_table.some_column
FROM other_table
WHERE table_name.matching_column = other_table.matching_column;
```

For example, suppose HR maintains a separate table of approved raises per department:

```sql
CREATE TABLE salary_adjustments (
    department_id INTEGER PRIMARY KEY,
    raise_percent NUMERIC NOT NULL
);

INSERT INTO salary_adjustments (department_id, raise_percent) VALUES
    (1, 8),   -- Engineering: 8% raise
    (2, 4),   -- Sales: 4% raise
    (3, 6);   -- Marketing: 6% raise
```

To apply each department's approved raise to every employee in that department, in one statement:

```sql
UPDATE employees
SET salary = employees.salary * (1 + salary_adjustments.raise_percent / 100)
FROM salary_adjustments
WHERE employees.department_id = salary_adjustments.department_id;
```

```
UPDATE 4
```

Conceptually, PostgreSQL matches each row of `employees` to the row(s) of `salary_adjustments` where the `WHERE` condition holds (much like a join, formally covered in Module 10), and for every matched pair, computes the `SET` expression using columns from *both* tables. An employee whose `department_id` doesn't appear anywhere in `salary_adjustments` simply isn't matched, and is left unchanged — `UPDATE ... FROM` only touches rows that find a match, exactly like a `WHERE` clause filters which rows to touch.

### `RETURNING` with `UPDATE`

Just like `INSERT`, `UPDATE` supports `RETURNING` to see the resulting rows without a follow-up `SELECT`:

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 1
RETURNING id, name, salary;
```

```
 id |    name     | salary
----+-------------+---------
  1 | Asha Rao    | 115500.00
  2 | Ben Ochieng | 108900.00
(2 rows)
```

`RETURNING` shows the **new**, post-update values — useful for confirming exactly what changed, especially when the new value was computed rather than a literal you already knew.

### The Danger of Omitting `WHERE`

`WHERE` is not a required clause syntactically — this statement is perfectly valid SQL:

```sql
UPDATE employees SET salary = 50000;
```

```
UPDATE 4
```

This overwrites **every single row's** `salary` to 50000 — Asha's, Ben's, Chen's, and Diego's, discarding every distinct salary that existed a moment ago, with no undo short of restoring from a backup or an open transaction (Module 14). Unlike a `SELECT` with no `WHERE` (which just harmlessly returns every row so you can look at it), an `UPDATE` with no `WHERE` **overwrites data that may be irreplaceable** — this is one of the most common and most damaging mistakes made against production databases, often caused by forgetting the `WHERE` clause entirely, or by a typo that comments it out or detaches it from the statement.

## Internal Working (Preview)

Conceptually, PostgreSQL processes an `UPDATE` like this:

```
 UPDATE statement
       │
       ▼
 Identify candidate rows: evaluate WHERE (and FROM's join condition, if present)
 against each row's CURRENT values
       │
       ▼
 For each qualifying row:
       │
       ├─▶ Compute each SET expression using that row's pre-update values
       ├─▶ Re-check every constraint (NOT NULL, CHECK, UNIQUE, FOREIGN KEY)
       │    against the row's NEW values, exactly as INSERT does
       │
       ▼
 If ANY qualifying row's new values fail ANY check → abort the WHOLE statement
       │
       ▼
 If ALL qualifying rows pass → PostgreSQL does NOT overwrite the old row in place;
 it writes a new row version and marks the old one obsolete (MVCC — MultiVersion
 Concurrency Control, covered in depth in Module 14), then updates any relevant indexes
```

An important internal detail: PostgreSQL doesn't literally overwrite bytes in the old row. It creates a **new row version** and marks the old one as no longer current — this is the mechanism (MVCC) that lets other transactions keep reading a consistent, unchanging view of the data while your update is in progress, without blocking them. You'll study this in full in Module 14, but it's worth knowing now that "update" at the SQL level does not necessarily mean "modify in place" at the storage level.

## Real-World Analogy

Think of `UPDATE` like editing a specific set of entries in a paper ledger with an eraser and pen — but a very disciplined clerk's version of it. Before making any change, the clerk first re-reads the rule book (the constraints from Module 5) to make sure the new value is still allowed (a salary can't become negative, an email can't collide with someone else's). The `WHERE` clause is the clerk's instruction for *which specific entries* to open the ledger to — say, "only entries belonging to the Engineering department." If you forget to give the clerk that instruction, they don't ask for clarification — they dutifully apply your change to every single entry in the entire ledger, because "update all of them" is a perfectly valid (if usually unintended) instruction. `UPDATE ... FROM` is like handing the clerk a second reference ledger (the raise-approval sheet) and saying "for each entry, look up the matching row over there and use its value," rather than telling the clerk the exact number yourself.

## Why UPDATE Was Designed This Way

`UPDATE` re-validates every constraint on the new row values, for the same reason `INSERT` does (Topic 1): the database's core guarantee is that data never sits in a state that violates its declared rules, and an update produces a *new* logical version of a row that must satisfy those rules just as much as a freshly inserted one. `WHERE` being optional (rather than mandatory) is a deliberate consistency choice, not an oversight: `SELECT`, `UPDATE`, and `DELETE` all share the same `WHERE` clause syntax and semantics — "no `WHERE` means every row qualifies" is one uniform rule across all three, rather than a special case invented just for `UPDATE`. The cost of that uniformity is that `UPDATE` (and `DELETE`, Topic 3) can be destructive in a way `SELECT` never is, which is exactly why disciplined habits (Best Practices, below) matter so much more for these two statements. `UPDATE ... FROM` exists because relational data is deliberately split across multiple tables (Module 2's relational model, and Module 5's foreign keys) rather than duplicated into one giant table — cross-table updates like "apply this department's approved raise" are a direct, everyday consequence of that normalized design, so the language needs a clean way to express them.

## Advantages

- **Precise, in-place modification** — only the rows and columns you specify change; a row's primary key, and every untouched column, remains exactly as it was.
- **Expressions can reference the row's own current values** — relative changes (`salary * 1.10`) are computed correctly per row, without you calculating each row's new value yourself beforehand.
- **`UPDATE ... FROM` avoids pulling data into an application** — a cross-table update that would otherwise require reading one table, computing values in application code, and issuing many individual updates can be expressed and executed as a single statement inside the database.
- **Constraints protect you from bad updates too** — the same `NOT NULL`/`CHECK`/`UNIQUE`/foreign key rules that protect `INSERT` protect `UPDATE`; you cannot accidentally update a row into an invalid state.

## Disadvantages / Limitations

- **No built-in "confirm before applying" step** — unlike some GUI tools, running an `UPDATE` in `psql` or a script applies it immediately (pending a transaction commit, Module 14); there's no native prompt asking "are you sure this will affect 4,000,000 rows?"
- **`WHERE` is optional, which is exactly what makes it dangerous** — the same flexibility that lets `WHERE` be omitted for a legitimate "update every row" case is what makes an accidentally-omitted `WHERE` catastrophic (see below).
- **`UPDATE ... FROM` syntax is a PostgreSQL/vendor extension, not standard SQL** — some other databases express the same idea differently (e.g., a `MERGE` statement or a joined subquery in the `SET` clause); if you need this logic to be portable across database vendors, check Module 22.

## Best Practices

- **Write and test the `WHERE` clause as a `SELECT` first.** Before running `UPDATE employees SET salary = 50000 WHERE department_id = 3;`, run `SELECT * FROM employees WHERE department_id = 3;` and visually confirm it returns exactly the rows you intend to change — then convert it to an `UPDATE` by keeping the identical `WHERE` clause.
- **Wrap risky updates in a transaction** (`BEGIN` ... `UPDATE` ... check the result ... `COMMIT` or `ROLLBACK`) so you can undo an update that turns out to be wrong before it's permanent — Module 14 covers this fully, but it's worth adopting the habit now.
- **Always double-check row counts.** PostgreSQL reports `UPDATE n` after every update — if you expected to change 3 rows and it reports `UPDATE 4000`, stop and investigate before doing anything else.
- **Use `RETURNING`** on important updates to see exactly what changed, in the same statement, rather than trusting your assumptions about which rows matched.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Running `UPDATE table SET column = value;` with no `WHERE`, intending to update only some rows | Updates every single row in the table — always double-check for a `WHERE` clause before running an `UPDATE`, especially one typed live against a real database. |
| Assuming `UPDATE ... FROM` behaves like an `INNER JOIN` that also updates unmatched rows | Rows in the target table with no match in the `FROM` table are simply left untouched, not set to `NULL` or errored — only matched rows are updated. |
| Writing `SET column = column + 1` believing it applies a single shared calculation to the whole table | Each qualifying row's own `column` value is used independently — every row gets its own value incremented by 1, not one calculation broadcast to all rows. |
| Forgetting that constraints are re-checked on update, and being surprised by a rejected `UPDATE` | An `UPDATE` producing a value that fails `NOT NULL`, `CHECK`, `UNIQUE`, or a foreign key is rejected exactly like a failing `INSERT` — the "new" value must satisfy every rule, not just the "old" one. |

## Interview Questions

1. **Q: What happens if you run `UPDATE employees SET salary = 50000;` with no `WHERE` clause?**
   A: Every row in the `employees` table has its `salary` set to 50000 — `WHERE` is optional, and its absence means "every row qualifies," identical to how a `SELECT` with no `WHERE` returns every row, except an `UPDATE` overwrites data rather than just displaying it.

2. **Q: How would you give every employee in a specific department a 10% raise in a single statement?**
   A: `UPDATE employees SET salary = salary * 1.10 WHERE department_id = <id>;` — the `WHERE` clause selects the affected rows, and `salary * 1.10` is computed per row using that row's own current salary.

3. **Q: What does `UPDATE ... FROM` let you do that a plain `UPDATE ... SET ... WHERE` cannot?**
   A: It lets the new value(s) in `SET`, and the condition in `WHERE`, reference columns from a second table, effectively joining the target table against another table during the update — useful for updates like "set each row's value based on a matching row in a reference table," which a single-table `UPDATE` has no way to express.

4. **Q: Are `NOT NULL`, `CHECK`, and foreign key constraints checked during `UPDATE`, or only during `INSERT`?**
   A: They're checked on both. `UPDATE` produces a new logical version of a row, and that new version must satisfy every declared constraint exactly as a freshly inserted row would — an update that would leave a row in an invalid state is rejected, and the entire statement fails.

## Summary

- `UPDATE table SET column = value, ... WHERE condition` modifies existing rows in place, changing only the specified columns.
- `SET` expressions can reference the row's own current values (`salary = salary * 1.10`), computed independently per matching row.
- `UPDATE ... FROM` lets you update a table's rows using values from a second table, matched via a `WHERE` condition — a PostgreSQL/vendor extension useful for cross-table updates.
- Omitting `WHERE` updates every row in the table — unlike a `WHERE`-less `SELECT`, this silently overwrites data and is one of the most common serious mistakes made against a live database.
- Every constraint from Module 5 is re-checked against a row's *new* values on every update, exactly as it is on insert.
- `RETURNING` shows the post-update values of affected rows in the same statement.
- Next, Topic 3 covers `DELETE` — removing rows entirely, where the "missing `WHERE`" danger is even more severe.
