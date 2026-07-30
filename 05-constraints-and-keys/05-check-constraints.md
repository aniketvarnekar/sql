# CHECK Constraints

## Learning Objectives

By the end of this section you should be able to:
- Write single-column and multi-column `CHECK` constraints to validate row data beyond type or uniqueness rules
- Explain where `CHECK` constraints are enforced relative to application-level validation, and why both often coexist
- Identify the specific limitations of `CHECK` constraints — particularly that they cannot reference other rows or other tables

## Prerequisites

- [NOT NULL and DEFAULT](01-not-null-and-default.md) — `CHECK` is best understood as a generalization of the same idea (a rule the database enforces on every write), extended from "is this present" to "does this satisfy an arbitrary condition."
- [Foreign Keys and Referential Integrity](04-foreign-keys-and-referential-integrity.md) — helps to have already seen how PostgreSQL enforces relationships between rows, so this topic's key limitation (that `CHECK` cannot do the same across rows or tables) has a clear contrast to point at.

## Motivation

`NOT NULL` guarantees a value is present. `UNIQUE` guarantees it isn't repeated. `PRIMARY KEY`/`FOREIGN KEY` guarantee identity and valid relationships. None of these can express something as simple as "a price must be greater than zero" or "an order's ship date can't be earlier than its order date." Plenty of business rules are really just logical conditions a row's values must satisfy — and `CHECK` constraints are how you tell the database to enforce those conditions itself, rather than trusting every piece of application code that writes to the table to remember to validate them.

## Problem Statement

Consider a `products` table with a `price` column and an `orders` table tracking dates, with no validation beyond data types:

```sql
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    price      NUMERIC(10, 2) NOT NULL
);

CREATE TABLE orders (
    order_id   SERIAL PRIMARY KEY,
    order_date DATE NOT NULL DEFAULT CURRENT_DATE,
    ship_date  DATE
);

INSERT INTO products (name, price) VALUES ('Desk Lamp', -19.99);
INSERT INTO orders (order_date, ship_date) VALUES ('2026-07-31', '2026-07-15');
```

Both inserts succeed without complaint:

```
 product_id |   name    | price
------------+-----------+--------
          1 | Desk Lamp | -19.99
(1 row)

 order_id | order_date | ship_date
----------+------------+------------
        1 | 2026-07-31 | 2026-07-15
(1 row)
```

A negative price for a product, and an order that apparently shipped *sixteen days before it was placed*, are both nonsensical in the real world, but `NUMERIC(10, 2)` and `DATE` are perfectly happy to store any value of the correct type and shape — the data type itself has no concept of "reasonable." Something needs to encode "the value must also satisfy this business rule," not just "the value must be this shape."

## Concept

### Single-Column CHECK

A `CHECK` constraint attaches a boolean expression to a column (or the whole row); PostgreSQL rejects any `INSERT`/`UPDATE` for which that expression evaluates to `FALSE`.

```sql
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    price      NUMERIC(10, 2) NOT NULL CHECK (price > 0)
);

INSERT INTO products (name, price) VALUES ('Desk Lamp', -19.99);
```

```
ERROR:  new row for relation "products" violates check constraint "products_price_check"
DETAIL:  Failing row contains (1, Desk Lamp, -19.99).
```

A valid price succeeds normally:

```sql
INSERT INTO products (name, price) VALUES ('Desk Lamp', 19.99);
```

```
INSERT 0 1
```

Exactly like `UNIQUE` and `FOREIGN KEY`, a `CHECK` constraint can be written as a named, table-level clause for a clearer error message and an easy way to reference it later:

```sql
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    price      NUMERIC(10, 2) NOT NULL,
    CONSTRAINT products_price_must_be_positive CHECK (price > 0)
);
```

`CHECK` expressions aren't limited to simple comparisons — they can use `IN`, pattern matching, or any boolean-producing expression:

```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    status   TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled'))
);

INSERT INTO orders (status) VALUES ('archived');
```

```
ERROR:  new row for relation "orders" violates check constraint "orders_status_check"
```

This is a common, useful pattern for constraining a text column to a fixed set of allowed values without introducing a separate lookup table.

### Multi-Column CHECK

