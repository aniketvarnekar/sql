# Module 07 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **SELECT and Projection** — choosing specific columns vs. `SELECT *`, `AS` aliasing, computed expression columns, and projection as a relational-algebra concept
- [x] **Filtering with WHERE** — `WHERE`'s per-row boolean evaluation, and the fundamental distinction between filtering (rows) and projecting (columns)
- [x] **Comparison and Logical Operators** — `=`, `<>`/`!=`, `<`, `>`, `<=`, `>=`, `AND`/`OR`/`NOT`, operator precedence and parentheses, and how compound conditions behave under three-valued logic
- [x] **Pattern Matching with LIKE** — `LIKE`/`ILIKE`, the `%` and `_` wildcards, escaping literal wildcard characters, and a brief pointer to `~` regex matching
- [x] **BETWEEN, IN, and IS NULL** — inclusive range checks, list/subquery membership, and the correct, NULL-safe way to test for missing values
- [x] **Sorting with ORDER BY** — single/multi-column sorts, `ASC`/`DESC`, position vs. name, `NULLS FIRST`/`NULLS LAST`, and sorting by an unprojected expression
- [x] **LIMIT, OFFSET, and DISTINCT** — pagination, its performance implications at scale, `DISTINCT`/`DISTINCT ON`, and the complete logical processing order of a `SELECT`

## Practical Connections

- **A reporting dashboard querying millions of rows** depends on exactly this module's toolkit working together: `WHERE` narrows the data down before anything else happens, `ORDER BY` presents it meaningfully, and `LIMIT` caps what's actually rendered on screen — without all three working in concert, "top 10 products this month" isn't answerable at all.
- **A search box on a real product listing page** is a direct application of pattern matching (Topic 4) combined with filtering (Topic 2) and pagination (Topic 7) — "find anything containing this text, 20 results per page."
- **A category filter dropdown, populated from real data rather than a hardcoded list**, is exactly what `SELECT DISTINCT category FROM products` answers — and picking a single representative "featured item" per category on a homepage is exactly the kind of problem `DISTINCT ON` was built for.
- **Every silent, hard-to-reproduce bug report along the lines of "this record used to show up and now it doesn't, for no reason"** is disproportionately likely to trace back to one of two things covered in this module: a `NULL` quietly resolving a condition to `UNKNOWN` instead of `TRUE` (Topics 3 and 5), or a missing `ORDER BY` making a `LIMIT`-ed result nondeterministic (Topics 6 and 7).

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Projection vs. filtering | Projection (`SELECT` list) chooses which *columns* appear; filtering (`WHERE`) chooses which *rows* survive. They're independent and freely combinable. |
| `WHERE` referencing a `SELECT` alias vs. `ORDER BY` referencing one | `WHERE` runs before the `SELECT` list is computed and cannot see its aliases; `ORDER BY` runs after and can. |
| `LIKE` vs. `ILIKE` | `LIKE` is case-sensitive and standard SQL; `ILIKE` ignores case and is a PostgreSQL-specific extension. |
| `=`/`<>` vs. `IS NULL`/`IS NOT NULL` | `=`/`<>` against `NULL` always evaluates to `UNKNOWN` and can never match a row; `IS NULL`/`IS NOT NULL` are dedicated predicates that always return a definite `TRUE`/`FALSE`. |
| `IN` list containing a `NULL` in a positive check vs. in a `NOT IN` check | A `NULL` in a plain `IN` list is harmless (it simply never matches, since `= NULL` is `UNKNOWN`); the same `NULL` inside a `NOT IN` list poisons the entire comparison to `UNKNOWN` for every row, silently returning zero rows. |
| `DISTINCT` vs. `DISTINCT ON` | `DISTINCT` removes fully duplicate rows, considering every projected column together; `DISTINCT ON (expr)` keeps exactly one row per distinct value of `expr`, chosen by a required leading `ORDER BY`. |
| Sorting by column position vs. column name in `ORDER BY` | Position (`ORDER BY 2`) is shorter but silently breaks if the `SELECT` list is later reordered or edited; naming columns explicitly is robust to those changes. |
| `LIMIT` alone vs. `LIMIT` with `ORDER BY` | `LIMIT` without `ORDER BY` returns an arbitrary, potentially inconsistent subset of rows; pairing it with `ORDER BY` is what makes the result deterministic and repeatable. |

## What's Next

This module gave you the complete core toolkit for asking questions of a single table: choosing columns (projection), choosing rows (filtering, with the full range of comparison, logical, pattern-matching, range, list, and `NULL` operators), presenting results in a meaningful order, and controlling how many rows come back. The full logical processing order established across these seven topics — `FROM → WHERE → SELECT → ORDER BY → LIMIT` — is a mental model you will keep leaning on for the rest of this course, since every later module only adds new stages to this same pipeline rather than replacing it. **Module 08 — Functions & Expressions** builds directly on top of this: the string, numeric, and date functions it covers, along with `CASE` expressions and `CAST`, all slot into the exact same `SELECT`-list and `WHERE`-clause positions this module just taught you to use, giving you dramatically more powerful things to put in those positions than plain column names and simple comparisons alone.
