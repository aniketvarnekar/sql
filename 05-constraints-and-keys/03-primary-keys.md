# Primary Keys

## Learning Objectives

By the end of this section you should be able to:
- Explain precisely what a `PRIMARY KEY` guarantees, in terms of the constraints it's built from
- Declare both single-column and composite (multi-column) primary keys
- Explain why a table may have at most one primary key
- Justify why nearly every table you design should have a primary key

## Prerequisites

- [NOT NULL and DEFAULT](01-not-null-and-default.md) and [UNIQUE Constraints](02-unique-constraints.md) — a primary key is best understood as a specific, stricter combination of both, so you need both individual pieces clear first.

## Motivation

Every table needs a reliable way to answer the question "which exact row do I mean?" Without one, "update Asha's order" or "delete this specific review" has no unambiguous target if two rows happen to look identical, or if the row you mean shifts depending on some column that could later be edited. A primary key is the one column (or set of columns) a table designates, up front, as *the* answer to "which row is this" — for every single row, forever.

## Problem Statement

Consider an `orders` table with no primary key at all:

```sql
CREATE TABLE orders (
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL,
    total_amount NUMERIC(10, 2) NOT NULL
);

INSERT INTO orders (customer_id, order_date, total_amount) VALUES
    (1, '2026-07-01', 149.99),
    (1, '2026-07-01', 149.99);
```

Both inserts succeed, and now the table contains two rows that are completely indistinguishable from each other:

```
 customer_id | order_date | total_amount
-------------+------------+--------------
           1 | 2026-07-01 |       149.99
           1 | 2026-07-01 |       149.99
(2 rows)
```

Was customer 1 charged once, or twice? Is this a genuine duplicate order placed by mistake, or two separate orders that happen to share every visible value? There is no column, or combination of columns, in this table that reliably identifies "row one" versus "row two" — and critically, there is no way to write `UPDATE orders SET total_amount = 0 WHERE ...` or `DELETE FROM orders WHERE ...` that targets exactly one of these rows without touching the other, because they are, as far as SQL is concerned, identical. A table needs some column (or combination) guaranteed to be both unique and always present, specifically so any single row can always be addressed unambiguously.

## Concept

### What a Primary Key Guarantees

A `PRIMARY KEY` is a constraint that guarantees two things simultaneously, for the column(s) it's declared on:

1. **Uniqueness** — no two rows may share the same value (identical to `UNIQUE`).
2. **Not-null** — the value can never be missing (identical to `NOT NULL`).

In other words, `PRIMARY KEY` is exactly `UNIQUE` and `NOT NULL` combined into a single declaration, plus one additional rule: **a table may have at most one primary key.** (You can have as many separate `UNIQUE` constraints as you like on a table, but only one of them — or one dedicated column — can be *the* primary key.)

```sql
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    order_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    total_amount NUMERIC(10, 2) NOT NULL
);

INSERT INTO orders (customer_id, total_amount) VALUES (1, 149.99);
INSERT INTO orders (customer_id, total_amount) VALUES (1, 149.99);

SELECT * FROM orders;
```

```
 order_id | customer_id | order_date | total_amount
----------+-------------+------------+--------------
        1 |           1 | 2026-07-31 |       149.99
        2 |           1 | 2026-07-31 |       149.99
(2 rows)
```

