# Character and String Types

## Learning Objectives

By the end of this section you should be able to:
- Distinguish `CHAR(n)`, `VARCHAR(n)`, and `TEXT`, and state what each guarantees about stored length
- Explain the storage implications of fixed-length vs. variable-length character data
- Explain why PostgreSQL treats `VARCHAR` and `TEXT` almost identically internally, unlike some other database systems
- Describe, at a basic level, what collation is and why it affects string comparison and sorting
- Judge when a declared length limit on a string column is actually meaningful versus cosmetic

## Prerequisites

- [Numeric Types](01-numeric-types.md) — this module's first topic; no new numeric concepts are needed here, but the same "pick the type that matches what the data actually is" judgment applies to strings.
- [Your First Query](../01-introduction/05-your-first-query.md) — you already used `TEXT` there for a `name` column without a full explanation of the alternatives; this topic supplies that explanation.

## Motivation

Text is everywhere in real schemas — names, emails, addresses, free-form notes, status codes, product descriptions. SQL gives you three different-looking ways to declare a text column, and it's tempting to assume they behave very differently or that picking the "wrong" one has serious performance consequences. In PostgreSQL specifically, that assumption is mostly false — but knowing exactly *why* it's false, and the one or two cases where the choice does matter, saves you from both a common beginner myth (that `CHAR`/`VARCHAR` are meaningfully faster) and a common beginner bug (assuming a length limit silently truncates instead of rejecting your data).

## Problem Statement

Imagine you're designing a table for a directory of countries, and you need a two-letter country code column (`'US'`, `'IN'`, `'GB'`) alongside a free-form notes column. If you declare the code column as `CHAR(2)` expecting it to always store exactly two visible characters, and someone later inserts `'US '` (with a trailing space, perhaps pasted from a spreadsheet) or a 3-letter code by mistake, what actually happens? And for the notes column — should it be `VARCHAR(500)`, `VARCHAR` with no limit, or `TEXT`? Does the choice affect how much disk space is used, or how fast queries against it run? These are exactly the questions this topic answers precisely, rather than by folklore.

## Concept

### `CHAR(n)` — Fixed-Length, Blank-Padded

`CHAR(n)` (also written `CHARACTER(n)`) declares a column that always *displays* as exactly `n` characters. If you insert a shorter string, PostgreSQL pads it with trailing spaces up to length `n`:

```sql
CREATE TABLE countries (
    code CHAR(2),
    name TEXT
);

INSERT INTO countries VALUES ('US', 'United States');
INSERT INTO countries VALUES ('IN', 'India');

SELECT code, length(code), code || '|' AS code_with_marker FROM countries;
```

```
 code | length | code_with_marker
------+--------+------------------
 US   |      2 | US|
 IN   |      2 | IN|
(2 rows)
```

Here both values happened to be exactly 2 characters, so there's no visible padding. But insert something shorter, and the padding becomes visible when concatenated against a marker:

```sql
CREATE TABLE demo_char (code CHAR(5));
INSERT INTO demo_char VALUES ('US');

SELECT code || '|' AS marked FROM demo_char;
```

```
   marked
-------------
 US   |
(1 row)
```

The stored value is physically padded with spaces to fill all 5 characters. Per the SQL standard, PostgreSQL treats these trailing padding spaces as **semantically insignificant**: they are disregarded when comparing two `character` values, and they are stripped off when a `character` value is converted to `TEXT` or `VARCHAR`. Insert something *longer* than the declared length, and PostgreSQL rejects it outright — it does not silently truncate:

```sql
INSERT INTO demo_char VALUES ('TOOLONG');
```

```
ERROR:  value too long for type character(5)
```

### `VARCHAR(n)` — Variable-Length, With a Limit

`VARCHAR(n)` (also written `CHARACTER VARYING(n)`) stores exactly the characters you give it, up to a maximum of `n` — no padding at all:

```sql
CREATE TABLE demo_varchar (code VARCHAR(5));

INSERT INTO demo_varchar VALUES ('US');       -- accepted, stored as exactly 'US', no padding
INSERT INTO demo_varchar VALUES ('TOOLONG');  -- rejected: 7 characters exceeds the limit of 5
```

```
ERROR:  value too long for type character varying(5)
```

