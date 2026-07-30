# Module 06 — Modifying Data

## Module Goal

By the end of this module, you will be able to confidently and safely change the data inside a table: add new rows, change existing values, remove rows, wipe a table clean, and handle the extremely common "insert this, but update it if it already exists" problem — all using PostgreSQL's exact syntax. You will also understand *why* each of these statements behaves the way it does internally, and why the constraints from Module 5 (Constraints & Keys) are not a separate topic from this one — they fire on almost every statement you write here. Everything from here on in the course (querying, joins, transactions) assumes you can already get realistic, safely-modified data into a table.

## Topics Covered in This Module

1. **[INSERT](01-insert.md)** — adding new rows: single-row, multi-row, `INSERT ... SELECT`, the `RETURNING` clause, and what happens when a constraint rejects an insert.
2. **[UPDATE](02-update.md)** — changing existing rows: the `SET` clause, updating multiple columns, `WHERE` referencing other columns, `UPDATE ... FROM`, and the danger of a missing `WHERE`.
3. **[DELETE](03-delete.md)** — removing rows: `DELETE ... WHERE`, the danger of `DELETE` with no `WHERE`, `DELETE ... USING`, `RETURNING`, and how foreign key `ON DELETE` actions interact with it.
4. **[TRUNCATE vs. DELETE](04-truncate-vs-delete.md)** — a full comparison of the two ways to empty a table: speed, transactional behavior, triggers, identity/sequence resets, and when to reach for each.
5. **[UPSERT with ON CONFLICT](05-upsert-on-conflict.md)** — PostgreSQL's `INSERT ... ON CONFLICT` clause for the "insert, or update if it already exists" problem, and why it's safer than checking first and inserting second.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 4 (Database & Table Design)** — this module assumes you already know how to create a table (`CREATE TABLE`), and understand what columns and data types are, since every example here modifies rows inside an already-defined table.
- **Module 5 (Constraints & Keys)** — this is the module most heavily reused here. `NOT NULL`, `UNIQUE`, `CHECK`, primary keys, and foreign keys (and their `ON DELETE`/`ON UPDATE` actions) are enforced automatically on every `INSERT`, `UPDATE`, and `DELETE` you run — you cannot write realistic modifying statements without understanding what happens when a constraint is violated.
- **Module 1 (Introduction)**, specifically [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md) and [Your First Query](../01-introduction/05-your-first-query.md) — this module is the deep dive into the DML category (`INSERT`/`UPDATE`/`DELETE`) first introduced there.

## How to Study This Module

Read Topics 1 through 3 (`INSERT`, `UPDATE`, `DELETE`) in order — they form one continuous story of "add, change, remove," and each one leans on constraint behavior established in Module 5, so don't skip past the "what happens when a constraint is violated" discussions even if they feel repetitive with Module 5. Topic 4 (`TRUNCATE` vs. `DELETE`) is a compact but important comparison — it's a favorite interview topic, so it's worth a careful read even though it's shorter than the others. Topic 5 (`UPSERT`) is the most advanced topic in this module and builds directly on both `INSERT` (Topic 1) and the `UNIQUE`/primary key constraints from Module 5 — read Topic 1 immediately before it. By the end of this module you should be able to look at any real-world "add a user," "edit a profile," "cancel an order," or "sync inventory counts" feature and know exactly which statement (and which clause of it) implements that behavior.
