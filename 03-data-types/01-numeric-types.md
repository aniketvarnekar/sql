# Numeric Types

## Learning Objectives

By the end of this section you should be able to:
- Distinguish exact numeric types (`SMALLINT`, `INTEGER`, `BIGINT`, `NUMERIC`/`DECIMAL`) from approximate, floating-point types (`REAL`, `DOUBLE PRECISION`)
- Explain, with a concrete example, why floating-point types must never be used for money or other values requiring exact decimal representation
- Use `SERIAL`, `BIGSERIAL`, and `GENERATED ... AS IDENTITY` to create auto-incrementing columns, and explain how `IDENTITY` differs from `SERIAL`
- Predict what happens when a value exceeds a numeric type's range or a `NUMERIC` column's declared precision
- Choose an appropriate numeric type for a given column, given its purpose and expected value range

## Prerequisites

- [Your First Query](../01-introduction/05-your-first-query.md) — that topic used `NUMERIC` and `SERIAL` in a `CREATE TABLE` statement without explaining *why* those types were chosen over alternatives; this topic completes that explanation.

## Motivation

Nearly every table you will ever design has at least one numeric column — an ID, a quantity, a price, a score, a count. Pick the wrong numeric type and one of two things eventually happens: either you waste storage and silently limit your system's growth (using a type too small for a value that later needs to be huge), or — far worse — you introduce a rounding error that slowly corrupts financial data in a way that is nearly invisible until an audit or a customer complaint surfaces it. Numeric type selection looks like a minor detail, but it is one of the few data type decisions in this module with genuinely high real-world stakes.

## Problem Statement

Suppose you're tracking a running account balance and you declare the column using a floating-point type:

```sql
CREATE TABLE bad_ledger (
    id SERIAL PRIMARY KEY,
    balance DOUBLE PRECISION
);

INSERT INTO bad_ledger (balance) VALUES (0.1);
UPDATE bad_ledger SET balance = balance + 0.2 WHERE id = 1;

SELECT balance, balance = 0.3 AS is_exactly_point_three FROM bad_ledger;
```

```
      balance       | is_exactly_point_three
---------------------+------------------------
 0.30000000000000004 | f
(1 row)
```

The balance is not exactly `0.3` — it's off by a sliver, and the equality check fails. This isn't a PostgreSQL bug; it's an unavoidable consequence of how `DOUBLE PRECISION` (an IEEE 754 binary floating-point type) represents numbers. Most decimal fractions, including simple ones like `0.1` and `0.2`, have no exact representation in binary — the same way `1/3` has no exact representation in base-10 decimal. In a loop of thousands of transactions, these tiny errors compound. For a line-of-business report this might be a rounding nuisance; for an account balance, an invoice total, or a tax calculation, it is a genuine correctness bug. This topic exists to make sure you never make this mistake, and to give you a full map of PostgreSQL's numeric types so you can pick correctly every time.

## Concept

### The Exact Integer Family

These types store whole numbers with no fractional part, exactly, as fixed-width binary integers:

| Type | Storage | Range |
|---|---|---|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 |
| `INTEGER` (`INT`) | 4 bytes | -2,147,483,648 to 2,147,483,647 |
| `BIGINT` | 8 bytes | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |

```sql
CREATE TABLE stock_levels (
    product_id INTEGER,
    quantity_on_hand SMALLINT
);

INSERT INTO stock_levels VALUES (1, 500);      -- accepted, fits in smallint
INSERT INTO stock_levels VALUES (2, 40000);    -- rejected: exceeds smallint range
```

```
ERROR:  smallint out of range
```

Nothing is truncated or silently clamped — an out-of-range value is always a hard error. This is important: PostgreSQL never quietly wraps an integer around or drops high-order bits the way some lower-level languages do; it refuses the insert outright, which is exactly the kind of loud, early failure you want when the alternative is silently wrong data.

### The Exact Decimal Type: `NUMERIC` / `DECIMAL`

