# Module 07 — Querying Basics

## Module Goal

By the end of this module, you will be able to write real, useful `SELECT` queries from scratch: choosing exactly the columns you want (and computing new ones), filtering rows down to the ones that matter with every comparison and logical operator SQL offers, matching text patterns, testing ranges, lists, and missing values, sorting results in exactly the order you need, and controlling how many rows come back. [Your First Query](../01-introduction/05-your-first-query.md) showed you one small, complete `SELECT` and promised that this module would return to explain, in full, the precise order in which a `SELECT` statement is actually evaluated — that promise is kept here. Nearly everything you will do with SQL from this point forward is an elaboration of the tools introduced in this module.

## Topics Covered in This Module

1. **[SELECT and Projection](01-select-and-projection.md)** — choosing specific columns vs. `SELECT *`, aliasing with `AS`, computed expression columns, and the relational-theory concept of projection.
2. **[Filtering with WHERE](02-filtering-with-where.md)** — the `WHERE` clause, how it evaluates a boolean expression per row, and the fundamental distinction between filtering and projecting.
3. **[Comparison and Logical Operators](03-comparison-and-logical-operators.md)** — `=`, `<>`/`!=`, `<`, `>`, `<=`, `>=`, `AND`, `OR`, `NOT`, operator precedence, and how these operators behave under `NULL`'s three-valued logic.
4. **[Pattern Matching with LIKE](04-pattern-matching-with-like.md)** — `LIKE`, PostgreSQL's `ILIKE`, the `%` and `_` wildcards, escaping literal wildcard characters, and a brief pointer to regular-expression matching.
5. **[BETWEEN, IN, and IS NULL](05-between-in-and-is-null.md)** — inclusive range checks with `BETWEEN`, membership checks with `IN`/`NOT IN`, and the correct way to test for missing values with `IS NULL`/`IS NOT NULL`.
6. **[Sorting with ORDER BY](06-sorting-with-order-by.md)** — single- and multi-column sorting, `ASC`/`DESC`, sorting by position vs. name, `NULLS FIRST`/`NULLS LAST`, and sorting by an expression that isn't even in the result.
7. **[LIMIT, OFFSET, and DISTINCT](07-limiting-and-distinct.md)** — capping and paginating results, removing duplicate rows with `DISTINCT` and PostgreSQL's `DISTINCT ON`, and why `LIMIT` needs `ORDER BY` to mean anything reliable.
8. **[Module Summary](08-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 01 (Introduction)**, in full — especially [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md) (this whole module is the deep dive into the DQL category, `SELECT`) and [Your First Query](../01-introduction/05-your-first-query.md) (the single example query this module dissects into its full generality).
- **Module 02 (Relational Model)** — specifically the relation/tuple/attribute vocabulary and the idea that column and row order carry no logical meaning on their own, from [Tables, Rows, and Columns](../02-relational-model/01-tables-rows-and-columns.md). Projection and filtering, covered here, are the practical SQL realization of that theory.
- **Module 03 (Data Types)** — you need to already know what `NULL` means and why ordinary two-valued logic doesn't apply to it, since several topics in this module (comparison operators, `IN`, `IS NULL`) depend on that directly.
- **Module 06 (Modifying Data)** — specifically [INSERT](../06-modifying-data/01-insert.md), since this module assumes you can already get a table populated with realistic rows to query against; this module's running example table is built with exactly that statement.

## How to Study This Module

Topics 1 and 2 are the foundation: projection (which columns) and filtering (which rows) are the two orthogonal ideas every other topic in this module refines. Read them first, and pay close attention to the logical processing order introduced in Topic 1 and completed in Topic 7 — it resolves a surprising number of "why doesn't this work" moments beginners hit (referencing a `SELECT` alias in `WHERE`, for instance). Topics 3 through 5 build out everything you can put inside a `WHERE` clause — comparison and logical operators, pattern matching, and the range/list/`NULL` checks — and are worth typing out yourself rather than just reading, since the `NULL`-related pitfalls in Topics 3 and 5 are exactly the kind of thing that looks obvious in isolation but causes real production bugs. Topics 6 and 7 shift from *which* rows to *how they're presented* — sorting, capping, and deduplicating — and Topic 7 closes the module by assembling the full logical pipeline (`FROM` → `WHERE` → `SELECT` → `ORDER BY` → `LIMIT`) into one complete picture, which the Module Summary then reinforces one more time.
