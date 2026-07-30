# MySQL Differences

## Learning Objectives

By the end of this section you should be able to:
- Translate a PostgreSQL `SERIAL`/`GENERATED ALWAYS AS IDENTITY` column into MySQL's `AUTO_INCREMENT` and explain the mechanical difference between them
- Explain why MySQL has historically had no native `FULL OUTER JOIN`, and rewrite one as a `UNION` of a `LEFT JOIN` and a `RIGHT JOIN`
- Compare `LIMIT` syntax between PostgreSQL and MySQL and identify where they agree and where they subtly differ
- Explain MySQL's default string-comparison case sensitivity and how it differs from PostgreSQL's default
- Contrast backtick identifier quoting with PostgreSQL's double-quote quoting, and translate `ON DUPLICATE KEY UPDATE` upserts to and from `ON CONFLICT`
- Describe, at a high level, what a MySQL storage engine is and why InnoDB is the default

## Prerequisites

- [Module 22 Overview](00-module-overview.md) — establishes that this entire module compares other databases against the PostgreSQL baseline built in Modules 1–21.
- [Primary Keys](../05-constraints-and-keys/03-primary-keys.md) — you need to already know what `SERIAL` and `GENERATED ALWAYS AS IDENTITY` do in PostgreSQL before comparing them to MySQL's `AUTO_INCREMENT`.
- [INSERT](../06-modifying-data/01-insert.md) — PostgreSQL's `INSERT ... ON CONFLICT` upsert is the baseline this topic translates into MySQL's `ON DUPLICATE KEY UPDATE`.
- [FULL OUTER JOIN and CROSS JOIN](../10-joins-and-set-operations/03-full-outer-join-and-cross-join.md) — you need to know precisely what a `FULL OUTER JOIN` returns before you can appreciate why its absence in MySQL is a real gap, not a cosmetic one.

## Motivation

MySQL is, by installed base, one of the most widely deployed relational databases in the world — it powers the majority of the traditional "LAMP stack" web, is the default choice behind countless content management systems, and is one of the two databases (alongside PostgreSQL) you are most likely to be asked about by name in a technical interview. If your entire mental model of SQL was built exclusively on PostgreSQL's exact syntax, sitting down at a MySQL prompt for the first time will produce a string of confusing errors for reasons that have nothing to do with your SQL knowledge and everything to do with a handful of concrete, learnable syntax and default-behavior differences.

## Problem Statement

Imagine you are asked, in an interview or on the job, to take a PostgreSQL script you wrote for this course and run it against a MySQL database instead:

```sql
CREATE TABLE customers (
    id    SERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL
);

INSERT INTO customers (email) VALUES ('asha@example.com')
ON CONFLICT (email) DO UPDATE SET email = EXCLUDED.email;

SELECT c.name, o.order_id
FROM customers c
FULL OUTER JOIN orders o ON o.customer_id = c.id;
```

Running this against MySQL fails on all three statements: `SERIAL` is not a MySQL type, `ON CONFLICT` is not MySQL syntax, and `FULL OUTER JOIN` does not exist in MySQL at all. None of these are advanced features — they are exactly the kind of everyday statements you have written dozens of times in this course. Without knowing MySQL's equivalents, you would have to rediscover each one by trial and error under time pressure. This topic hands you the translation table up front.

## Concept

### Auto-Increment Columns: `AUTO_INCREMENT` vs. `SERIAL` / `IDENTITY`

In PostgreSQL, an auto-incrementing integer column is declared with `SERIAL` (a shorthand that creates a backing sequence object, covered in [Primary Keys](../05-constraints-and-keys/03-primary-keys.md)) or, in modern PostgreSQL style, `GENERATED ALWAYS AS IDENTITY`. MySQL uses a column attribute, `AUTO_INCREMENT`, directly on an integer type:

```sql
-- PostgreSQL
CREATE TABLE customers (
    id    SERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL
);

-- MySQL
CREATE TABLE customers (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

| Aspect | PostgreSQL (`SERIAL`) | MySQL (`AUTO_INCREMENT`) |
|---|---|---|
| Mechanism | Creates a genuine separate **sequence object** in the database, and wires the column's default to `nextval()` on it | A counter attribute stored as part of the table's own metadata — not a separate database object you can query independently |
| Can you have more than one per table? | Only meaningfully useful on one column, but nothing stops you from wiring multiple columns to separate sequences | Exactly one `AUTO_INCREMENT` column per table, and it must be a key (usually the primary key) |
| Resetting the counter | `ALTER SEQUENCE customers_id_seq RESTART WITH 1;` | `ALTER TABLE customers AUTO_INCREMENT = 1;` |
| Gaps after a rolled-back transaction | Yes — a sequence value consumed by a rolled-back `INSERT` is never reused, exactly like PostgreSQL's `SERIAL` (see [Primary Keys](../05-constraints-and-keys/03-primary-keys.md)) | Yes, with `InnoDB` — behaves the same way in this respect |
| Underlying concept | "A separate, independently-managed number generator the column's default happens to call" | "A property of the column itself" |

The practical takeaway: the *concept* — a database-generated, unique, ever-increasing integer you never supply manually — is identical. Only the mechanism and the exact keyword differ, and that pattern ("same concept, different keyword and internal mechanism") repeats throughout this entire module.

### The Missing `FULL OUTER JOIN`

PostgreSQL supports `FULL OUTER JOIN` natively (see [Module 10](../10-joins-and-set-operations/03-full-outer-join-and-cross-join.md)) — it returns every row from both tables, with `NULL`s filling in wherever a match is missing on either side. MySQL, as of the versions in common production use, has **no `FULL OUTER JOIN` keyword at all**. Attempting one produces a syntax error:

```
ERROR 1064 (42000): You have an error in your SQL syntax; ... near 'FULL OUTER JOIN orders o ON o.customer_id = c.id' at line 1
```

The standard workaround rebuilds the same result by combining a `LEFT JOIN` and a `RIGHT JOIN` with `UNION`, since a full outer join is logically equivalent to "everything the left join produces, plus whatever the right join produces that the left join didn't":

```sql
-- MySQL workaround for FULL OUTER JOIN
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id

UNION

SELECT c.name, o.order_id
FROM customers c
RIGHT JOIN orders o ON o.customer_id = c.id;
```

```
  name  | order_id
--------+----------
 Asha   |      101
 Asha   |      104
 Ben    |     NULL
 NULL   |      109
(4 rows)
```

`UNION` (not `UNION ALL`) is essential here — it deduplicates the rows that both the `LEFT JOIN` and the `RIGHT JOIN` produce in common (every row where both sides genuinely matched), which is exactly the overlap you'd otherwise double-count. This is a direct, practical application of the set-operation knowledge from [Module 10](../10-joins-and-set-operations/03-full-outer-join-and-cross-join.md).

### `LIMIT`: Mostly the Same, With a Twist

PostgreSQL and MySQL both support the same core `LIMIT` syntax:

```sql
-- Identical in both PostgreSQL and MySQL
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 5;
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 5 OFFSET 10;
```

MySQL additionally supports a comma shorthand PostgreSQL does not have: `LIMIT offset, row_count`.

```sql
-- MySQL-only shorthand — equivalent to LIMIT 5 OFFSET 10
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 10, 5;
```

The two numbers are easy to transpose by accident (is it "skip 10, take 5" or "skip 5, take 10"?), which is precisely why this course's examples always use the unambiguous `LIMIT n OFFSET m` form — it also happens to be the form that is portable to MySQL, since MySQL accepts both, while PostgreSQL only accepts the `OFFSET` form.

### String Comparison and Case Sensitivity

By default, PostgreSQL's text comparisons are **case-sensitive**: `'Asha' = 'asha'` evaluates to `false`. MySQL's default **collation** (the rules that govern how strings are compared and sorted) is commonly a case-*insensitive* one (historically `utf8_general_ci` or `utf8mb4_general_ci` — the trailing `ci` stands for "case-insensitive"), so the same comparison behaves differently:

```sql
-- PostgreSQL (default collation)
SELECT 'Asha' = 'asha';
```
```
 ?column?
----------
 f
