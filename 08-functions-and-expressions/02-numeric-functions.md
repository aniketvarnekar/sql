# Numeric Functions

## Learning Objectives

By the end of this section you should be able to:
- Round, ceiling, and floor numeric values, and explain the difference between the three
- Compute absolute values, remainders, and powers using `ABS`, `MOD`/`%`, and `POWER`
- Predict the exact result *type* of an arithmetic expression, including the difference between integer division (which truncates) and numeric division (which doesn't)
- Explain, briefly, what `random()` returns and why it's non-deterministic

## Prerequisites

- [String Functions](01-string-functions.md) — not a strict dependency, but this topic assumes the same comfort with calling a function inside a `SELECT` list established there.
- Module 3 — Data Types, specifically the distinction between `INTEGER`/`BIGINT` (whole numbers) and `NUMERIC`/`DECIMAL` (exact fractional numbers) — this topic's central lesson (division result types) is meaningless without that distinction already in place.

## Motivation

Numbers stored in a table are rarely the numbers you want to *show* someone. A stored price needs tax applied and rounding to two decimal places for a receipt. A stored quantity needs dividing into full shipping boxes, with leftovers handled separately. And one of SQL's most common silent bugs — a report showing `0` where it should show `0.5` — comes directly from not knowing which arithmetic operations truncate and which don't. Numeric functions and a solid grasp of arithmetic result types are what let you compute correct, presentable numbers without ever second-guessing what the database just handed you.

## Problem Statement

Suppose you're given a small product catalog:

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    price NUMERIC(10,2),
    quantity_in_stock INTEGER
);

INSERT INTO products (name, price, quantity_in_stock) VALUES
    ('Wireless Mouse',       19.99, 143),
    ('Mechanical Keyboard',  89.50,  37),
    ('USB-C Hub',            24.99,   0);
```

You need three things: a tax-inclusive price rounded to the nearest cent, the number of shipping boxes needed if each box holds 12 units (with any leftover accounted for separately), and confirmation of exactly how many units are left over. Get the arithmetic or rounding wrong here and a customer sees an incorrect charge, or a warehouse under-orders boxes — small numeric mistakes with very real consequences.

## Concept

### `ROUND()` — Rounding to a Given Precision

```sql
SELECT ROUND(21.638675, 2);   -- 21.64
SELECT ROUND(96.88375, 2);    -- 96.88
SELECT ROUND(27.050175, 2);   -- 27.05
SELECT ROUND(4.5);            -- 5   (no second argument = round to nearest whole number)
```

`ROUND(value, decimal_places)` rounds `value` to the given number of decimal places. If `decimal_places` is omitted, it rounds to the nearest integer. Applied to the catalog with an 8.25% tax rate:

```sql
SELECT name, price, ROUND(price * 1.0825, 2) AS price_with_tax
FROM products;
```

```
        name         | price | price_with_tax
----------------------+-------+----------------
 Wireless Mouse       | 19.99 |          21.64
 Mechanical Keyboard  | 89.50 |          96.88
 USB-C Hub            | 24.99 |          27.05
(3 rows)
```

### `CEIL()` / `FLOOR()` — Rounding Toward a Direction

Unlike `ROUND()`, which rounds to the *nearest* value, `CEIL()` (also spelled `CEILING()`) always rounds **up** and `FLOOR()` always rounds **down**, regardless of how close the fractional part is to the next whole number.

```sql
SELECT CEIL(11.01);   -- 12
SELECT FLOOR(11.99);  -- 11
```

A realistic use: how many boxes of 12 are needed to ship the entire stock of Wireless Mice, given a partial box still counts as a full box?

```sql
SELECT name, quantity_in_stock,
       quantity_in_stock / 12.0            AS raw_boxes,
       CEIL(quantity_in_stock / 12.0)      AS boxes_needed,
       FLOOR(quantity_in_stock / 12.0)     AS full_boxes
FROM products
WHERE name = 'Wireless Mouse';
```

```
      name      | quantity_in_stock | raw_boxes | boxes_needed | full_boxes