These two orders now have identical-looking business data, but they are unambiguously distinct rows — `order_id` 1 and `order_id` 2 — and `UPDATE orders SET total_amount = 0 WHERE order_id = 1;` affects exactly one of them, with zero ambiguity. `SERIAL` (PostgreSQL's auto-incrementing integer type) makes this effortless: every new row gets the next integer automatically, so `order_id` is guaranteed unique without any manual bookkeeping.

Attempting a duplicate primary key value fails immediately:

```sql
INSERT INTO orders (order_id, customer_id, total_amount) VALUES (1, 2, 50.00);
```

```
ERROR:  duplicate key value violates unique constraint "orders_pkey"
DETAIL:  Key (order_id)=(1) already exists.
```

And attempting `NULL` fails too, exactly as it would for any `NOT NULL` column:

```sql
INSERT INTO orders (order_id, customer_id, total_amount) VALUES (NULL, 2, 50.00);
```

```
ERROR:  null value in column "order_id" of relation "orders" violates not-null constraint
```

### Single-Column Primary Keys

The `orders` example above is a single-column primary key — one column, `order_id`, alone identifies every row. This is the most common case, and `SERIAL PRIMARY KEY` (or, in more modern PostgreSQL style, `GENERATED ALWAYS AS IDENTITY PRIMARY KEY`) is the standard way to write it when the key is a database-generated surrogate value (surrogate vs. natural keys are covered fully in Topic 6 of this module).

### Composite (Multi-Column) Primary Keys

Sometimes no single column is a natural identifier, but a *combination* of columns is. Consider an `order_items` table recording which products appear on which order:

```sql
CREATE TABLE order_items (
    order_id   INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity   INTEGER NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 100, 2, 24.99),
    (1, 200, 1, 99.99),
    (2, 100, 1, 24.99);
```

```
SELECT * FROM order_items;
```

```
 order_id | product_id | quantity | unit_price
----------+------------+----------+------------
        1 |        100 |        2 |      24.99
        1 |        200 |        1 |      99.99
        2 |        100 |        1 |      24.99
(3 rows)
```

Order 1 legitimately contains multiple products, and product 100 legitimately appears on multiple orders — so neither `order_id` alone nor `product_id` alone can be the primary key. But the *pair* — "this specific product, on this specific order" — is exactly one row, every time, which is precisely what the business rule "a product can only be listed once per order" requires:

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (1, 100, 5, 24.99);
```

```
ERROR:  duplicate key value violates unique constraint "order_items_pkey"
DETAIL:  Key (order_id, product_id)=(1, 100) already exists.
```

If order 1 needs *more* of product 100, the correct operation is `UPDATE order_items SET quantity = quantity + 5 WHERE order_id = 1 AND product_id = 100;`, not a second row for the same pair — the composite primary key is precisely what makes that the only valid way to express "more of the same item."

A composite primary key also automatically enforces `NOT NULL` on every column it includes:

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (3, NULL, 1, 10.00);
```

```
ERROR:  null value in column "product_id" of relation "order_items" violates not-null constraint
```

### Exactly One Primary Key Per Table

A table may have many `UNIQUE` constraints, but only one `PRIMARY KEY`:

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    email       TEXT UNIQUE NOT NULL,
    ssn         TEXT PRIMARY KEY  -- invalid: a second primary key
);
```

```
ERROR:  multiple primary keys for table "customers" are not allowed
```

`email` here is unique and mandatory too, but it is declared as a `UNIQUE` constraint, not a second primary key — a table can have several columns that are each individually capable of identifying a row (these are sometimes called "candidate keys"), but only one of them is formally designated *the* primary key. Choosing which candidate becomes the primary key, versus which stay as ordinary `UNIQUE` constraints, is exactly the judgment call explored in Topic 6 (Natural vs. Surrogate Keys).

## Internal Working (Preview)

Declaring `PRIMARY KEY` is syntactic sugar that does three things at once:

```
PRIMARY KEY (col1 [, col2, ...])
        │
        ├──▶ Adds NOT NULL to every listed column
        │
        ├──▶ Creates a UNIQUE constraint (and its backing B-tree index) across the listed column(s) together
        │
        └──▶ Registers this constraint, specifically, as the table's designated primary key in PostgreSQL's system catalog