(1 row)
```

```sql
-- MySQL (typical default collation)
SELECT 'Asha' = 'asha';
```
```
+-----------------+
| 'Asha' = 'asha' |
+-----------------+
|               1 |
+-----------------+
```

This is a genuinely dangerous gap to be unaware of: a `WHERE email = 'Asha@Example.com'` filter that PostgreSQL treats as an exact, case-sensitive match may silently match differently-cased rows on a MySQL database using a case-insensitive collation, and vice versa if you deliberately choose a case-sensitive one (`utf8mb4_bin` or `_cs` collations). Collations are configurable per column, table, and database in MySQL — the point here is only that the *default* differs from PostgreSQL's default, so code that "just works" on one can behave subtly differently on the other if you never checked.

### Identifier Quoting: Backticks vs. Double Quotes

PostgreSQL quotes unusually-named identifiers (table/column names with reserved words, spaces, or mixed case that must be preserved) with double quotes, as covered in [Your First Query](../01-introduction/05-your-first-query.md). MySQL uses **backticks** for the same purpose:

```sql
-- PostgreSQL
SELECT "order", "Customer Name" FROM "Orders";

-- MySQL
SELECT `order`, `Customer Name` FROM `Orders`;
```

MySQL can be configured (via the `ANSI_QUOTES` SQL mode) to accept double quotes for identifiers instead — but in its default mode, double quotes in MySQL are just an alternate way to write a *string literal*, identical to single quotes. This is the exact opposite of PostgreSQL's rule (single quotes for literals, double quotes for identifiers, no exceptions), and mixing the two up when moving between the databases is one of the most common sources of confusing, hard-to-spot bugs.

### Upserts: `ON DUPLICATE KEY UPDATE` vs. `ON CONFLICT`

PostgreSQL's upsert syntax, `INSERT ... ON CONFLICT`, is covered in Module 6. MySQL solves the same "insert this, or update it if a conflicting unique/primary key already exists" problem with `ON DUPLICATE KEY UPDATE`:

```sql
-- PostgreSQL
INSERT INTO customers (id, email)
VALUES (1, 'asha@example.com')
ON CONFLICT (id) DO UPDATE SET email = EXCLUDED.email;

-- MySQL
INSERT INTO customers (id, email)
VALUES (1, 'asha@example.com')
ON DUPLICATE KEY UPDATE email = VALUES(email);
```

| Aspect | PostgreSQL `ON CONFLICT` | MySQL `ON DUPLICATE KEY UPDATE` |
|---|---|---|
| Which constraint triggers it | You name the specific conflicting column/constraint (`ON CONFLICT (id)`) | Any unique key or primary key violation on the row triggers it — you don't name which one |
| Referring to the rejected row's incoming values | `EXCLUDED.column_name` | `VALUES(column_name)` |
| Doing nothing on conflict | `ON CONFLICT (id) DO NOTHING` | No direct equivalent — you'd `UPDATE` a column to its own current value as a no-op, or handle it at the application layer |

The concept — "avoid a race between checking whether a row exists and then deciding whether to insert or update it" — is identical in both databases; only the clause name and the way you reference the incoming row's values differ.

### Storage Engines: InnoDB, Briefly

Uniquely among the databases in this module, MySQL has a **pluggable storage engine** architecture — the same `CREATE TABLE` syntax can be backed by different underlying storage implementations, chosen per table with an `ENGINE = ...` clause. **InnoDB** has been the default engine for many years and is what virtually all production MySQL tables use: it supports transactions (Module 14), foreign keys (Module 5), and row-level locking. An older engine, **MyISAM**, predates InnoDB and lacks all three of those — it is still occasionally encountered in legacy schemas but should not be chosen for new tables. PostgreSQL has no equivalent concept: it has a single storage engine, and this "which engine is this table using?" question simply never arises.

```sql
CREATE TABLE customers (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
) ENGINE = InnoDB;
```

## Internal Working

The `FULL OUTER JOIN` workaround is a useful window into how a query planner actually executes a join. Conceptually:

```
 LEFT JOIN customers → orders     RIGHT JOIN customers → orders
 (every customer,                 (every order,
  matching order or NULL)          matching customer or NULL)
        │                                  │
        └──────────────┬───────────────────┘
                        ▼
                     UNION
        (rows appearing in both results,
         i.e. genuine matches, are de-duplicated;
         unmatched rows from either side survive once)
                        │
                        ▼
              Full outer join result