-----------------+-------------------+-----------+---------------+------------
 Wireless Mouse  |               143 | 11.9166... |            12 |         11
(1 row)
```

`CEIL` correctly reports that 12 boxes are required (11 full boxes plus a partial 12th), while `FLOOR` reports only the 11 *completely* full boxes — two genuinely different, both useful, answers depending on what you're asking.

### `ABS()` — Absolute Value

```sql
SELECT ABS(-42.5);   -- 42.5
SELECT ABS(42.5);    -- 42.5
```

A common use is measuring "distance from target" without caring about direction — e.g., how far each product's price is from a $25 target price, regardless of whether it's over or under:

```sql
SELECT name, price, ABS(price - 25) AS distance_from_target
FROM products;
```

```
        name         | price | distance_from_target
----------------------+-------+-----------------------
 Wireless Mouse       | 19.99 |                  5.01
 Mechanical Keyboard  | 89.50 |                 64.50
 USB-C Hub            | 24.99 |                  0.01
(3 rows)
```

### `MOD()` / `%` — Remainder

`MOD(dividend, divisor)` and the `%` operator are equivalent — both return the remainder left over after integer division.

```sql
SELECT MOD(143, 12);   -- 11
SELECT 143 % 12;       -- 11
```

Combined with the boxing example above: 143 units, boxes of 12, gives 11 full boxes (`FLOOR(143/12.0)`) with 11 units left over (`143 % 12`) — those 11 leftover units are exactly what makes the 12th box a partial one.

```sql
SELECT name, quantity_in_stock,
       quantity_in_stock / 12          AS full_boxes,
       quantity_in_stock % 12          AS leftover_units
FROM products
WHERE name = 'Wireless Mouse';
```

```
      name      | quantity_in_stock | full_boxes | leftover_units
-----------------+-------------------+------------+-----------------
 Wireless Mouse  |               143 |         11 |              11
(1 row)
```

(Note `quantity_in_stock / 12` here — both integers — already truncates to `11` without needing `FLOOR()` at all. More on exactly why in the next section.)

### `POWER()` — Exponentiation

```sql
SELECT POWER(2, 10);     -- 1024
SELECT POWER(1.0825, 3); -- 1.268392... (compounding 8.25% growth over 3 periods)
```

`POWER(base, exponent)` raises `base` to `exponent`. A practical use is compound growth calculations — e.g., projecting a price after three consecutive years of an 8.25% increase:

```sql
SELECT name, price, ROUND(price * POWER(1.0825, 3), 2) AS price_in_3_years
FROM products
WHERE name = 'Mechanical Keyboard';
```

```
         name         | price | price_in_3_years
-----------------------+-------+-------------------
 Mechanical Keyboard   | 89.50 |           113.55
