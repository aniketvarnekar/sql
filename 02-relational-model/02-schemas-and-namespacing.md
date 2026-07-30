# Schemas and Namespacing

## Learning Objectives

By the end of this section you should be able to:
- Define what a schema is and explain how it differs from a database
- Explain the role of PostgreSQL's default `public` schema and the `search_path`
- Create a custom schema with `CREATE SCHEMA` and create/query tables inside it using qualified `schema.table` names
- Explain why namespacing tables into schemas matters as a database grows to serve multiple applications or teams

## Prerequisites

- [Tables, Rows, and Columns](01-tables-rows-and-columns.md) — you need a solid grasp of what a table (relation) is before learning how tables are grouped into higher-level containers.
- [Setting Up PostgreSQL](../01-introduction/04-setting-up-postgresql.md) — you should have a running `psql` session connected to the `sql_course` database, since every example below runs against it.

## Motivation

Every table you've created so far has lived in one flat space, with no grouping at all. That's fine for a handful of practice tables, but real databases regularly host dozens or hundreds of tables, often belonging to different applications, teams, or subsystems that happen to share the same PostgreSQL server for cost or operational simplicity. Without some way to group related tables together and keep unrelated ones from colliding by name, a growing database becomes an unmanageable flat pile. Schemas are the relational model's answer to that problem, and understanding them precisely will save you from a specific, common class of "why is my query touching the wrong table" confusion later.

## Problem Statement

Imagine a single company database serving two teams. The billing team wants a table called `invoices` to track money owed by customers. Separately, the sales team also wants a table called `invoices` — to them, an "invoice" means a sales quote sent to a prospect, a completely different concept with different columns. Both teams are told to share the same PostgreSQL database (for good reasons: joining data across teams occasionally, one server to operate instead of many). Without any grouping mechanism, both teams are forced to either negotiate over who "owns" the name `invoices` or resort to ugly workarounds like prefixing every table name (`billing_invoices`, `sales_invoices`) by hand, forever, across every table they ever create. Neither option scales cleanly, and neither gives the database itself any real understanding that these are two logically separate collections of tables.

## Concept

### The Hierarchy: Server → Database → Schema → Table

A PostgreSQL server can host many databases (Module 4 covers `CREATE DATABASE` in depth). Inside a single database, tables aren't the top-level organizing unit — **schemas** are. A **schema** is a named container inside a database that groups together related tables (and other objects: views, sequences, functions, and more). One database can contain many schemas, and each schema can contain many tables.

```
 PostgreSQL server
  └── database: sql_course
       ├── schema: public          (the default schema)
       │    └── tables: books, color_swatches, ...
       ├── schema: billing
       │    └── tables: invoices, payments
       └── schema: sales
            └── tables: invoices, quotes
```

Notice both `billing` and `sales` can have a table named `invoices` without any conflict — because a table's true, unambiguous identity within a database is not just its name, but its name *combined with* the schema it lives in. This is the core idea of **namespacing**: the schema acts as a namespace, and a name only has to be unique *within* its own namespace, not across the entire database.

It's worth being precise about a distinction that's easy to blur: a **database** is the top-level isolation boundary (no single query can span two databases — Module 4 covers this), while a **schema** is a grouping *within* one database. You will very often see both used to solve a similar-sounding problem ("keep things separate"), but they solve it at different scales and with different trade-offs, covered later in this topic.

### The Default `public` Schema

Every table you've created so far — `books`, `color_swatches`, and everything from Module 01 — was actually created inside a schema, even though you never named one. That's because every new PostgreSQL database starts with a schema named `public`, and unqualified names (`CREATE TABLE books (...)` rather than `CREATE TABLE public.books (...)`) resolve into `public` by default. Confirm this yourself:

```sql
SELECT schemaname, tablename
FROM pg_tables
WHERE tablename = 'books';
```

```
 schemaname | tablename
------------+-----------
 public     | books
(1 row)
```

You can also ask PostgreSQL directly what governs this default resolution:

```sql
SHOW search_path;
```

```
   search_path
-----------------
 "$user", public
(1 row)
```

The **`search_path`** is an ordered list of schemas PostgreSQL checks, in order, when you reference a table without qualifying it by schema. `"$user"` means "a schema matching your current username, if one exists" (used rarely, mostly by convention in some multi-tenant setups); if no such schema exists, PostgreSQL moves on to the next entry — `public` — and that's where `CREATE TABLE books (...)` actually lands.

### Creating and Using a Custom Schema

Creating a new schema is a single statement:

```sql
CREATE SCHEMA billing;
```

Tables can now be created directly inside it by qualifying the table name with the schema name, separated by a dot:

```sql
CREATE TABLE billing.invoices (
    id             SERIAL PRIMARY KEY,
    customer_name  TEXT NOT NULL,
    amount         NUMERIC NOT NULL,
    issued_on      DATE NOT NULL DEFAULT CURRENT_DATE
);

INSERT INTO billing.invoices (customer_name, amount) VALUES
    ('Acme Corp', 4200.00),
    ('Globex Inc', 1875.50);

SELECT * FROM billing.invoices;
```

```
 id | customer_name | amount  | issued_on
----+---------------+---------+------------
  1 | Acme Corp     | 4200.00 | 2026-07-31
  2 | Globex Inc    | 1875.50 | 2026-07-31
(2 rows)
```

Now create a second schema for the sales team, with its own, entirely different `invoices` table:

```sql
CREATE SCHEMA sales;

CREATE TABLE sales.invoices (
    id           SERIAL PRIMARY KEY,
    prospect     TEXT NOT NULL,
    quoted_price NUMERIC NOT NULL,
    valid_until  DATE
);

INSERT INTO sales.invoices (prospect, quoted_price, valid_until) VALUES
    ('Initech LLC', 9999.00, '2026-09-30');

SELECT * FROM sales.invoices;
```

```
 id |  prospect   | quoted_price | valid_until
----+-------------+--------------+-------------
  1 | Initech LLC |      9999.00 | 2026-09-30
(1 row)
```

Both `billing.invoices` and `sales.invoices` coexist in the same `sql_course` database, with completely different columns, without the slightest conflict — because their full, qualified identities are different, even though their unqualified table name is the same.

### What Happens If You Don't Qualify the Name

With both `billing.invoices` and `sales.invoices` now in existence, try the unqualified form:

```sql
SELECT * FROM invoices;
```

```
ERROR:  relation "invoices" does not exist
```

PostgreSQL refuses, rather than guessing — because `invoices` doesn't exist in any schema on the current `search_path` (which is `"$user", public`, and neither `billing` nor `sales` is on it). This is a deliberate safety property: an unqualified name is only ever resolved unambiguously, by walking the `search_path` in order and stopping at the *first* schema that contains a matching name — it never silently searches every schema in the database and picks one. If you temporarily add a schema to your session's `search_path`:

```sql
SET search_path TO billing, public;

SELECT * FROM invoices;
```

```
 id | customer_name | amount  | issued_on
----+---------------+---------+------------
  1 | Acme Corp     | 4200.00 | 2026-07-31
  2 | Globex Inc    | 1875.50 | 2026-07-31
(2 rows)
```

Now the unqualified name `invoices` resolves to `billing.invoices`, because `billing` was placed first in the `search_path` for this session. This is exactly why relying on an adjusted `search_path` in shared, multi-schema environments is risky (more on this in Best Practices below) — the same unqualified query text can mean two entirely different tables depending on session state that isn't visible in the query itself.

### Namespacing at Scale — Why It Matters

The two-team example above is deliberately small; the real motivation shows up as a database grows:

- **Avoiding name collisions without ugly prefixing.** Instead of `billing_invoices` and `sales_invoices` littering every table name with a manual, error-prone naming convention, each team's tables live cleanly under their own schema, and the schema name itself carries that meaning.
- **Permission granularity.** PostgreSQL lets you grant or revoke access at the schema level, not just per table:
  ```sql
  GRANT USAGE ON SCHEMA billing TO billing_app_user;
  GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA billing TO billing_app_user;
  ```
  This lets an organization mirror its real team boundaries directly in database permissions — the billing application's database user can be scoped to only the `billing` schema, unable to even see what tables exist inside `sales`. Module 19 (Security & Access Control) covers `GRANT`/`REVOKE` in full depth; this is simply the schema-level entry point to that system.
- **Logical organization mirroring how an organization actually thinks.** A large application's schemas often map directly onto its subsystems — `auth`, `billing`, `reporting`, `audit` — which makes exploring an unfamiliar database dramatically easier than hunting through one flat list of hundreds of tables.
- **Cleaner bulk operations.** An entire schema and everything inside it can be dropped in one statement (`DROP SCHEMA reporting CASCADE;`), useful for tearing down a subsystem's tables without hunting them all down individually — though, as with any `CASCADE`, this should be used deliberately (Module 4 covers `CASCADE` semantics in depth).

### Introspecting Schemas

You can list every schema in the current database directly:

```sql
\dn
```

```
  List of schemas
   Name   |  Owner
----------+----------
 billing  | postgres
 public   | postgres
 sales    | postgres
(3 rows)
```

Or with pure SQL, querying PostgreSQL's system catalog (previewed briefly in Module 01 and covered properly across this course):