A `CHECK` constraint can reference more than one column in the same row, enforcing a relationship *between* two of that row's own values:

```sql
CREATE TABLE orders (
    order_id   SERIAL PRIMARY KEY,
    order_date DATE NOT NULL DEFAULT CURRENT_DATE,
    ship_date  DATE,
    CONSTRAINT ship_date_not_before_order_date CHECK (ship_date IS NULL OR ship_date >= order_date)
);

INSERT INTO orders (order_date, ship_date) VALUES ('2026-07-31', '2026-07-15');
```

```
ERROR:  new row for relation "orders" violates check constraint "ship_date_not_before_order_date"
DETAIL:  Failing row contains (1, 2026-07-31, 2026-07-15).
```

Note the explicit `ship_date IS NULL OR ...` — `ship_date` is nullable (an order that hasn't shipped yet has no ship date), and the constraint must account for that: a bare `ship_date >= order_date` would incorrectly reject every unshipped order, because comparing `NULL >= order_date` evaluates to `NULL` (neither true nor false), and PostgreSQL only rejects a row when a `CHECK` expression evaluates to *exactly* `FALSE` — a `NULL` result is treated as passing. This is a subtle but important consequence of `NULL`'s three-valued logic (Module 3), and it's worth internalizing: **a `CHECK` constraint does not implicitly enforce `NOT NULL`** — a `CHECK` on a nullable column is silently skipped whenever the column is actually `NULL`, unless you write the expression to explicitly handle that case.

A valid row succeeds:

```sql
INSERT INTO orders (order_date, ship_date) VALUES ('2026-07-15', '2026-07-31');
INSERT INTO orders (order_date, ship_date) VALUES ('2026-07-31', NULL);
```

```
INSERT 0 1
INSERT 0 1
```

Another common multi-column example is a `quantity` and `unit_price` pair on `order_items`, ensuring both are sensible together:

```sql
CREATE TABLE order_items (
    order_id   INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity   INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
    PRIMARY KEY (order_id, product_id)
);
```

Here, two separate single-column `CHECK`s are used rather than one multi-column expression, since the two rules (`quantity > 0` and `unit_price >= 0`) are logically independent of each other — use a multi-column `CHECK` specifically when the rule genuinely relates two or more columns together (like the `ship_date`/`order_date` comparison above), not merely because two unrelated rules happen to live on the same table.

### Where CHECK Constraints Are Enforced

A `CHECK` constraint is enforced entirely **inside the database**, on every `INSERT` and `UPDATE`, regardless of what application, script, or person is performing the write. This is the same principle behind every constraint in this module: a rule enforced at the database level applies universally — to your main application, to a one-off maintenance script, to a bulk data import, and to anyone connecting directly with `psql` — whereas a rule enforced only in application code applies only to *that* application, and only if every code path through it remembers to check.

This does not mean application-level validation becomes pointless. It typically serves a different, complementary purpose:

| Layer | Typical purpose | Example |
|---|---|---|
| **Application-level validation** | Give the user immediate, friendly feedback before ever sending a request to the database (e.g., a web form highlighting "price must be positive" the instant you type it). | Client-side/server-side form validation, often duplicating database rules for a better user experience. |
| **Database-level `CHECK` constraint** | The final, unconditional guarantee — enforced no matter what wrote the data, and the last line of defense against bugs, race conditions, or a completely different application writing to the same table later. | `CHECK (price > 0)` on the `products` table itself. |

In a mature system, both layers typically coexist: application-level checks give a fast, friendly error message; the database-level `CHECK` constraint guarantees the rule can never actually be violated, no matter what bypasses or bugs exist upstream.

### Limitations: CHECK Cannot Reference Other Rows or Other Tables

A `CHECK` constraint's expression is evaluated **only against the single row being inserted or updated** — it has no ability to look at any other row in the same table, or at any other table at all.

```sql
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    total_amount NUMERIC(10, 2) NOT NULL
        CHECK (total_amount <= (SELECT SUM(price) FROM products))  -- NOT ALLOWED
);
```

```
ERROR:  cannot use subquery in check constraint
```

PostgreSQL explicitly forbids this — a `CHECK` constraint cannot contain a subquery, and therefore cannot express rules like "this order's total must not exceed the customer's account balance" (which requires looking at the `customers` table) or "no more than 3 orders per customer per day" (which requires looking at other rows in `orders` itself). Rules that depend on other rows or other tables require different tools entirely — most commonly triggers (Module 18) or application-level transaction logic (Module 14) — `CHECK` constraints are strictly for validating a single row's own values, in isolation.

## Internal Working (Preview)

```
INSERT/UPDATE proposes a row
        │
        ▼
Row's final column values are assembled (defaults applied, etc.)
        │
        ▼
For each CHECK constraint on the table:
   Evaluate the boolean expression against this row's values only
        │
   ┌────┴─────────────┐
 FALSE            TRUE or NULL
   │                    │
 REJECT              WRITE ROW
```

Because the expression only ever needs the current row's own values (never other rows or tables), evaluating a `CHECK` constraint is extremely cheap — no index lookup, no scan, just a boolean expression evaluated against values already in hand. This is precisely why PostgreSQL restricts `CHECK` to single-row expressions: allowing arbitrary cross-row or cross-table lookups inside a constraint checked on every write would be both semantically complicated (what if the row you're comparing against changes in the same transaction?) and prohibitively expensive at scale.