(1 row)
```

### Arithmetic Operators and Their Result Types — The Critical Part

This is the single most important thing to internalize in this topic. **The result type of a division depends on the types of both operands, not on the "true" mathematical answer.**

```sql
SELECT 7 / 2;     -- 3
SELECT 7 / 2.0;   -- 3.5000000000000000
SELECT 7.0 / 2;   -- 3.5000000000000000
```

When **both operands are integers**, PostgreSQL performs **integer division**, which **truncates toward zero** — it discards the fractional part entirely, it does not round. `7 / 2` is `3`, not `3.5` and not `4`. This is a direct consequence of the result needing to fit back into an integer type, which has no way to represent a fractional part at all.

The moment **either operand is a `NUMERIC`** (as `2.0` and `7.0` are — a literal written with a decimal point is `NUMERIC`, not `INTEGER`), PostgreSQL promotes the whole expression to numeric division, which preserves the fractional part. The unusually long string of decimal digits (`3.5000000000000000`) is not a mistake — PostgreSQL's numeric division computes a generous, fixed number of decimal places by default so that repeating or high-precision results (e.g., `10 / 3.0` → `3.3333333333333333`) don't lose accuracy; trailing zeros simply pad an exact result out to that same fixed width.

```sql
SELECT 10 / 3;     -- 3                     (integer division, truncated)
SELECT 10 / 3.0;   -- 3.3333333333333333    (numeric division, precise)
```

If you have two integer columns but need a precise fractional result, cast at least one operand explicitly (Topic 5 of this module covers casting in full):

```sql
SELECT quantity_in_stock / 12::NUMERIC AS precise_boxes FROM products;
-- or equivalently:
SELECT quantity_in_stock::NUMERIC / 12 AS precise_boxes FROM products;
```

This single truncation behavior is responsible for an enormous number of real-world bugs — "average" calculations silently truncating to `0`, percentage calculations coming out as `0%` instead of a fraction — anytime a developer divides two integer columns expecting a fractional answer.

### `random()` — Briefly

`random()` returns a pseudo-random `DOUBLE PRECISION` value greater than or equal to `0` and less than `1`, and returns a **different** value every time it's called — its output can't be shown as a fixed, reproducible example the way the deterministic functions above can.

```sql
SELECT random();
-- e.g. 0.7052341982374621  (a different value on every execution)
```

To get a random integer in a specific range, combine it with `FLOOR()`: `FLOOR(random() * 100)` gives a random whole number from `0` to `99`. Random-value generation is used sparingly in most reporting SQL and is mentioned here mainly so its presence in a query elsewhere in this course doesn't come as a surprise — it is not a topic with much further depth on its own.

## Internal Working (Preview)

Numeric functions and arithmetic operators are resolved by PostgreSQL's operator/function overload resolution during query parsing: the engine looks at the declared types of both operands and picks the specific internal implementation (integer arithmetic vs. numeric arithmetic vs. floating-point arithmetic) that matches. This is why `7 / 2` and `7 / 2.0` don't just look different — they invoke genuinely different internal code paths, one operating on machine integers directly (fast, but capped at whole-number results) and one operating on PostgreSQL's arbitrary-precision `NUMERIC` representation (slower per-operation, but exact and fractional).

```
 Parser sees:  7 / 2         →  both INTEGER  →  integer division path   → 3
 Parser sees:  7 / 2.0       →  INTEGER, NUMERIC → operand promoted → numeric division path → 3.5000000000000000
