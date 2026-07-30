# NULL and Three-Valued Logic

## Learning Objectives

By the end of this section you should be able to:
- State precisely what `NULL` represents, and explain why it is not the same as zero, an empty string, or `FALSE`
- Explain three-valued logic (`TRUE`, `FALSE`, `UNKNOWN`) and produce the truth tables for `AND`, `OR`, and `NOT` including `UNKNOWN`
- Explain why `NULL = NULL` does not evaluate to `TRUE`, and use `IS NULL`/`IS NOT NULL` correctly instead
- Predict how `NULL` behaves in arithmetic expressions, aggregate functions, and `WHERE`/boolean conditions
- Recognize and avoid the most common real-world bugs `NULL` causes, including the `NOT IN` trap

## Prerequisites

- [Boolean and Other Types](04-boolean-and-other-types.md) — that topic introduced `BOOLEAN`'s third state briefly; this topic explains that state, and `NULL` generally, in full depth.

## Motivation

`NULL` is, by a wide margin, the single most misunderstood concept in SQL for people learning it — including many experienced programmers coming from languages where a "null"/"None"/"nil" value behaves like an ordinary, comparable value. In SQL, it does not. Misunderstanding `NULL` produces bugs that are unusually dangerous precisely because they don't crash or throw an error — they silently return the wrong number of rows, the wrong aggregate total, or the wrong filtered result, and the query looks completely reasonable while doing it. Every concept in the rest of this course — filtering, joining, aggregating — interacts with `NULL`, so getting a precise mental model right here pays off for the entire remainder of the course.

## Problem Statement

Suppose you have a small table of employees, some of whom haven't been assigned to a department yet:

```sql
CREATE TABLE staff (
    id SERIAL PRIMARY KEY,
    name TEXT,
    department TEXT
);

INSERT INTO staff (name, department) VALUES
    ('Asha', 'Sales'),
    ('Ben', 'Engineering'),
    ('Chen', NULL);
```

You want everyone *not* in Sales, so you write the obvious query:

```sql
SELECT name FROM staff WHERE department <> 'Sales';
```

```
 name
------
 Ben
(1 row)
```

Chen is missing. Intuitively, "no department listed" should surely count as "not Sales" — but it doesn't appear. This isn't a bug in PostgreSQL; it's the direct, correct consequence of what `NULL` means and how comparisons involving it behave. By the end of this topic you'll be able to explain exactly why Chen was excluded, and how to write the query so it includes him if that's genuinely what you want.

## Concept

### What `NULL` Actually Means

`NULL` represents the **absence of a known value** — "unknown," "not applicable," or "not yet recorded." It is not zero, not an empty string, and not `FALSE`. These are all different, concrete values; `NULL` is the deliberate *absence* of any value at all:

```sql
SELECT NULL = 0 AS null_equals_zero,
       NULL = '' AS null_equals_empty_string,
       NULL IS NULL AS is_actually_null;
```

```
 null_equals_zero | null_equals_empty_string | is_actually_null
-------------------+---------------------------+------------------
                    |                           | t
(1 row)
```

Notice the first two columns are themselves blank — not `f` (false) — because comparing anything to `NULL` with `=` doesn't produce `FALSE`, it produces `NULL` itself (more precisely, the third truth value, `UNKNOWN`, described next). Only `IS NULL` reliably tests for `NULL`, and it does return a real `TRUE`.

### Three-Valued Logic: `TRUE`, `FALSE`, `UNKNOWN`

Ordinary boolean logic has two values. SQL logic has three, because any comparison involving an unknown value can't be truthfully resolved to either `TRUE` or `FALSE` — it must resolve to a third value, **`UNKNOWN`** (which PostgreSQL displays as `NULL` when you `SELECT` a boolean expression directly, since `UNKNOWN` and boolean `NULL` are the same underlying concept).

`AND`, `OR`, and `NOT` are all defined over all three values:

**`AND`**