## Real-World Analogy

Think of airport security screening a single passenger's bag: the scanner checks *that bag's own contents* against a fixed rule set ("no liquids over 100ml," "no sharp objects") — it never needs to know what's in anyone else's bag, or how many bags have already passed through today, to make its decision. That's a `CHECK` constraint: a self-contained rule evaluated purely against the one item in front of it. A rule like "this passenger has already checked in three bags today, reject a fourth" requires looking at *other* records (a passenger's whole day's history) — that's a fundamentally different kind of check, requiring a different system entirely (a database trigger or application logic, not a `CHECK` constraint), exactly as this topic's limitation describes.

## Why CHECK Constraints Were Designed This Way

`NOT NULL`, `UNIQUE`, and `FOREIGN KEY` each encode one specific, common shape of business rule, but no fixed set of built-in constraints could ever anticipate every rule a real schema needs ("price must be positive," "a discount percentage must be between 0 and 100," "a valid status must be one of five specific words"). `CHECK` exists as an escape hatch: a way to express *arbitrary* row-level logic directly in the schema, without needing a new dedicated SQL keyword for every possible business rule. Its restriction to single-row evaluation follows the same design principle seen throughout this module — a constraint checked on every single write should be cheap and predictable to evaluate; anything requiring visibility into other rows or other tables is a fundamentally more expensive and more subtle problem (what should happen if two transactions are both trying to satisfy a cross-row rule at the same moment?), and SQL hands that class of problem to triggers and application logic instead, deliberately keeping `CHECK` itself simple and fast.

## Advantages

- **Expresses arbitrary business rules directly in the schema** — validation logic lives next to the data it protects, rather than scattered across application code.
- **Enforced universally, on every write, regardless of what wrote it** — the same guarantee every other constraint in this module provides.
- **Cheap to evaluate** — since it only ever examines the row being written, `CHECK` adds minimal overhead compared to constraints that require index lookups.
- **Self-documenting** — a `CHECK (price > 0)` in a `CREATE TABLE` statement tells any future reader the business rule immediately, without needing to read application code to discover it.

## Disadvantages / Limitations

- **Cannot reference other rows or other tables** — any rule that depends on aggregate data, another row, or another table cannot be expressed as a `CHECK` constraint at all; it needs triggers (Module 18) or application-transaction logic (Module 14).
- **Complex expressions can be harder to read than equivalent application code** — a deeply nested `CHECK` expression buried in a `CREATE TABLE` statement is sometimes less discoverable than a clearly named validation function in application code, especially for very elaborate rules.
- **Adding a `CHECK` to a populated table requires every existing row to already satisfy it** — exactly like `NOT NULL` (Topic 1), PostgreSQL validates all existing data before accepting the new constraint, which can require a data cleanup pass first.
- **Changing a `CHECK` constraint's logic requires dropping and re-adding it** — there's no in-place "modify the expression"; you must `DROP CONSTRAINT` and `ADD CONSTRAINT` with the new expression.