```

## Real-World Analogy

Think of integer division like splitting a stack of whole cookies among people, with no cookie ever cut into pieces: 7 cookies among 2 people means each person gets 3, and the leftover cookie is simply set aside, unrepresented, unless someone explicitly asks "how many are left over?" (`MOD`/`%`). Numeric division is like weighing out an amount of flour by mass instead of counting whole units: 7 units among 2 gives exactly 3.5 units each, because the underlying quantity (a `NUMERIC`) doesn't have to come in whole increments to begin with.

## Why These Functions and Rules Were Designed This Way

Integer types exist specifically to represent whole numbers exactly and efficiently — a database column typed `INTEGER` is a promise that every value in it *is* a whole number. Integer division truncating rather than silently switching to a fractional result respects that promise: the result of an integer-typed expression stays an integer, full stop. If integer division instead auto-promoted to a fractional result, the type of an expression would depend on the *values themselves* (whether the division happens to come out even), which would make types unpredictable from the query text alone — a violation of SQL's principle that an expression's type is knowable from its declared operand types, independent of the actual data. `NUMERIC`'s arbitrary, generous decimal scale on division reflects the opposite design goal: `NUMERIC` exists specifically for cases (money, precise measurements) where losing precision silently is unacceptable, so its division defaults toward *more* precision rather than less.

## Advantages

- **Predictable types.** Knowing the operand types tells you the result type before running anything — no surprises about whether a query will hand back a whole number or a fraction.
- **Exactness for `NUMERIC`.** Unlike floating-point (`REAL`/`DOUBLE PRECISION`) arithmetic, `NUMERIC` arithmetic (including its functions like `ROUND`) is exact for base-10 decimal values, which is why `NUMERIC` is the standard choice for money.
- **A small, composable toolkit.** `ROUND`, `CEIL`, `FLOOR`, `ABS`, `MOD`, and `POWER` cover the overwhelming majority of everyday numeric transformations without needing anything more specialized.

## Disadvantages / Limitations

- **Integer division truncation is an easy trap.** It's silent — no error, no warning — which is precisely why it causes so many real bugs; a query can run "successfully" and simply be wrong.
- **`NUMERIC` arithmetic is slower than integer or floating-point arithmetic** at large scale, because it isn't backed by a single native CPU instruction the way fixed-width integer/float math is — usually a non-issue, but worth knowing if you're doing heavy numeric computation over huge row counts.
- **`random()` breaks reproducibility.** A query using `random()` cannot be re-run to produce identical results, which matters for anything needing to be auditable or debuggable step by step.

## Best Practices

- Whenever a division's result needs to be fractional, deliberately cast at least one operand to `NUMERIC` rather than relying on the values happening to divide evenly.
- Use `ROUND(value, 2)` (or the appropriate number of places) before displaying any monetary value — don't display raw, unrounded arithmetic results to end users.
- Reach for `CEIL`/`FLOOR` specifically when you need a *directional* rounding guarantee (e.g., "always round up for how many containers are needed"), not `ROUND`, which can round either direction depending on the fractional value.
- Never use `random()` where reproducibility or auditability matters.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Dividing two `INTEGER` columns and expecting a fractional result | Integer division truncates toward zero; `7 / 2` is `3`, not `3.5` — you must cast at least one operand to `NUMERIC` to get a fractional answer. |
| Using `ROUND()` when a directional guarantee is actually needed | `ROUND(11.01)` gives `11`, which under-counts if the real need was "how many containers, given any partial container still counts as a full one" — that calls for `CEIL`, not `ROUND`. |
| Assuming `MOD`/`%` only works conceptually for "leftover items," not realizing it's the same operation used for things like even/odd checks (`n % 2 = 0`) | `MOD`/`%` is general-purpose remainder computation — it applies anywhere a remainder concept is useful, not just physical leftover counting. |
| Expecting `random()` to return the same value across two calls in the same query | Each call to `random()` independently generates a new value — two separate `random()` calls in one row will almost always differ from each other. |

## Interview Questions

1. **Q: What does `SELECT 7 / 2;` return in PostgreSQL, and why?**
   A: `3`. Both operands are integers, so PostgreSQL performs integer division, which truncates the result toward zero rather than returning a fractional value — the result type must remain an integer.

2. **Q: How would you force a division between two integer columns to return a precise fractional result?**
   A: Cast at least one operand to `NUMERIC` (or another fractional type), e.g. `column_a::NUMERIC / column_b`, which promotes the whole expression to numeric division.

3. **Q: What's the difference between `ROUND()`, `CEIL()`, and `FLOOR()`?**
   A: `ROUND()` rounds to the nearest value at a given precision (up or down, whichever is closer). `CEIL()` always rounds up, regardless of how small the fractional part is. `FLOOR()` always rounds down, regardless of how large the fractional part is.

4. **Q: Why is `NUMERIC` generally preferred over floating-point types for monetary calculations?**
   A: `NUMERIC` represents decimal values exactly, since it's stored as an exact base-10 representation, whereas floating-point types (`REAL`/`DOUBLE PRECISION`) use a binary approximation that can introduce small rounding errors — unacceptable for money, where exactness matters.

## Summary

- `ROUND` rounds to the nearest value at a given precision; `CEIL` always rounds up; `FLOOR` always rounds down — they are not interchangeable.
- `ABS` strips sign; `MOD`/`%` returns a remainder; `POWER` raises to an exponent.
- The most important rule in this topic: division between two `INTEGER` operands truncates toward zero, while division involving a `NUMERIC` operand preserves the fractional result — the operand *types*, not the mathematically "true" answer, determine what you get back.
- Cast explicitly to `NUMERIC` whenever you need a fractional division result from integer-typed columns.
- `random()` is non-deterministic by design and unsuitable anywhere reproducibility matters.
