# Module 10 — Joins and Set Operations

## Module Goal

By the end of this module, you will be able to combine data spread across multiple tables — the single most common thing you do in real SQL, because well-designed relational schemas deliberately split data into many narrow, related tables rather than one giant one (Module 4; Module 5 — Constraints & Keys). You will understand every kind of join (`INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, `CROSS`, and self-joins), how to chain several tables together in one query, and how to combine entire result sets vertically with `UNION`, `INTERSECT`, and `EXCEPT`. Almost nothing you build after this module — reporting queries, subqueries (Module 11), window functions (Module 16) — makes sense without this foundation, because almost no real question is answerable from a single table alone.

## Topics Covered in This Module

1. **[INNER JOIN](01-inner-join.md)** — combining rows from two tables based on a matching condition, explicit `JOIN...ON` syntax vs. old-style comma/`WHERE` joins, and why `INNER JOIN` excludes non-matching rows.
2. **[LEFT and RIGHT OUTER JOIN](02-left-and-right-outer-join.md)** — preserving every row from one side with `NULL`s filling in for non-matches, why `LEFT JOIN` dominates in practice, and the "find rows with no match" pattern.
3. **[FULL OUTER JOIN and CROSS JOIN](03-full-outer-join-and-cross-join.md)** — preserving unmatched rows from both sides at once, the Cartesian product produced by `CROSS JOIN`, and the danger of an accidental cross join.
4. **[Self-Joins](04-self-joins.md)** — joining a table to itself using aliases, for hierarchical data (employee/manager) and finding pairs or duplicates within a single table.
5. **[Joining Multiple Tables](05-multiple-table-joins.md)** — chaining three or more tables, aliasing conventions for readability, and how the query planner — not the order you write joins in — decides actual execution order.
6. **[UNION, INTERSECT, and EXCEPT](06-union-intersect-except.md)** — combining entire result sets vertically instead of horizontally, `UNION` vs. `UNION ALL`, and the column-count/type-compatibility rule that governs all of them.
7. **[Module Summary](07-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 5 — Constraints & Keys**, specifically primary keys and foreign keys. A join condition is, in the overwhelming majority of real queries, exactly the relationship a foreign key already declares (a `customer_id` column in an `orders` table pointing back to `customers.customer_id`). If you're fuzzy on what a foreign key guarantees, joins will feel like arbitrary column-matching rather than "reconnecting data that a foreign key already promised was related."
- **Module 7 — Querying Basics**, specifically `SELECT`, `WHERE`, `ORDER BY`, and `LIMIT`. Every join in this module is written inside an otherwise ordinary `SELECT` statement, and `WHERE`/`ORDER BY` compose with joined results exactly as they did with a single table — this module doesn't reintroduce those clauses, it assumes you can already read them fluently.
- **Module 4 — Database & Table Design**, for general comfort with `CREATE TABLE` and the idea that data is deliberately split across multiple tables rather than duplicated into one.

## How to Study This Module

Topics 1–3 (`INNER`, `LEFT`/`RIGHT`, `FULL OUTER`/`CROSS`) form one continuous idea — four flavors of the same `JOIN...ON` syntax that differ only in which unmatched rows they keep — so read them back-to-back rather than spacing them out, and run every example yourself; join behavior is far easier to internalize by watching rows appear and disappear than by reading a description of it. Topic 4 (self-joins) is a small but important twist on the same syntax and is worth slowing down for, since it's a common point of confusion the first time you see it. Topic 5 (multiple tables) is mostly a readability and reasoning discipline built on Topics 1–4, not new syntax. Topic 6 (`UNION`/`INTERSECT`/`EXCEPT`) is conceptually distinct from everything before it in this module — joins combine tables *side-by-side* (more columns), set operations combine result sets *top-to-bottom* (more rows) — so don't let the fact that it's in the same module blur that distinction.

### The Running Schema

Every topic in this module reuses the same three-table schema: a store's customers, the orders they place, and the individual line items within each order. This mirrors a completely ordinary real-world design: rather than repeating a customer's name and email on every single line item ever ordered, the data is split across three related tables and reconnected on demand with joins.

```
   customers                         orders                              order_items
  ┌───────────────┐                 ┌─────────────────────────┐         ┌──────────────────────────┐
  │ customer_id PK│◄───────┐        │ order_id       PK       │◄───┐    │ order_item_id     PK      │
  │ name          │        │        │ customer_id    FK ──────┼────┘    │ order_id          FK ─────┼──┐
  │ email         │        └────────┼─(references customers) │         │ product_name              │  │
  │ city          │        1     ∞  │ order_date              │  1   ∞  │ quantity                  │  │
  └───────────────┘                 │ status                  │         │ unit_price                │  │
                                     └─────────────────────────┘         └──────────────────────────-┘  │
                                                                                                          │
                                     (order_items.order_id references orders.order_id) ───────────────────┘
```

- One customer can place **many** orders; every order belongs to **at most one** customer (`orders.customer_id` is nullable, to allow for a guest checkout with no linked customer account — this detail matters in Topic 3).
- One order can contain **many** order items; every order item belongs to **exactly one** order.

This is precisely the one-to-many relationship pattern from Module 5's foreign keys, appearing twice in a row — and reconnecting these split-apart tables back into one readable result is exactly what this entire module teaches you to do.
