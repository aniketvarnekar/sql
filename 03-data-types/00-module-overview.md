# Module 03 — Data Types

## Module Goal

By the end of this module, you will know exactly which PostgreSQL data type to reach for when you design a column, and — just as importantly — why the wrong choice quietly causes bugs, wasted storage, or corrupted financial data years later. You'll understand the difference between exact and approximate numbers, the three ways PostgreSQL stores text, how it separates a "wall-clock reading" from a "point in time," a tour of booleans and PostgreSQL's more exotic built-in types, and — the single most consequential topic in this module — what `NULL` really means and why it breaks ordinary two-valued logic. Every `CREATE TABLE` you write from Module 4 onward depends on the judgment this module builds.

## Topics Covered in This Module

1. **[Numeric Types](01-numeric-types.md)** — `INTEGER`/`SMALLINT`/`BIGINT`, exact (`NUMERIC`/`DECIMAL`) vs. approximate (`REAL`/`DOUBLE PRECISION`) numbers, `SERIAL`/`BIGSERIAL`/`IDENTITY` for auto-incrementing columns, overflow behavior, and choosing the right numeric type.
2. **[Character and String Types](02-character-and-string-types.md)** — `CHAR(n)` vs. `VARCHAR(n)` vs. `TEXT`, fixed vs. variable-length storage, why PostgreSQL treats `VARCHAR` and `TEXT` almost identically internally, and collation basics.
3. **[Date and Time Types](03-date-and-time-types.md)** — `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMP WITH TIME ZONE` vs. `WITHOUT`, `INTERVAL`, and why timezone-aware timestamps matter.
4. **[Boolean and Other Types](04-boolean-and-other-types.md)** — `BOOLEAN`'s three states, and a practical tour of `UUID`, `ARRAY`, and `ENUM` (with a brief note that `JSON`/`JSONB` exist, deferred to a later module).
5. **[NULL and Three-Valued Logic](05-null-and-three-valued-logic.md)** — what `NULL` actually means, three-valued logic, why `NULL = NULL` isn't `TRUE`, and how `NULL` interacts with arithmetic, aggregates, and boolean expressions.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 01 — Introduction**, in full: you need a working PostgreSQL setup and `psql` session ([Setting Up PostgreSQL](../01-introduction/04-setting-up-postgresql.md)) to run every example in this module, and you already used `TEXT`, `NUMERIC`, and `SERIAL` without a full explanation in [Your First Query](../01-introduction/05-your-first-query.md) — this module is where those types are finally explained properly.
- **Module 02 — Relational Model**: you need the idea, established there, that every column in a relational table has one single, declared, enforced data type, and that this typing is part of what makes a database *relational* rather than an unstructured pile of values. This module is the detailed, practical follow-through on that promise: it catalogs the actual types PostgreSQL lets you declare and the consequences of each choice.

## How to Study This Module

Read Topics 1 and 2 (numeric and string types) carefully and type out every example — nearly every table you will ever design uses one of these two families constantly, and the mistakes they cover (floating-point money, silent truncation assumptions) are extremely common in real systems. Topic 3 (date and time) rewards careful reading even though it feels procedural at first — the `TIMESTAMP` vs. `TIMESTAMPTZ` distinction is one of the most consequential and most frequently mishandled decisions in real schema design. Topic 4 is a lighter tour; skim the parts that don't apply to your immediate needs, but don't skip the reasoning behind `ENUM` vs. a lookup table, since that judgment call reappears in later modules. Topic 5, `NULL` and three-valued logic, is the topic to re-read even after you think you understand it — it is short in surface area but is responsible for more subtle, hard-to-spot bugs in real-world SQL than any other single concept in this course, and it will keep mattering all the way through joins, subqueries, and aggregation in later modules.
