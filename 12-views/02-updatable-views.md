# Updatable Views

## Learning Objectives

By the end of this section you should be able to:
- State the precise conditions under which PostgreSQL automatically allows `INSERT`/`UPDATE`/`DELETE` through a view
- Create a simple, automatically updatable view and confirm that writes through it correctly affect the base table
- Explain the problem `WITH CHECK OPTION` solves, and add it to a view that should enforce it
- Explain, in terms of ambiguity rather than just "it's a rule," why join views and aggregate views are not automatically updatable
- State that `INSTEAD OF` triggers are the general-purpose escape hatch for writing through any view, without yet needing to build one

## Prerequisites

- [Creating and Using Views](01-creating-and-using-views.md) — this topic assumes you're comfortable creating a view and querying it; here we start writing through one.
- Module 6 (Modifying Data) — you need to already be fluent with `INSERT`, `UPDATE`, and `DELETE` syntax against ordinary tables, since every example below runs those same statements against a view instead.
- Module 9 (Aggregation) and Module 10 (Joins & Set Operations) — the second half of this topic explains why views built on `GROUP BY` or multi-table joins resist automatic updatability, which only makes sense if you're already comfortable with what those queries do to the shape of the result.

## Motivation

A view that can only be read from is useful, but it's only half the story. Plenty of real scenarios call for a restricted, simplified *write* interface too: a support team that should only ever see and edit orders in "pending" status, a regional team that should only be able to add customers tagged to their own region, or a self-service tool that should let a caller update exactly one narrow slice of a table without exposing (or risking) the rest of it. If a view could only be queried, none of these would be possible without falling back to raw table access — which defeats the entire point of building a simplified, restricted interface in the first place.

## Problem Statement

Suppose a fulfillment team should be able to see and update only customers based in Chennai — adding new ones, correcting details, but never touching customers elsewhere. A first attempt might be a plain filtered view:

```sql
CREATE VIEW chennai_customers AS
SELECT id, name, city, signup_date
FROM customers
WHERE city = 'Chennai';
```

Two questions immediately follow, and neither has an obvious answer without digging into how PostgreSQL actually treats this view: can this team `INSERT` a new customer through it? And if they `UPDATE` a row so its `city` no longer equals `'Chennai'`, what happens — does the row just quietly vanish from their view while still existing, now out of their intended scope, in the real table?

## Concept

### The Conditions for Automatic Updatability

PostgreSQL will automatically translate `INSERT`, `UPDATE`, and `DELETE` statements run against a view into equivalent statements against its base table, **but only when the view is "simple" enough that the mapping from a view row back to exactly one base-table row is unambiguous.** Concretely, a view is automatically updatable only if all of the following hold:

- It has **exactly one table (or another updatable view) in its `FROM` clause** — no joins.
- It contains **no `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`, `OFFSET`, `UNION`/`INTERSECT`/`EXCEPT`, or `WITH`** at the top level.
- Its select list contains **no aggregate functions, window functions, or set-returning functions**.
- Its select list columns map directly to columns of the base table (simple expressions like `name`, or `name AS customer_name`, are fine; computed expressions like `price * 1.1` are readable but not writable through).

The `chennai_customers` view above satisfies every one of these — a single table, no aggregation, no set operations, and every output column maps directly to a real `customers` column. It is automatically updatable.

### Writing Through a Simple, Updatable View

```sql
INSERT INTO chennai_customers (name, city)
VALUES ('Meera', 'Chennai');
```

```
INSERT 0 1
```

PostgreSQL rewrites this into an `INSERT INTO customers (name, city) VALUES ('Meera', 'Chennai')` and it lands as a real row in the base `customers` table — confirmed by querying the base table directly:

```sql
SELECT * FROM customers WHERE name = 'Meera';
```

```
 id  | name  |  city   | signup_date
-----+-------+---------+-------------
 105 | Meera | Chennai | 2026-07-31
(1 row)
```

`UPDATE` and `DELETE` work the same way:

```sql
UPDATE chennai_customers SET city = 'Mumbai' WHERE name = 'Priya';
```

```
UPDATE 1
```