| AND | TRUE | FALSE | UNKNOWN |
|---|---|---|---|
| **TRUE** | TRUE | FALSE | UNKNOWN |
| **FALSE** | FALSE | FALSE | FALSE |
| **UNKNOWN** | UNKNOWN | FALSE | UNKNOWN |

**`OR`**

| OR | TRUE | FALSE | UNKNOWN |
|---|---|---|---|
| **TRUE** | TRUE | TRUE | TRUE |
| **FALSE** | TRUE | FALSE | UNKNOWN |
| **UNKNOWN** | TRUE | UNKNOWN | UNKNOWN |

**`NOT`**

| NOT | Result |
|---|---|
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | UNKNOWN |

The pattern worth internalizing: `FALSE AND anything` is always `FALSE`, and `TRUE OR anything` is always `TRUE` — these two "short-circuit" straight past an `UNKNOWN`, because the outcome is already determined regardless of what the unknown value turns out to be. But `TRUE AND UNKNOWN`, `FALSE OR UNKNOWN`, and `NOT UNKNOWN` all remain `UNKNOWN` — there's genuinely not enough information to resolve them either way.

### Why `NULL = NULL` Is Not `TRUE`

`NULL` means "an unknown value" — not a specific, identifiable value like `0` or `''`. Asking whether one unknown value equals another unknown value can't honestly be answered `TRUE` or `FALSE` — the only truthful answer is "unknown":

```sql
SELECT NULL = NULL AS comparison_result;
```

```
 comparison_result
--------------------
```

```
(1 row)
```

The result is `UNKNOWN` (displayed blank), not `TRUE`. This is deliberately different from `IS NULL`, which is not a comparison operator at all — it's a special predicate that directly asks "is this value absent?" and always returns a definite `TRUE` or `FALSE`:

```sql
SELECT NULL IS NULL AS is_null_check,
       5 IS NULL AS five_is_null_check;
```

```
 is_null_check | five_is_null_check
----------------+---------------------
 t              | f
(1 row)
```

**Always use `IS NULL` / `IS NOT NULL` to test for `NULL`** — never `= NULL` or `<> NULL`, both of which always evaluate to `UNKNOWN` and therefore never match any row in a `WHERE` clause, no matter what.

### Why the `WHERE` Clause Excluded Chen

A `WHERE` clause keeps a row only when its condition evaluates to exactly `TRUE` — rows where the condition evaluates to `FALSE` *or* `UNKNOWN` are both excluded. This is the precise mechanism behind the Problem Statement's bug:

```sql
SELECT name, department, department <> 'Sales' AS comparison_result
FROM staff;
```

```
 name |  department  | comparison_result
------+---------------+--------------------
 Asha | Sales         | f
 Ben  | Engineering   | t
 Chen |               |
(3 rows)
```

Chen's `department` is `NULL`, so `NULL <> 'Sales'` evaluates to `UNKNOWN` — not `TRUE`, not `FALSE`. Since `WHERE` only keeps rows where the condition is `TRUE`, Chen is excluded, exactly like a row that failed the condition outright. If you actually want "not Sales, including rows with no department recorded at all," you must say so explicitly:

```sql
SELECT name FROM staff WHERE department <> 'Sales' OR department IS NULL;
```

```
 name
------
 Ben
 Chen
(2 rows)
```

### `NULL` in Arithmetic

Any arithmetic expression involving `NULL` produces `NULL` — the reasoning being that if one operand is unknown, the result of any calculation involving it is also unknown:

```sql
SELECT 5 + NULL AS sum_result,
       NULL * 100 AS product_result,
       NULL || 'text' AS concat_result;
```

```
 sum_result | product_result | concat_result
-------------+-----------------+----------------
             |                 |
(1 row)
```

All three are `NULL`, not zero or an error. This matters directly for computed columns: a `total = price * quantity` calculation silently becomes `NULL` for any row where either `price` or `quantity` is unrecorded, rather than raising an error you'd notice immediately.

### `NULL` in Aggregate Functions

Aggregate functions (covered fully in Module 09 — Aggregation) have specific, easy-to-misremember rules around `NULL`:

```sql
CREATE TABLE payroll (
    id SERIAL PRIMARY KEY,
    employee TEXT,
    bonus NUMERIC
);

INSERT INTO payroll (employee, bonus) VALUES
    ('Asha', 500),
    ('Ben', NULL),
    ('Chen', 300);

SELECT
    COUNT(*)      AS count_star,
    COUNT(bonus)  AS count_bonus_column,
    SUM(bonus)    AS total_bonus,
    AVG(bonus)    AS average_bonus
FROM payroll;
```

```
 count_star | count_bonus_column | total_bonus | average_bonus
-------------+---------------------+--------------+----------------
           3 |                   2 |          800 |         400
(1 row)
```

`COUNT(*)` counts rows regardless of `NULL` values (all 3 rows count); `COUNT(bonus)` counts only rows where `bonus` is *not* `NULL` (2 rows); `SUM` and `AVG` silently skip `NULL` values entirely rather than treating them as zero — Ben's missing bonus doesn't drag the average down to `266.67`, it's simply excluded from the calculation, giving `400` (the average of just `500` and `300`). A further, easily-missed detail: `SUM` over a group where *every* value is `NULL` (or over zero rows) returns `NULL`, not `0` — worth checking for explicitly with `COALESCE` (covered fully in Module 08 — Functions & Expressions) if your application logic assumes a numeric total is always present.

### The `NOT IN` Trap

This is one of the most notorious real-world `NULL` bugs. Suppose you want employees not in a list of departments pulled from a subquery:

```sql
CREATE TABLE closed_departments (name TEXT);
INSERT INTO closed_departments VALUES ('Sales'), (NULL);

SELECT name FROM staff
WHERE department NOT IN (SELECT name FROM closed_departments);
```

```
 name
------
(0 rows)
```

Zero rows — even Ben, who is definitely in Engineering, not Sales. `NOT IN (subquery)` is logically equivalent to `<> ALL (subquery)`, meaning "not equal to every single value returned." Since one of those values is `NULL`, every row's comparison against that `NULL` evaluates to `UNKNOWN`, and an `AND` chain (`ALL`) containing even one `UNKNOWN` alongside `TRUE`s can never resolve to a definite `TRUE` — so no row ever qualifies, for every single row in the outer table, silently. This is a genuine, well-known production bug pattern: if there's any chance a `NOT IN` subquery could return a `NULL`, prefer `NOT EXISTS` instead (covered in Module 11 — Subqueries), which does not have this failure mode.

### `IS DISTINCT FROM` — NULL-Safe Equality

PostgreSQL provides `IS DISTINCT FROM` and `IS NOT DISTINCT FROM` as NULL-safe alternatives to `<>` and `=` — they always return a definite `TRUE`/`FALSE`, treating two `NULL`s as equal to each other rather than unknown:

```sql
SELECT NULL IS NOT DISTINCT FROM NULL AS null_safe_equality,
       NULL = NULL AS ordinary_equality;
```

```
 null_safe_equality | ordinary_equality
---------------------+--------------------
 t                   |
(1 row)
```

## Internal Working (Preview)

PostgreSQL does not store a special sentinel *value* to represent `NULL` inside a column's data — instead, each row (each "tuple," in storage terminology) carries a small **null bitmap** in its header, with one bit per column, marking whether that column's value is present or absent for this particular row:

```
 Row header                              Row data
 ┌─────────────────────────────┐   ┌───────────────────────────────┐
 │ null bitmap: [0][1][0][0]    │   │  (column 2's actual bytes are  │
 │  col1  col2  col3  col4       │   │   simply omitted from storage  │
 │  (0=present, 1=null)          │   │   entirely when its bit is 1)  │
 └─────────────────────────────┘   └───────────────────────────────┘
```

When a column's null bit is set, PostgreSQL doesn't store any bytes for that column's value at all in that row — there is nothing there to compare, which is precisely why comparison operators can't produce a real `TRUE`/`FALSE` against it and must fall back to `UNKNOWN`.

## Real-World Analogy

