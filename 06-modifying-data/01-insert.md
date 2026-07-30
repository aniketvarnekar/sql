# INSERT

## Learning Objectives

By the end of this section you should be able to:
- Write single-row and multi-row `INSERT` statements using explicit column lists
- Use `INSERT ... SELECT` to copy or derive rows from an existing query instead of typing literal values
- Use PostgreSQL's `RETURNING` clause to get inserted values back without a second query
- Predict exactly what happens — and what error PostgreSQL raises — when an insert violates a `NOT NULL`, `CHECK`, or `UNIQUE` constraint

## Prerequisites

- **Module 4 (Database & Table Design)** — you need a table already defined (columns and types) before you can insert into it.
- **Module 5 (Constraints & Keys)** — `NOT NULL`, `UNIQUE`, `CHECK`, and primary/foreign keys are enforced on every insert; this topic assumes you know what each constraint means, and focuses on how they behave *at insert time*.
- [Your First Query](../01-introduction/05-your-first-query.md) — you've already seen a basic multi-row `INSERT`; this topic goes much deeper on the statement.

## Motivation

A table with no rows is just an empty shape — every piece of real, useful data that will ever exist in a database gets there through one statement: `INSERT`. Whether it's a single new user signing up, a nightly batch job loading a million rows from another system, or a report table being populated from the results of a calculation, it all comes down to variations on `INSERT`. Getting comfortable with its full syntax — and with exactly how it fails when data doesn't meet the rules you defined in Module 5 — is the foundation of every data-modifying task in this module.

## Problem Statement

Suppose you've just created the `employees` table from Module 4/5:

```sql
CREATE TABLE departments (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE employees (
    id             SERIAL PRIMARY KEY,
    name           TEXT NOT NULL,
    email          TEXT UNIQUE,
    department_id  INTEGER REFERENCES departments(id),
    salary         NUMERIC CHECK (salary > 0),
    hire_date      DATE NOT NULL DEFAULT CURRENT_DATE
);
```

It's completely empty. You need to:
- Add a single new hire.
- Add a whole batch of new hires from a hiring event, in one statement, without running `INSERT` five separate times.
- Populate an archive table from rows that already exist in `employees`, without retyping every value by hand.
- Immediately know the auto-generated `id` of a row you just inserted, so your application can use it right away.
- Understand exactly what PostgreSQL tells you when someone tries to insert an employee with no name, a negative salary, or an email that's already taken.

`INSERT` — in its several forms — is the answer to every one of these.

## Concept

### Anatomy of `INSERT`

```sql
INSERT INTO table_name (column1, column2, ...)
VALUES (value1, value2, ...);
```

- `INSERT INTO table_name` — which table receives the new row(s).
- `(column1, column2, ...)` — the **explicit column list**: which columns you're supplying values for, and in what order. This is optional (PostgreSQL will assume *all* columns, in table-definition order, if omitted) but strongly recommended — see Best Practices.
- `VALUES (...)` — the actual value(s), positionally matched to the column list.

Any column you don't mention in the column list gets its **default value** — either an explicit `DEFAULT` from the column definition (like `hire_date`'s `CURRENT_DATE`), the special auto-increment behavior of `SERIAL` (for `id`), or `NULL` if the column has no default and allows nulls.

### Single-Row Insert

```sql
INSERT INTO departments (name) VALUES ('Engineering');

INSERT INTO employees (name, email, department_id, salary)
VALUES ('Asha Rao', 'asha@example.com', 1, 95000);
```