This is a very common point of confusion: **`VARCHAR(n)` does not silently truncate on `INSERT`** — it raises an error, exactly like `CHAR(n)` does, when the value exceeds the declared length. Truncation *does* happen, silently, only when you explicitly `CAST` a value to a `VARCHAR(n)`:

```sql
SELECT 'TOOLONG'::VARCHAR(5) AS explicitly_cast;
```

```
 explicitly_cast
------------------
 TOOLO
(1 row)
```

An explicit cast is a deliberate instruction ("force this into this shape"), so PostgreSQL truncates rather than erroring — but a plain `INSERT` against a table column is not treated as an implicit cast, and errors instead. Keep these two behaviors distinct in your head: **insert against a declared column length → error; explicit `CAST`/`::type` → silent truncation.**

### `TEXT` — Variable-Length, Unlimited

`TEXT` stores a string of any length, with no declared maximum at all. It is a PostgreSQL extension to the SQL standard (the standard only defines `CHARACTER`/`CHARACTER VARYING`), though many other database systems now provide an equivalent unlimited-length type under various names.

```sql
CREATE TABLE demo_text (notes TEXT);

INSERT INTO demo_text VALUES ('Any length of free text is fine, however long it needs to be.');
```

### Why PostgreSQL Treats `VARCHAR` and `TEXT` Nearly Identically

In many older or more storage-conscious database systems, `CHAR(n)` was genuinely faster than `VARCHAR(n)`, because fixed-width fields let the engine calculate a row's exact byte layout in advance. PostgreSQL does not work that way: internally, `CHAR(n)`, `VARCHAR(n)`, and `TEXT` are all stored using the same underlying variable-length representation (informally called `varlena`: a short length header followed by the actual bytes). `VARCHAR(n)` is implemented as this same variable-length storage plus a length check enforced at insert/update time; `TEXT` is exactly that storage with no length check at all. There is no meaningful storage or performance advantage to `CHAR(n)` in PostgreSQL — if anything, `CHAR(n)`'s mandatory padding and stripping logic on every read and write makes it very slightly *more* work, not less. PostgreSQL's own documentation is explicit about this: in most situations, `TEXT` or `VARCHAR` without a length limit should be used instead of `CHAR(n)`, and a length constraint — when you genuinely want to enforce a maximum — is usually better expressed as a `VARCHAR(n)` or as a `CHECK` constraint (covered in Module 05 — Constraints & Keys) than relied upon for any storage benefit.

### String Comparison and Collation, Briefly

