# Module 02 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Tables, Rows, and Columns** — the formal vocabulary of relations, tuples, and attributes; what makes a collection of data a valid relation; why row and column order carry no logical meaning
- [x] **Schemas and Namespacing** — what a schema is, the default `public` schema and `search_path`, creating and qualifying custom schemas, and why namespacing matters as databases grow
- [x] **The Relational Model and Codd's Rules** — Codd's 1970 paper, relations as mathematical sets versus SQL's permissive practical behavior, a plain-English tour of the twelve rules, and why SQL only approximates the pure relational model

## Practical Connections

- **A reporting dashboard querying millions of rows** relies on the exact property this module established: your `SELECT` statement describes *what* result you want, never *how* to retrieve it, and the database is free to physically reorganize, index, and reorder that data for performance without ever changing what your query logically returns.
- **A growing company running dozens of internal tools against one shared PostgreSQL server** depends on schemas to keep each tool's tables logically separated and independently permissioned, without needing a separate database (and its harder isolation trade-offs) for every single tool.
- **A data engineer investigating why a "unique" email column somehow has three rows with a blank email** is running directly into this module's honest theoretical nuance: SQL's `NULL` is never considered equal to another `NULL`, so a `UNIQUE` constraint alone does not prevent multiple missing values — a very real, very common production surprise this module explained in advance.
- **Any tool that auto-generates documentation or a visual schema diagram for a database** is only possible because of Codd's Rule 4 — the database's own structure (tables, columns, schemas) is stored as ordinary, queryable relational data, not some separate, opaque metadata format.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Relation (theory) vs. table (SQL) | A relation is formally a *set* of tuples — no duplicates possible by definition. A SQL table permits duplicate rows by default (bag semantics) unless a `PRIMARY KEY` or `UNIQUE` constraint forces true set-like uniqueness. |
| Database vs. schema | A database is the top-level isolation boundary — no query spans two databases. A schema is a namespace *within* one database, grouping related tables; tables in different schemas of the same database can still be joined in one query. |
| Row/column order in output vs. a guaranteed property of the data | The order rows or columns are displayed in (via `ORDER BY`, or the column list in `SELECT`) is a presentation choice made at query time — it says nothing about any inherent, guaranteed order of the underlying relation, which the model treats as an unordered set. |
| `NULL` allowed twice in a `UNIQUE` column vs. "duplicate values rejected" | `UNIQUE` rejects two rows with the same *known* value, but `NULL` represents "unknown," and `NULL = NULL` is never true under SQL's three-valued logic — so multiple `NULL`s are permitted by default, which is easy to mistake for a bug. |
| Codd's 1970 relational model paper vs. Codd's 1985 twelve rules | The 1970 paper introduced the relational model itself. The 1985 rules, published fifteen years later, were a separate, later contribution: a concrete checklist for testing whether a product genuinely implements that model. |

## What's Next

This module gave you the precise theoretical vocabulary and mental model — relations as unordered sets of tuples, schemas as namespaces, and the historical rules that define (and are only approximated by) "relational" — that every remaining module quietly assumes. **Module 03 — Data Types** builds directly on top of it: having established that every attribute draws its values from a defined domain, the next module makes that idea concrete, covering PostgreSQL's numeric, string, date/time, and boolean types in depth, along with the precise semantics of `NULL` that this module's discussion of Codd's Rule 3 already began to surface.