Neither statement mentions `id` (it's `SERIAL` — PostgreSQL assigns the next value automatically) or `hire_date` (it has a `DEFAULT CURRENT_DATE`, so it's filled in with today's date unless you override it).

### Multi-Row Insert

Instead of running `INSERT` once per row, PostgreSQL lets you supply multiple parenthesized value lists in a single statement:

```sql
INSERT INTO departments (name) VALUES
    ('Sales'),
    ('Marketing'),
    ('Support');

INSERT INTO employees (name, email, department_id, salary) VALUES
    ('Ben Ochieng', 'ben@example.com',   1, 88000),
    ('Chen Wei',    'chen@example.com',  2, 76000),
    ('Diego Marin', 'diego@example.com', 3, 61000);
```

This inserts three rows into `employees` in one statement. This matters for more than convenience — a single multi-row `INSERT` is dramatically faster than the same number of single-row inserts, because it's parsed, planned, and (by default) committed as one operation instead of many round trips between your application and the database (more in Internal Working below).

Querying the result:

```sql
SELECT id, name, department_id, salary FROM employees ORDER BY id;
```

```
 id |     name     | department_id | salary
----+--------------+---------------+--------
  1 | Asha Rao     |             1 |  95000
  2 | Ben Ochieng  |             1 |  88000
  3 | Chen Wei     |             2 |  76000
  4 | Diego Marin  |             3 |  61000
(4 rows)
```

### `INSERT ... SELECT` — Inserting Derived or Copied Data

Every example so far used literal values you typed by hand (`VALUES (...)`). But very often, the rows you want to insert are the *result of a query against existing data* — copying rows into an archive table, seeding one table from another, or inserting rows computed from other rows. PostgreSQL lets you replace `VALUES (...)` with any `SELECT` statement, as long as the number and types of columns it returns match the column list you're inserting into:

```sql
INSERT INTO table_name (column1, column2, ...)
SELECT expr1, expr2, ...
FROM some_other_table
WHERE condition;
```

For example, suppose you keep an `employees_archive` table with the same shape as `employees`, for staff who've left:

```sql
CREATE TABLE employees_archive (
    id             INTEGER PRIMARY KEY,
    name           TEXT NOT NULL,
    email          TEXT,
    department_id  INTEGER,
    salary         NUMERIC,
    hire_date      DATE,
    archived_on    DATE NOT NULL DEFAULT CURRENT_DATE
);
```

To copy every employee currently in the `Support` department into the archive:

```sql
INSERT INTO employees_archive (id, name, email, department_id, salary, hire_date)
SELECT id, name, email, department_id, salary, hire_date
FROM employees
WHERE department_id = 3;
```

No literal values were typed at all — the `SELECT` supplies the rows, row by row, exactly as `VALUES` would have. This also works for *deriving* new values rather than only copying them verbatim — for instance, giving every employee a one-time bonus row in a separate `bonuses` table computed as a percentage of their current salary:

```sql
INSERT INTO bonuses (employee_id, bonus_amount)
SELECT id, salary * 0.05
FROM employees
WHERE salary > 70000;
```

Here `salary * 0.05` is computed per row by the `SELECT`, and the computed value — not a copied column — is what gets inserted. `INSERT ... SELECT` is one of the most common ways large amounts of realistic data move between tables, and it never requires the data to leave the database or pass back through your application at all.

### The `RETURNING` Clause

A very common need immediately after inserting a row is to know what was actually stored — especially auto-generated values like a `SERIAL` id or a computed `DEFAULT`. Without any special clause, you'd have to run a second `SELECT` to look the row back up. PostgreSQL (this is a PostgreSQL-specific convenience — not part of every database) lets you append `RETURNING` directly to the `INSERT` statement to get those values back in the very same round trip:

```sql
INSERT INTO employees (name, email, department_id, salary)
VALUES ('Farah Nasser', 'farah@example.com', 2, 82000)
RETURNING id, hire_date;
```

```
 id | hire_date
----+------------
  5 | 2026-07-31
(1 row)
```

You can return any column, an expression, or literally everything with `RETURNING *`:

```sql
INSERT INTO employees (name, email, department_id, salary)
VALUES ('Grace Liu', 'grace@example.com', 1, 91000)
RETURNING *;
```

```
 id |   name    |       email        | department_id | salary | hire_date
----+-----------+---------------------+----------------+--------+------------
  6 | Grace Liu | grace@example.com  |              1 |  91000 | 2026-07-31
(1 row)
```

`RETURNING` also works with multi-row inserts (returning one row of output per row inserted) and, as later topics show, with `UPDATE` and `DELETE` too.

### What Happens When a Constraint Is Violated

Every constraint from Module 5 is checked on every row an `INSERT` tries to add. If a row fails a check, PostgreSQL **rejects that entire statement** — including every other row in the same multi-row insert — and raises an error. No partial insert happens.

**`NOT NULL` violation** — trying to insert an employee with no name:

```sql
INSERT INTO employees (email, department_id, salary)
VALUES ('noname@example.com', 1, 60000);
```

```
ERROR:  null value in column "name" of relation "employees" violates not-null constraint
DETAIL:  Failing row contains (7, null, noname@example.com, 1, 60000, 2026-07-31).
```

**`CHECK` violation** — trying to insert a negative salary:

```sql
INSERT INTO employees (name, email, department_id, salary)
VALUES ('Hana Suzuki', 'hana@example.com', 1, -5000);
```

```
ERROR:  new row for relation "employees" violates check constraint "employees_salary_check"
DETAIL:  Failing row contains (7, Hana Suzuki, hana@example.com, 1, -5000, 2026-07-31).
```

**`UNIQUE` violation** — trying to insert an email that already exists:

```sql
INSERT INTO employees (name, email, department_id, salary)
VALUES ('Ian Osei', 'asha@example.com', 2, 70000);
```

```
ERROR:  duplicate key value violates unique constraint "employees_email_key"
DETAIL:  Key (email)=(asha@example.com) already exists.
```

**Foreign key violation** — trying to insert an employee into a department that doesn't exist:

```sql
INSERT INTO employees (name, email, department_id, salary)
VALUES ('Jae Park', 'jae@example.com', 999, 70000);
```

```
ERROR:  insert or update on table "employees" violates foreign key constraint "employees_department_id_fkey"
DETAIL:  Key (department_id)=(999) is not present in table "departments".
```

In every case: the statement fails, nothing is inserted, and the error message tells you precisely which constraint was violated and which value caused it — this is the DBMS enforcing the rules you declared in Module 5, centrally and automatically, exactly as introduced back in Module 1's discussion of what a DBMS gives you over plain files.

## Internal Working (Preview)

When PostgreSQL executes an `INSERT`, roughly this happens per statement:

```
 INSERT statement
       │
       ▼
 Parse & resolve column list / VALUES or SELECT source
       │
       ▼
 For each candidate row:
       │
       ├─▶ Apply column DEFAULTs for any omitted column
       ├─▶ Check NOT NULL constraints
       ├─▶ Check CHECK constraints
       ├─▶ Check UNIQUE / PRIMARY KEY constraints (via an index lookup)
       ├─▶ Check FOREIGN KEY constraints (a lookup against the referenced table)
       │
       ▼
 If ANY row in the statement fails ANY check → abort the WHOLE statement, no rows written
       │
       ▼
 If ALL rows pass → write all rows to the table's storage, update indexes,
 return RETURNING output (if any)
```

A key detail: a single `INSERT` statement — even a multi-row one — is atomic as a unit (this is a preview of Module 14's transaction guarantees). If row 3 of a 10-row `INSERT` fails a `CHECK` constraint, none of the 10 rows are inserted, not even the 2 that would have passed. This is why the error messages above show `DETAIL: Failing row contains (...)` — PostgreSQL is telling you exactly which candidate row caused the whole batch to be rejected.

`UNIQUE` and `PRIMARY KEY` checks are enforced using the index PostgreSQL automatically builds for those constraints (Module 13 covers indexes in depth) — this is *why* uniqueness checks stay fast even as a table grows to millions of rows: PostgreSQL doesn't scan the whole table to check for a duplicate, it does an index lookup.

## Real-World Analogy

Think of `INSERT` like filing a new form at a government records office. A clerk (the DBMS) checks your form before it's accepted into the filing cabinet: is every required field filled in (`NOT NULL`)? Does the value in the "age" field make sense, e.g. not negative (`CHECK`)? Does this national ID number already belong to someone else on file (`UNIQUE`)? Does the "department code" you wrote actually correspond to a real department (foreign key)? If the clerk finds *any* problem, they hand the entire form back to you unfiled and tell you exactly which field is wrong — they don't file the "good" fields and discard the "bad" one. `INSERT ... SELECT` is like handing the clerk an entire pre-filled ledger to re-file into a new cabinet, rather than writing out each form by hand. `RETURNING` is the clerk immediately stamping your copy with the file number they assigned, instead of making you come back later to ask what number your form got.

## Why INSERT Was Designed This Way

`INSERT` enforces every constraint *before* committing any change, and rejects the entire statement on any failure, because a relational database's core promise (established in Module 1 and formalized in Module 14's ACID guarantees) is that data is never left in a state that violates the rules you declared. If PostgreSQL instead inserted the rows that passed and silently skipped the ones that failed, your table's data could quietly drift out of sync with your application's assumptions — a bug that might not surface until much later, and would be far harder to trace back to its cause than an immediate, specific error at insert time. The `INSERT ... SELECT` form exists because SQL is declarative (Module 1, Topic 2): rather than writing a loop in application code that reads rows one at a time and issues one `INSERT` per row, you describe *which rows, in what shape* you want inserted, and let the database's engine perform the whole operation internally, in one pass, without ever shipping the intermediate data back and forth. `RETURNING` exists purely as a PostgreSQL convenience to avoid an unnecessary second round trip for information the database already computed while inserting (like a `SERIAL` id) — many other databases require a separate query for this.

## Advantages

- **Constraint enforcement is automatic and centralized** — you never need application code to re-check "is this salary positive?" before every insert; the database guarantees it, always, for every caller.
- **Multi-row inserts are efficient** — one parse/plan/commit cycle for many rows, instead of the overhead of many separate statements.
- **`INSERT ... SELECT` avoids moving data through your application** — copying or deriving thousands of rows between tables happens entirely inside the database engine.
- **`RETURNING` saves a round trip** — you get generated values (like auto-increment IDs) back immediately, without a follow-up query.

## Disadvantages / Limitations

- **All-or-nothing per statement can be inconvenient** — if you need to insert 1,000 rows and skip only the ones that violate a constraint (rather than rejecting the whole batch), a plain `INSERT` won't do that; you'd need to either pre-filter the data yourself or use more advanced techniques (like `ON CONFLICT`, Topic 5) for specific conflict cases.
- **Multi-row `INSERT` with very many rows can hit statement-size or memory practicality limits** — for truly massive bulk loads, PostgreSQL's specialized `COPY` command (outside this module's scope) is generally preferred over an enormous single `INSERT`.
- **`INSERT ... SELECT` still validates every constraint per row**, so it offers no way to bypass the rules from Module 5 — this is a deliberate limitation, not a bug, but it means a large copy can still fail entirely on one bad row deep into the source data.