`NUMERIC` and `DECIMAL` are exact synonyms in PostgreSQL — the SQL standard defines both names, and PostgreSQL treats them identically. `NUMERIC` stores numbers with a user-specified **precision** (total number of significant digits) and **scale** (digits after the decimal point), and — critically — represents the value *exactly*, in base 10, with no binary rounding error:

```sql
CREATE TABLE ledger (
    id SERIAL PRIMARY KEY,
    balance NUMERIC(12, 2)  -- up to 12 total digits, 2 after the decimal point
);

INSERT INTO ledger (balance) VALUES (0.1);
UPDATE ledger SET balance = balance + 0.2 WHERE id = 1;

SELECT balance, balance = 0.3 AS is_exactly_point_three FROM ledger;
```

```
 balance | is_exactly_point_three
---------+------------------------
    0.30 | t
(1 row)
```

With `NUMERIC`, `0.1 + 0.2` is *exactly* `0.3` — no rounding drift, because the value is stored as an exact base-10 representation rather than a binary approximation. `NUMERIC` without any precision/scale (just `NUMERIC` or `NUMERIC` with no arguments) stores values of any precision up to PostgreSQL's implementation limit, useful when you genuinely don't want to constrain the number of digits — but in practice, declaring an explicit precision and scale (like `NUMERIC(12, 2)` for currency with cents) is almost always better, because it documents the expected shape of the data and catches mistakes early:

```sql
INSERT INTO ledger (balance) VALUES (123456789012.34); -- 14 total digits, exceeds precision 12
```

```
ERROR:  numeric field overflow
DETAIL:  A field with precision 12, scale 2 must round to an absolute value less than 10^10.
```

### The Approximate Floating-Point Family

`REAL` (single precision, 4 bytes, ~6 decimal digits of precision) and `DOUBLE PRECISION` (double precision, 8 bytes, ~15 decimal digits of precision) are IEEE 754 binary floating-point types — the same representation used by most programming languages' native floating-point numbers. They store an approximation of a value, trading exactness for compact storage and very fast hardware-native arithmetic:

```sql
CREATE TABLE sensor_readings (
    id SERIAL PRIMARY KEY,
    temperature_celsius REAL,
    precise_measurement DOUBLE PRECISION
);

INSERT INTO sensor_readings (temperature_celsius, precise_measurement)
VALUES (21.5, 3.14159265358979);
```

Floating-point types are entirely appropriate for scientific measurements, sensor data, or anything where a value is inherently approximate anyway (a temperature reading has measurement error far larger than any floating-point rounding error) and where raw arithmetic speed across huge datasets matters more than exactness to the last digit. They are never appropriate for money, quantities of discrete countable things, or any value where two representations of "the same" number must compare equal.

### `SERIAL`, `BIGSERIAL`, and `IDENTITY` — Auto-Incrementing Columns

`SERIAL` isn't a true data type — it's PostgreSQL shorthand that expands into an `INTEGER` column, a backing **sequence** object (a separate database object that generates the next integer on demand), and a `DEFAULT` that calls that sequence:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_name TEXT
);
```

is exactly equivalent to:

```sql
CREATE SEQUENCE orders_id_seq;
CREATE TABLE orders (
    id INTEGER NOT NULL DEFAULT nextval('orders_id_seq') PRIMARY KEY,
    customer_name TEXT
);
ALTER SEQUENCE orders_id_seq OWNED BY orders.id;
```

| Variant | Underlying integer type | Range |
|---|---|---|
| `SMALLSERIAL` | `SMALLINT` | 1 to 32,767 |
| `SERIAL` | `INTEGER` | 1 to 2,147,483,647 |
| `BIGSERIAL` | `BIGINT` | 1 to 9,223,372,036,854,775,807 |

Because `SERIAL` is really just "an integer column with a default," nothing stops you from inserting an explicit value that skips or collides with the sequence:

```sql
INSERT INTO orders (id, customer_name) VALUES (9999, 'Manual Insert');
INSERT INTO orders (customer_name) VALUES ('Auto Insert'); -- sequence still tries to produce 1, 2, 3...
```

This can desynchronize the sequence from the actual maximum `id` in the table, eventually causing a duplicate-key error when the sequence catches up to a manually-inserted value.

The modern, SQL-standard-compliant alternative is an **identity column**, using `GENERATED ALWAYS AS IDENTITY` or `GENERATED BY DEFAULT AS IDENTITY`:

```sql
CREATE TABLE orders_v2 (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_name TEXT
);