Think of a paper survey form with a "favorite color" question. Leaving it **blank** (`NULL`) is fundamentally different from writing **"none"** (a real, specific text value) or circling **"zero"** if the question were numeric — blank means the respondent's actual preference is simply not known to you, not that they affirmatively have no color preference. Now imagine trying to compare two respondents' blank answers and asking "did they answer the same way?" — you cannot truthfully say yes (you don't know they'd have picked the same color) or no (you don't know they wouldn't have) — the only honest answer is "unknown," which is exactly why `NULL = NULL` isn't `TRUE`.

## Why NULL and Three-Valued Logic Were Designed This Way

Before `NULL` was formalized as part of the relational model, database designers commonly used a real sentinel value to mean "missing" — an empty string, a `-1`, or a magic date like `1900-01-01` — and every one of these hacks was ambiguous (is `-1` a genuine negative quantity or "missing"?) and required every single query author to remember and re-apply the convention correctly. Edgar Codd, whose relational model underpins this entire course (Module 02), argued that missing or inapplicable information needed to be represented as a distinct, unambiguous marker rather than overloading a real, in-range data value — this is exactly what `NULL` is. Once `NULL` is allowed as a legitimate column state, ordinary two-valued (`TRUE`/`FALSE`) logic is no longer sufficient to evaluate every possible comparison, because some comparisons genuinely cannot be resolved without information you don't have — three-valued logic is the minimal, logically consistent extension needed to keep SQL's comparison and filtering semantics honest once `NULL` is in play, rather than forcing an arbitrary (and therefore misleading) `TRUE` or `FALSE` answer onto a genuinely unknown comparison.

## Advantages

- **`NULL` cleanly represents genuinely missing or inapplicable data** without overloading any real, in-range value as an ad-hoc "missing" sentinel — avoiding the ambiguity that plagued pre-relational data storage.
- **Three-valued logic is internally consistent** — every comparison and boolean expression involving `NULL` has one well-defined, principled outcome, even if that outcome is `UNKNOWN` rather than a definite answer.
- **Aggregate functions ignoring `NULL` by default is usually what you actually want** — an average of the *known* values is generally more useful than one artificially dragged toward zero by unrecorded data.

## Disadvantages / Limitations

- **Three-valued logic is genuinely harder to reason about than plain boolean logic**, and it is easy to write a query that looks correct, runs without error, and returns a subtly wrong result — as the `WHERE department <> 'Sales'` and `NOT IN` examples both demonstrate.
- **`NULL`'s behavior is inconsistent with how "null"/"None" behaves in most general-purpose programming languages**, where it commonly compares equal to itself — this mismatch is a very frequent source of confusion for people newer to SQL specifically.
- **The `NOT IN` trap is a real, recurring production bug pattern** — a subquery that can return even a single `NULL` silently poisons the entire `NOT IN` comparison for every outer row, with no error or warning at query time.

## Best Practices

- Always use `IS NULL` / `IS NOT NULL` to test for `NULL` — never `= NULL` or `<> NULL`, both of which always evaluate to `UNKNOWN` and can never match a row.
- When you want a condition like "not X, including missing values" to actually include rows with `NULL`, say so explicitly with `OR column IS NULL`, rather than assuming a `<>`/`!=` comparison covers it.
- Prefer `NOT EXISTS` over `NOT IN` when the subquery's result could plausibly contain a `NULL` (Module 11 covers this trade-off in depth) — `NOT EXISTS` does not have the silent-empty-result failure mode.
- Use `COALESCE(value, default)` (Module 08) wherever a `NULL` should be treated as a specific fallback value for display or calculation purposes, rather than letting it silently propagate through arithmetic.
- Be deliberate about whether a column should allow `NULL` at all — a `NOT NULL` constraint (Module 05) is often the simplest way to eliminate an entire category of NULL-related ambiguity for a column that should always have a value.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `WHERE column = NULL` to find `NULL` rows | This always evaluates to `UNKNOWN`, never `TRUE`, so it matches zero rows regardless of the data — use `WHERE column IS NULL` instead. |
| Assuming `WHERE column <> 'X'` includes rows where `column` is `NULL` | The comparison evaluates to `UNKNOWN` for `NULL` rows, and `WHERE` only keeps rows where the condition is `TRUE` — `NULL` rows are silently excluded unless you add `OR column IS NULL`. |
| Using `NOT IN (subquery)` when the subquery could return a `NULL` | If any value in the subquery's result is `NULL`, the entire `NOT IN` comparison evaluates to `UNKNOWN` for every outer row, silently returning zero rows — use `NOT EXISTS` instead. |
| Assuming `COUNT(column)` and `COUNT(*)` always return the same number | `COUNT(*)` counts every row; `COUNT(column)` counts only rows where that specific column is not `NULL` — they diverge whenever the column has any missing values. |
| Assuming `SUM`/`AVG` treat `NULL` as zero | Both functions simply skip `NULL` values in their calculation; a `SUM` over a group where every value is `NULL` (or over zero rows) returns `NULL`, not `0`. |