```

Because a primary key is backed by a unique index exactly like a `UNIQUE` constraint (Topic 2), looking up a row by its primary key is fast even in a very large table — this is also why the primary key is overwhelmingly the most common column (or column set) used in `JOIN` conditions (Module 10) and `WHERE` clauses targeting a specific row: it's both semantically "the" identifier and mechanically the fastest possible lookup path.

## Real-World Analogy

A primary key is like a national passport number, or a car's Vehicle Identification Number (VIN). Two people can share a name; two cars can share a color, make, and model year — but no two passports share a number, and no passport is ever issued without one. That single field is specifically designed to be the unambiguous answer to "which exact person/vehicle is this," even when every other visible attribute happens to match someone or something else. A composite primary key is like a seat assignment on a flight: neither the row letter (`A`) nor the row number (`14`) alone identifies a unique seat on the whole plane, but the pair `14A` does.

## Why Primary Keys Were Designed This Way

The relational model requires every row in a table to be distinguishable — a table is formally a *set* of rows, and in a true set, no two elements can be identical (Module 2 covers this formally). Without a designated identifying column, two rows with identical values across every column would be indistinguishable, and the model's very basis breaks down. `PRIMARY KEY` exists to make this guarantee explicit and mechanical: rather than relying on "every column together happens to always differ" (which real data frequently violates, as the `orders` Problem Statement showed), the table designer designates one column, or minimal set of columns, whose sole job is guaranteeing distinctness. Combining `UNIQUE` and `NOT NULL` into a single named concept, rather than leaving developers to manually apply both separately, also gives every table a single, predictable, and referenceable identity — which becomes essential the moment other tables need to *point at* a specific row, the subject of the next topic, Foreign Keys.

## Advantages

- **Unambiguous row identification** — every row can always be targeted precisely by its primary key value, in `UPDATE`, `DELETE`, and `JOIN` operations.
- **Fast lookups by default** — backed by a unique index, so finding a row by primary key remains efficient regardless of table size.
- **Foundation for relationships between tables** — foreign keys (Topic 4) exist specifically to reference another table's primary key, making multi-table designs possible.
- **Self-documenting intent** — a `PRIMARY KEY` declaration tells anyone reading the schema, immediately, "this is how a row in this table is identified," without needing to infer it from application code.

## Disadvantages / Limitations

- **Choosing the wrong primary key is costly to fix later** — if other tables have already created foreign keys referencing it (Topic 4), changing a primary key on a live table is a significant, carefully coordinated migration, not a quick edit.
- **A composite primary key is more cumbersome to reference** — any foreign key pointing at a two-or-more-column primary key must itself supply all of those columns, which is more verbose than referencing a single surrogate column (Topic 6 expands on this trade-off).
- **Not a substitute for `CHECK` or format validation** — a primary key guarantees uniqueness and presence, nothing about whether the value is well-formed or sensible (a primary key of `TEXT` type happily accepts `'garbage'` as long as it's unique and non-null).

## Best Practices

- Give virtually every table a primary key — even a table that "feels like" it doesn't need one (a pure logging table, a many-to-many join table) benefits from unambiguous row addressing, and most schema tools, ORMs-adjacent tooling, and replication mechanisms assume one exists.
- Prefer a single-column primary key when there's no strong reason for a composite one — it keeps foreign keys elsewhere in the schema simple (Topic 6 goes deeper on this specific trade-off).
- Reserve composite primary keys for tables whose entire identity genuinely *is* the combination of two (or more) foreign keys, such as a join table like `order_items` connecting orders and products.
- Never treat "no two rows have matched yet in my testing" as equivalent to "a primary key isn't necessary" — a primary key is a guarantee enforced by the database for every future row, not an observation about current data.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a table can have multiple primary keys if it has multiple unique-looking columns | A table may have exactly one primary key. Other unique, mandatory columns should be declared as separate `UNIQUE NOT NULL` constraints, not additional primary keys. |
| Believing `PRIMARY KEY (a, b)` means `a` and `b` are each independently unique | Exactly like composite `UNIQUE`, it's the combination that's guaranteed unique — either column can repeat on its own across different values of the other. |
| Designing a table without any primary key "for simplicity" | Every row then risks being indistinguishable from another if all visible columns happen to match, and there's no reliable way to `UPDATE`/`DELETE` a single specific row or to have another table reference it via foreign key. |
| Forgetting that a primary key column is automatically `NOT NULL` and trying to also declare it `NOT NULL` explicitly, or being confused when omitting it still causes an error | `PRIMARY KEY` already implies `NOT NULL` — no additional declaration is needed, and omitting a value for it (without a `DEFAULT`) will always fail. |

## Interview Questions

1. **Q: What exactly does `PRIMARY KEY` guarantee, and how does that differ from a plain `UNIQUE` constraint?**
   A: A `PRIMARY KEY` guarantees both uniqueness and non-null-ness for its column(s) — it's equivalent to `UNIQUE NOT NULL` combined, plus the rule that a table may only have one. A plain `UNIQUE` constraint alone only guarantees uniqueness; it can still allow `NULL` (in fact, multiple `NULL`s, as covered in Topic 2) unless separately marked `NOT NULL`.

2. **Q: When would you choose a composite primary key over a single surrogate column?**
   A: When a row's identity is genuinely and only defined by the combination of two or more values — most commonly a join/junction table connecting two other tables (like `order_items` identified by `(order_id, product_id)`), where the pair itself is the entire meaning of the row and there's no independent concept of "an order_item" outside that pairing.

3. **Q: Can a table exist without a primary key in PostgreSQL? Should it?**
   A: Syntactically, yes — PostgreSQL does not require a primary key. In practice, it's strongly discouraged for nearly every table: without one, rows can become indistinguishable from each other if all other columns match, targeted updates/deletes on a single row become unreliable, and other tables have no reliable column to build a foreign key reference against.

## Summary

- A `PRIMARY KEY` combines `UNIQUE` and `NOT NULL` into a single guarantee: every row is both distinctly and reliably identifiable by it.
- Primary keys can be single-column (the common case) or composite, spanning multiple columns whose combination — not either column alone — is guaranteed unique.
- A table may have exactly one primary key, even if multiple columns could individually serve as candidates; other candidates become plain `UNIQUE` constraints instead.
- A primary key is backed by a unique index, giving both the integrity guarantee and fast lookups by that key.
- Nearly every table should have a primary key — it's the foundation that makes precise row targeting possible and is required for another table to reference a specific row via a foreign key, covered next.