## Best Practices

- **Always specify an explicit column list**, even when inserting into every column. `INSERT INTO employees VALUES (...)` silently breaks the moment someone adds, removes, or reorders a column in the table; `INSERT INTO employees (name, email, ...) VALUES (...)` keeps working correctly regardless of the table's column order.
- **Prefer multi-row `INSERT` over many single-row inserts** when adding several related rows at once — fewer round trips, one atomic unit.
- **Use `RETURNING` instead of a follow-up `SELECT`** whenever you need a value the database just generated (like a `SERIAL` id) — it's strictly more efficient and can't race with another concurrent insert changing what "the last inserted row" means.
- **Test constraint-violating inserts deliberately** during development (e.g., try inserting a duplicate email on purpose) so you know exactly what error your application needs to handle, rather than discovering it in production.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Omitting the column list and relying on positional order (`INSERT INTO employees VALUES (...)`) | Breaks silently (or inserts values into the wrong columns) the moment the table's column order changes — always name columns explicitly. |
| Assuming a multi-row `INSERT` inserts "the good rows" and skips "the bad ones" | A single `INSERT` statement is all-or-nothing — if any one row violates a constraint, the entire statement is rejected and zero rows are inserted. |
| Forgetting that `SERIAL`/auto-generated columns should not be listed in the column list with a manual value (unless intentionally overriding it) | Manually supplying an `id` value defeats the point of `SERIAL` and can create duplicate-key conflicts later when the sequence catches up to a manually-inserted value. |
| Using a second `SELECT` after `INSERT` to fetch the row you just inserted | Unnecessary and slightly race-prone under concurrent access — use `RETURNING` in the same statement instead. |

