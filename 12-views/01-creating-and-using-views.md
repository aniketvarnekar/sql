# Creating and Using Views

## Learning Objectives

By the end of this section you should be able to:
- Explain what a view is in precise terms — a saved, named query, not a copy of data
- Write `CREATE VIEW ... AS SELECT ...` and query the resulting view exactly as you would a table
- Modify a view's definition safely with `CREATE OR REPLACE VIEW`, and state the exact restriction it operates under
- Remove a view with `DROP VIEW`, including when `CASCADE` is required
- Explain, with a concrete example, how views help with reuse, abstraction, and simplifying complex joins/aggregations for the people who need to query the data

## Prerequisites

This is the first topic in this module, so it leans directly on the module-level prerequisites: comfort reading and writing multi-table joins (Module 10) and `GROUP BY` aggregation (Module 9), since the motivating example below combines both. It also assumes the `CREATE`/`DROP` DDL rhythm from Module 4 (Database & Table Design).

## Motivation

Every non-trivial database eventually accumulates a handful of queries that get run over and over, by different people, in slightly different forms — "give me each customer's total completed spend," "show me this month's revenue by category," "list only the active accounts in this region." If every person who needs that answer writes their own version of the query from scratch, you get small, silent divergences: one person filters out cancelled orders, another forgets to; one person joins on the right key, another makes an off-by-one mistake in a date range. A view lets you write the query *once*, give it a name, and let everyone — including tools that only know how to run `SELECT * FROM some_name` — query that single, agreed-upon definition instead.

## Problem Statement

Suppose you're tracking `customers`, `orders`, and `order_items` (an order can contain several line items, each for a product at a given quantity and price), and the business question "how much has each customer actually spent on completed orders?" comes up constantly — from a support tool, a monthly report, and an analyst's ad-hoc queries. Without a view, every one of those consumers writes something like this from scratch:

```sql
SELECT
    c.id AS customer_id,
    c.name AS customer_name,
    COUNT(DISTINCT o.id) AS total_orders,
    SUM(oi.quantity * oi.unit_price) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY c.id, c.name;
```

That's a three-table join with a filter and an aggregation — not something you want to ask every analyst and every internal tool to retype correctly, every time, forever. If the support tool's author forgets the `WHERE o.status = 'completed'` filter, their numbers will quietly disagree with the monthly report's numbers, and nobody will notice until a customer complains their total looks wrong.

## Concept

### A View Is a Saved Query, Not Stored Data