## Best Practices

- Use `CHECK` for any rule that can be evaluated purely from the row's own column values — value ranges, allowed sets of values, or logical relationships between two of the row's own columns.
- Give every `CHECK` constraint an explicit, descriptive name (`CONSTRAINT products_price_must_be_positive CHECK (price > 0)`) rather than relying on PostgreSQL's auto-generated name — it makes the eventual constraint-violation error message meaningful at a glance.
- Remember that `CHECK` on a nullable column silently passes when the value is `NULL` — if the rule should also forbid `NULL`, pair the `CHECK` with an explicit `NOT NULL`, don't assume `CHECK` alone covers it.
- Keep application-level validation for user-experience purposes (fast, friendly feedback), but never treat it as a substitute for the equivalent database-level `CHECK` — assume any application-only rule will eventually be bypassed by something (a script, a different application, a bug).
- For rules that genuinely need to look at other rows or tables, don't try to force them into a `CHECK` constraint — reach for a trigger (Module 18) instead.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Trying to reference another table or use a subquery inside a `CHECK` constraint | PostgreSQL explicitly disallows subqueries in `CHECK` expressions — cross-row and cross-table validation requires triggers or application logic instead. |
| Assuming `CHECK` automatically also enforces `NOT NULL` on the columns it references | A `CHECK` expression that evaluates to `NULL` (which happens whenever one of its referenced columns is `NULL`, in many common expressions) is treated as passing, not failing — `CHECK` and `NOT NULL` are independent and often need to be combined deliberately. |
| Relying only on application-level validation and skipping the database-level `CHECK` | Any other application, script, or direct database access that bypasses the application layer can insert data violating the intended rule — only a database-level constraint guarantees it universally. |
| Writing an overly complex `CHECK` expression that's hard to read or maintain | Extremely elaborate row-level logic is a sign the rule might belong in a trigger or stored function (Module 18) instead, where it can be named, commented, and tested more clearly than a dense inline boolean expression. |

## Interview Questions

1. **Q: What is a `CHECK` constraint, and how is it different from `NOT NULL` or `UNIQUE`?**
   A: A `CHECK` constraint enforces an arbitrary boolean expression against a row's own column values — it's a general-purpose validation tool, versus `NOT NULL` (which only enforces presence) and `UNIQUE` (which only enforces non-repetition). `CHECK` can express rules like value ranges, allowed value sets, and relationships between two columns in the same row.

2. **Q: Can a `CHECK` constraint enforce that an order's total never exceeds the customer's account credit limit, where the credit limit is stored in a separate `customers` table?**
   A: No. `CHECK` constraints cannot reference other tables or contain subqueries — they only evaluate the single row being written. That kind of cross-table rule requires a trigger (Module 18) or application-level transactional logic (Module 14) instead.

3. **Q: If a table has `CHECK (discount_percent BETWEEN 0 AND 100)` and `discount_percent` is nullable, what happens when a row is inserted with `discount_percent` left as `NULL`?**
   A: The insert succeeds. The `CHECK` expression evaluates to `NULL` (not `FALSE`) when `discount_percent` is `NULL`, and PostgreSQL only rejects a row when a `CHECK` expression evaluates to exactly `FALSE` — a `NULL` result is treated as satisfying the constraint. If `NULL` should also be disallowed, the column needs an explicit `NOT NULL` constraint as well.

## Summary

- `CHECK` constraints validate arbitrary boolean conditions against a row's own column values, filling the gap left by `NOT NULL`, `UNIQUE`, and foreign keys.
- They can reference a single column (`price > 0`) or multiple columns in the same row (`ship_date >= order_date`).
- `CHECK` is enforced entirely at the database level, on every write, regardless of what application or script performed it — application-level validation is a complementary user-experience layer, not a substitute.
- `CHECK` constraints cannot reference other rows or other tables — no subqueries are allowed; cross-row or cross-table validation requires triggers or application logic instead.
- A `CHECK` on a nullable column silently passes when the value is `NULL`, since the expression evaluates to `NULL` rather than `FALSE` — pair it with `NOT NULL` explicitly if that's not the intended behavior.
