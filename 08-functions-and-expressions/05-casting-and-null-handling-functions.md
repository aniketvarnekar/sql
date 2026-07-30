# Casting and NULL-Handling Functions

## Learning Objectives

By the end of this section you should be able to:
- Convert a value from one type to another using `CAST(... AS type)` and PostgreSQL's `::` shorthand
- Explain the difference between implicit and explicit type conversion, and when PostgreSQL allows each
- Use `COALESCE` to supply a default value in place of `NULL`
- Use `NULLIF` to turn a specific value into `NULL`
- Combine `COALESCE` and `NULLIF` to solve two extremely common real-world problems: avoiding division-by-zero errors and providing sensible defaults for missing data

## Prerequisites

- [CASE Expressions](04-case-expressions.md) — `COALESCE` and `NULLIF` are, conceptually, shorthand for patterns you could otherwise write with `CASE`; seeing `CASE` first makes that connection clear.
- Module 3 — Data Types, specifically its coverage of `NULL` as "absence of a value" and the rule that most operations involving `NULL` themselves produce `NULL` — this entire topic exists to give you tools for working around that propagation when it isn't what you want.

## Motivation

Two extremely common, extremely different problems show up in almost every real query: "this column's *type* isn't quite what I need for this operation" and "this column is sometimes empty, and I need to plan for that." Casting solves the first problem; `COALESCE` and `NULLIF` solve the second. Nearly every non-trivial report you'll ever write in SQL uses at least one of these three tools — they are not niche functions, they are load-bearing infrastructure for correct, crash-free queries.

## Problem Statement

Consider an `inventory` table:

```sql
CREATE TABLE inventory (
    id SERIAL PRIMARY KEY,
    product_name TEXT,
    sku TEXT,
    units_sold INTEGER,
    units_returned INTEGER,
    discount_pct NUMERIC
);

INSERT INTO inventory (product_name, sku, units_sold, units_returned, discount_pct) VALUES
    ('Wireless Mouse',      '00120', 120,    4, 0.10),
    ('Mechanical Keyboard', '00089',   0,    0, NULL),
    ('USB-C Hub',           '00045',  45, NULL, 0.00);
```

Two immediate problems: `sku` is stored as `TEXT` (to preserve leading zeros), but a report needs to compute "next SKU number" as an integer — a straight arithmetic operation on a `TEXT` column will fail outright. Separately, you're asked to compute a **return rate** (`units_returned / units_sold`) for every product, and to show `0` instead of a blank whenever `discount_pct` is missing. The Mechanical Keyboard sold **zero** units — dividing by that will crash the whole query with a division-by-zero error unless you specifically guard against it. Both problems — a type mismatch, and unplanned-for missing data — are exactly what this topic's tools exist to solve.

## Concept

### `CAST(... AS type)` and the `::` Shorthand

`CAST(expression AS type)` converts a value from one type to another. PostgreSQL also supports a shorter, non-standard `expression::type` syntax that does exactly the same thing.

```sql
SELECT CAST(sku AS INTEGER)      FROM inventory WHERE product_name = 'Wireless Mouse';  -- 120
SELECT sku::INTEGER              FROM inventory WHERE product_name = 'Wireless Mouse';  -- 120 (identical result)
SELECT CAST('2026-07-31' AS DATE);                                                       -- 2026-07-31
SELECT (19.999)::NUMERIC(10,2);                                                          -- 20.00
```

Both forms are fully equivalent in PostgreSQL — `CAST(... AS ...)` is the SQL-standard spelling (portable to other databases, Module 22), while `::` is a PostgreSQL-specific shorthand that's extremely common in PostgreSQL code you'll encounter, but won't run unmodified on most other database products.

A practical use — computing the next SKU number, which requires treating the stored text as an integer first:

```sql
SELECT product_name, sku, sku::INTEGER + 1 AS next_sku_number
FROM inventory;
```

```
      product_name      |  sku  | next_sku_number
--------------------------+--------+------------------
 Wireless Mouse           | 00120  |              121
 Mechanical Keyboard      | 00089  |               90
 USB-C Hub                | 00045  |               46
(3 rows)
```

### Implicit vs. Explicit Casting