INSERT INTO orders_v2 (id, customer_name) VALUES (9999, 'Manual Insert');
```

```
ERROR:  cannot insert into column "id"
DETAIL:  Column "id" is an identity column defined as GENERATED ALWAYS.
HINT:  Use OVERRIDING SYSTEM VALUE to override.
```

`GENERATED ALWAYS AS IDENTITY` refuses manual values by default (protecting you from exactly the desynchronization problem above), unless you explicitly opt in with `OVERRIDING SYSTEM VALUE`. `GENERATED BY DEFAULT AS IDENTITY` behaves more like `SERIAL` — it allows manual overrides without extra syntax — but is still backed by proper sequence metadata that PostgreSQL tracks more robustly than a bare `SERIAL` default. For new schemas, `GENERATED ALWAYS AS IDENTITY` is the generally preferred choice.

### Choosing the Right Numeric Type

| If the column is... | Use |
|---|---|
| A row-identifying auto-incrementing key | `INTEGER`/`BIGINT` with `GENERATED ALWAYS AS IDENTITY` (or `BIGSERIAL` in older code) |
| Money, prices, account balances, tax amounts | `NUMERIC(p, s)` with an explicit precision/scale — never `REAL`/`DOUBLE PRECISION` |
| A simple count or quantity with a known small range | `SMALLINT` or `INTEGER` |
| A count that could plausibly grow into the billions (e.g., a global event counter) | `BIGINT` |
| A scientific measurement, sensor value, or statistical computation | `REAL` or `DOUBLE PRECISION` |
| A percentage or ratio needing exact decimal comparisons | `NUMERIC` |

## Internal Working (Preview)

`SMALLINT`, `INTEGER`, and `BIGINT` are stored as fixed-width two's-complement binary integers — exactly the same representation a CPU's arithmetic unit works with natively, which is why comparisons and arithmetic on them are extremely fast.

`REAL` and `DOUBLE PRECISION` follow the IEEE 754 layout: a sign bit, an exponent, and a mantissa (significand):

```
 DOUBLE PRECISION (64 bits total)
 ┌─┬───────────────┬──────────────────────────────────────────────────┐
 │S│   Exponent    │                    Mantissa                      │
 │1│    11 bits    │                     52 bits                      │
 └─┴───────────────┴──────────────────────────────────────────────────┘
