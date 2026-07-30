# Module 04 — Database & Table Design

## Module Goal

By the end of this module, you will be able to create and remove entire databases, and design, modify, and remove the tables that live inside them, using PostgreSQL's Data Definition Language (DDL). You will understand not just the syntax of `CREATE`, `ALTER`, `DROP`, and `TRUNCATE`, but the structural consequences of each — what happens to storage, to dependent objects, and to concurrent users when you run them. This module is where the abstract idea of "a table" from earlier modules becomes something you can actually stand up, reshape, and tear down with confidence, and it is the direct foundation for Module 5, where you'll make the tables you build here actually enforce data integrity.

## Topics Covered in This Module

1. **[Creating and Dropping Databases](01-creating-and-dropping-databases.md)** — `CREATE DATABASE` and its common options, `DROP DATABASE`, why you can't drop a database you're connected to, and how to list existing databases.
2. **[Creating Tables](02-creating-tables.md)** — The full `CREATE TABLE` syntax, designing a table from a real-world requirement, and where data types and constraints fit into a column definition.
3. **[Altering Tables](03-altering-tables.md)** — Adding, dropping, and renaming columns; changing a column's data type safely; renaming a table; and a first look at adding/dropping constraints.
4. **[Dropping and Truncating Tables](04-dropping-and-truncating-tables.md)** — `DROP TABLE`, `CASCADE` vs. `RESTRICT` when other objects depend on a table, and how `TRUNCATE` differs structurally from `DROP`.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 01 — Introduction**, in full: you need a working PostgreSQL installation and `psql` session ([Setting Up PostgreSQL](../01-introduction/04-setting-up-postgresql.md)), the `CREATE TABLE` → `INSERT` → `SELECT` loop ([Your First Query](../01-introduction/05-your-first-query.md)), and the DDL/DML/DQL categorization ([Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md)) — this entire module lives inside the DDL category.
- **Module 02 — Relational Model**: you need the vocabulary of tables, rows, columns, and relations that module establishes formally — this module assumes you already know what a table logically *is* and focuses on how to actually bring one into existence.
- **Module 03 — Data Types**: you need to already know PostgreSQL's core data types (numeric, text, date/time, boolean) — this module uses them constantly in column definitions but does not re-teach what they mean.

## How to Study This Module

Read Topics 1 and 2 carefully and type out every example — they are the foundation for everything else in this module and for every table you will ever build in this course. Topic 3 (altering tables) is just as important in real practice, since schemas evolve constantly once an application is live; pay particular attention to the discussion of why some column-type changes are dangerous on large tables, since that judgment separates a careful engineer from one who causes an outage. Topic 4 is shorter but conceptually sharp — the distinction between `DROP`, `TRUNCATE`, and `CASCADE`/`RESTRICT` is a frequent source of real-world mistakes, so don't skim it. This module deliberately does not teach constraints (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`) in depth or the full `DELETE` vs. `TRUNCATE` trade-off — both get dedicated treatment in Module 5 (Constraints & Keys) and Module 6 (Modifying Data) respectively. Here, you're learning to build and reshape the container; what rules the container enforces, and how its contents change over time, come next.
