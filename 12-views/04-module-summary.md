# Module 12 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Creating and Using Views** — `CREATE VIEW` as a saved, named query; querying a view exactly like a table; `CREATE OR REPLACE VIEW` and its column-compatibility restriction; `DROP VIEW` with `CASCADE`; and how views deliver reuse, abstraction, and simplification for complex joins and aggregations.
- [x] **Updatable Views** — the exact conditions under which PostgreSQL allows `INSERT`/`UPDATE`/`DELETE` through a view automatically, `WITH CHECK OPTION` for preventing rows from escaping a filtered view's scope, why join and aggregate views resist automatic updatability, and `INSTEAD OF` triggers as the general escape hatch.
- [x] **Materialized Views** — `CREATE MATERIALIZED VIEW`, the difference between a stored result and a recomputed one, `REFRESH MATERIALIZED VIEW` (and `CONCURRENTLY`'s unique-index requirement), the staleness-versus-speed trade-off, and choosing between a regular view, a materialized view, and a plain table.

## Practical Connections

- **Reporting dashboards** at any real scale almost always sit on top of views (for a simple, stable query interface) or materialized views (when the underlying aggregation is too expensive to recompute on every page load) rather than raw, repeated ad-hoc SQL scattered across an application's codebase.
- **Restricted self-service tools** — a support console that lets an operator update only certain rows, or a partner-facing interface that exposes only a filtered slice of data — commonly rely on updatable, `WITH CHECK OPTION`-protected views to enforce their boundaries directly in the database, rather than trusting application code alone to enforce them.
- **Nightly or hourly batch-driven analytics** (end-of-day revenue summaries, usage reports generated once and read many times through the day) are a natural fit for materialized views, refreshed on a predictable schedule after each batch load completes.
- **Layered views built on other views** let a team build increasingly high-level, business-friendly abstractions (a "customer summary" view feeding into a "top customers" view, for instance) without duplicating the underlying join and aggregation logic at each layer.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Regular view vs. materialized view | A regular view stores only the query and recomputes it on every reference; a materialized view physically stores the query's result and only updates it when explicitly refreshed. |
| `CREATE OR REPLACE VIEW` vs. `DROP VIEW` + `CREATE VIEW` | `CREATE OR REPLACE VIEW` can only append new output columns; changing, removing, or reordering existing columns requires dropping and recreating the view. |
| A view being queryable vs. a view being updatable | Every view can be queried with `SELECT`; only "simple" views (single table, no aggregation/`DISTINCT`/`GROUP BY`/set operations) are automatically writable with `INSERT`/`UPDATE`/`DELETE`. |
| `WITH CHECK OPTION` vs. ordinary constraints | `WITH CHECK OPTION` only enforces that a written row still satisfies the view's own `WHERE` clause; it is not a replacement for `NOT NULL`, `UNIQUE`, `CHECK`, or foreign key constraints (Module 5), which are enforced independently at the table level. |
| `REFRESH MATERIALIZED VIEW` vs. `REFRESH MATERIALIZED VIEW CONCURRENTLY` | The plain form takes a lock that blocks readers for the duration of the refresh; `CONCURRENTLY` avoids that block but requires a unique index and does more work to compute a row-level diff. |

## What's Next

This module gave you a full toolkit for treating a saved query as a first-class database object: a plain view for reuse and abstraction, an updatable view (with `WITH CHECK OPTION`) for controlled writes, and a materialized view for turning an expensive, frequently-read query into a cheap one. **Module 13 — Indexes** builds directly on top of this: you'll learn how B-tree indexes work, how the query planner decides whether to use one, and how to read an `EXPLAIN` plan — including how to index a materialized view's stored data to make reads against it even faster, and how to reason about whether an expensive query (view-wrapped or not) is actually a good candidate for indexing versus materializing in the first place.