PostgreSQL sometimes converts a value's type automatically, without you writing `CAST` or `::` anywhere — this is **implicit casting**. It happens specifically when a value's type is not yet fixed and context makes the intended type unambiguous:

```sql
SELECT '5' + 3;
```

```
 ?column?
----------
        8
(1 row)
```

Here, `'5'` is a string *literal* with no fixed type until it's used in an expression — because it appears next to an integer in an addition, PostgreSQL resolves it as an integer automatically, with no explicit cast written. This is easy to mistake for "SQL freely converts text to numbers whenever needed" — it does not. The moment the value comes from an actual `TEXT`-typed **column** (not an untyped literal), that automatic resolution no longer applies:

```sql
SELECT sku + 1 FROM inventory WHERE product_name = 'Wireless Mouse';
```

```
ERROR:  operator does not exist: text + integer
```

`sku` is a genuinely, already-typed `TEXT` column — PostgreSQL will not silently guess that you meant to treat its contents as a number, because a `TEXT` column could validly contain something that isn't numeric at all. This is exactly why the earlier example explicitly wrote `sku::INTEGER + 1` — the explicit cast is required once a value already has a concrete, non-numeric type.

### `COALESCE()` — First Non-`NULL` Value

`COALESCE(value1, value2, ..., valueN)` evaluates its arguments left to right and returns the **first one that isn't `NULL`** — if every argument is `NULL`, the result is `NULL`.

```sql
SELECT product_name, discount_pct, COALESCE(discount_pct, 0) AS effective_discount
FROM inventory;
```

```
      product_name      | discount_pct | effective_discount
--------------------------+---------------+----------------------
 Wireless Mouse           |         0.10 |                 0.10
 Mechanical Keyboard      |               |                 0.00
 USB-C Hub                |         0.00 |                 0.00
(3 rows)
```

The Mechanical Keyboard's genuinely missing `discount_pct` becomes a usable `0` in the report, while USB-C Hub's *already-zero* `discount_pct` (a real, known value — "explicitly no discount") is left untouched, exactly as it should be — `COALESCE` only steps in for actual `NULL`s, never for a real value that merely happens to equal zero. `COALESCE` accepts more than two arguments and can express a full fallback chain, e.g. `COALESCE(preferred_price, list_price, 0)` — try the first, then the second, then finally a hard default.

### `NULLIF()` — Turning a Specific Value Into `NULL`

`NULLIF(value1, value2)` returns `NULL` if `value1` equals `value2`, and otherwise returns `value1` unchanged. It is, in effect, the inverse operation of `COALESCE`: instead of replacing `NULL` with something else, it replaces something specific with `NULL`.

```sql
SELECT NULLIF(0, 0);     -- NULL
SELECT NULLIF(5, 0);     -- 5
SELECT NULLIF('N/A', 'N/A');  -- NULL
```

### Combining Both — Avoiding Division-by-Zero

Dividing by an actual zero raises a hard error in PostgreSQL — it does not silently return `Infinity` or `NULL` the way some other systems or spreadsheet software might. `NULLIF` is the standard way to prevent this: convert the *specific dangerous value* (`0`) into `NULL` *before* the division happens, so the division sees a `NULL` divisor instead of a literal zero — and dividing by `NULL` simply produces `NULL`, not an error.

```sql
SELECT product_name,
       units_sold,
       units_returned,
       ROUND(COALESCE(units_returned, 0)::NUMERIC / NULLIF(units_sold, 0), 4) AS return_rate
FROM inventory;
```

```
      product_name      | units_sold | units_returned | return_rate
--------------------------+------------+-----------------+--------------
 Wireless Mouse           |        120 |              4 |      0.0333
 Mechanical Keyboard      |          0 |              0 |
 USB-C Hub                |         45 |                |      0.0000
(3 rows)
```

Walking through each row:
- **Wireless Mouse**: `units_sold` is a real, non-zero `120`, so `NULLIF(120, 0)` just returns `120` unchanged; the division proceeds normally, giving `4 / 120 ≈ 0.0333`.
- **Mechanical Keyboard**: `units_sold` is genuinely `0` — without `NULLIF`, this row would raise `ERROR: division by zero` and abort the *entire query*, not just this row. `NULLIF(0, 0)` converts it to `NULL`, so the division becomes `0 / NULL`, which quietly evaluates to `NULL` — a sensible "not applicable, no units were sold" result rather than a crash.
- **USB-C Hub**: `units_returned` is `NULL` (unknown, not necessarily zero); `COALESCE(NULL, 0)` supplies `0` so the numerator is well-defined, and the division proceeds normally to `0.0000`.