```

A database that *does* support `FULL OUTER JOIN` natively (PostgreSQL) executes this same logical result directly in one pass inside its join engine, without materializing two intermediate result sets and de-duplicating them — which is one concrete reason the MySQL workaround, while correct, tends to be less efficient on large tables than a native full outer join would be.

## Real-World Analogy

Comparing PostgreSQL and MySQL is like comparing American and British English. The grammar (standard SQL: `SELECT`, `WHERE`, `JOIN`, `GROUP BY`) is shared and mutually intelligible — a British and an American speaker understand each other's core sentences without translation. But specific vocabulary genuinely differs ("boot" vs. "trunk," "flat" vs. "apartment" — here, `AUTO_INCREMENT` vs. `SERIAL`, backticks vs. double quotes), and a few concepts simply don't map one-to-one (some idioms exist in one dialect with no exact equivalent in the other, much like MySQL having no direct `FULL OUTER JOIN` keyword at all). Fluency in one dialect gets you most of the way in the other; it does not make the differences disappear.

## Why These Differences Exist

MySQL and PostgreSQL grew out of different priorities. MySQL was built in the mid-1990s with an emphasis on being fast and simple to deploy for web applications, which shaped early decisions like a lightweight, table-metadata-based auto-increment counter rather than a fully independent sequence object, and a pluggable storage engine model that let it prioritize either speed (MyISAM, historically) or transactional safety (InnoDB) per table. PostgreSQL, descended more directly from academic research into the relational model, has historically prioritized standards compliance and correctness — a genuine sequence object, strict default case sensitivity matching the SQL standard's literal comparison semantics, and a single, deeply transactional storage layer. Neither approach is objectively "wrong" — they reflect different trade-offs made over decades, both of which happened to succeed enormously in different corners of the industry (MySQL in web-hosting and open-source CMS platforms, PostgreSQL in applications valuing strict correctness and advanced SQL features).

## Advantages

- **Recognizing "same concept, different keyword" makes new databases far less intimidating** — nearly every apparent MySQL "gotcha" in this topic is a renamed or slightly-reshaped version of something you already understand deeply from PostgreSQL.
- **Knowing the `FULL OUTER JOIN` workaround deepens your understanding of joins generally** — building it by hand from `LEFT JOIN`, `RIGHT JOIN`, and `UNION` forces you to internalize what a full outer join actually *means*, rather than treating it as an opaque keyword.
- **Awareness of collation defaults prevents a genuine, hard-to-diagnose class of bug** — case-sensitivity mismatches between environments are notorious for passing all tests in one setup and silently misbehaving in another.

## Disadvantages / Limitations

- **The workarounds are not free** — the `UNION`-based full outer join, in particular, typically costs more (two scans plus a deduplication pass) than a database with native support for the same query.
- **Collation and engine choices are configurable, not fixed** — everything described here as a MySQL "default" can be changed per database, table, or column, so any specific MySQL instance you encounter in the wild may have been configured away from what's shown here; always verify rather than assume.
- **This topic does not cover every MySQL divergence** — it deliberately focuses on the differences most likely to bite someone moving from PostgreSQL; MySQL has many smaller behavioral quirks (its historically looser handling of invalid dates, differences in `GROUP BY` strictness across versions, and others) that are out of scope here.

## Best Practices

- When you must write a `FULL OUTER JOIN`-shaped query against MySQL, wrap the `UNION` workaround in a view so the awkward two-query structure is written once and reused, rather than duplicated at every call site.
- If case-sensitive comparisons matter to your application (e.g., comparing tokens, hashes, or case-meaningful codes), explicitly pin the collation (`COLLATE utf8mb4_bin` in MySQL) rather than relying on whatever the server's default happens to be.
- When reading an unfamiliar MySQL schema, run `SHOW CREATE TABLE table_name;` early — it reveals the storage engine, auto-increment counter's current value, and any explicit collation overrides in one place.
- Prefer the explicit `LIMIT n OFFSET m` form over MySQL's `LIMIT m, n` comma shorthand even when writing MySQL-only code — it reads unambiguously and happens to also be valid PostgreSQL.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `FULL OUTER JOIN` against MySQL and assuming it's just "not implemented yet" in a specific version | It is a structural gap in MySQL as a product, not a version limitation — the `LEFT JOIN`/`RIGHT JOIN`/`UNION` workaround is the standard, permanent solution. |
| Assuming string equality behaves identically across a PostgreSQL and a MySQL environment without checking collations | MySQL's common default collations are case-insensitive while PostgreSQL's default comparison is case-sensitive — identical-looking `WHERE` clauses can match different sets of rows. |
| Using double quotes for a string literal out of habit in MySQL (e.g., `WHERE name = "Asha"`) and having it work, then carrying that habit back into PostgreSQL | It works in MySQL's default mode only because MySQL treats double quotes as equivalent to single quotes there; PostgreSQL enforces the standard rule strictly (double quotes are identifiers only) and will throw an error or, worse, silently reference an unintended identifier. |
| Assuming `ON DUPLICATE KEY UPDATE` and `ON CONFLICT` are interchangeable syntax with a find-and-replace | The clauses differ in how you name the conflicting key (or don't, in MySQL's case) and how you reference the incoming row's values (`EXCLUDED.col` vs. `VALUES(col)`) — a literal keyword swap will not run. |

## Interview Questions

1. **Q: Why does MySQL not support `FULL OUTER JOIN`, and how do you get the same result?**
   A: MySQL simply does not implement the `FULL OUTER JOIN` keyword. The equivalent result is built by taking a `UNION` of a `LEFT JOIN` (every row from the left table, matched or not) and a `RIGHT JOIN` (every row from the right table, matched or not) between the same two tables — `UNION` deduplicates the rows that both queries return in common, leaving exactly the full outer join's result.

2. **Q: What is the practical difference between PostgreSQL's `SERIAL` and MySQL's `AUTO_INCREMENT`?**
   A: `SERIAL` is PostgreSQL shorthand that creates a genuinely separate sequence database object and wires the column's default value to it. `AUTO_INCREMENT` is a MySQL column attribute stored as part of the table's own metadata rather than a separate object. Both produce the same practical behavior — an automatically assigned, ever-increasing unique integer — through different internal mechanisms.

3. **Q: A query that compares two strings for equality returns different results on a PostgreSQL server and a MySQL server, with identical data. What is the most likely cause, and how would you fix it?**
   A: The most likely cause is a difference in default string collation — PostgreSQL's default comparisons are case-sensitive, while many common MySQL default collations are case-insensitive. The fix is to make the comparison behavior explicit rather than relying on defaults: pin a specific collation on the MySQL side (e.g. `COLLATE utf8mb4_bin` for case-sensitive comparison), or normalize case explicitly in the query (e.g. with a lowercasing function) on whichever side needs to match the other's behavior.

## Summary

- MySQL's `AUTO_INCREMENT` and PostgreSQL's `SERIAL`/`GENERATED ALWAYS AS IDENTITY` solve the same problem — automatic unique key generation — through different internal mechanisms (a table attribute vs. a separate sequence object).
- MySQL has no native `FULL OUTER JOIN`; the standard workaround is `UNION` of a `LEFT JOIN` and a `RIGHT JOIN` on the same table pair.
- `LIMIT ... OFFSET ...` is shared syntax between the two databases; MySQL additionally accepts a `LIMIT offset, count` shorthand that PostgreSQL does not.
- MySQL's common default collations are case-insensitive for string comparison, unlike PostgreSQL's case-sensitive default — this is a real source of silent behavioral differences, not just cosmetic syntax.
- Identifier quoting uses backticks in MySQL versus double quotes in PostgreSQL, and MySQL's default mode treats double quotes as string literals, the opposite of PostgreSQL's rule.
- `ON DUPLICATE KEY UPDATE` (MySQL) and `ON CONFLICT` (PostgreSQL) both implement upsert semantics but differ in how the conflicting key and the incoming row's values are referenced.