This succeeds — but notice what it just did. `Priya`'s row was updated to `city = 'Mumbai'` in the real `customers` table, and the view's own `WHERE city = 'Chennai'` clause means that row **no longer appears in `chennai_customers` at all**. The fulfillment team, who should only ever be touching Chennai customers, was just allowed to move a row entirely out of their own scope of visibility with a single `UPDATE`. This is the problem `WITH CHECK OPTION` exists to close.

### WITH CHECK OPTION

Adding `WITH CHECK OPTION` to a view's definition tells PostgreSQL to reject any `INSERT` or `UPDATE` through the view that would produce a row failing the view's own `WHERE` condition:

```sql
CREATE OR REPLACE VIEW chennai_customers AS
SELECT id, name, city, signup_date
FROM customers
WHERE city = 'Chennai'
WITH CHECK OPTION;
```

Now the same update is rejected outright, before it ever touches the base table:

```sql
UPDATE chennai_customers SET city = 'Mumbai' WHERE name = 'Priya';
```

```
ERROR:  new row violates check option for view "chennai_customers"
DETAIL:  Failing row contains (id, Priya, Mumbai, 2026-01-14).
```

The same protection applies to `INSERT`: trying to insert a row through `chennai_customers` with `city = 'Mumbai'` fails for the identical reason — the row that would result doesn't satisfy the view's `WHERE city = 'Chennai'` clause. `WITH CHECK OPTION` is what turns a filtered, writable view into a genuine boundary rather than a one-way window that writes can silently escape through. (When views are stacked on top of each other, PostgreSQL also supports `LOCAL` and `CASCADED` check options to control whether only the immediate view's condition is checked or every view's condition in the stack — the default, `CASCADED`, checks all of them.)

### Why Join and Aggregate Views Aren't Automatically Updatable

Recall `customer_order_summary` from the previous topic:

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

Trying to write through it fails immediately:

```sql
UPDATE customer_order_summary SET total_spent = 7000 WHERE customer_id = 1;
```

```
ERROR:  cannot update view "customer_order_summary"
DETAIL:  Views containing GROUP BY are not automatically updatable.
HINT:  To enable updating the view, provide an INSTEAD OF UPDATE trigger or an unconditional ON UPDATE DO INSTEAD rule.
```

This isn't an arbitrary restriction — it reflects a genuine ambiguity. `total_spent` is the sum of `quantity * unit_price` across a variable number of `order_items` rows belonging to a variable number of `orders` rows. If you set `total_spent = 7000`, there is no defined way to translate that single number back into specific changes to specific `order_items` rows — should one item's price change? Should a new phantom row be invented? Should every item's price be scaled proportionally? SQL has no built-in answer, so PostgreSQL refuses rather than guess.

The same ambiguity shows up with joins even without any aggregation. Consider a view joining `orders` to `customers`:

```sql
CREATE VIEW order_details AS
SELECT o.id AS order_id, o.status, c.name AS customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

This view fails the "exactly one table in `FROM`" rule, so it is not automatically updatable either — even though, unlike the aggregate case, each output row *does* correspond to exactly one `orders` row and one `customers` row. PostgreSQL's automatic updatability check doesn't attempt to prove that a specific join is safe on a case-by-case basis; the "single table only" rule is a blanket, conservative line, because verifying safety for every possible join shape in general would require far more complex reasoning than the automatic mechanism is designed to do.

### INSTEAD OF Triggers — the General Escape Hatch

None of this means join and aggregate views can *never* be written through — it means PostgreSQL won't do it automatically. The general mechanism for making an arbitrarily complex view writable is an **`INSTEAD OF` trigger**: procedural code you attach to the view that intercepts an `INSERT`, `UPDATE`, or `DELETE` attempt and explicitly decides how it maps back onto one or more base tables — for example, an `INSTEAD OF UPDATE` trigger on `order_details` could decide that updating `status` through the view should update the corresponding row in `orders`, while updating `customer_name` should update the corresponding row in `customers`. Writing one is a full topic in its own right and is covered in depth in Module 18 (Procedures, Functions & Triggers). For now, the important takeaway is simply that this escape hatch exists and is the answer whenever a view you need to write through doesn't qualify for automatic updatability.

## Internal Working (Preview)

An `INSERT`/`UPDATE`/`DELETE` against a view goes through the same rewrite-rule mechanism described in Topic 1 for `SELECT`, but with an extra check performed before the rewrite is even attempted:

```
UPDATE chennai_customers SET city = 'Mumbai' WHERE name = 'Priya'
        │
        ▼
