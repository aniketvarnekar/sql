# SELECT and Projection

## Learning Objectives

By the end of this section you should be able to:
- Choose specific columns in a `SELECT` list instead of relying on `SELECT *`
- Give a column or a computed expression a readable name with `AS`
- Write computed/expression columns that don't exist as stored columns at all
- Explain the relational-theory concept of *projection* and connect it to what the `SELECT` list actually does
- State, at a high level, the logical order in which a full `SELECT` statement is evaluated

## Prerequisites

- [Your First Query](../01-introduction/05-your-first-query.md) — you've already written `SELECT name, salary FROM employees WHERE department = 'Sales' ORDER BY salary DESC`; this topic is the deep, general treatment of the `SELECT` list piece of that statement, and previews how all its clauses relate.
- [Tables, Rows, and Columns](../02-relational-model/01-tables-rows-and-columns.md) — you need the *relation*/*attribute* vocabulary and, specifically, the idea that a relation's attributes are a named set with no inherent position, since that theory is exactly what this topic's `SELECT` mechanics implement in practice.

## Motivation

Every `SELECT` you will ever write starts with the same question: *which columns, exactly, do I want back?* It's tempting to always answer "all of them" — but real queries almost never want the entire row, and often want values that don't exist as columns at all (a discounted price, a full name built from two fields, a flag computed from a date). Learning to shape the `SELECT` list deliberately — not just reflexively typing `*` — is the first and most foundational skill of writing real SQL, and it's also your first hands-on encounter with a piece of formal relational theory: projection.

## Problem Statement

Suppose your team maintains a catalog of products for an internal ordering tool. A colleague asks you three separate, very ordinary questions:

1. "Just give me the product names and prices — I don't need the internal IDs or stock counts cluttering the output."
2. "Can you show me what each product would cost with 8% tax added? We don't store a taxed price column anywhere."
3. "The column headers `name` and `price` are fine for us, but can the tax-inclusive one be labeled something a non-technical teammate would understand at a glance?"

None of these are answerable with `SELECT *` — that returns every stored column, verbatim, under its stored name, whether you asked for it or not. Answering all three requires deliberately shaping what the `SELECT` list returns: which columns, which computed values, and under what names.

## Concept

### Setting Up This Module's Running Example

Every topic in this module reuses the same table: a small product catalog. Set it up once here; later topics simply refer back to it.

```sql
CREATE TABLE products (
    id               SERIAL PRIMARY KEY,
    name             TEXT NOT NULL,
    category         TEXT,
    price            NUMERIC(10,2) NOT NULL,
    stock_quantity   INTEGER,
    discontinued_on  DATE
);

INSERT INTO products (name, category, price, stock_quantity, discontinued_on) VALUES
    ('Wireless Mouse',        'Electronics',      24.99, 150,  NULL),
    ('USB-C Cable',           'Electronics',       9.99, 300,  NULL),
    ('Mechanical Keyboard',   'Electronics',      89.99,  45,  NULL),
    ('Standing Desk',         'Furniture',       349.00,  12,  NULL),
    ('Office Chair',          'Furniture',       199.50,   0,  NULL),
    ('Desk Lamp',             'Furniture',        34.95,  60,  '2024-11-01'),
    ('Notebook Pack of 5',    'Office Supplies',   6.49, 500,  NULL),
    ('Stapler',               'Office Supplies',  12.00, NULL, NULL),
    ('Whiteboard Marker Set', 'Office Supplies',   8.75, 220,  NULL),
    ('Coffee Mug',            'Kitchen',          11.25,  80,  NULL),
    ('Electric Kettle',       'Kitchen',          45.00,   0,  '2023-06-15'),
    ('Blender_Pro 2000',      'Kitchen',          79.99,  15,  NULL),
    ('100% Cotton Towel',     'Home Goods',       15.00,  40,  NULL),
    ('Uncategorized Widget',  NULL,                3.50, 1000, NULL);
```

This is exactly the `INSERT` statement covered in depth in [INSERT](../06-modifying-data/01-insert.md) — a single multi-row insert populating 14 rows in one statement, with `id` assigned automatically by `SERIAL` (1 through 14, in the order written above) and `stock_quantity`/`discontinued_on` left as `NULL` wherever no value is given. Deliberately: some products have no `category` (`NULL`), some have no known `stock_quantity` (`NULL`), and two have a `discontinued_on` date while the rest are still active (`NULL`). This variety is what makes later topics in this module (pattern matching, range checks, `NULL` handling) demonstrable with real, believable data instead of a contrived toy example.

### `SELECT *` vs. Naming Columns

```sql
SELECT * FROM products WHERE category = 'Electronics';
```

```
 id |        name         |  category   | price | stock_quantity | discontinued_on
----+----------------------+-------------+-------+-----------------+------------------
  1 | Wireless Mouse       | Electronics | 24.99 |             150 |
  2 | USB-C Cable          | Electronics |  9.99 |             300 |
  3 | Mechanical Keyboard  | Electronics | 89.99 |              45 |
(3 rows)
```

`*` means "every column, in the table's defined order, under its stored name." It's convenient for a quick, exploratory look at unfamiliar data — which is exactly how it's used casually throughout this course — but naming columns explicitly is what real, maintained queries should do:

```sql
SELECT name, price
FROM products
WHERE category = 'Electronics';
```

```
        name         | price
----------------------+-------
 Wireless Mouse       | 24.99
 USB-C Cable          |  9.99
 Mechanical Keyboard  | 89.99
(3 rows)
```

Only the two requested columns come back, in the order requested — which need not match the table's stored column order at all. `SELECT price, name` would return the same two columns with `price` first, even though `name` is declared before `price` in the table.

### Aliasing Columns with `AS`

A column's stored name isn't always the name you want in the output — especially for computed values, which have no stored name at all. `AS` renames a column or expression in the result set:

```sql
SELECT name AS product_name, price AS unit_price
FROM products
WHERE category = 'Electronics';
```

```
    product_name     | unit_price
----------------------+------------
 Wireless Mouse       |      24.99
 USB-C Cable          |       9.99
 Mechanical Keyboard  |      89.99
(3 rows)
```

`AS` is technically optional — `SELECT name product_name` works identically — but writing `AS` explicitly is strongly preferred: without it, a typo or a missing comma between two column names can silently be misread as "alias the first as the second" instead of the two-columns list you meant, which is confusing to spot on review. If an alias needs characters SQL wouldn't normally allow in a plain identifier (spaces, mixed reserved words), wrap it in double quotes: `AS "Product Name"`.

### Computed / Expression Columns

The `SELECT` list is not limited to columns that exist in the table — it can contain any valid SQL expression, evaluated once per row:

```sql
SELECT
    name,
    price,
    price * 1.08 AS price_with_tax
FROM products
WHERE category = 'Kitchen';
```

```
       name         | price | price_with_tax
---------------------+-------+-----------------
 Coffee Mug          | 11.25 |         12.1500
 Electric Kettle     | 45.00 |         48.6000
 Blender_Pro 2000    | 79.99 |         86.3892
(3 rows)
```

`price_with_tax` exists nowhere in storage — it's computed fresh, per row, at query time from `price * 1.08`. Notice the exact values: `79.99 * 1.08` is `86.3892` precisely, with no floating-point rounding error, because `price` is `NUMERIC` — the same exact decimal arithmetic behavior Module 3 (Data Types) explains is why `NUMERIC` is the right choice for anything involving money.

Text expressions work the same way. PostgreSQL's `||` operator concatenates strings:

```sql
SELECT name || ' (' || category || ')' AS label
FROM products
WHERE category = 'Electronics';
```

```
                label
--------------------------------------
 Wireless Mouse (Electronics)
 USB-C Cable (Electronics)
 Mechanical Keyboard (Electronics)
(3 rows)
```

A computed column can reference multiple stored columns, literals, arithmetic, string concatenation, or (as later modules cover) built-in functions — anything that produces one value per row is fair game for the `SELECT` list.

### Projection: The Relational-Theory Behind What `SELECT`'s Column List Does

[Tables, Rows, and Columns](../02-relational-model/01-tables-rows-and-columns.md) established that a relation is formally a set of tuples over a set of named attributes, and that an attribute's *position* carries no logical meaning — only its *name* does. Choosing a subset of a relation's attributes to keep, discarding the rest, is a formal operation in relational algebra called **projection** (traditionally written with the Greek letter π). The `SELECT` list — the part of the statement between the keyword `SELECT` and `FROM` — is SQL's practical implementation of exactly this operation: a *vertical slice* of the table, keeping only the named columns (or computed values) you ask for, for every row that survives filtering.

This is deliberately contrasted with the *other* fundamental relational operation — choosing which *rows* (tuples) to keep, called **selection** in relational algebra, which is what the `WHERE` clause implements (Topic 2 of this module covers it in full). Projection acts on columns; selection acts on rows. They're independent, composable operations: you can project without filtering (`SELECT name, price FROM products`, every row, only two columns), filter without projecting away anything (`SELECT * FROM products WHERE category = 'Kitchen'`, every column, only some rows), or do both at once, as every example above already does.

### A First Look at the Logical Processing Order of a `SELECT`

You write a `SELECT` statement with `SELECT` first and `FROM` second — but that is not the order the database actually evaluates it in. Conceptually, PostgreSQL processes a full `SELECT` in roughly this sequence:

```
FROM   → identify the source table(s)
WHERE  → keep only the rows that satisfy the condition   (selection)
SELECT → compute the requested columns/expressions for each surviving row  (projection)
ORDER BY → sort the resulting rows
LIMIT  → cap how many rows are returned
```

`FROM` is resolved first, `WHERE` filters rows before any column is ever computed, and only then does the `SELECT` list's projection happen — which is why a `WHERE` clause can reference `stock_quantity` for filtering even in a query whose `SELECT` list never returns that column at all: filtering happens on the full row, before projection throws anything away. This module's [LIMIT, OFFSET, and DISTINCT](07-limiting-and-distinct.md) topic returns to this exact pipeline once every stage has a topic of its own, and assembles the complete picture.

## Internal Working (Preview)

For a query like `SELECT name, price FROM products WHERE category = 'Kitchen'`, the engine's conceptual pipeline looks like this:

```
 products table (all 14 rows, all 6 columns, on disk)
        │
        ▼
 Locate the table via the system catalog
        │
        ▼
 Evaluate category = 'Kitchen' against every row
        │            (rows that don't qualify are discarded here — before
        │             projection ever runs)
        ▼
 3 qualifying rows remain, still with all 6 columns internally
        │
        ▼
 Evaluate the SELECT list (name, price) against each surviving row
        │            (id, category, stock_quantity, discontinued_on are
        │             computed/read but never included in the output)
        ▼
 Final result: 3 rows, 2 columns
```

The engine doesn't necessarily do this as three separate, wasteful full passes over the data in a real execution plan — the query planner (Module 20 covers this in depth) is free to interleave or reorder the *physical* steps for efficiency, as long as the *logical* result is identical to this conceptual order. That distinction — logical meaning vs. physical execution strategy — is precisely the "what, not how" declarative principle established back in Module 1's discussion of what a DBMS provides.

## Real-World Analogy

Think of projection like requesting an abridged company directory. The full employee record on file might include a home address, an emergency contact, a salary, and a performance history — but the printed directory handed out to every desk only shows a name and an extension number. Nothing about the underlying employee records changed; a specific, deliberate subset of attributes was chosen to appear in this particular output, in this particular order, possibly under friendlier labels ("Ext." instead of "phone_extension"). That's projection: the same underlying data, a chosen vertical slice of it, presented for a specific purpose.

## Why Projection Was Designed This Way

SQL is declarative: you describe *what* result shape you want, not the mechanical steps to build it. Projection is the cleanest expression of that philosophy — instead of writing a loop that reads every column of every row and manually discards the ones you don't need, you simply *name* the columns (or expressions) you want, and the database guarantees the result contains exactly those, nothing more. This also directly enables the physical/logical independence formalized in [The Relational Model and Codd's Rules](../02-relational-model/03-the-relational-model-and-codds-rules.md): because a query's `SELECT` list identifies wanted attributes *by name*, not by position or physical storage layout, the database is completely free to reorganize how it physically stores those columns (for compression, for performance) without ever invalidating a single query that names them.

## Advantages

- **Reduced data transfer and processing** — returning 2 columns instead of 6 (or 60) means less data computed, sent over the network, and handled by whatever reads the result.
- **Resilience to schema change** — a query that names its columns explicitly keeps working, unchanged, even if the table later gains new columns; `SELECT *` silently changes its output shape the moment the table's structure changes.
- **Readability and intent** — `SELECT name, price` tells a reader exactly what the query is for; `SELECT *` tells them nothing about what the caller actually needed.
- **Enables values that don't exist in storage at all** — computed/expression columns let a query answer questions (`price_with_tax`, a concatenated label) without ever altering the table's actual structure.

## Disadvantages / Limitations

- **Verbosity** — naming many columns explicitly is more typing than `*`, especially for quick, throwaway exploration of unfamiliar data (where `SELECT *` remains genuinely useful).
- **Computed expressions cost CPU per row** — a `SELECT` list expression like `price * 1.08` is recomputed for every row every time the query runs; for something reused constantly, a view (Module 12) can encapsulate the computation instead of repeating it in every query.
- **Aliases are not visible everywhere in the same statement** — as the next section explains, an alias defined in the `SELECT` list cannot be used in that same query's `WHERE` clause, which surprises people used to languages where a variable, once assigned, is usable immediately afterward.

## Best Practices

- Avoid `SELECT *` in application code, views, or anything that will be maintained over time — name columns explicitly, even if that list is currently identical to every column in the table.
- Always use `AS` explicitly when aliasing, even though it's technically optional — it makes column renaming unambiguous to any future reader (including you, months later).
- Give computed columns clear, descriptive aliases (`price_with_tax`, not `col1`) — the reader of the result set has no other way to know what an unlabeled computed column represents.
- Reach for `SELECT *` freely during interactive, exploratory queries against data you don't yet know well — that's exactly the situation it's suited for.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Referencing a `SELECT`-list alias inside that same query's `WHERE` clause (e.g. `SELECT price * 1.08 AS taxed FROM products WHERE taxed > 50`) | `WHERE` is evaluated *before* the `SELECT` list, per the logical processing order — the alias doesn't exist yet at the point `WHERE` runs, so PostgreSQL raises a "column does not exist" error. |
| Assuming `SELECT *`'s column order is a stable contract | Column order reflects the table's current definition order, which can change if columns are added, dropped, or a table is recreated — code that depends positionally on `SELECT *`'s output is fragile. |
| Treating `AS` as required syntax rather than a readability convention | `SELECT price total` (no `AS`) is valid and aliases `price` as `total` — but omitting `AS` makes a genuine typo (a missing comma between two intended separate columns) much harder to spot on review. |
| Believing a computed column changes the underlying table | `price * 1.08 AS price_with_tax` exists only in the result set of that one query — nothing is written back to `products`; the table is completely unmodified. |

## Interview Questions

1. **Q: What is "projection" in relational theory, and what part of a `SELECT` statement implements it?**
   A: Projection is the relational-algebra operation of choosing a subset of a relation's named attributes (columns) to keep, discarding the rest, without regard to their storage position. The `SELECT` list — the columns and expressions named between `SELECT` and `FROM` — is SQL's implementation of this operation.

2. **Q: Why can't a `WHERE` clause reference an alias defined in that same query's `SELECT` list?**
   A: Because of SQL's logical processing order, `WHERE` is evaluated before the `SELECT` list is computed — the alias simply doesn't exist yet at the point rows are being filtered. `ORDER BY`, by contrast, runs after `SELECT` and can reference its aliases (covered in this module's sorting topic).

3. **Q: What are the trade-offs between `SELECT *` and naming columns explicitly?**
   A: `SELECT *` is faster to type and convenient for quick, ad hoc exploration of unfamiliar data, but its output silently changes shape whenever the table's columns change, transfers more data than may be needed, and communicates nothing about what a query actually requires. Explicit column lists are more verbose but are readable, resilient to schema changes, and transfer only what's needed — the right default for any query meant to be maintained.

## Summary

- The `SELECT` list determines which columns (or computed expressions) appear in the result — this is *projection*, the relational-algebra operation of choosing a vertical slice of a relation's attributes.
- `SELECT *` returns every column under its stored name and current position; naming columns explicitly is more verbose but far more resilient and readable, and is the right default outside of quick, exploratory queries.
- `AS` renames a column or expression in the output; it's optional syntactically but should be used explicitly for clarity.
- The `SELECT` list can contain arbitrary expressions — arithmetic, string concatenation, and more — computed fresh per row, without altering the underlying table.
- SQL's logical processing order runs roughly `FROM → WHERE → SELECT → ORDER BY → LIMIT`; because `WHERE` runs before `SELECT`, a `WHERE` clause cannot reference a `SELECT`-list alias — this module's final topics return to this pipeline in full.