## Interview Questions

1. **Q: Why does `NULL = NULL` evaluate to `NULL` (or `UNKNOWN`) instead of `TRUE`?**
   A: `NULL` represents an unknown value, not a specific value like `0` or `''`. Asking whether one unknown value equals another unknown value can't be truthfully answered `TRUE` or `FALSE` — the only logically honest result is `UNKNOWN`, which SQL's three-valued logic represents and which `SELECT` displays as `NULL`. `IS NULL`, by contrast, is a distinct predicate (not a comparison) that always returns a definite `TRUE`/`FALSE`.

2. **Q: What is three-valued logic, and why does SQL need it?**
   A: Three-valued logic extends ordinary boolean logic (`TRUE`/`FALSE`) with a third value, `UNKNOWN`, needed because any comparison involving a `NULL` (unknown) operand can't be honestly resolved to `TRUE` or `FALSE`. `AND`, `OR`, and `NOT` are all defined to handle `UNKNOWN` consistently — for example, `FALSE AND UNKNOWN` is `FALSE` (already determined regardless of the unknown), while `TRUE AND UNKNOWN` remains `UNKNOWN` (genuinely unresolved).

3. **Q: What is the difference between `COUNT(*)` and `COUNT(column_name)`?**
   A: `COUNT(*)` counts every row in the result set, regardless of whether any column is `NULL`. `COUNT(column_name)` counts only the rows where that specific column's value is not `NULL`. They return the same number only when the column being counted has no `NULL` values at all.

4. **Q: What is the `NOT IN` / `NULL` trap, and how do you avoid it?**
   A: `NOT IN (subquery)` is logically equivalent to `<> ALL (subquery)`. If the subquery's result set contains even one `NULL`, every row's comparison against that `NULL` evaluates to `UNKNOWN`, which poisons the entire `ALL` condition so that it can never resolve to `TRUE` for any row — the query silently returns zero rows, with no error. The standard fix is to use `NOT EXISTS` with a correlated subquery instead, which does not exhibit this failure mode regardless of `NULL`s in the compared data.

## Summary

- `NULL` means "unknown" or "absent" — it is not zero, not an empty string, and not `FALSE`; it is the deliberate lack of any value at all.
- SQL uses three-valued logic (`TRUE`, `FALSE`, `UNKNOWN`) because comparisons involving `NULL` cannot be honestly resolved to a definite `TRUE`/`FALSE`.
- `NULL = NULL` evaluates to `UNKNOWN`, not `TRUE` — always use `IS NULL`/`IS NOT NULL` to test for `NULL` explicitly.
- `WHERE` only keeps rows where a condition is exactly `TRUE`; rows evaluating to `FALSE` or `UNKNOWN` are both silently excluded, which is why comparisons against `NULL` frequently drop rows a beginner expects to see.
- Arithmetic involving `NULL` always yields `NULL`; `COUNT(*)` includes `NULL` rows while `COUNT(column)` excludes them; `SUM`/`AVG` skip `NULL` values rather than treating them as zero.
- `NOT IN` against a subquery that can return `NULL` silently returns zero rows for every outer row — prefer `NOT EXISTS` whenever that risk exists.