This single line demonstrates the two functions' complementary roles perfectly: `COALESCE` fills in for *missing* data (a `NULL` numerator that should be treated as zero), while `NULLIF` guards against a *dangerous real value* (an actual zero denominator) by turning it into `NULL` before it can cause a crash.

## Internal Working (Preview)

Both `COALESCE` and `NULLIF` are, internally, shorthand the SQL standard defines in terms of `CASE`:

```sql
-- COALESCE(a, b, c) is equivalent to:
CASE
    WHEN a IS NOT NULL THEN a
    WHEN b IS NOT NULL THEN b
    ELSE c
END

-- NULLIF(a, b) is equivalent to:
CASE WHEN a = b THEN NULL ELSE a END
```

The database engine evaluates arguments to `COALESCE` **left to right and stops as soon as it finds a non-`NULL` value** — later arguments are not evaluated at all once a non-`NULL` result is found, which matters if a later argument would be expensive to compute or would itself error. Casting (`CAST`/`::`) is resolved at parse time whenever possible: the engine determines the target type and either performs a straightforward binary conversion (e.g., extracting an integer from a `TEXT` string's digit characters) or rejects the statement immediately if no valid conversion path exists for that type pair.

## Real-World Analogy

`COALESCE` works like a form with a fallback chain of contacts: "Call the customer's mobile number; if that's not on file, call their home number; if that's not on file either, call the office main line" — you take the first one that's actually available, skipping past ones that are missing. `NULLIF` works like a data-entry policy that says "if someone enters the placeholder value 'N/A' or literally zero, treat it as if nothing was entered at all" — converting a specific, known "junk" value into a clean "unknown," so later steps that specifically check for "unknown" (like a division guard) behave correctly instead of being fooled by an entered placeholder that looks like real data.

## Why These Functions Were Designed This Way

`NULL` exists to represent "value unknown or not applicable" — a state deliberately distinct from any real value, including zero (Module 3 covers this distinction in depth). Because so many operations propagate `NULL` (an expression touching `NULL` usually becomes `NULL` itself), SQL needs a standard, declarative way to say "substitute a default here" (`COALESCE`) and "treat this specific value as if it were unknown" (`NULLIF`) without writing a full `CASE` block every single time — both are common enough patterns that the SQL standard defines them as their own concise functions, even though (as shown above) they're formally just `CASE` in disguise. This keeps everyday `NULL`-handling readable and idiomatic rather than verbose.

## Advantages

- **Prevents entire classes of runtime errors.** Division-by-zero, in particular, is a hard error in PostgreSQL — `NULLIF` is the standard, idiomatic guard against it.
- **Concise compared to the equivalent `CASE` expression.** Both functions express a common pattern in a fraction of the code a hand-written `CASE` block would need.
- **Composable.** `COALESCE` and `NULLIF` nest naturally inside larger expressions (as shown in the combined return-rate example), and inside each other, without any special syntax.

## Disadvantages / Limitations

- **`COALESCE`'s arguments must be comparable/compatible types**, or PostgreSQL will raise a type error — you can't casually `COALESCE` a number and a piece of unrelated text together without an explicit cast reconciling them.
- **`NULLIF` only guards against one specific, known "dangerous" value at a time.** If a divisor could be dangerous for multiple different values (not just zero), `NULLIF` alone isn't sufficient and a full `CASE` expression is clearer.
- **Casting can silently lose information.** Casting `NUMERIC(10,2)` to `NUMERIC(10,0)`, or a long string to a shorter `VARCHAR(n)`, truncates or rounds — PostgreSQL doesn't warn you mid-query that precision was lost unless the truncation is severe enough to raise an error outright.

## Best Practices

- Default to `NULLIF(divisor, 0)` any time a divisor comes from a column or computed expression that *could* plausibly be zero — treat it as a required safety habit, not an optional nicety.
- Use `COALESCE` at the exact point a default is semantically meaningful — don't blanket every column with `COALESCE(column, 0)` "just in case," since that can quietly mask a genuinely missing value that a report should have flagged instead.
- Prefer `CAST(... AS ...)` over `::` in any SQL you expect might need to run on a non-PostgreSQL database someday (Module 22); use `::` freely in PostgreSQL-only code, where it's idiomatic and far more common in practice.
- Never rely on implicit casting of literals as a substitute for understanding your column types — it works for untyped literals like `'5'`, but not for genuinely `TEXT`-typed columns, and leaning on it can create confusion about what's actually happening.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Dividing by a column without guarding against zero, assuming SQL returns `Infinity` or `NULL` automatically | PostgreSQL raises a hard `ERROR: division by zero` for integer/numeric division by a literal zero, aborting the entire query — you must guard with `NULLIF(divisor, 0)` explicitly. |
| Using `COALESCE(column, 0)` on every nullable numeric column reflexively | This can silently convert a genuinely missing, should-be-investigated value into a falsely "normal-looking" zero, hiding a data quality problem instead of surfacing it. |
| Assuming `'5' + 3` proves that text columns can always be used in arithmetic | That specific case works only because `'5'` is an untyped literal resolved by context; an actual `TEXT`-typed column with the same contents still requires an explicit cast. |
| Forgetting that `NULLIF(a, b)` compares for equality, not "a is null" | `NULLIF` doesn't check whether `a` is `NULL` — it checks whether `a` *equals* `b`, then produces `NULL` if so. Confusing this with a `NULL`-check is a common misreading of what the function does. |

## Interview Questions

1. **Q: How would you safely compute `numerator / denominator` in SQL when `denominator` might be zero?**
   A: Wrap the denominator in `NULLIF(denominator, 0)`. If `denominator` is `0`, `NULLIF` converts it to `NULL`, and dividing by `NULL` produces `NULL` instead of raising a division-by-zero error — the row's result becomes "not applicable" rather than crashing the whole query.

2. **Q: What's the difference between `CAST(x AS type)` and `x::type` in PostgreSQL?**
   A: They are functionally identical — both convert `x` to the given type. `CAST(... AS ...)` is the SQL-standard syntax, portable across database vendors; `::` is a PostgreSQL-specific shorthand for the same operation and won't run unmodified on most other databases.

3. **Q: Why does `SELECT '5' + 3;` work, but `SELECT text_column + 3;` fail with an error, if `text_column` contains `'5'`?**
   A: `'5'` as a bare literal has no fixed type until context resolves it — next to an integer in addition, PostgreSQL implicitly interprets it as an integer. A `TEXT`-typed column, however, already has a concrete, fixed type that could legitimately hold non-numeric text, so PostgreSQL requires an explicit cast (`text_column::INTEGER`) rather than guessing.

4. **Q: What does `COALESCE(a, b, c)` return if `a` is `NULL` and `b` is also `NULL`?**
   A: It returns `c` (or `NULL` if `c` is also `NULL`) — `COALESCE` evaluates its arguments left to right and returns the first one that is not `NULL`, falling through the entire list if necessary.

## Summary

- `CAST(expr AS type)` and `expr::type` are equivalent ways to explicitly convert a value's type; `CAST` is standard SQL, `::` is PostgreSQL-specific shorthand.
- Implicit casting only happens automatically for untyped literals resolved by context (like `'5' + 3`) — genuinely typed columns require an explicit cast for cross-type operations.
- `COALESCE(a, b, c, ...)` returns the first non-`NULL` argument, evaluated left to right — the standard way to supply a default for missing data.
- `NULLIF(a, b)` returns `NULL` if `a` equals `b`, otherwise returns `a` unchanged — the standard way to turn a specific dangerous or placeholder value into `NULL`.
- The combined pattern `COALESCE(numerator, 0) / NULLIF(denominator, 0)` is one of the most common idioms in real-world SQL: it fills in missing numerators and prevents division-by-zero errors in one expression.
- Both `COALESCE` and `NULLIF` are formally shorthand for specific `CASE` patterns — understanding `CASE` first makes both of them easier to reason about precisely.
