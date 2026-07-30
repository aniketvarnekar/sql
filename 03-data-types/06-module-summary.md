# Module 03 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Numeric Types** — exact integer types (`SMALLINT`/`INTEGER`/`BIGINT`), exact `NUMERIC`/`DECIMAL`, approximate `REAL`/`DOUBLE PRECISION`, `SERIAL`/`BIGSERIAL`/`IDENTITY` auto-increment, overflow behavior, and how to choose the right type.
- [x] **Character and String Types** — `CHAR(n)` vs. `VARCHAR(n)` vs. `TEXT`, fixed vs. variable-length storage, why PostgreSQL treats `VARCHAR`/`TEXT` almost identically, and a brief look at collation.
- [x] **Date and Time Types** — `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ` vs. `TIMESTAMP WITHOUT TIME ZONE`, `INTERVAL`, and why timezone-aware storage matters.
- [x] **Boolean and Other Types** — `BOOLEAN`'s three states, plus a practical tour of `UUID`, `ARRAY`, and `ENUM`, with a brief note that `JSON`/`JSONB` exist and are covered fully later.
- [x] **NULL and Three-Valued Logic** — what `NULL` really means, three-valued logic and its truth tables, `IS NULL`/`IS NOT NULL`, and how `NULL` interacts with arithmetic, aggregates, and boolean expressions.

## Practical Connections

- A billing system that stores prices and balances in `NUMERIC` rather than `DOUBLE PRECISION` avoids the slow, silent accumulation of rounding errors that would otherwise show up as unexplained discrepancies in a financial audit months later.
- A global application serving users across many time zones relies entirely on `TIMESTAMPTZ` to guarantee that "when did this event happen" means the same real-world instant no matter which server, region, or user's browser recorded or displays it.
- A reporting dashboard aggregating millions of rows depends on precise `NULL` semantics — a report that silently drops or miscounts rows with missing data (because of a `<>` comparison or a `NOT IN` subquery) can produce numbers that look plausible while being quietly wrong, which is far more dangerous than a query that fails loudly.
- A distributed system accepting data from many independent, occasionally offline clients relies on `UUID` identifiers precisely because no single central sequence generator is reachable from every client at all times.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| `NUMERIC`/`DECIMAL` vs. `REAL`/`DOUBLE PRECISION` | `NUMERIC` stores exact base-10 values with no rounding error, required for money and exact totals; `REAL`/`DOUBLE PRECISION` are fast IEEE 754 binary approximations, unsuitable for exact equality or financial data. |
| `SERIAL` vs. `GENERATED ALWAYS AS IDENTITY` | Both auto-increment an integer column via a backing sequence, but `SERIAL`'s plain `DEFAULT`-based implementation allows accidental manual-value overrides that can desynchronize the sequence, while `IDENTITY` (with `ALWAYS`) rejects manual inserts by default. |
| `CHAR(n)` vs. `VARCHAR(n)` vs. `TEXT` | `CHAR(n)` is fixed-length and blank-padded; `VARCHAR(n)` is variable-length with an enforced maximum; `TEXT` is variable-length with no maximum — and in PostgreSQL specifically, all three share the same internal storage, so there is no performance advantage to `CHAR(n)`. |
| `TIMESTAMP` vs. `TIMESTAMPTZ` | `TIMESTAMP` stores a bare wall-clock reading with no time zone awareness at all; `TIMESTAMPTZ` stores a genuinely unambiguous instant, normalized to UTC internally and converted for display based on session time zone. |
| `NULL` vs. zero / empty string / `FALSE` | `NULL` represents an unknown or absent value; `0`, `''`, and `FALSE` are all concrete, specific values — conflating them is a frequent source of incorrect filtering and calculation logic. |
| `COUNT(*)` vs. `COUNT(column)` | `COUNT(*)` counts every row regardless of `NULL`s; `COUNT(column)` counts only rows where that specific column is not `NULL`. |
| `ENUM` vs. a lookup table with a foreign key | `ENUM` enforces a closed set of values with no join required but needs a schema change to add new values; a lookup table requires a join but lets new valid values be added with a plain `INSERT`. |

## What's Next

This module gave you a precise, practical map of PostgreSQL's core data types — numeric, string, date/time, boolean, and PostgreSQL's more specialized types — plus the single most important semantic wrinkle underlying all of them: what `NULL` means and how three-valued logic governs every comparison, aggregate, and boolean expression you write from here on. **Module 04 — Database & Table Design** builds directly on this: you'll use every type from this module inside real `CREATE TABLE` statements, learn how to alter a table's structure safely once it's live, and understand the structural consequences of dropping or truncating a table — turning the type judgment you just built into actual, standing database objects.
