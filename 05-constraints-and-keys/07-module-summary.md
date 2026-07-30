# Module 05 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **NOT NULL and DEFAULT** — mandatory columns, automatic fallback values, how the two interact, and adding `NOT NULL` to an already-populated table
- [x] **UNIQUE Constraints** — single-column and composite uniqueness, PostgreSQL's "every `NULL` is distinct" behavior under `UNIQUE`, and the fact that `UNIQUE` is implemented via an underlying index
- [x] **Primary Keys** — the `UNIQUE` + `NOT NULL` guarantee combined, single-column vs. composite keys, and why one per table is the rule
- [x] **Foreign Keys and Referential Integrity** — the `REFERENCES` syntax, preventing orphaned rows, and all four `ON DELETE`/`ON UPDATE` actions worked through with concrete scenarios
- [x] **CHECK Constraints** — single- and multi-column validation expressions, where enforcement happens relative to application-level checks, and the inability to reference other rows or tables
- [x] **Natural vs. Surrogate Keys** — defining both, weighing stability/simplicity/performance/meaning-leakage, and when each is the right default

## Practical Connections

- **A signup form's "email already registered" error** is very likely a `UNIQUE` constraint violation surfacing from the database, not (or not only) an application-level check — exactly the race-condition-proof guarantee Topic 2 discussed.
- **An e-commerce checkout that refuses to let you delete an account with existing order history** is `ON DELETE RESTRICT` (Topic 4) doing its job — protecting historical, legally/financially relevant records from accidental deletion.
- **A reporting dashboard joining millions of order rows to a customers table with confidence that every `customer_id` is valid** is relying entirely on referential integrity (Topic 4) holding — without it, that join would need defensive `NULL`-handling logic everywhere, for data that should never have been possible in the first place.
- **A products catalog letting you correct a mistyped SKU with a single, unremarkable `UPDATE`** is only that simple because the catalog's primary key is a surrogate `product_id` (Topic 6), not the SKU itself — otherwise that correction would cascade through every table referencing it.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| `NOT NULL` vs. `DEFAULT` | `NOT NULL` forbids a column from ever being empty; `DEFAULT` only supplies a value when a column is *omitted* from an `INSERT` — an explicit `NULL` still violates `NOT NULL` regardless of any default. |
| `UNIQUE` vs. `PRIMARY KEY` | `UNIQUE` alone still allows any number of `NULL`s and a table can have many `UNIQUE` constraints; `PRIMARY KEY` adds `NOT NULL` on top of uniqueness, and a table may have only one. |
| Composite `UNIQUE`/`PRIMARY KEY` vs. uniqueness on each column individually | `UNIQUE (a, b)` or `PRIMARY KEY (a, b)` only guarantees the *combination* is unique — either column alone can still repeat across different values of the other. |
| `RESTRICT`/`NO ACTION` vs. `CASCADE` vs. `SET NULL` | `RESTRICT`/`NO ACTION` blocks deleting a parent with existing children; `CASCADE` deletes/updates the children along with the parent; `SET NULL` clears the relationship but keeps the child row — the right choice depends entirely on whether the child row has independent meaning without its parent. |
| `CHECK` constraint vs. application-level validation | A `CHECK` constraint is enforced by the database on every write, from any source; application-level validation only protects writes that go through that specific application, and is best used for fast user feedback, not as the sole guarantee. |
| Natural key vs. surrogate key | A natural key is a real-world, business-meaningful identifier (an SKU, an email); a surrogate key is a database-generated value with no meaning outside the database (a `SERIAL` integer, a `UUID`) — surrogate keys are the common default specifically because they never need external correction. |

## What's Next

This module gave every table in your schema the ability to enforce its own correctness: mandatory values, uniqueness, unambiguous identity, honest relationships between tables, and arbitrary row-level business rules. With tables that can now be trusted to hold correct, consistent data, **Module 06 — Modifying Data** turns to actually working with that data at scale over time: the full depth of `INSERT`, `UPDATE`, and `DELETE`, the difference between `DELETE` and `TRUNCATE`, and "upsert" patterns for inserting-or-updating in a single statement — all of it now operating against tables that will reject bad data before it's ever written, rather than silently accepting it.