A **view** is a named object in the database that stores a `SELECT` statement — not the rows that statement currently returns. Every time you query a view, PostgreSQL re-runs the underlying `SELECT` against the current data in the base tables. This is the single most important fact about views: **a regular view holds zero rows of its own.** (Module 12's Topic 3 covers *materialized* views, which are the deliberate exception to this rule.)

Turning the query above into a view:

```sql
CREATE VIEW customer_order_summary AS
SELECT
    c.id AS customer_id,
    c.name AS customer_name,
    COUNT(DISTINCT o.id) AS total_orders,
    SUM(oi.quantity * oi.unit_price) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY c.id, c.name;
```

`CREATE VIEW` is DDL (Module 1, Topic 3) — it changes the database's structure (it adds a new named object to the catalog), even though what it stores is a query rather than a table shape.

### Querying a View Like a Table

Once created, `customer_order_summary` can be queried with ordinary `SELECT` — no special syntax, no indication to the reader that it isn't a real table:

```sql
SELECT *
FROM customer_order_summary
ORDER BY total_spent DESC;
```

```
 customer_id | customer_name | total_orders | total_spent
-------------+---------------+--------------+-------------
           1 | Asha          |            2 |        6342
           3 | Priya         |            1 |        1546
(2 rows)
```

Notice two things about this result. First, it reflects only *completed* orders, because that filter is baked into the view — nobody querying `customer_order_summary` can accidentally forget it. Second, customers with no completed orders (a customer whose only order was cancelled, or who has no orders at all) don't appear at all, because the `JOIN`s and `WHERE` clause inside the view exclude them, exactly as they would in the raw query.

You can filter, sort, and even join a view with other tables or views, exactly as with a real table:

```sql
SELECT customer_name, total_spent
FROM customer_order_summary
WHERE total_spent > 2000
ORDER BY total_spent DESC;
```

```
 customer_name | total_spent
---------------+-------------
 Asha          |        6342
(1 row)
```

Nothing about this query needs to know that `customer_order_summary` is a view rather than a table — that's the entire point.

### CREATE OR REPLACE VIEW

Requirements change, and a view's definition will need to evolve. Rather than dropping and recreating a view (which would also require recreating anything that depends on it), PostgreSQL lets you redefine it in place:

```sql
CREATE OR REPLACE VIEW customer_order_summary AS
SELECT
    c.id AS customer_id,
    c.name AS customer_name,
    COUNT(DISTINCT o.id) AS total_orders,
    SUM(oi.quantity * oi.unit_price) AS total_spent,
    ROUND(SUM(oi.quantity * oi.unit_price)::numeric / COUNT(DISTINCT o.id), 2) AS avg_order_value
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY c.id, c.name;
```

This succeeds because the new query only *adds* a column (`avg_order_value`) at the end — every existing column keeps the same name, position, and data type. `CREATE OR REPLACE VIEW` has one hard restriction: **it cannot remove, reorder, or change the data type of an existing output column.** Trying to remove `total_orders` from the same view fails outright:

```sql
CREATE OR REPLACE VIEW customer_order_summary AS
SELECT
    c.id AS customer_id,
    c.name AS customer_name,
    SUM(oi.quantity * oi.unit_price) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY c.id, c.name;
```

```
ERROR:  cannot drop columns from view
```

To make that kind of change, you must `DROP` the view and `CREATE` it fresh (covered next) — PostgreSQL refuses to silently change a column's identity out from under anything that might already depend on it.

### DROP VIEW

Removing a view uses the same shape as removing a table:

```sql
DROP VIEW IF EXISTS customer_order_summary;
```

`IF EXISTS` avoids an error if the view was never created (or was already dropped) — useful in scripts that might run more than once. If another view depends on the one you're dropping, PostgreSQL refuses by default:

```sql
CREATE VIEW top_customers AS
SELECT * FROM customer_order_summary WHERE total_spent > 5000;

DROP VIEW customer_order_summary;
```

```
ERROR:  cannot drop view customer_order_summary because other objects depend on it
DETAIL:  view top_customers depends on view customer_order_summary
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```

As with `DROP TABLE`, adding `CASCADE` drops the dependent objects along with it:

```sql
DROP VIEW customer_order_summary CASCADE;
```

This also silently drops `top_customers` — use it deliberately, not reflexively, since it can remove objects you forgot depended on the one you're targeting.

### Why Views Help: Reuse, Abstraction, and Simplification

Three concrete benefits fall out of everything above:

- **Reuse** — the join-plus-aggregation logic for "customer order summary" is written once, in one place. Every consumer — a report, a dashboard, an ad-hoc query, another view — gets the same answer, because they're all running the exact same underlying query.
- **Abstraction** — someone querying `customer_order_summary` doesn't need to know it spans three tables, or that it filters on order status, or that it uses `COUNT(DISTINCT ...)` to avoid double-counting orders with multiple line items. They see a simple, flat result: customer, order count, total spent. The complexity is still there — it's just been moved behind a name.
- **Simplifying access for less SQL-fluent consumers** — a business intelligence tool, a less experienced analyst, or a support engineer writing a one-off query can `SELECT * FROM customer_order_summary WHERE customer_name = 'Asha'` without ever writing a `JOIN` or a `GROUP BY` themselves. This same idea, applied to restricting which *columns* or *rows* a user can see rather than simplifying a join, is also the basis of a common security pattern covered in depth in Module 19 (Security & Access Control): granting access to a view instead of the underlying tables.

You can list every view in the current database, and inspect one, through PostgreSQL's catalog views:

```sql
SELECT viewname FROM pg_views WHERE schemaname = 'public';
```

```
        viewname
------------------------
 customer_order_summary
(1 row)
```

```sql
\d customer_order_summary
```

```
                        View "public.customer_order_summary"
     Column      |  Type   | Collation | Nullable | Default
------------------+---------+-----------+----------+---------
 customer_id      | integer |           |          |
 customer_name    | text    |           |          |
 total_orders     | bigint  |           |          |
 total_spent      | numeric |           |          |
```

## Internal Working (Preview)

A view does not store a query as text that gets re-parsed on every use. When you run `CREATE VIEW`, PostgreSQL parses the `SELECT` once and stores its internal representation — a **rewrite rule** — in the system catalogs. When a later query references the view, the planner substitutes the view's stored definition directly into the outer query before optimizing anything, a process usually called **view expansion** or **inlining**:

```
SELECT customer_name, total_spent
FROM customer_order_summary
WHERE total_spent > 2000
        │
        ▼  (rewriter substitutes the view's stored definition in place of its name)
SELECT customer_name, total_spent
FROM (
    SELECT c.id AS customer_id, c.name AS customer_name,
           COUNT(DISTINCT o.id) AS total_orders,
           SUM(oi.quantity * oi.unit_price) AS total_spent
    FROM customers c
    JOIN orders o ON o.customer_id = c.id
    JOIN order_items oi ON oi.order_id = o.id
    WHERE o.status = 'completed'
    GROUP BY c.id, c.name
) AS customer_order_summary
WHERE total_spent > 2000
        │
        ▼
   Query planner optimizes the *entire combined query* as one unit
   (it does not run the view's query first and filter the result second)
```

This matters in practice: the planner is free to push your outer `WHERE total_spent > 2000` down, reorder joins, or choose different access paths than it would if the view's query were genuinely run in isolation first. A view is a convenience for *you*, the person writing SQL — it is not, by itself, a performance boundary or a separately-executed step. That distinction is exactly why an expensive view queried often doesn't get any faster just because it's wrapped in a view — a concern Topic 3 addresses directly with materialized views.

## Real-World Analogy

A view is like a restaurant menu item, not a pre-made dish sitting in a refrigerator. The menu lists "Grilled Salmon Plate" as a simple, single thing you can order — you don't need to know the kitchen actually combines salmon from the walk-in, vegetables from a separate prep station, and sauce from yet another station every single time. Ordering the same dish tomorrow doesn't reheat yesterday's plate — the kitchen (the query planner) recombines the same fresh raw ingredients (the base tables) again, following the same recipe (the stored query), every time it's ordered. The menu name is an abstraction over real, re-executed work — exactly like a view.

## Why Views Were Designed This Way

The relational model (Module 2) treats the result of a query over relations as itself a relation — a table of rows and columns, with no distinction in *shape* from a base table. Views are the direct, practical consequence of that theoretical closure property: if a query's output is "just another relation," it's natural to let the database name that relation and let you query it identically to any base table, without inventing a separate, second kind of object with different rules. This is also why views compose so cleanly — a view can be built on top of another view (as `top_customers` was built on top of `customer_order_summary` above), because from SQL's point of view, there's no structural difference between querying a table and querying a view.

## Advantages

- **Single source of truth for a query's logic** — a filter, join, or calculation defined once in a view can't silently drift between different consumers the way copy-pasted SQL can.
- **Insulates consumers from schema complexity** — a three-table join with an aggregation becomes a single flat name; consumers who only need the answer don't need to understand (or maintain) how it's derived.
- **A foundation for access control** — granting `SELECT` on a view instead of the underlying tables is a standard way to expose only certain rows or columns to certain users (Module 19 covers the permission side of this in depth).
- **Composable** — views can reference other views, letting you build progressively higher-level abstractions (a "top customers" view built on a "customer summary" view, for instance) without duplicating logic at each layer.
- **Cheap to change** — `CREATE OR REPLACE VIEW` lets you evolve a view's logic (within its column-compatibility constraint) without touching every place that queries it.

## Disadvantages / Limitations

- **No inherent performance benefit** — as the Internal Working section showed, a view is expanded into the outer query at query time; an expensive aggregation wrapped in a view is exactly as expensive every time it's queried. Views solve a reuse/abstraction problem, not a performance problem (Topic 3's materialized views address performance directly).
- **Adds a layer of indirection when debugging** — reading an `EXPLAIN` plan (Module 20) for a query built on several nested views can be harder to trace back to the original SQL than a single flat query would be.
- **Silent breakage on base-table changes** — dropping or renaming a column that a view depends on breaks the view; PostgreSQL will refuse the underlying `ALTER`/`DROP` in some cases, but in others (e.g., a `SELECT *` view whose base table gains an incompatible new column type) the failure only surfaces the next time the view is queried.
- **The column-compatibility restriction on `CREATE OR REPLACE VIEW`** means genuine restructuring (removing/reordering columns) always requires a `DROP`/`CREATE` cycle, which in turn requires thinking through `CASCADE` and everything that currently depends on the view.