```

Because the mantissa represents a binary fraction, only numbers expressible as a sum of powers of two can be stored exactly — `0.5`, `0.25`, `0.125` are exact; `0.1` is not, the same way `1/3` has no finite decimal expansion.

`NUMERIC`, by contrast, is stored as a variable-length sequence of base-10000 "digit groups" plus metadata recording the weight (where the decimal point falls) and the declared precision/scale — it is arithmetic performed in decimal, digit-group by digit-group, in software rather than directly by the CPU's binary floating-point unit. This is why `NUMERIC` arithmetic is exact but measurably slower per operation than `INTEGER` or `DOUBLE PRECISION` arithmetic — you are trading raw speed for guaranteed correctness.

## Real-World Analogy

Think of `DOUBLE PRECISION` like measuring a plank of wood with a tape measure marked only in imperial fractions (halves, quarters, eighths, sixteenths of an inch) — most measurements land close to a mark, but a measurement of exactly "one-tenth of an inch" simply cannot be represented on that tape; you always round to the nearest mark the tape actually has. `NUMERIC` is like an accountant's ledger that records amounts to the exact cent, in base 10, the same base your currency actually uses — there is no rounding to the nearest available mark, because the ledger's "marks" are defined in the same units as the money itself.

## Why Numeric Types Were Designed This Way

The SQL standard deliberately separates *exact numeric types* (`INTEGER`, `NUMERIC`/`DECIMAL`) from *approximate numeric types* (`REAL`, `DOUBLE PRECISION`) because these solve fundamentally different problems and no single representation is good at both. Exact types exist because financial and counting data must never silently drift — a database's entire value proposition, established in Module 01, is that stored data can be trusted and rules are enforced centrally; a numeric type that silently introduces rounding error would undermine that guarantee for an entire category of data. Approximate types exist because scientific and engineering computation needs speed and enormous dynamic range (from the extremely small to the extremely large) far more than it needs exactness to the last digit, and IEEE 754 floating-point hardware is a mature, universal, and extremely fast standard for that trade-off. PostgreSQL exposes both families rather than picking one, and leaves the choice — and its consequences — squarely to you as the schema designer.

## Advantages

- **Exact types guarantee correctness for discrete and financial data** — an `INTEGER` count or a `NUMERIC` balance means exactly what it says, with no representational drift, which is essential wherever "close enough" is not good enough.
- **Approximate types are compact and extremely fast** — `DOUBLE PRECISION` arithmetic runs directly on CPU floating-point hardware, making it ideal for large-scale scientific or statistical computation.
- **`SERIAL`/`IDENTITY` remove an entire class of manual bookkeeping** — you never have to compute or coordinate the "next" ID yourself; the database guarantees uniqueness and ordering for you.
- **Explicit range types (`SMALLINT`/`INTEGER`/`BIGINT`) let you document and enforce expected value ranges at the schema level**, catching bugs (like an accidentally negative quantity growing absurdly large) as a loud error instead of silent corruption.

## Disadvantages / Limitations

- **`NUMERIC` arithmetic is slower than native integer or floating-point arithmetic**, because it is performed in software, digit-group by digit-group, rather than directly by CPU hardware — for very high-volume numerical computation where exactness genuinely doesn't matter, this cost is real.
- **Floating-point types cannot be safely compared for exact equality**, which rules them out for anything requiring precise thresholds or exact totals — a design constraint you must remember every time you reach for `REAL`/`DOUBLE PRECISION`.
- **`SERIAL` (as opposed to `IDENTITY`) permits accidental manual overrides that can desynchronize the sequence**, a foot-gun that newer schemas should avoid by preferring `GENERATED ALWAYS AS IDENTITY`.
- **Choosing too small an integer range (e.g., `SMALLINT` for an ID space that later needs billions of rows) requires a disruptive type change later** — Module 04 covers how column type changes on a live table can be costly and risky, which makes it worth erring slightly larger up front for any column expected to grow substantially.

## Best Practices

- Always use `NUMERIC` with an explicit precision and scale for money, taxes, prices, and any value where "off by a fraction of a cent, compounded over millions of rows" is unacceptable.
- Default to `INTEGER` for ordinary counts and IDs; only reach for `BIGINT` when you have good reason to expect the value could exceed about 2.1 billion (a very high-traffic ID column, for instance).
- Prefer `GENERATED ALWAYS AS IDENTITY` over `SERIAL` in new schemas — it provides the same auto-increment behavior with stronger protection against manual-value desynchronization.
- Never use `REAL`/`DOUBLE PRECISION` for any value that will be summed, compared for equality, or displayed as an exact total to a user (invoices, balances, inventory counts).
- When in doubt about the correct integer size, prefer the next size up rather than the smallest one that "just fits" today — resizing a heavily-used column later is more disruptive than a few extra bytes per row now.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Storing money in `REAL` or `DOUBLE PRECISION` | Binary floating-point cannot represent most decimal fractions exactly, causing rounding drift that compounds over many transactions — always use `NUMERIC` for money. |
| Assuming an out-of-range `INSERT` truncates or wraps the value | PostgreSQL always raises a hard error (`out of range` / `numeric field overflow`) rather than silently truncating or wrapping an integer or numeric value. |
| Declaring bare `NUMERIC` with no precision/scale "just to be safe" | It works, but it forfeits useful early error-catching — an explicit precision/scale documents and enforces the expected shape of the data. |
| Manually inserting explicit values into a `SERIAL` column without considering the sequence | This can desynchronize the sequence from the table's actual maximum ID, eventually causing duplicate-key errors; `GENERATED ALWAYS AS IDENTITY` prevents this by default. |
| Comparing two floating-point values with `=` and expecting exact equality | Floating-point arithmetic accumulates tiny representational errors; use `NUMERIC` if exact equality matters, or an explicit tolerance/rounding comparison if floating-point is unavoidable. |

## Interview Questions

1. **Q: Why should `NUMERIC` be used instead of `DOUBLE PRECISION` for storing monetary values?**
   A: `DOUBLE PRECISION` is an IEEE 754 binary floating-point type that can only exactly represent numbers expressible as sums of powers of two — most decimal fractions, including simple ones like `0.1`, have no exact binary representation and are stored as a close approximation. Over many arithmetic operations these tiny errors compound into visible discrepancies. `NUMERIC` stores values as an exact base-10 representation with a defined precision and scale, so decimal arithmetic on it is exact, which is required for money, invoices, and any value where an exact total matters.

2. **Q: What is the difference between `SERIAL` and `GENERATED ALWAYS AS IDENTITY`?**
   A: `SERIAL` is shorthand that creates an integer column, a backing sequence, and a default expression calling that sequence — but because it's implemented as an ordinary default, you can freely insert explicit values into the column, which can desynchronize the sequence from the table's real data. `GENERATED ALWAYS AS IDENTITY` is the SQL-standard-compliant equivalent that, by default, rejects explicit manual inserts into that column (raising an error unless you use `OVERRIDING SYSTEM VALUE`), giving stronger protection against that class of bug. `GENERATED BY DEFAULT AS IDENTITY` allows manual values like `SERIAL` does, while still using the more standards-compliant identity metadata.

3. **Q: What happens if you try to insert a value that's too large for a column's declared numeric type?**
   A: PostgreSQL raises a hard error and rejects the insert entirely — for integer types, an "out of range" error; for `NUMERIC(p, s)`, a "numeric field overflow" error when the value's digit count exceeds the declared precision. Values are never silently truncated, rounded, or wrapped around.

4. **Q: When would you legitimately choose `REAL` or `DOUBLE PRECISION` over `NUMERIC`?**
   A: When the data is inherently approximate anyway (scientific measurements, sensor readings, statistical computations) and raw arithmetic speed or storage compactness across a very large dataset matters more than exactness to the last digit — the measurement error in a physical sensor reading, for example, typically dwarfs any floating-point representational error, making the trade-off acceptable.

## Summary

- PostgreSQL's exact integer family (`SMALLINT`, `INTEGER`, `BIGINT`) stores whole numbers as fixed-width binary integers with no rounding; out-of-range inserts always raise a hard error, never silent truncation.
- `NUMERIC`/`DECIMAL` (exact synonyms) store base-10 values with a declared precision and scale, representing decimal fractions exactly — the required choice for money and any value needing exact decimal comparison.
- `REAL` and `DOUBLE PRECISION` are IEEE 754 binary floating-point types — fast and compact, but unable to exactly represent most decimal fractions, making them unsuitable for money or exact equality comparisons.
- `SERIAL`/`BIGSERIAL` are shorthand for an integer column plus a backing sequence and default; `GENERATED ALWAYS AS IDENTITY` is the modern, safer, SQL-standard-compliant alternative that guards against accidental manual-value desynchronization.
- Choosing a numeric type is a real design decision with real consequences — pick exact types for anything counted or paid, and approximate types only for genuinely approximate, high-volume computation.
