# Module 19 — Security & Access Control

## Module Goal

By the end of this module, you will understand how a relational database actually enforces "who is allowed to do what" — not as an application-level afterthought, but as a first-class part of the database engine itself. You will create and manage roles, grant and revoke precise privileges on tables, schemas, and databases, understand exactly what SQL injection is and how parameterized queries eliminate it structurally, and learn to apply the principle of least privilege so that a leaked credential or a bug never has more power than it strictly needs. This module turns the DCL category — introduced in passing back in Module 1 — into something you can actually design and defend with.

## Topics Covered in This Module

1. **[Users and Roles](01-users-and-roles.md)** — `CREATE ROLE`/`CREATE USER`, why a "user" is just a role with the `LOGIN` privilege in PostgreSQL, role membership and inheritance, `ALTER ROLE`, and why grouping permissions into roles scales far better than granting directly to individuals.
2. **[GRANT and REVOKE](02-grant-and-revoke.md)** — granting specific privileges (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, and DDL-related privileges) on specific objects (tables, schemas, databases) to roles, `REVOKE` to remove them, column-level privilege granularity, and how to check what a role can currently do.
3. **[SQL Injection](03-sql-injection.md)** — precisely what SQL injection is, a worked example of a vulnerable query being exploited, parameterized queries/prepared statements as the structural fix, and why input escaping alone is a fragile substitute for true parameterization.
4. **[Principle of Least Privilege](04-principle-of-least-privilege.md)** — granting only the minimum permissions a role actually needs, concrete examples for an application connection user and a reporting user, and how this bounds the damage of a compromised credential.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 1 (Introduction), Topic 3 — [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md):** DCL (Data Control Language) was introduced there as one of the five statement categories, with `GRANT`/`REVOKE` given as its example commands. This module is the full depth treatment of that category.
- **Module 6 (Modifying Data), in full — [Module Overview](../06-modifying-data/00-module-overview.md), [INSERT](../06-modifying-data/01-insert.md), [UPDATE](../06-modifying-data/02-update.md):** `INSERT`, `UPDATE`, and `DELETE` are exactly the DML privileges this module grants and revokes — you cannot reason about "what should this role be allowed to do" without already knowing what each of those statements actually does to data.
- **Module 4 (Database & Table Design) — [Module Overview](../04-database-and-table-design/00-module-overview.md), [Creating and Dropping Databases](../04-database-and-table-design/01-creating-and-dropping-databases.md):** privileges in this module are granted *on* databases, schemas, and tables — objects Module 4 taught you to create. You need to already know what a schema and a database are as containers before you can meaningfully restrict access to them.
- General comfort with `SELECT`/`WHERE` from [Your First Query](../01-introduction/05-your-first-query.md) is assumed throughout, particularly in the SQL injection topic's worked examples.

## How to Study This Module

Read Topics 1 and 2 together and in order — roles are the "who," and `GRANT`/`REVOKE` are the "what they can do," and neither is useful without the other. Type out every `CREATE ROLE` and `GRANT` example against a real PostgreSQL instance and then actually try to violate the permissions you just set up (e.g., connect as the restricted role and attempt a forbidden statement) — seeing PostgreSQL's `permission denied` error firsthand is far more instructive than reading about it. Topic 3 (SQL Injection) is conceptually the most important topic in this module for anyone who will ever write code that builds SQL from user input — read it slowly, and make sure you understand *why* parameterization is a structural fix rather than a defensive habit. Topic 4 (Principle of Least Privilege) is where everything in this module becomes a single, memorable design rule — read it last, since it directly reuses the `GRANT`/`REVOKE` vocabulary from Topic 2 and the risk model from Topic 3 to explain *why* you'd bother being this careful in the first place.