```sql
SELECT schema_name FROM information_schema.schemata
ORDER BY schema_name;
```

```
 schema_name
--------------
 billing
 information_schema
 pg_catalog
 public
 sales
(5 rows)
```

Two schemas you didn't create show up here: `pg_catalog`, PostgreSQL's own internal system catalog (the tables that describe every table, column, and constraint in the database — including its own), and `information_schema`, a standardized, cross-vendor view over that same catalog information. Both exist in every PostgreSQL database automatically, and both are themselves proof that "schema" really is just "a named group of tables" — PostgreSQL uses the exact same mechanism to organize its own internal bookkeeping tables that you just used to organize `billing` and `sales`.

## Internal Working (Preview)

Schema resolution is itself powered by ordinary-looking catalog tables:

```
 CREATE SCHEMA billing;
        │
        ▼
 A new row is added to PostgreSQL's pg_namespace catalog,
 recording the schema's name and owner
        │
        ▼
 CREATE TABLE billing.invoices (...);
        │
        ▼
 A new row is added to pg_class (which tracks every table,
 among other objects), with a reference (relnamespace)
 pointing at billing's pg_namespace row
        │
        ▼
 An unqualified reference to "invoices" triggers PostgreSQL
 to walk the session's search_path, schema by schema, checking
 pg_namespace + pg_class for the first match — stopping at the
 first schema (in search_path order) that contains a table
 with that name
```

This is the same underlying mechanism, at a smaller scale, that Topic 3 of this module discusses as Codd's Rule 4 (a dynamic, queryable online catalog) — the database's own structural metadata (which schemas exist, which tables live in each) is itself stored as ordinary relational data, queryable with the same `SELECT` you use for everything else, as the `information_schema.schemata` query above demonstrated directly.

## Real-World Analogy

Picture a large shared office building housing several departments — Finance, Sales, Engineering — each with its own supply closet. Each closet can have a shelf labeled "Invoices" without any confusion, because nobody refers to it as just "the Invoices shelf" — they say "Finance's Invoices shelf" or "Sales's Invoices shelf." The department name is the namespace; the shelf label alone is ambiguous, but department name plus shelf label together is always unique and unambiguous. If someone walks in and asks for "the Invoices shelf" without saying which department, a well-run building doesn't guess — it asks "which department's?", exactly as PostgreSQL raises `relation "invoices" does not exist` rather than silently picking one when the name isn't qualified and doesn't resolve via `search_path`.

## Why This Design Was Chosen

Namespacing exists because real organizations rarely map cleanly onto a single, flat collection of uniquely-named things — teams evolve independently, reuse familiar names for locally meaningful concepts, and need administrative boundaries that mirror their actual structure. Rather than forcing every table in a database to compete for one global namespace (pushing teams toward brittle manual prefixing conventions), PostgreSQL implements schemas as first-class namespaces with their own catalog entries, permission model, and resolution rules. This also cleanly serves the same logical/physical independence theme introduced in Module 01: an application referring to `billing.invoices` doesn't need to know or care that `sales.invoices` even exists, and a DBA can reorganize, rename, or drop an entire schema's tables without touching unrelated schemas' definitions at all.

## Advantages

- **Eliminates naming collisions cleanly** — unrelated tables can share the same name across schemas without any conflict or manual prefixing convention.
- **Enables fine-grained, team-aligned permissions** — access can be granted or revoked per schema, letting database permissions mirror real organizational boundaries (Module 19).
- **Improves discoverability in large databases** — schemas group related tables logically, making it far easier for a newcomer (or a query tool) to understand a database's structure at a glance, compared to one flat list of hundreds of tables.
- **Supports safe, scoped bulk operations** — an entire schema's objects can be dropped, backed up, or migrated as a logical unit.

## Disadvantages / Limitations

- **An extra conceptual layer for beginners** — understanding "database vs. schema vs. table" as three distinct levels takes deliberate learning, and the terms are easy to conflate at first (some other database products, discussed briefly in Module 22, use "schema" and "database" almost interchangeably, which adds to the confusion when moving between systems).
- **Qualifying names is more verbose** — writing `billing.invoices` everywhere is more typing than `invoices`, and teams must decide how disciplined to be about always qualifying versus relying on `search_path`.
- **`search_path` misconfiguration causes confusing bugs** — because an unqualified name's meaning depends on session state (the current `search_path`) rather than being fixed by the query text alone, the same-looking query can silently resolve to a different table under a different session configuration, which can be a genuinely difficult class of bug to track down.
- **Schemas don't provide the same hard isolation a separate database does** — cross-schema joins and foreign keys within the same database are fully possible (and often desirable), but that also means "separating" two teams' tables into different schemas doesn't prevent one team's query from accidentally joining into another schema's tables if permissions allow it; true hard isolation requires separate databases (Module 4), not just separate schemas.

