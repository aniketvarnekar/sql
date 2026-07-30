# Module 02 — Relational Model

## Module Goal

By the end of this module, you will understand the theory that everything else in this course rests on: what a table actually *is* in formal terms, how tables are organized into named groups called schemas, and the academic and historical reasoning — originating with Edgar F. Codd — that shaped the relational databases you use every day. This module is deliberately more conceptual than hands-on: it gives you the precise vocabulary (relation, tuple, attribute, schema) and the mental model (sets, not lists; logical structure independent of physical storage) that later modules will keep referring back to, especially when you reach constraints, normalization, and query design.

## Topics Covered in This Module

1. **[Tables, Rows, and Columns](01-tables-rows-and-columns.md)** — The formal vocabulary of relations, tuples, and attributes; what makes a collection of data a valid relation; why row order and column order don't matter in the relational model.
2. **[Schemas and Namespacing](02-schemas-and-namespacing.md)** — What a schema is, PostgreSQL's default `public` schema, creating custom schemas, qualifying names as `schema.table`, and why namespacing matters as databases grow.
3. **[The Relational Model and Codd's Rules](03-the-relational-model-and-codds-rules.md)** — Edgar Codd's 1970 paper and 1985 twelve rules, relations as mathematical sets versus SQL's more permissive behavior in practice, and why SQL is only an approximation of the pure relational model.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 01 — Introduction**, in full. Specifically: you need the database/DBMS distinction and the definition of a relational database from [What Is a Database and a DBMS?](../01-introduction/01-what-is-a-database-and-a-dbms.md), the declarative nature of SQL from [What Is SQL?](../01-introduction/02-what-is-sql.md), and a working `psql` session with the `sql_course` database from [Setting Up PostgreSQL](../01-introduction/04-setting-up-postgresql.md), since every example here is run against it. The `CREATE TABLE` → `INSERT` → `SELECT` loop from [Your First Query](../01-introduction/05-your-first-query.md) is assumed throughout — this module explains *what* that loop was actually operating on, in formal terms.

## How to Study This Module

Read Topic 1 first and slowly — the vocabulary it introduces (relation, tuple, attribute) is used without re-explanation in every module for the rest of this course, including when this course later says "row" or "column" as the informal, everyday equivalent. Topic 2 is more practical and can be read a little faster, but don't skip the discussion of `search_path`, since misunderstanding it is a common source of "why is my query hitting the wrong table" confusion later. Topic 3 is the most academic topic in the module — it will not make you write a single new line of working SQL — but it explains *why* SQL behaves the way it does in cases that otherwise look like bugs or inconsistencies (duplicate rows being allowed, `NULL` behaving strangely with `UNIQUE`), so treat it as essential background rather than optional trivia. Come back to Topic 3 after Module 5 (Constraints & Keys) if any of it feels abstract on a first read — it will click harder once you've actually declared a primary key yourself.