## Best Practices

- Give views clear, descriptive names that describe *what* they return (`customer_order_summary`, not `v1` or `temp_view`) — a view's name is its entire interface to everyone who isn't you.
- Avoid `SELECT *` inside a view's definition; list columns explicitly, so an unrelated change to a base table (a new column added elsewhere) doesn't silently and unexpectedly change the view's output shape.
- Keep each view focused on one clear purpose rather than building a single sprawling "do everything for everyone" view — it's easier to reason about, test, and safely evolve several small views than one that tries to satisfy every consumer at once.
- Before dropping a view, check `pg_depend` (or just attempt the `DROP` without `CASCADE` first and read the error) to see what depends on it, rather than reaching for `CASCADE` reflexively.
- Document a non-obvious view's purpose and any filtering assumptions it bakes in (e.g., "excludes cancelled orders") with `COMMENT ON VIEW customer_order_summary IS '...'`, since that assumption is invisible to anyone just reading `SELECT * FROM customer_order_summary`.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a view stores its own copy of data | A regular view stores only the query; it is re-executed against current base-table data on every reference. It has no data of its own to go stale — Topic 3 (Materialized Views) is the object that actually stores a result. |
| Trying to reorder or remove a column with `CREATE OR REPLACE VIEW` | PostgreSQL only allows appending new columns at the end; changing, removing, or reordering existing output columns requires `DROP VIEW` followed by a fresh `CREATE VIEW`. |
| Dropping a view that other views depend on without `CASCADE` | PostgreSQL refuses by default and reports the dependency; either resolve the dependent views explicitly or use `CASCADE` deliberately, understanding it will drop them too. |
| Writing `SELECT *` inside a view definition | The view's output columns become whatever the base table currently has, in whatever order — an unrelated later change to the base table (a new column) silently changes the view's shape for every consumer. |

