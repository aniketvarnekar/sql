# Boolean and Other Types

## Learning Objectives

By the end of this section you should be able to:
- Declare and use a `BOOLEAN` column, including its `NULL`/unknown state
- Recognize `UUID` columns and explain why they're used as an alternative to auto-incrementing IDs
- Recognize `ARRAY` types and read/write a simple array column
- Recognize `ENUM` (user-defined enumerated) types and explain the trade-off between an `ENUM` and a plain lookup table
- State that PostgreSQL also has native `JSON`/`JSONB` types, without needing to use them yet

## Prerequisites

- [Date and Time Types](03-date-and-time-types.md) — this module's previous topic; no direct dependency, but continues the same type-selection judgment.
- This topic briefly touches `NULL` as it applies to `BOOLEAN`; the full explanation of `NULL` and three-valued logic is the next topic, [NULL and Three-Valued Logic](05-null-and-three-valued-logic.md) — a short forward-reference is noted below where relevant.

## Motivation

Beyond numbers, strings, and dates, real schemas frequently need a simple yes/no flag, a globally unique identifier that doesn't depend on a central counter, a small bundled list of values, or a column restricted to one of a fixed, known set of options. PostgreSQL provides dedicated, well-designed types for every one of these needs — `BOOLEAN`, `UUID`, `ARRAY`, and `ENUM` — and knowing they exist (and when to reach for each) prevents you from reinventing weaker versions of them out of integers and text.

## Problem Statement

Picture three small but common design decisions:
1. You need an "is this account active?" flag. You could store `0`/`1` in an `INTEGER` column (a very common habit carried over from systems without a native boolean type) — but does `0` mean false, or does it mean something else entirely to a future reader of the schema?
2. You need to assign a unique ID to a record generated on a mobile device that's currently offline, with no way to ask a central database for "the next" sequence value — a `SERIAL`/`IDENTITY` column can't help here, since it fundamentally depends on a single, central counter.
3. You need an order's status restricted to exactly `'pending'`, `'shipped'`, `'delivered'`, or `'cancelled'` — using a plain `TEXT` column for this invites typos (`'Shiped'`) that silently create a new, invalid status with no error at all.

Each of these has a purpose-built PostgreSQL type, covered below.

## Concept

### `BOOLEAN` — True, False, or Unknown

`BOOLEAN` stores exactly one of three states: `TRUE`, `FALSE`, or `NULL` (meaning "unknown/not recorded" — the next topic covers precisely what this third state means and how it behaves in logic). PostgreSQL accepts several convenient literal spellings on input, all normalized to `t`/`f` on output:

```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    username TEXT,
    is_active BOOLEAN
);

INSERT INTO accounts (username, is_active) VALUES
    ('asha', TRUE),
    ('ben', 'false'),
    ('chen', 'yes'),
    ('devi', NULL);   -- activation status not yet recorded

SELECT username, is_active FROM accounts;
```

```
 username | is_active
----------+-----------
 asha     | t
 ben      | f
 chen     | t
 devi     |
(4 rows)
```

Note that `devi`'s `is_active` is blank in the output — that's `NULL`, not `FALSE`. A column declared `BOOLEAN` without `NOT NULL` can always be in this third, unrecorded state; the next topic explains exactly why treating it as equivalent to `FALSE` is a common and serious mistake.

### `UUID` — A Universally Unique Identifier

A `UUID` (Universally Unique Identifier) is a 128-bit value, conventionally displayed as 32 hexadecimal digits grouped with hyphens (e.g., `a1b2c3d4-e5f6-4890-ab12-3456789abcde`), designed so that independently generated values are, for all practical purposes, guaranteed never to collide — with no coordination between the systems generating them:

```sql
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_name TEXT
);

INSERT INTO devices (device_name) VALUES ('Warehouse Scanner 12');

SELECT * FROM devices;
```