Is this view "simple" enough for an unambiguous 1-row-to-1-row mapping?
 (single table in FROM? no GROUP BY/DISTINCT/aggregates/set ops?)
        │
   ┌────┴────┐
  YES        NO
   │          │
   ▼          ▼
Rewrite into  Refuse at the rule-rewrite stage, before touching any data —
an equivalent  unless an INSTEAD OF trigger is defined, in which case that
UPDATE against  trigger's procedural logic runs instead of the automatic
the base table  rewrite (Module 18)
   │
   ▼
If WITH CHECK OPTION is present, additionally verify the resulting row
still satisfies the view's WHERE clause — reject with an error if not
```

The key detail is that this decision is made structurally, by inspecting the view's definition, not by looking at the actual data being written. That's why the `GROUP BY` error above appears instantly, regardless of which row or value you attempted to update — PostgreSQL doesn't need to look at any data to know the mapping is undefined in general.

## Real-World Analogy

Think of a company mailroom. A view over a single employee's own inbox is simple: if you leave a corrected memo in that slot, it's obvious whose inbox it belongs to — one slot, one recipient, no ambiguity. `WITH CHECK OPTION` is like a mailroom rule that refuses to let you drop a memo addressed to someone else into a slot reserved for one department — if the memo's addressee doesn't match the slot's own labeling rule, it's rejected on the spot rather than filed somewhere it shouldn't be.

Now imagine a "monthly department totals" report, stapled together by combining and summing several employees' individual expense reports. If you tried to hand the mailroom a correction — "change this department's total to $7,000" — the mailroom has no way to know which original expense report, filed under which original employee, that correction is supposed to modify. It would have to guess, so instead it refuses automatically. The only way to make that correction stick is to have an actual person (an `INSTEAD OF` trigger) read the correction and manually decide which underlying expense report(s) to actually change.

## Why Updatable Views Were Designed This Way

PostgreSQL's guiding principle here is safety through conservatism: automatic updatability is only granted when the mapping between a view row and a single base-table row is *provably* one-to-one and unambiguous. Anything murkier — a join that might be one-to-many, an aggregate that collapses many rows into one — risks silently corrupting data in an undefined way if the database simply guessed at a translation. Refusing outright, with a clear error, is safer than inventing an implicit rule that might surprise you in a way you wouldn't discover until data was already wrong. This is consistent with the same integrity-first philosophy behind constraints (Module 5) and behind requiring explicit, deliberate statements for destructive operations (Module 6): PostgreSQL prefers a loud, immediate failure over silent, ambiguous behavior.

## Advantages

- **Lets you build restricted write interfaces without application code** — a single-table, filtered, `WITH CHECK OPTION`-protected view gives a class of user real `INSERT`/`UPDATE`/`DELETE` access, scoped to exactly the rows they should touch, entirely inside the database.
- **`WITH CHECK OPTION` closes a real, easy-to-miss hole** — without it, a filtered writable view can be used to move a row silently out of (or into) its intended scope; with it, that's a hard error instead of a silent data-integrity surprise.
- **The automatic-updatability check is structural, not data-dependent** — you get an immediate, predictable answer ("this view type is/isn't updatable") from the view's definition alone, without needing to reason about specific data to know whether a given `UPDATE` will be accepted.

## Disadvantages / Limitations

- **Automatic updatability is narrow** — it excludes joins and aggregations, which are exactly the kinds of views most worth creating in the first place (Topic 1's entire motivating example, `customer_order_summary`, is not automatically updatable).
- **`WITH CHECK OPTION` only checks the view's own filter condition** — it is not a substitute for real constraints (Module 5) or permission checks (Module 19); it guarantees a written row satisfies the view's `WHERE` clause, nothing more.
- **`INSTEAD OF` triggers reintroduce procedural code** — once you need one, you've traded the declarative simplicity that made the view attractive in the first place for hand-written logic that must be written, tested, and maintained like any other program code.

## Best Practices

- Don't rely on updatability as an accident of a view's shape — if a view is meant to be written through, design it deliberately as a single-table, filter-only view, and add `WITH CHECK OPTION` explicitly rather than assuming the default behavior is safe.
- Verify whether a view is actually updatable by querying `information_schema.views` (`is_updatable`, `is_insertable_into`) rather than assuming from its definition — it's a precise, checkable fact, not a judgment call.
- Add `WITH CHECK OPTION` to any filtered view exposed to a user or team who could otherwise write a row out of (or into) a scope they shouldn't have — treat its absence on a filtered writable view as a bug, not an oversight to fix later.
- Reach for `INSTEAD OF` triggers only when a genuinely complex view needs to be writable — if a simpler, single-table updatable view (or plain, direct table access guarded by permissions, Module 19) would satisfy the requirement, prefer that over the added complexity of trigger logic.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming any view supports `INSERT`/`UPDATE`/`DELETE` | Only "simple" views — single table in `FROM`, no `DISTINCT`/`GROUP BY`/`HAVING`/aggregates/set operations — are automatically updatable. Anything else requires an `INSTEAD OF` trigger. |
| Building a filtered, writable view without `WITH CHECK OPTION` | Without it, an `UPDATE` (or `INSERT`) through the view can silently produce a row that no longer satisfies the view's own filter — the row "escapes" the intended scope without any error. |
| Trying to `UPDATE` a column derived from an aggregate or expression | There is no defined way to translate a change to a computed value (a `SUM`, a `COUNT`, an arithmetic expression) back into a change on specific base-table rows; PostgreSQL rejects this at the rule-rewrite stage. |
| Forgetting that join views need an `INSTEAD OF` trigger for writes | Even a join where every output row maps to exactly one row per base table is still rejected automatically, because PostgreSQL's blanket rule requires a single table in `FROM` — it does not attempt to verify join-specific safety case by case. |

## Interview Questions

1. **Q: What are the minimum conditions for a PostgreSQL view to be automatically updatable?**
   A: Exactly one table (or another updatable view) in its `FROM` clause, no `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`/`OFFSET`, set operations, or `WITH`, and a select list free of aggregate functions, window functions, and set-returning functions, with each output column mapping directly to a base-table column.

2. **Q: What problem does `WITH CHECK OPTION` solve, and what's a concrete scenario where its absence causes a real issue?**
   A: Without it, a row written through a filtered, updatable view can end up failing that view's own `WHERE` condition — effectively "escaping" the scope the view was meant to enforce, silently. For example, a view scoped to one region's customers would otherwise let someone `UPDATE` a customer's region to something outside that scope without any error, even though the whole point of the view was to restrict them to that one region.

3. **Q: Why can't you `UPDATE` a column in a view that comes from `SUM(...)` or a multi-table join, even though the underlying data clearly changes something?**
   A: Because there's no unambiguous way to map the change back to specific base-table rows. An aggregate like `SUM` collapses many rows into one output value — changing that value has no defined effect on the individual rows it summed. A join can combine rows from multiple tables in ways PostgreSQL's automatic mechanism doesn't attempt to verify are safe on a case-by-case basis, so it rejects all joins uniformly rather than reasoning about specific ones.

4. **Q: If a view genuinely needs to support writes but doesn't qualify for automatic updatability, what's the general solution?**
   A: An `INSTEAD OF` trigger attached to the view, which intercepts `INSERT`/`UPDATE`/`DELETE` attempts and runs custom procedural logic to decide exactly how the change should be applied to one or more base tables. This is covered in depth in Module 18 (Procedures, Functions & Triggers).

## Summary

- PostgreSQL automatically allows writes through a view only when it's "simple": a single table in `FROM`, no aggregation, no `DISTINCT`/`GROUP BY`/`HAVING`, and no set operations, with output columns mapping directly to base-table columns.
- `WITH CHECK OPTION` closes a real gap in filtered, writable views — it rejects any `INSERT`/`UPDATE` that would produce a row failing the view's own `WHERE` clause, instead of letting that row silently leave the view's intended scope.
- Join views and aggregate views aren't automatically updatable because there is no unambiguous way to map a change back to specific base-table rows — the restriction reflects a genuine mapping problem, not an arbitrary limitation.
- `INSTEAD OF` triggers are the general, unrestricted escape hatch for writing through any view, at the cost of reintroducing hand-written procedural logic; they're covered in full in Module 18.
- The next topic, Materialized Views, shifts focus from *writability* to *performance* — a different (and, for regular views, entirely separate) axis of how a view behaves.