## Interview Questions

1. **Q: What is a view, and how is it fundamentally different from a table?**
   A: A view is a named, saved `SELECT` query stored in the database's catalog. It holds no data of its own — every time it's queried, the underlying query is executed (typically inlined into the outer query by the planner) against the current contents of its base tables. A table, by contrast, physically stores rows.

2. **Q: Can `CREATE OR REPLACE VIEW` always be used to change a view's definition?**
   A: No — it can only be used if the new query's output columns are compatible with the existing ones: the same column names, in the same order, with the same data types, up through however many columns the current view has. New columns may be appended at the end. Removing, reordering, or retyping an existing column requires dropping the view and recreating it.

3. **Q: Give three concrete reasons a team might create a view instead of just sharing the raw SQL.**
   A: Reuse (one definition instead of copy-pasted, potentially diverging SQL), abstraction (consumers see a simple named result instead of needing to understand a multi-table join or aggregation), and access control (granting `SELECT` on a view lets you expose specific rows/columns without granting access to the full underlying tables).

4. **Q: If View B is defined using View A, what happens if you run `DROP VIEW A` without `CASCADE`?**
   A: PostgreSQL refuses the drop and reports that View B depends on View A, along with a hint to use `CASCADE`. Running `DROP VIEW A CASCADE` would drop both View A and View B.

## Summary

- A view is a saved, named `SELECT` query — not a copy of data — created with `CREATE VIEW name AS SELECT ...` and queried exactly like a table.
- `CREATE OR REPLACE VIEW` lets you evolve a view's definition in place, but only by appending new columns; removing, reordering, or retyping existing columns requires dropping and recreating the view.
- `DROP VIEW` follows the same `IF EXISTS`/`CASCADE` pattern as `DROP TABLE`; `CASCADE` is required (and must be used deliberately) when other views depend on the one being dropped.
- Views exist to solve reuse and abstraction problems — a single definition of "what this data means" that every consumer shares — not performance problems; the underlying query is genuinely re-executed (and typically inlined) every time.
- The next topic, Updatable Views, addresses the other half of working with views: under exactly what conditions you can `INSERT`/`UPDATE`/`DELETE` through one, and why most interesting views (joins, aggregates) can't be written through automatically.