```
                  id                  |      device_name
--------------------------------------+------------------------
 f47ac10b-58cc-4372-a567-0e02b2c3d479 | Warehouse Scanner 12
(1 row)
```

`gen_random_uuid()` is PostgreSQL's built-in function (available natively from PostgreSQL 13 onward; earlier versions require enabling the `pgcrypto` extension with `CREATE EXTENSION pgcrypto`) for generating a random UUID. Unlike `SERIAL`/`IDENTITY`, no central sequence is involved at all — any number of independent systems (offline mobile clients, separate database shards, distributed services) can each generate their own UUIDs with negligible risk of two of them ever matching.

### `ARRAY` — A Column Holding a List of Values

PostgreSQL allows (almost) any data type to have an array variant, written as the base type followed by square brackets:

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[]
);

INSERT INTO articles (title, tags) VALUES
    ('Understanding Indexes', ARRAY['databases', 'performance', 'sql']);

SELECT title, tags, tags[1] AS first_tag
FROM articles;
```

```
         title           |               tags                | first_tag
--------------------------+-----------------------------------+------------
 Understanding Indexes    | {databases,performance,sql}       | databases
(1 row)
```

```sql
SELECT title FROM articles WHERE 'performance' = ANY (tags);
```

```
         title
------------------------
 Understanding Indexes
(1 row)
```

Array indexing in PostgreSQL is 1-based (`tags[1]` is the *first* element, not the second), and `= ANY (array_column)` is the standard way to test whether a value appears anywhere in an array.

### `ENUM` — A Fixed, Named Set of Values

A PostgreSQL `ENUM` is a custom type you define once, listing every value it's allowed to hold — any attempt to store something outside that list fails immediately:

```sql
CREATE TYPE order_status AS ENUM ('pending', 'shipped', 'delivered', 'cancelled');

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);