## Interview Questions

1. **Q: What happens if row 3 of a 10-row `INSERT` statement violates a `CHECK` constraint?**
   A: The entire statement fails and none of the 10 rows are inserted — not even the 9 that would have passed. `INSERT` is all-or-nothing per statement; PostgreSQL never partially applies a multi-row insert.

2. **Q: What is `RETURNING`, and why is it useful?**
   A: A PostgreSQL-specific clause appended to `INSERT` (and `UPDATE`/`DELETE`) that returns the actual values of the affected row(s) in the same statement — most commonly used to retrieve auto-generated values like a `SERIAL` primary key without a separate follow-up `SELECT`.

3. **Q: How does `INSERT ... SELECT` differ from `INSERT ... VALUES`, and when would you use it?**
   A: `VALUES` supplies literal, hand-typed rows; `SELECT` supplies rows computed from an existing query against the database's own data. You'd use `INSERT ... SELECT` to copy rows between tables (e.g., archiving) or to insert values derived from existing data (e.g., a bonus computed as a percentage of an existing salary), without pulling that data out to an application and back.

4. **Q: Why does PostgreSQL reject an entire insert rather than inserting the valid rows and reporting an error for the invalid ones?**
   A: Because a relational database guarantees your data never ends up in a state that violates its declared constraints; silently inserting some rows and skipping others could leave your table in a state your application doesn't expect, with no clear signal that anything was skipped. Failing the whole statement makes the failure explicit and immediate.

## Summary

- `INSERT INTO table (columns) VALUES (...)` adds one or more new rows; always specify the column list explicitly.
- Multiple rows can be inserted in a single statement by supplying multiple parenthesized value lists, which is both more convenient and more efficient than repeated single-row inserts.
- `INSERT ... SELECT` inserts rows produced by a query instead of literal values — used for copying data between tables or inserting values derived from existing rows.
- PostgreSQL's `RETURNING` clause returns the actual inserted values (including auto-generated ones like `SERIAL` ids) in the same round trip as the insert.
- Every `NOT NULL`, `CHECK`, `UNIQUE`, and foreign key constraint from Module 5 is checked on every row of every insert; if any row fails any check, the entire statement is rejected and nothing is written.
- Next, Topic 2 covers `UPDATE` — changing the values of rows that already exist.
