# String Functions

## Learning Objectives

By the end of this section you should be able to:
- Concatenate values using both `CONCAT()` and the `||` operator, and explain how they differ when a `NULL` is involved
- Measure, case-convert, and trim string values
- Extract a portion of a string with `SUBSTRING`, replace text with `REPLACE`, and split a delimited string with `SPLIT_PART`
- Locate a substring's position with `POSITION` and `STRPOS`
- Recognize why position-based string functions can silently break on inconsistently formatted data

## Prerequisites

This is the first topic in Module 08, so there is no prior topic within this module to depend on. From earlier in the course, this topic assumes Module 7 — Querying Basics (you'll be calling every function shown here from inside a `SELECT` list or a `WHERE` condition, exactly as covered there) and Module 3 — Data Types (specifically, a solid grasp of the `TEXT`/`VARCHAR`/`CHAR` types these functions operate on). It's also worth having read [Your First Query](../01-introduction/05-your-first-query.md), which first introduced choosing specific columns in a `SELECT` list — the same projection mechanics apply once those columns are wrapped in a function call.

## Motivation

Stored text is almost never in exactly the shape you need to display it, compare it, or split it apart. A customer's name might have stray leading whitespace from a web form. An email address might be stored in mixed case even though comparisons should be case-insensitive. A single "full name" column might need to become "first name" and "last name" for a report. None of this requires changing how the data is stored — it requires transforming it *at query time*, which is exactly what string functions are for.

## Problem Statement

Suppose you're handed a `contacts` table populated by a web form with no input validation:

```sql
CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    full_name TEXT,
    email TEXT,
    phone TEXT
);

INSERT INTO contacts (full_name, email, phone) VALUES
    ('  Asha Verma ', 'ASHA.VERMA@EXAMPLE.COM', '+1-415-555-0134'),
    ('Ben Okafor',    'ben.okafor@example.com', '+1-415-555-0198'),
    ('Chen Wei',      'chen.wei@example.com',   '+86-10-5550-0122');
```

A plain `SELECT * FROM contacts;` hands you back exactly what was stored — including the stray spaces around `'  Asha Verma '` and the shouting-case email. If you need a clean display name, a lowercase email for a reliable comparison, or just the username portion before the `@`, you have two choices: fix it in whatever application code reads this data, every single place it's read, or fix it once, at the query, using string functions. SQL's string functions exist so you never have to do the former.

## Concept

### `CONCAT()` and the `||` Operator

Both join multiple values into one string. PostgreSQL supports both the ANSI-standard `||` operator and the `CONCAT()` function, and they behave *differently* around `NULL`.

```sql
SELECT full_name || ' <' || email || '>' AS via_operator FROM contacts;
```

```
             via_operator
---------------------------------------
   Asha Verma  <ASHA.VERMA@EXAMPLE.COM>
 Ben Okafor <ben.okafor@example.com>
 Chen Wei <chen.wei@example.com>
(3 rows)
```

```sql
SELECT CONCAT(full_name, ' <', email, '>') AS via_concat FROM contacts;
```

The output here is identical for these three rows because none of the values are `NULL`. The difference only appears once a `NULL` is involved:

```sql
SELECT '  Hello' || NULL || 'World';
```

```
 ?column?
----------

(1 row)
```

The entire expression becomes `NULL` — this is standard SQL behavior: any operator that touches a `NULL` operand produces `NULL` (Module 3 covers why `NULL` propagates this way). `CONCAT()`, however, is a PostgreSQL convenience function that treats `NULL` arguments as empty strings instead of poisoning the whole result:

```sql
SELECT CONCAT('  Hello', NULL, 'World');
```

```
   concat
------------
   HelloWorld
(1 row)
```

This single difference is a common source of "why did my whole concatenated column go blank?" bugs — covered again in Common Mistakes below.

### `LENGTH()` — Measuring a String

`LENGTH(string)` returns the number of *characters* in a string.

```sql
SELECT full_name, LENGTH(full_name) FROM contacts;
```

```
   full_name    | length
-----------------+--------
   Asha Verma    |     13
 Ben Okafor       |     10
 Chen Wei         |      8
(3 rows)
```

Notice Asha's row is 13, not 10 — `'  Asha Verma '` includes two leading spaces and one trailing space, all of which count. This is precisely why `LENGTH()` is often paired with `TRIM()` (next) before being trusted for anything meaningful, like a "name must be under 50 characters" check.

### `UPPER()` / `LOWER()` — Case Conversion

```sql
SELECT email, LOWER(email) AS normalized_email FROM contacts;
```

```
          email          | normalized_email
--------------------------+-------------------------
 ASHA.VERMA@EXAMPLE.COM   | asha.verma@example.com
 ben.okafor@example.com   | ben.okafor@example.com
 chen.wei@example.com     | chen.wei@example.com
(3 rows)
```

This matters more than it looks: if you compared `email = 'asha.verma@example.com'` directly against the stored value, Asha's row would **not** match, because SQL string equality is case-sensitive by default. Wrapping both sides in `LOWER()` (`LOWER(email) = LOWER('Asha.Verma@Example.com')`) makes the comparison case-insensitive regardless of how either value happens to be cased.

### `TRIM()` — Removing Leading/Trailing Characters

```sql
SELECT full_name, TRIM(full_name) AS trimmed, LENGTH(TRIM(full_name)) FROM contacts;
```

```
   full_name    |  trimmed   | length
-----------------+------------+--------
   Asha Verma    | Asha Verma |     10
 Ben Okafor       | Ben Okafor |     10
 Chen Wei         | Chen Wei   |      8
(3 rows)
```

`TRIM(string)` removes leading and trailing whitespace by default. It does **not** touch whitespace in the middle of a string — `TRIM('  Asha  Verma  ')` still leaves the double space between "Asha" and "Verma" untouched, because `TRIM` only strips from the two ends. PostgreSQL also supports a fuller standard-SQL form for trimming characters other than spaces:

```sql
SELECT TRIM(BOTH 'x' FROM 'xxHelloxx');    -- 'Hello'
SELECT TRIM(LEADING '0' FROM '00042');     -- '42'
SELECT TRIM(TRAILING '.' FROM 'file.txt.'); -- 'file.txt'
```

PostgreSQL also offers `LTRIM(string, characters)` and `RTRIM(string, characters)` as shorthand for `TRIM(LEADING ...)` and `TRIM(TRAILING ...)` respectively.

### `SUBSTRING()` — Extracting Part of a String

`SUBSTRING(string FROM start FOR length)` extracts a piece of a string. Positions are **1-indexed** (the first character is position 1, not 0).

```sql
SELECT phone, SUBSTRING(phone FROM 4 FOR 3) AS area_code FROM contacts;
```

```
       phone       | area_code
--------------------+-----------
 +1-415-555-0134    | 415
 +1-415-555-0198    | 415
 +86-10-5550-0122    | 10-
(3 rows)
```

Look closely at Chen Wei's row: extracting "characters 4 through 6" gives `'10-'`, not an area code, because that phone number uses an international format (`+86-10-...`) with a different structure than the US-formatted numbers above it. This is a deliberately included trap — see Common Mistakes below for why fixed-position extraction like this is fragile in real data.

PostgreSQL also accepts the more familiar function-call form: `SUBSTRING(phone, 4, 3)` is equivalent to `SUBSTRING(phone FROM 4 FOR 3)`. If `FOR length` is omitted, `SUBSTRING` extracts from the start position to the end of the string.

### `REPLACE()` — Substituting Text

```sql
SELECT phone, REPLACE(phone, '-', '') AS digits_only FROM contacts;
```

```
       phone       | digits_only
--------------------+---------------
 +1-415-555-0134    | +14155550134
 +1-415-555-0198    | +14155550198
 +86-10-5550-0122   | +861055500122
(3 rows)
```

`REPLACE(string, from, to)` replaces **every** occurrence of `from` with `to` — not just the first one. This is a common assumption mismatch for anyone coming from a language where "replace" defaults to the first match only; in SQL, if you want to replace only one occurrence you need more targeted logic (typically `SUBSTRING` plus concatenation, or a regular-expression function, which is out of scope for this topic).

### `SPLIT_PART()` — Splitting a Delimited String

```sql
SELECT email,
       SPLIT_PART(email, '@', 1) AS username,
       SPLIT_PART(email, '@', 2) AS domain
FROM contacts;
```

```
          email          | username  |    domain
--------------------------+-----------+---------------
 ASHA.VERMA@EXAMPLE.COM   | ASHA.VERMA| EXAMPLE.COM
 ben.okafor@example.com   | ben.okafor| example.com
 chen.wei@example.com     | chen.wei  | example.com
(3 rows)
```

`SPLIT_PART(string, delimiter, field_number)` splits `string` on every occurrence of `delimiter` and returns the `field_number`-th piece (1-indexed). If the delimiter never appears in the string, field 1 returns the whole original string and any field number beyond 1 returns an empty string — it does not error.

### `POSITION()` and `STRPOS()` — Locating a Substring

Both return the 1-indexed position of the first occurrence of a substring, or `0` if it isn't found at all. They differ only in syntax:

```sql
SELECT email, POSITION('@' IN email) AS at_position FROM contacts;
SELECT email, STRPOS(email, '@')     AS at_position FROM contacts;
```

```
          email          | at_position
--------------------------+-------------
 ASHA.VERMA@EXAMPLE.COM   |          12
 ben.okafor@example.com   |          11
 chen.wei@example.com     |           9
(3 rows)
```

`POSITION(substring IN string)` uses the SQL-standard infix syntax; `STRPOS(string, substring)` is PostgreSQL's function-call equivalent with the arguments in the opposite order. Both are commonly used as a cheap existence check (`POSITION('@' IN email) > 0`) when you only care *whether* a substring exists, without needing full pattern-matching.

### Putting It Together

A realistic cleanup query combining several of the above, to produce presentable output from the raw table:

```sql
SELECT
    TRIM(full_name)                              AS clean_name,
    LOWER(email)                                  AS normalized_email,
    SPLIT_PART(LOWER(email), '@', 2)              AS email_domain,
    REPLACE(phone, '-', '')                       AS phone_digits
FROM contacts
ORDER BY clean_name;
```

```
 clean_name  |    normalized_email     | email_domain |  phone_digits
-------------+--------------------------+---------------+---------------
 Asha Verma  | asha.verma@example.com  | example.com   | +14155550134
 Ben Okafor  | ben.okafor@example.com  | example.com   | +14155550198
 Chen Wei    | chen.wei@example.com    | example.com   | +861055500122
(3 rows)
```

Nothing about the underlying table changed — every transformation happened at query time, on the fly.

## Internal Working (Preview)

String functions run during the **projection/evaluation** phase of query execution — after rows have been located (and, if a `WHERE` clause references a function, after or during row filtering), the database computes each function's result per row, using its internal, highly-optimized (typically C-language) implementation. Text in PostgreSQL is stored and compared according to a configured **collation** (the rules that define character ordering and case) and character **encoding** (almost always UTF-8 by default) — `LENGTH()` counts characters according to that encoding, not raw bytes, so a string containing multi-byte characters still reports its character count correctly, not its byte count (a separate function, `OCTET_LENGTH()`, reports bytes if you ever need that instead).

```
 Row source  ──▶  WHERE filter (may evaluate functions here)  ──▶  Column list evaluated
 (table scan                                                       (functions computed
  or index)                                                        per surviving row)
```

Critically, none of this changes what's stored on disk. `UPPER(email)` computes an uppercase version of the value *for that query's result set only* — the stored row is untouched unless you explicitly run an `UPDATE` (Module 6) to overwrite it.

## Real-World Analogy

Think of string functions like the "Find & Replace" and "Text to Columns" tools in spreadsheet software, or a copy editor marking up a manuscript. The copy editor doesn't rewrite the author's original notebook — they produce a *cleaned-up version* for publication: trimming stray marks (`TRIM`), standardizing capitalization per house style (`UPPER`/`LOWER`), splitting a run-on sentence into two columns of a table (`SPLIT_PART`), and swapping one word for another throughout a whole document (`REPLACE`). The original manuscript (your stored row) stays exactly as it was; only the presented, edited copy (the query result) reflects the changes.

## Why String Functions Were Designed This Way

The relational model favors storing data in its most atomic, unambiguous form (this is a preview of *normalization*, covered fully in Module 15) rather than pre-splitting or pre-formatting it into every shape some future report might need. An email address is naturally one value, not two pre-split "username" and "domain" columns — splitting it is a *presentation* concern, not a *storage* concern. Because SQL is declarative (Module 1, Topic 2), you describe the transformation you want (`SPLIT_PART(email, '@', 1)`) and the engine computes it on demand, for exactly the rows and columns you asked for, without you ever having to denormalize the table just to make a report easier to write.

## Advantages

- **No redundant storage.** You don't need a separate `username` column just in case some report needs it — it's computed on the fly from the one true `email` column, so there's no risk of the derived column drifting out of sync with the source.
- **Centralized, consistent logic.** Every query that needs a normalized email uses the same `LOWER(email)` expression, rather than each application reimplementing (and potentially getting slightly wrong) its own normalization logic.
- **Performance-optimized by the engine.** String functions are implemented in the database's native code, generally faster than reading raw rows into an application and manipulating text there.

## Disadvantages / Limitations

- **Not automatically indexable.** An index on `email` doesn't speed up a `WHERE LOWER(email) = ...` search — the database can't use a plain index when the *column* is wrapped in a function, because the index stores the original values, not the transformed ones (PostgreSQL's expression/functional indexes, covered in Module 13, solve this, but they must be created deliberately).
- **Fragile against inconsistent data.** As Chen Wei's phone number demonstrated, position-based extraction (`SUBSTRING(phone FROM 4 FOR 3)`) silently produces garbage when the input format isn't perfectly uniform — it doesn't error, it just gives you a wrong answer that looks plausible.
- **Locale/collation sensitivity.** Case conversion and string comparison behavior can vary subtly depending on the database's configured collation (for example, how accented characters sort or compare) — usually a non-issue, but worth knowing it exists if you ever see unexpected case-insensitive comparison results.

## Best Practices

- Normalize at the boundary when a value will *always* be compared case-insensitively (e.g., store emails already lowercased on insert) rather than wrapping every future query in `LOWER()` — this also makes an index on the column directly useful again.
- When you must query using a function repeatedly on a large table, create an expression index (Module 13) rather than accepting a full table scan on every query.
- Prefer `POSITION`/`STRPOS` for a simple "does this substring exist" check; reach for pattern-matching tools (`LIKE`, regular expressions — outside this topic's scope) only when you need actual wildcards, not fixed substrings.
- Don't rely on fixed character positions (`SUBSTRING(x FROM 4 FOR 3)`) to parse structured data like phone numbers or codes unless you've verified every row genuinely follows the same fixed format.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `||` and `CONCAT()` behave identically with `NULL` | `||` returns `NULL` if *any* operand is `NULL`, silently blanking the entire expression; `CONCAT()` treats `NULL` arguments as empty strings and keeps the rest of the result intact. Picking the wrong one causes reports to go unexpectedly blank. |
| Assuming `TRIM()` removes all whitespace, including in the middle of a string | `TRIM()` only strips characters from the two *ends* of a string. `'a  b'` stays `'a  b'` after `TRIM()` — the double space in the middle is untouched. |
| Assuming `REPLACE()` only replaces the first match | PostgreSQL's `REPLACE()` replaces **every** occurrence of the target substring, which surprises people used to languages where "replace" defaults to the first match only. |
| Trusting fixed-position `SUBSTRING` on real-world data without verifying uniform formatting | As shown with Chen Wei's international phone number, a format that varies row-to-row causes `SUBSTRING` to return technically-valid-looking but semantically wrong output, with no error raised. |

## Interview Questions

1. **Q: What's the difference between `||` and `CONCAT()` in PostgreSQL, specifically around `NULL`?**
   A: `||` is the standard SQL concatenation operator; if any operand is `NULL`, the entire result is `NULL`. `CONCAT()` is a PostgreSQL function that treats `NULL` arguments as empty strings, so the rest of the concatenation still comes through. Use `CONCAT()` (or wrap operands in `COALESCE`) when you need concatenation to be resilient to missing values.

2. **Q: You need to split a `'lastname,firstname'` column into two separate output columns. What function would you use, and what's a risk to watch for?**
   A: `SPLIT_PART(column, ',', 1)` and `SPLIT_PART(column, ',', 2)`. The risk is inconsistent input — if some rows don't contain a comma at all, field 1 silently returns the whole string and field 2 returns an empty string rather than raising an error, so the output can look plausible while being wrong for those rows.

3. **Q: Why doesn't a plain B-tree index on a `TEXT` column speed up a query filtering with `WHERE LOWER(column) = 'x'`?**
   A: The index stores the column's original, unmodified values. Wrapping the column in `LOWER()` in the `WHERE` clause means the database must compute `LOWER()` for every row before it can compare, which the plain index can't do — it would need a functional/expression index built specifically on `LOWER(column)` (Module 13) to be usable here.

## Summary

- `||` and `CONCAT()` both join strings, but differ critically on `NULL`: `||` propagates `NULL`, `CONCAT()` ignores it.
- `LENGTH()` counts characters (including whitespace) exactly as stored; `TRIM()` strips leading/trailing whitespace only, never the middle of a string.
- `UPPER()`/`LOWER()` normalize case, most commonly to make comparisons case-insensitive.
- `SUBSTRING`, `REPLACE`, `SPLIT_PART`, and `POSITION`/`STRPOS` extract, replace, split, and locate text respectively — all are computed at query time and never alter the stored row.
- Position-based extraction is only as reliable as the uniformity of your underlying data — always verify formatting consistency before trusting fixed offsets.
- None of these functions can be used efficiently by a plain index unless you build a dedicated expression index (Module 13) for the exact expression you query by.
