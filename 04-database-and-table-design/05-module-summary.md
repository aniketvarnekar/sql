# Module 04 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Creating and Dropping Databases** — `CREATE DATABASE` with `OWNER`, `ENCODING`, `TEMPLATE`, and `CONNECTION LIMIT`; listing databases with `\l` and `pg_database`; `DROP DATABASE` (with `IF EXISTS`); why a database in use cannot be dropped until you disconnect (or force it with `WITH (FORCE)`).
- [x] **Creating Tables** — the full `CREATE TABLE` syntax, `IF NOT EXISTS`, designing a table's columns from a real-world requirement, column ordering, schema-qualified names, identifier quoting rules, and a conceptual preview of where constraints attach.
- [x] **Altering Tables** — `ADD COLUMN`, `DROP COLUMN`, `RENAME COLUMN`, `ALTER COLUMN ... TYPE ... USING`, `RENAME TO` for tables, why type changes can require an expensive, lock-heavy full-table rewrite, and a preview of `ADD CONSTRAINT`/`DROP CONSTRAINT`.
- [x] **Dropping and Truncating Tables** — `DROP TABLE` (with `IF EXISTS`), `CASCADE` vs. `RESTRICT` for dependent objects, and the structural contrast between `DROP TABLE` (removes structure and data) and `TRUNCATE TABLE` (removes only data, keeps structure).

## Practical Connections

- **Every application's onboarding or environment-setup script** relies on exactly the `CREATE DATABASE` / `CREATE TABLE IF NOT EXISTS` patterns from Topics 1 and 2 — spinning up a fresh, correctly structured database for a new developer or a test environment is this module's content, run start to finish.
- **A live application's schema evolving over time** — adding a feature that needs a new column, widening a column that turned out to be too small, renaming something that was poorly named originally — is exactly the territory of Topic 3 (`ALTER TABLE`), including the very real operational judgment call of when a type change is safe to run immediately versus when it needs a planned maintenance window because of table-rewrite locking.
- **Nightly data pipelines and staging tables** commonly rely on `TRUNCATE TABLE` (Topic 4) to reset a scratch table's contents every run without ever having to redefine its structure, while genuinely retired tables get permanently removed with `DROP TABLE`.
- **Schema cleanup in a mature system with many interdependent views and foreign keys** requires exactly the judgment Topic 4 introduced: reading what a table's dependents actually are before deciding between manually removing them first or accepting a `CASCADE`'s wider blast radius.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| `DROP DATABASE` refusing vs. `DROP TABLE` refusing | `DROP DATABASE` refuses because of an active *connection* to that database (you must disconnect, or force-disconnect others). `DROP TABLE` (by default, `RESTRICT`) refuses because of *other database objects* (views, foreign keys) depending on it — a different reason entirely, solved by removing dependents first or using `CASCADE`, not by disconnecting anyone. |
| `ALTER COLUMN ... TYPE` that's instant vs. one that rewrites the whole table | Whether PostgreSQL can prove the change is representation-compatible using only catalog metadata (instant, e.g. widening `NUMERIC` precision) versus needing to convert every stored value to a new representation (a full, lock-heavy table rewrite, e.g. `TEXT` to `INTEGER`). |
| `DROP TABLE` vs. `TRUNCATE TABLE` | `DROP TABLE` removes the table's structure and all its data — the table ceases to exist. `TRUNCATE TABLE` removes only the rows; the table's columns, types, and constraints remain fully intact and immediately usable. |
| `CASCADE` on `DROP TABLE` vs. `CASCADE` on `TRUNCATE TABLE` | Both follow the same "also affect dependent objects" pattern, but the dependents differ in kind: `DROP TABLE CASCADE` also drops dependent views/constraints; `TRUNCATE ... CASCADE` also empties other tables that reference this one via a foreign key. |
| A column constraint vs. a table constraint (previewed here, taught fully in Module 5) | A column constraint is written directly after one column's type and applies only to that column; a table constraint is its own separate entry in `CREATE TABLE` (or added via `ALTER TABLE ADD CONSTRAINT`) and can span multiple columns at once. |

## What's Next

Module 04 gave you full control over the *shape* of your data: creating and removing databases, designing and creating tables from real requirements, reshaping existing tables safely as needs change, and removing tables or their contents when they're no longer needed. What it deliberately left underdeveloped is *enforcement* — right now, nothing stops a `products` row from having a negative price, a missing name, or an `order_items` row pointing at a product that doesn't exist. **Module 05 — Constraints & Keys** builds directly on top of every table you now know how to create and alter, teaching `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, and `NOT NULL` in full — the rules that turn a correctly shaped table into one that actively protects the integrity of the data stored inside it.
