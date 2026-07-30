# Module 12 — Views

## Module Goal

By the end of this module, you will be able to save a query as a reusable, named database object with `CREATE VIEW`, query it exactly as if it were a table, and reason precisely about when PostgreSQL will let you write (`INSERT`/`UPDATE`/`DELETE`) through that view automatically versus when it won't. You will also understand materialized views — the mechanism for physically storing an expensive query's result so it doesn't have to be recomputed on every read — and how to decide between a regular view, a materialized view, and a plain table for a given problem. Views are the primary tool the relational model gives you for hiding complexity (joins, aggregations, filters) behind a simple, stable, reusable interface, and this module sets up direct groundwork for Module 18's triggers (the general escape hatch for writing through complex views) and Module 13's indexing decisions (materialized views can be indexed exactly like real tables).

## Topics Covered in This Module

1. **[Creating and Using Views](01-creating-and-using-views.md)** — `CREATE VIEW` as a saved, named query; querying a view like a table; `CREATE OR REPLACE VIEW`; `DROP VIEW`; why views help with reuse, abstraction, and simplifying complex joins/aggregations for the people who need to query the data.
2. **[Updatable Views](02-updatable-views.md)** — which views PostgreSQL allows `INSERT`/`UPDATE`/`DELETE` through automatically, `WITH CHECK OPTION`, why join and aggregate views are not automatically updatable, and `INSTEAD OF` triggers as the general escape hatch.
3. **[Materialized Views](03-materialized-views.md)** — `CREATE MATERIALIZED VIEW`, the difference between a stored result and a recomputed one, `REFRESH MATERIALIZED VIEW` (and `CONCURRENTLY`), the staleness trade-off, and when a materialized view is the right tool versus a regular view or a plain table.
4. **[Module Summary](04-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 9 (Aggregation)** — `GROUP BY`, `HAVING`, and the aggregate functions (`SUM`, `COUNT`, `AVG`, etc.) are used constantly in this module's examples, since one of the most common reasons to create a view is to hide an aggregation behind a simple name. You also need to already understand *why* a `GROUP BY` query collapses many rows into one, because that same collapsing is exactly why aggregate views resist being automatically updatable (Topic 2).
- **Module 10 (Joins & Set Operations)** — most views worth creating wrap a multi-table join (customers joined to orders joined to order line items, for example), so you need to be comfortable reading and writing `INNER JOIN`/`LEFT JOIN` queries before this module's examples will make sense.
- **Module 6 (Modifying Data)** — Topic 2 of this module (Updatable Views) runs `INSERT`, `UPDATE`, and `DELETE` statements directly against views, so you need to already be comfortable with the syntax and behavior of all three.
- **Module 5 (Constraints & Keys)** — understanding primary keys and foreign keys is what lets you reason about whether a join view has an unambiguous, one-row-to-one-row mapping back to a single base table row — the exact question that determines updatability in Topic 2.
- **Module 4 (Database & Table Design)**, specifically its `CREATE`/`ALTER`/`DROP` patterns — views follow an analogous lifecycle (`CREATE VIEW`, `CREATE OR REPLACE VIEW`, `DROP VIEW ... CASCADE`), and this module assumes that DDL rhythm is already familiar.

## How to Study This Module

Start with Topic 1 — it is the conceptual foundation for the entire module (a view is a saved query, not stored data) and every later topic depends on that mental model being solid before adding nuance. Topic 2 (Updatable Views) is the topic most people get subtly wrong in practice — read it slowly, actually run the examples where a row silently "leaks" out of a filtered view, and make sure the `WITH CHECK OPTION` fix genuinely clicks before moving on. Topic 3 (Materialized Views) is conceptually simpler than Topic 2 but is where real production performance decisions get made — pay close attention to the staleness trade-off discussion, since choosing a materialized view when you actually needed always-current data (or vice versa) is a common and costly real-world mistake. By the end of this module you should be able to look at any reporting dashboard, admin panel, or "simplified read-only interface" in a real system and correctly guess whether it's backed by a plain view, an updatable view, or a materialized view.