INSERT INTO orders (status) VALUES ('shipped');
INSERT INTO orders (status) VALUES ('Shiped');  -- typo, not a defined label
```

```
ERROR:  invalid input value for enum order_status: "Shiped"
```

The typo is caught immediately as a hard error, exactly where a plain `TEXT` column would have silently accepted it. A subtler benefit: `ENUM` values sort in the order they were *declared* when the type was created, not alphabetically — so `ORDER BY status` naturally produces `pending, shipped, delivered, cancelled` order (the real-world lifecycle order) rather than alphabetical order, without any extra `CASE` expression.

Adding a new label later requires `ALTER TYPE order_status ADD VALUE 'returned';` — straightforward, but worth knowing that in some PostgreSQL versions a newly added enum value cannot be used within the very same transaction that added it, a minor operational quirk worth being aware of. Because of this rigidity, many teams prefer a plain lookup table with a foreign key (covered in Module 05 — Constraints & Keys, and Module 10 — Joins) over an `ENUM` whenever the set of valid values is expected to change reasonably often — a lookup table's rows can be added with a simple `INSERT`, no schema migration required.

### A Brief Note on `JSON`

PostgreSQL also has native `JSON` and `JSONB` column types for storing semi-structured, document-like data directly inside a relational column — useful when a value's shape genuinely varies row-to-row in a way a fixed set of columns can't cleanly express. This module doesn't cover them further; they get full, dedicated treatment later in the course, in the Advanced SQL module. For now, it's enough to know the types exist and what problem they're aimed at.

## Internal Working (Preview)

`BOOLEAN` is stored as a single byte. `UUID` is stored as a fixed 16-byte binary value (not as a 36-character string, even though it displays as one), which is compact and fast to compare and index. `ARRAY` values are stored using PostgreSQL's variable-length storage, prefixed with metadata recording the array's dimensions and element count, followed by the elements themselves. `ENUM` values are stored internally as a small fixed-width identifier (conceptually similar to an internal integer), which is mapped to and from its human-readable label via an entry PostgreSQL keeps in its system catalog — this is exactly how PostgreSQL knows the "declared order" of enum labels for sorting purposes, since that ordering is recorded as catalog metadata at creation time rather than derived from the labels' spelling.

## Real-World Analogy

`BOOLEAN` is like a physical light switch that can be on, off, or — if the switch is simply unplugged from any wiring — genuinely undetermined, rather than forcibly defaulting to "off." `UUID` is like a serial number stamped independently by many different factories around the world using a scheme specifically designed so that no two factories, without ever communicating, will ever stamp the same number — unlike a single ticket-dispensing machine (a sequence/`SERIAL`) that can only ever hand out one "next" number at a time from one physical location. `ENUM` is like a multiple-choice form with pre-printed checkboxes — you can only select one of the printed options, unlike a blank line (`TEXT`) where anyone could write anything, including a typo.

## Why These Types Were Designed This Way

PostgreSQL is deliberately more extensible than the bare SQL standard requires — it allows genuinely new types (`ENUM`, and arrays of arbitrary element types) to be defined by the schema designer rather than restricting you to a small, fixed built-in catalog. This reflects the same underlying principle from Module 02 that a relational column's type is a real, enforced constraint on what can be stored — `ENUM` simply lets *you* define a brand-new, narrowly-scoped type when none of the built-in ones expresses your business rule precisely enough, rather than forcing you to approximate that rule with a looser type (`TEXT`) plus separate, easy-to-forget application-level validation. `UUID` exists because the relational model's usual answer to "give every row a unique identifier" (a centrally-generated sequence) assumes a single coordinating authority, which breaks down in distributed or offline-capable systems — `UUID` provides a mathematically sound way to keep the same guarantee (uniqueness) without that central coordination requirement.

## Advantages

- **`BOOLEAN` is self-documenting** — a column typed `BOOLEAN` unambiguously means true/false/unknown, unlike an `INTEGER` flag where `0`/`1` conventions must be remembered or looked up.
- **`UUID` requires no central coordination to guarantee uniqueness**, making it well suited to distributed systems, offline-capable clients, or merging data from multiple independently-operating sources.
- **`ARRAY` lets you store a small, naturally list-shaped value without a separate table**, which is convenient when the list is genuinely simple and doesn't need its own rich set of attributes.
- **`ENUM` enforces a closed set of valid values at the database layer itself**, catching typos and invalid states the instant they're inserted rather than downstream in application logic or, worse, silently in a report.

## Disadvantages / Limitations

- **`UUID` values are effectively random**, which can hurt index locality and clustering performance compared to sequential integer IDs on very large, heavily-indexed tables — a trade-off covered in more depth in Module 13 (Indexes).
- **`ARRAY` columns make it harder to query, join, or constrain individual elements** compared to modeling a genuine one-to-many relationship as its own related table (Module 10 — Joins, Module 15 — Normalization) — arrays are best reserved for small, simple, rarely-queried-by-individual-element lists.
- **`ENUM` is rigid**: adding, removing, or reordering values requires a schema change (`ALTER TYPE`), and some operations on enum types have transactional quirks — a plain lookup table with a foreign key is often more flexible when the set of valid values changes with any regularity.
- **`JSON`/`JSONB` (mentioned only briefly here) trade the enforced structure of ordinary columns for flexibility** — a trade-off significant enough that the course devotes a full later module to it rather than covering it properly here.

## Best Practices

- Prefer `BOOLEAN` over an `INTEGER` 0/1 flag for any true/false column — it documents its own meaning and cannot be confused with an actual numeric quantity.
- Use `UUID` primary keys when records may be generated outside a single central database (offline clients, multiple independent services, data merged from separate systems); otherwise, a `BIGINT`/`IDENTITY` key is usually simpler and more index-friendly.
- Reserve `ARRAY` columns for genuinely simple, small, rarely-individually-queried lists (like a handful of tags); model anything resembling a real one-to-many relationship as a separate related table instead.
- Choose `ENUM` when the set of valid values is small and genuinely stable (e.g., the days of the week); prefer a lookup table with a foreign key when the set of valid values is expected to grow or change over the application's lifetime.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `INTEGER` (0/1) instead of `BOOLEAN` for a true/false flag | It works but loses self-documentation and type safety — nothing stops an accidental `2` from being inserted, whereas `BOOLEAN` only ever holds `TRUE`, `FALSE`, or `NULL`. |
| Assuming a `BOOLEAN` column can only ever be `TRUE` or `FALSE` | Unless declared `NOT NULL`, it can also be `NULL` (unrecorded/unknown) — a state that behaves differently from `FALSE` in `WHERE` clauses, covered fully in the next topic. |
| Reaching for `ARRAY` to model a genuine one-to-many relationship with its own attributes | A related row (e.g., "order line items," each with its own quantity and price) usually belongs in its own table linked by a foreign key, not bundled into an array column — arrays are best for simple, flat lists. |
| Assuming an `ENUM`'s values sort alphabetically | `ENUM` values sort in the order they were declared when the type was created, which is often intentionally the real-world lifecycle order, not alphabetical order. |
| Treating `JSON`/`JSONB` as a shortcut to avoid designing proper columns | It's a legitimate tool for genuinely variable-shaped data, but using it to avoid schema design entirely forfeits the type-checking and query clarity ordinary typed columns provide — covered fully in a later, dedicated module. |

## Interview Questions

1. **Q: What are the three possible values of a PostgreSQL `BOOLEAN` column?**
   A: `TRUE`, `FALSE`, and — unless the column is declared `NOT NULL` — `NULL`, meaning the value is simply unrecorded or unknown. `NULL` is not equivalent to `FALSE`; it behaves distinctly in comparisons and boolean logic, covered in full in the next topic.

2. **Q: Why would you choose a `UUID` primary key over a `SERIAL`/`IDENTITY` integer key?**
   A: `UUID` values can be generated independently by any number of systems (offline clients, separate services, distributed shards) with a negligible chance of collision, requiring no central coordinating sequence. `SERIAL`/`IDENTITY` depend on a single sequence generator, which works well for a single centralized database but doesn't fit systems that need to generate valid, unique IDs without contacting a central authority first.

3. **Q: What's the trade-off between using an `ENUM` type versus a lookup table with a foreign key for something like an order's status?**
   A: An `ENUM` enforces a closed, valid set of values directly at the type level with no extra join required, and its declared order is preserved for sorting — but adding or changing values requires a schema change (`ALTER TYPE`), which can be operationally inconvenient if the set of valid statuses changes often. A lookup table with a foreign key requires an extra join to use but allows new valid values to be added with a plain `INSERT`, no schema migration needed — generally preferable when the set of values is expected to evolve.

4. **Q: Does PostgreSQL support storing a document-like, variably-shaped value directly in a column?**
   A: Yes — PostgreSQL has native `JSON` and `JSONB` types for exactly this purpose, letting you store semi-structured data that doesn't fit a fixed set of typed columns. Full coverage of querying and indexing them is deferred to a later, dedicated module, since it's a large enough topic to warrant its own treatment.

## Summary

- `BOOLEAN` stores `TRUE`, `FALSE`, or (unless constrained otherwise) `NULL` — self-documenting and safer than an integer 0/1 convention.
- `UUID` is a 128-bit identifier that can be generated independently by many systems with negligible collision risk, making it well suited to distributed or offline-capable ID generation, at some cost to index locality on very large tables.
- `ARRAY` lets a column hold a list of values of any base type, useful for small, simple, flat lists — but a genuine one-to-many relationship usually belongs in its own related table instead.
- `ENUM` restricts a column to a fixed, named, ordered set of values, catching invalid input immediately — at the cost of requiring a schema change to add new values, which is why a lookup table is often preferred for frequently-changing value sets.
- PostgreSQL also has native `JSON`/`JSONB` types for semi-structured data — worth knowing they exist now, with full depth deferred to a later, dedicated module.
