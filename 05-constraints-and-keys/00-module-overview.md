# Module 05 — Constraints & Keys

## Module Goal

By the end of this module, you will be able to make a table *enforce its own rules* — guaranteeing that mandatory data is never missing, that values that must be unique actually are, that every row can be uniquely and reliably identified, that relationships between tables can never point at nothing, and that values satisfy whatever business logic you attach to them. Modules 1–4 taught you how to describe a table's *shape* (its columns and their types). This module teaches you how to describe a table's *rules* — the difference between a table that merely stores whatever it's given and a table that actively protects the correctness of its own data, without any application code having to police it.

## Topics Covered in This Module

1. **[NOT NULL and DEFAULT](01-not-null-and-default.md)** — Making columns mandatory, giving columns automatic fallback values, and how the two interact.
2. **[UNIQUE Constraints](02-unique-constraints.md)** — Enforcing that a column (or combination of columns) never repeats, and PostgreSQL's specific handling of `NULL` under `UNIQUE`.
3. **[Primary Keys](03-primary-keys.md)** — The single identifying constraint every table should have: what it guarantees, single-column vs. composite keys, and why exactly one per table.
4. **[Foreign Keys and Referential Integrity](04-foreign-keys-and-referential-integrity.md)** — Linking tables together safely, preventing "orphaned" rows, and the four `ON DELETE`/`ON UPDATE` actions that control what happens when a referenced row disappears.
5. **[CHECK Constraints](05-check-constraints.md)** — Attaching arbitrary row-level validation logic directly to the database, beyond what a data type or uniqueness rule alone can express.
6. **[Natural vs. Surrogate Keys](06-natural-vs-surrogate-keys.md)** — Choosing between a real-world attribute and a database-generated identifier as a table's primary key, and the trade-offs of each.
7. **[Module Summary](07-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 4 — Database & Table Design**, especially its "Creating Tables" and "Altering Tables" topics: constraints are written *inside* `CREATE TABLE` column definitions, or attached afterward with `ALTER TABLE ... ADD CONSTRAINT`. This module assumes you're already comfortable with both statement forms and focuses purely on the constraint clauses that slot into them.
- **Module 3 — Data Types**, especially its coverage of `NULL` semantics: several of this module's topics (`NOT NULL`, `UNIQUE`, primary keys) are really precise statements about when `NULL` is and isn't allowed, so you need to already understand what `NULL` *means* (the absence of a value, not zero or empty string) before this module can meaningfully restrict it.
- **Module 1 — Introduction**, specifically [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md): every statement in this module is DDL — you're still defining structure, just a more rule-rich structure than Module 4 covered on its own.

## How to Study This Module

Read Topics 1 through 5 in order — they build on each other and on a single running example schema (a `customers` / `orders` / `order_items` / `products` set of tables) that grows one constraint at a time as you move through the module. Topic 3 (Primary Keys) and Topic 4 (Foreign Keys) are the conceptual core of the entire module and of relational database design generally — expect to reread Topic 4's `ON DELETE`/`ON UPDATE` section more than once, since choosing the wrong action there is one of the most common real-world schema design mistakes. Topic 6 (Natural vs. Surrogate Keys) is a judgment-call topic rather than a syntax topic — it matters most once you start designing your own schemas from scratch rather than following a prescribed one, so treat it as a lens to revisit, not a fact to memorize once and forget.