## Best Practices

- Prefer explicit, schema-qualified names (`billing.invoices`) in application code, migrations, and views — even when the unqualified name would currently resolve correctly via `search_path` — because it makes the query's meaning unambiguous regardless of session configuration, and immune to a future schema being added ahead of it on the `search_path`.
- Use schemas to reflect genuine logical or organizational boundaries (by team, by subsystem, by bounded concern) rather than creating them arbitrarily — a schema per team or per major subsystem is a common, sensible convention.
- Don't rely on a customized `search_path` as your primary mechanism for "separating" two teams' identically-named tables in a shared, multi-schema database — it's convenient for interactive, ad hoc work, but fragile as a long-term contract, since it depends on per-session state rather than on the query itself.
- Reach for a separate schema when you need logical grouping and scoped permissions within one database; reach for a separate database (Module 4) when you need hard isolation, including no possibility of accidental cross-querying at all.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `CREATE SCHEMA` creates a new database. | A schema is a namespace *within* a single database, not a new database. Tables in different schemas of the same database can still be joined together in a single query; tables in genuinely different databases cannot. |
| Assuming `public` is special or protected in a way other schemas aren't. | `public` is simply the schema that ships by default and sits on the default `search_path` — it has no inherent extra protection. In fact, older PostgreSQL versions made `public` writable by any authenticated user by default, which was a real, commonly cited security misconfiguration; modern PostgreSQL versions have tightened this default. |
| Creating a schema and tables inside it, then being confused that another database user "can't see" the tables even after granting table-level `SELECT`. | Accessing objects inside a schema also requires the `USAGE` privilege on the schema itself (`GRANT USAGE ON SCHEMA billing TO some_user;`) — table-level grants alone aren't sufficient if the user can't even "enter" the schema namespace to begin with. |
| Writing unqualified table names in shared migration scripts and assuming they'll always resolve the same way. | Unqualified names resolve according to the *current session's* `search_path`, which can differ between environments, users, or over time as schemas are added — qualified names remove this ambiguity entirely. |

## Interview Questions

1. **Q: What is a schema, and how does it differ from a database?**
   A: A schema is a named namespace *within* a single database that groups related tables (and other objects like views and sequences). A database is the top-level isolation boundary — a single connection accesses exactly one database, and no query can span two databases. Multiple schemas can exist inside one database, and tables in different schemas of the same database can still be joined in a single query, unlike tables in separate databases.

2. **Q: What is PostgreSQL's default schema, and how does an unqualified table name get resolved to it?**
   A: The default schema is `public`. When a table name is referenced without a schema qualifier, PostgreSQL resolves it by walking the session's `search_path` (an ordered list of schemas, `"$user", public` by default) and using the first schema on that list containing a matching table name.

3. **Q: Why might a large organization use schemas instead of just prefixing table names, e.g. `billing_invoices` vs. `sales_invoices`?**
   A: Schemas provide a real namespace enforced and understood by the database itself, not just a naming convention humans have to remember to follow consistently. This enables schema-level permission grants (scoping an application's database user to only its own schema), schema-level bulk operations (dropping or migrating a whole schema at once), and avoids the ever-growing, error-prone prefixing that manual naming conventions require as more teams share a database.

4. **Q: If you create tables named `invoices` in both a `billing` schema and a `sales` schema, what happens if you run `SELECT * FROM invoices;` without qualifying it?**
   A: It depends entirely on the session's current `search_path`. If neither schema is on the `search_path`, PostgreSQL raises an error that the relation doesn't exist. If exactly one of them is on the `search_path`, the unqualified name resolves to that one. If both were somehow reachable, PostgreSQL resolves to whichever schema appears first in `search_path` order — it never guesses across multiple equally-valid matches silently in an ambiguous way.

## Summary

- A **schema** is a named namespace within a single database that groups related tables (and other objects); it sits between "database" and "table" in PostgreSQL's organizational hierarchy.
- Every PostgreSQL database starts with a default schema named **`public`**; unqualified table names resolve according to the session's **`search_path`**, an ordered list of schemas PostgreSQL checks in sequence.
- `CREATE SCHEMA schema_name;` creates a new namespace; tables inside it are created and referenced with the qualified form `schema_name.table_name`.
- Schemas let identically-named tables coexist without conflict, enable schema-level permission grants that mirror real team boundaries, and improve discoverability — but they are not a substitute for a separate database when true hard isolation is required.
- Prefer explicit, schema-qualified names in real application code and migrations rather than depending on `search_path`, which is convenient interactively but fragile as a long-term contract.