When PostgreSQL compares or sorts two strings (`<`, `>`, `ORDER BY`, etc.), it does so according to a **collation** — a set of rules defining alphabetical order, case sensitivity, and how accented or non-ASCII characters sort relative to plain ones. A database has a default collation (usually inherited from your operating system's locale settings at database creation time), and you can override it per-column or per-query with an explicit `COLLATE` clause:

```sql
SELECT 'apple' < 'Banana' AS default_collation_result;

SELECT 'apple' < 'Banana' COLLATE "C" AS byte_order_result;
```

Under a typical locale-aware collation, lowercase and uppercase letters interleave the way a dictionary would sort them; under the `"C"` collation (plain byte-value ordering), all uppercase letters sort before any lowercase letter, because uppercase ASCII codes are numerically smaller. This distinction rarely matters for simple English-only data, but it becomes important once you sort or compare user-generated, multi-language, or mixed-case text — a topic revisited in later modules whenever sorting behavior needs to be precise.

### When Length Limits Actually Matter

Given that PostgreSQL gets no storage or speed benefit from a shorter declared length, a length limit is purely a **data-validation** tool, not a performance one. It's worth declaring when:
- The length is a genuine, fixed business rule you want the database itself to enforce early (a two-letter ISO country code, a fixed-format product SKU).
- You want `psql` or a client tool to visibly document the expected shape of the data just by reading the schema.

It's not worth agonizing over when:
- You're guessing at a "reasonable" limit for something like a name or comment field — an arbitrary `VARCHAR(255)` (a very common but largely superstition-driven convention carried over from other systems) provides no PostgreSQL-specific benefit over plain `TEXT`, and can force an awkward schema change later if a legitimately longer value shows up.

## Internal Working (Preview)

All three character types share PostgreSQL's general variable-length ("varlena") storage format: a short header recording the value's length in bytes, immediately followed by the string's actual byte content.

```
 varlena storage layout (conceptual)
 ┌────────────────────┬───────────────────────────────────────┐
 │  length header      │              string bytes             │
 │ (1 or 4 bytes)       │        (UTF-8 encoded, no padding      │
 │                      │         for varchar/text)             │
 └────────────────────┴───────────────────────────────────────┘
```

For `CHAR(n)`, the stored bytes include the trailing space padding up to `n` characters; PostgreSQL strips that padding back off automatically whenever the value is compared to another `character` value or converted to `TEXT`/`VARCHAR`, so the padding is largely invisible to you unless you go looking for it (as in the `code || '|'` example above). Very long string values (beyond about 2KB) are handled by PostgreSQL's `TOAST` mechanism, which transparently compresses and/or moves the value to a separate storage area — this applies identically to `CHAR`, `VARCHAR`, and `TEXT`, reinforcing that there is no meaningful internal distinction between them beyond the length check.

## Real-World Analogy

`CHAR(n)` is like a paper form with a fixed-width printed grid of boxes for each letter — if your answer is shorter than the number of boxes, you're implicitly leaving the remaining boxes blank (padding), and if your answer is longer than the number of boxes, the form simply won't accept it. `VARCHAR(n)` and `TEXT` are like writing on ruled notebook paper instead — you use exactly as much space as your answer needs, with `VARCHAR(n)` having a printed margin you're not allowed to write past, and `TEXT` being a page with no margin at all.

## Why Character Types Were Designed This Way

The SQL standard originally defined `CHARACTER(n)` for an era when many real-world data sources — punch cards, fixed-format mainframe records, early file formats — genuinely were fixed-width, and a database needed a type that faithfully modeled "always exactly this many characters, padded if necessary." `CHARACTER VARYING(n)` was added for the (now far more common) case of realistically variable-length text with an enforced upper bound. PostgreSQL's own contribution, `TEXT`, reflects a later, pragmatic recognition that most real text data (names, descriptions, comments) has no natural fixed maximum at all, and that forcing an artificial one provides no engine-level benefit once a database's storage layer is built around variable-length values uniformly, as PostgreSQL's is. This is consistent with the relational model's promise, from Module 02, that a column's type constrains *what kind* of data can be stored — length limits are a validation choice layered on top of that guarantee, not a distinct storage mechanism.

## Advantages

- **`CHAR(n)`** faithfully models genuinely fixed-width data and documents that fixed shape directly in the schema, which is useful when interoperating with external systems that expect fixed-width fields.
- **`VARCHAR(n)`** enforces a hard business-rule-driven maximum length at the database layer itself, catching oversized input the moment it's inserted rather than downstream.
- **`TEXT`** imposes no artificial constraint, which avoids future schema changes when a "reasonable" guessed limit turns out to be wrong, and matches PostgreSQL's internal storage model exactly with zero overhead.
- **Uniform internal storage** across all three types means you rarely need to worry about a performance trade-off when choosing between them in PostgreSQL — the choice can be driven purely by what the data validation rules actually require.

## Disadvantages / Limitations

- **`CHAR(n)`'s padding behavior is a frequent source of surprise** — values that look different in a naive string comparison in application code can compare equal once trailing spaces are disregarded, and vice versa if you don't account for it.
- **A `VARCHAR(n)`/`CHAR(n)` limit that's too small forces a disruptive schema change later** (Module 04 covers altering column types), so an overly conservative guess can cost you more than not declaring a limit at all.
- **Relying on a length limit as your only validation** provides no protection against other malformed input (empty strings, wrong character sets, invalid formats) — genuine business-rule validation usually needs a `CHECK` constraint (Module 05) in addition to, or instead of, a bare length limit.
- **Collation-aware comparisons can be slower than raw byte comparisons** (the `"C"` collation), a performance detail that rarely matters until you're sorting or comparing very large volumes of text.

## Best Practices

- Default to `TEXT` (or `VARCHAR` with no length specified) for most string columns unless you have a genuine, fixed business rule to enforce.
- Reserve `CHAR(n)` for truly fixed-width data (ISO codes, fixed-format identifiers) — and even then, many teams prefer `VARCHAR(n)` or `TEXT` with a `CHECK` constraint instead, to avoid the padding-comparison surprise entirely.
- If a maximum length is a real business rule (e.g., "usernames must be 30 characters or fewer"), express it with `VARCHAR(n)` or a `CHECK (length(username) <= 30)` constraint — not as a guess at "how long text usually is."
- Never assume an `INSERT` against a length-limited column will truncate — assume it will error, and validate/truncate explicitly in your application logic or with an explicit cast if truncation is genuinely the desired behavior.
- Be explicit about collation (`COLLATE`) when sort order or case sensitivity must be consistent regardless of the server's locale configuration, especially for user-facing sorted lists.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `VARCHAR(n)` silently truncates an over-length value on `INSERT` | It raises a hard error (`value too long for type character varying(n)`) instead — silent truncation only happens on an explicit `CAST`. |
| Believing `CHAR(n)` is faster than `VARCHAR`/`TEXT` in PostgreSQL | All three share the same internal variable-length storage representation; `CHAR(n)`'s padding/stripping logic makes it, if anything, marginally more work, not less. |
| Comparing `CHAR(n)` values in application code without accounting for padding | PostgreSQL disregards trailing spaces when comparing `character` values internally, but naive string equality in other contexts (e.g., exporting raw bytes) may not, leading to mismatches. |
| Declaring `VARCHAR(255)` everywhere out of habit | This is a superstition carried over from other systems; in PostgreSQL it provides no storage or performance benefit over `TEXT`, and can force an inconvenient schema change if a longer value legitimately shows up. |
| Treating a declared length limit as sufficient data validation | A length limit only bounds size — it says nothing about format, character set, or business meaning; real validation usually needs a `CHECK` constraint (Module 05) alongside it. |

## Interview Questions

1. **Q: What is the practical difference between `CHAR(n)`, `VARCHAR(n)`, and `TEXT` in PostgreSQL?**
   A: `CHAR(n)` stores a value padded with trailing spaces to exactly `n` characters, rejecting anything longer; `VARCHAR(n)` stores exactly the given characters up to a maximum of `n`, rejecting anything longer with no padding; `TEXT` stores a string of any length with no declared maximum. Internally, PostgreSQL stores all three using the same variable-length representation, so in PostgreSQL specifically there is no meaningful storage or speed difference between `VARCHAR` and `TEXT` — the only real difference is whether a length constraint is enforced.

2. **Q: Does inserting a string longer than a column's `VARCHAR(n)` limit truncate it?**
   A: No — a plain `INSERT` or `UPDATE` against a `VARCHAR(n)` column raises an error if the value exceeds `n` characters. Truncation only happens when the value is explicitly cast, e.g. `'toolongvalue'::VARCHAR(5)`, which is a deliberate instruction to force the value into that shape rather than an implicit conversion.

3. **Q: Why does PostgreSQL's documentation recommend `TEXT` over `CHAR(n)` in most cases?**
   A: Because PostgreSQL gets no performance or storage benefit from `CHAR(n)`'s fixed-width model — all character types share the same underlying variable-length storage — while `CHAR(n)` adds the extra behavior of padding values with trailing spaces and stripping that padding on comparison/conversion, which is a frequent source of subtle bugs with no compensating benefit in this particular database engine.

4. **Q: What is collation, and why does it matter for string columns?**
   A: Collation is the set of rules a database uses to compare and sort text — determining alphabetical order, case sensitivity, and how accented or non-ASCII characters rank. Different collations can produce different sort orders and comparison results for the same two strings (for example, whether uppercase letters sort before or interleave with lowercase ones), so it matters whenever consistent, correct sorting or comparison of text is required, especially across different locales or languages.

## Summary

- `CHAR(n)` is fixed-length and blank-padded; `VARCHAR(n)` is variable-length with an enforced maximum; `TEXT` is variable-length with no maximum at all.
- PostgreSQL stores all three using the same internal variable-length representation, so there is no meaningful performance or storage advantage to `CHAR(n)` in this database, unlike in some other systems.
- A plain `INSERT`/`UPDATE` exceeding a declared length limit always raises an error; silent truncation only happens via an explicit `CAST`.
- Collation determines how PostgreSQL compares and sorts text; the default is inherited from locale settings, and can be overridden with `COLLATE` when consistent behavior across locales matters.
- Default to `TEXT` unless you have a genuine, fixed business rule for a maximum length — and prefer a `CHECK` constraint over a bare length limit when the rule is more nuanced than "no longer than N characters."
