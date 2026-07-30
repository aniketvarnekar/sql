# Creating Tables

## Learning Objectives

By the end of this section you should be able to:
- Write a complete `CREATE TABLE` statement with multiple columns, each with an appropriate data type
- Use `IF NOT EXISTS` to make table creation safely re-runnable
- Take a real-world requirement and translate it into a sensible set of columns and types
- Explain, at a conceptual level, where constraints fit into a column definition, without yet needing their full syntax or behavior

## Prerequisites

- [Creating and Dropping Databases](01-creating-and-dropping-databases.md) — you need a database to create tables inside; this topic assumes you have one open (e.g., `sql_course`).
- Module 03 — Data Types: you need to already know PostgreSQL's core data types (`INTEGER`, `NUMERIC`, `TEXT`/`VARCHAR`, `BOOLEAN`, `DATE`, `TIMESTAMP`, and so on) — this topic uses them throughout but does not re-teach what each one means or when to prefer one over another.

## Motivation

`CREATE TABLE` is the single most consequential statement in this entire module — arguably in the whole course. Every `INSERT`, every `SELECT`, every `JOIN` you will ever write depends on a table having been correctly designed first. A well-designed table makes every later query natural to write; a poorly designed one (wrong types, missing columns, an awkward shape) creates friction you'll fight against for the lifetime of the application. This topic is where you learn to go from "I need to store some information about X" to a precise, correct `CREATE TABLE` statement.

## Problem Statement

Imagine you've been asked to build the data layer for a small online bookstore's product catalog. Before writing a single query, you need to answer: What information does each product actually need to hold? What type is each piece of information? Which pieces are mandatory versus optional? Get this wrong — say, storing a price as text, or forgetting a "quantity in stock" column — and every feature built on top of this table (search, checkout, inventory reports) inherits that mistake. `CREATE TABLE` is where that design decision becomes a concrete, permanent (until altered) database structure.

## Concept

### The Full `CREATE TABLE` Syntax

The general shape of a `CREATE TABLE` statement is:

```sql
CREATE TABLE [IF NOT EXISTS] table_name (
    column1_name  data_type  [column_constraint ...],
    column2_name  data_type  [column_constraint ...],
    ...
    [table_constraint ...]
);
```

Breaking this down piece by piece:

| Piece | Meaning |
|---|---|
| `CREATE TABLE` | The DDL keyword that begins the statement (Module 1's DDL/DML/DQL categorization). |
| `IF NOT EXISTS` | Optional. If a table with this name already exists, PostgreSQL silently does nothing instead of raising an error. |
| `table_name` | The name you're giving the new table — must be unique within its schema (the default schema is `public`, covered briefly below). |
| `column_name data_type` | Each column's name and the type of value it will hold — repeated once per column, comma-separated. |
| `column_constraint` | An optional rule attached directly to one column (e.g., `NOT NULL`) — introduced conceptually later in this topic, taught fully in Module 5. |
| `table_constraint` | An optional rule that can span multiple columns (e.g., a primary key made of two columns together) — also deferred to Module 5. |

### A Real Design Example: A Bookstore's `products` Table

Let's actually design a table from a requirement, rather than looking at syntax in isolation. Suppose the requirement is:

> "Store each product we sell: a name, its price, how many are currently in stock, whether it's currently available for sale, and when it was added to the catalog."

Translating each requirement into a column:

| Requirement | Column name | Data type | Reasoning |
|---|---|---|---|
| A unique identifier for each product | `id` | `SERIAL` | An auto-incrementing integer PostgreSQL fills in for you, as seen in Module 1 — every table generally needs some reliable way to refer to one specific row. |
| A name | `name` | `TEXT` | Product names vary widely in length; `TEXT` places no artificial length cap (Module 3 covers `TEXT` vs. `VARCHAR(n)` in depth). |
| A price | `price` | `NUMERIC(10, 2)` | Money must be exact — `NUMERIC` avoids floating-point rounding errors; `(10, 2)` allows up to 10 total digits with exactly 2 after the decimal point, enough for prices up to 99,999,999.99. |
| How many are in stock | `quantity_in_stock` | `INTEGER` | A whole number of physical units — never fractional. |
| Whether it's currently available | `is_available` | `BOOLEAN` | A true/false flag is exactly what `BOOLEAN` represents. |
| When it was added | `created_at` | `TIMESTAMP` | Records a specific point in time (Module 3 covers date/time types and their precision). |

Put together as an actual `CREATE TABLE` statement:

```sql
CREATE TABLE IF NOT EXISTS products (
    id                 SERIAL,
    name               TEXT,
    price              NUMERIC(10, 2),
    quantity_in_stock  INTEGER,
    is_available       BOOLEAN,
    created_at         TIMESTAMP
);
```

Run in `psql`:

```
sql_course=# CREATE TABLE IF NOT EXISTS products (
sql_course(#     id                 SERIAL,
sql_course(#     name               TEXT,
sql_course(#     price              NUMERIC(10, 2),
sql_course(#     quantity_in_stock  INTEGER,
sql_course(#     is_available       BOOLEAN,
sql_course(#     created_at         TIMESTAMP
sql_course(# );
CREATE TABLE
```

Notice `psql` shows a continuation prompt (`sql_course(#`) for every line until the closing `;` — exactly the multi-line statement behavior described in Module 1.

Confirm the table's shape with the `\d` meta-command (introduced in Module 1):

```
sql_course=# \d products
                                        Table "public.products"
       Column       |            Type             | Collation | Nullable |             Default
---------------------+------------------------------+-----------+----------+----------------------------------
 id                  | integer                     |           | not null | nextval('products_id_seq'::regclass)
 name                | text                        |           |          |
 price               | numeric(10,2)               |           |          |
 quantity_in_stock   | integer                     |           |          |
 is_available        | boolean                     |           |          |
 created_at          | timestamp without time zone |           |          |
```

Notice `id` already shows `not null` and a default value — this is a side effect of `SERIAL`, which PostgreSQL implements internally as an `INTEGER` column paired with an auto-incrementing sequence object and an implicit `NOT NULL` (fully explained in Module 3 and Module 5).

### Where Constraints Fit (Preview Only)

You may have expected `id` to be explicitly marked as a way to uniquely identify each row, or `name` to be required on every product. Those rules — "this value must be unique," "this column can never be empty," "this value must reference a row in another table" — are called **constraints**, and they can be attached directly inside a `CREATE TABLE` statement, immediately after a column's data type:

```sql
CREATE TABLE products (
    id     SERIAL PRIMARY KEY,
    name   TEXT NOT NULL,
    price  NUMERIC(10, 2)
);
```

Here, `PRIMARY KEY` and `NOT NULL` are constraints. At this stage, all you need to know is:
- A **column constraint** is written directly after a column's type and applies to just that column.
- A **table constraint** is written as its own separate entry (often at the end, after all columns), and can apply across multiple columns at once.
- Constraints are what turn a table from "a shape that happens to hold whatever you put into it" into "a shape that actively refuses data that would break your rules."

The complete catalog of constraint types (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `NOT NULL`) and exactly how each behaves is the entire subject of Module 5 (Constraints & Keys). This topic deliberately stops at "constraints exist, and here's roughly where they go" so you can focus fully on shaping columns first.

### Column Ordering

Columns appear in the order you list them, and that order is preserved in the table's metadata — it's what `SELECT * FROM products` returns by default, and what `\d products` displays. However, column order has no bearing on how you can *query* the table: `SELECT price, name FROM products` is completely valid and returns columns in whatever order you ask for (as already demonstrated in Module 1's `SELECT name, salary` example) — the table's declared order is just a *default* display order, not a constraint on querying.

Column order can have a minor effect on physical storage size in PostgreSQL, because of a low-level detail called data alignment padding (grouping same-sized types together can reduce wasted space per row on disk). This is a micro-optimization, not a correctness concern, and is not worth agonizing over while learning — it's mentioned only so you're not surprised if you ever see it discussed in a performance-focused context (Module 20).

### Schema-Qualified Names

Every table actually lives inside a **schema** — a named namespace within a database that groups related objects together. If you don't specify one, PostgreSQL uses the default schema, called `public`. This is why `\d products` above printed `Table "public.products"` — that's the fully qualified name. You can always write it out explicitly:

```sql
CREATE TABLE public.products (
    id SERIAL,
    name TEXT
);
```

For this course, you can safely rely on the `public` default throughout — explicit schema management (creating and using multiple schemas deliberately) is a more advanced organizational topic outside this module's scope, but it's useful to recognize `schema.table` notation when you see it.

### Naming Rules and Quoting

Unquoted table and column names in PostgreSQL are automatically lowercased and must start with a letter or underscore, followed by letters, digits, or underscores. `CREATE TABLE Products` and `CREATE TABLE products` create the *same* table name (`products`) — PostgreSQL folds unquoted identifiers to lowercase. If you genuinely need mixed case, spaces, or a reserved word as a name, you must double-quote it every single time you reference it:

```sql
CREATE TABLE "Products" (
    "Product Name" TEXT
);
```

This is rarely a good idea — it forces exact-case, exact-quoting discipline on every future query — and is mentioned here mainly so that if you ever see a name that looks unusually capitalized or spaced, you know it was deliberately double-quoted at creation time. Prefer plain, lowercase, underscore-separated names (`quantity_in_stock`, not `"Quantity In Stock"`).

## Internal Working (Preview)

When `CREATE TABLE` runs, conceptually:

```
CREATE TABLE products (...)
        │
        ▼
 PostgreSQL validates every column's data type and constraint syntax
        │
        ▼
 A new row is added to the system catalog pg_class (recording the table itself)
 and pg_attribute (recording each column, its type, and its position)
        │
        ▼
 If any column is SERIAL, a backing sequence object is also created
 (this is what produced the products_id_seq you saw in \d products)
        │
        ▼
 Empty physical storage is allocated for the table's future rows
```

The table does not yet contain any rows — `CREATE TABLE` only establishes structure and metadata (recorded in the catalog), exactly as previewed for DDL statements generally in Module 1. Nothing is populated until `INSERT` (Module 6) runs against it.

## Real-World Analogy

Designing and running a `CREATE TABLE` statement is like designing and building a filing cabinet drawer from scratch before anyone files a single document into it. You decide how many labeled slots (columns) the drawer will have, what kind of document each slot is meant to hold (its data type — a slot sized for photographs vs. one sized for invoices), and which slots are mandatory versus optional (constraints, previewed here). Only once the drawer exists with its labeled slots can anyone actually start filing real documents (`INSERT`) into it.

## Why `CREATE TABLE` Was Designed This Way

Requiring an explicit, upfront structure — rather than letting a table's shape emerge implicitly from whatever the first row inserted happens to look like — is a direct consequence of the relational model's insistence (Module 2) that every row in a table share the exact same well-defined columns and types. This upfront rigidity is precisely what lets the DBMS validate every future `INSERT` and `UPDATE` automatically, without your application ever having to re-check "does this value make sense for this column" itself — the same "centralized rule enforcement" advantage of relational databases introduced in Module 1. The separation of column constraints (attached to one column) from table constraints (spanning multiple columns) mirrors the reality that some rules are inherently about a single value, while others (like "the combination of `order_id` and `product_id` must be unique together") are inherently about a relationship between values — a distinction Module 5 explores fully.

## Advantages

- **Structure is validated once, enforced forever** — every future row must conform to the declared columns and types, without your application needing to re-verify this itself.
- **Self-documenting** — a well-named, well-typed `CREATE TABLE` statement (or its `\d` output) tells any future reader exactly what data the table is meant to hold, without needing to inspect actual rows.
- **`IF NOT EXISTS` makes setup scripts safely repeatable** — running the same setup script twice doesn't error out the second time, which matters for onboarding new environments or CI pipelines.

## Disadvantages / Limitations

- **Upfront rigidity** — every row must fit the declared shape; a genuinely variable, unpredictable structure (e.g., wildly different attributes per product category) doesn't fit naturally into a single fixed set of columns without workarounds (Module 21 covers using JSON columns for this specific situation).
- **Changing your mind later has a cost** — unlike a spreadsheet where you can freely add a column mid-use, altering an existing table's structure once it holds real data requires careful `ALTER TABLE` operations (Topic 3 of this module), some of which are expensive or risky on large, live tables.
- **No enforcement of business rules by default** — a bare `CREATE TABLE` with no constraints, as shown in the first example above, will happily accept a negative price or a duplicate product; only adding constraints (Module 5) closes those gaps.

## Best Practices

- Design columns from the actual requirement, not from whatever occurs to you first — write out, in plain language, every piece of information the row needs to hold, then map each to a name and a type, as done in this topic's worked example.
- Use plain, lowercase, underscore-separated names for tables and columns (`quantity_in_stock`) and avoid double-quoted, case-sensitive identifiers — they create ongoing friction in every query that touches them.
- Choose the most specific, correct data type for each column now, rather than defaulting to `TEXT` for everything — Module 3 exists precisely because type choice has real consequences (validation, storage size, available operations).
- Use `IF NOT EXISTS` in setup scripts you expect to re-run (onboarding, CI, local resets), but not as a substitute for actually knowing whether a table is supposed to already exist in a one-off, deliberate creation.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming column order in `CREATE TABLE` restricts how you can `SELECT` later. | Declared column order only affects `SELECT *` and metadata display order; any `SELECT` can name columns in any order regardless of how the table was defined. |
| Using `TEXT` or a generic type for every column "to keep things simple." | Defeats the purpose of typed columns — you lose validation (a "price" column that accepts arbitrary text), lose type-specific operations (date arithmetic, numeric comparison), and often waste storage. |
| Forgetting that unquoted identifiers are lowercased, then being confused why `"Products"` (double-quoted) doesn't match a table created as `Products` (unquoted, which became `products`). | Unquoted names are folded to lowercase automatically; double-quoted names are taken completely literally, case included — mixing the two conventions causes "table does not exist"-style errors that are confusing until you know this rule. |
| Expecting `CREATE TABLE` alone to enforce rules like "price must be positive" or "name is required." | A bare `CREATE TABLE` with no constraints enforces only data *types*, not business rules — those require explicitly added constraints, covered in Module 5. |

## Interview Questions

1. **Q: Walk through how you would design a table for a given real-world requirement.**
   A: List every distinct piece of information the requirement mentions, then for each one choose a clear column name and the most specific correct data type available (an integer for counts, `NUMERIC` for exact monetary values, `BOOLEAN` for true/false flags, `TIMESTAMP` for points in time, `TEXT` for free-form strings). Only after the shape is right do you consider which columns need constraints to enforce rules like uniqueness or mandatory presence.

2. **Q: Does the order in which columns are declared in `CREATE TABLE` limit how you can query them later?**
   A: No. Declared order only determines the default display order (used by `SELECT *` and tools like `\d`) — any `SELECT` statement can list columns in any order you choose, regardless of the table's underlying declared order.

3. **Q: What does `IF NOT EXISTS` do in a `CREATE TABLE` statement, and when is it appropriate to use?**
   A: It makes PostgreSQL silently skip table creation (instead of raising an error) if a table with that name already exists. It's appropriate in setup or provisioning scripts meant to be safely re-run multiple times (for example, in a fresh test environment), but shouldn't be used as a substitute for genuinely knowing whether the table is expected to already exist.

4. **Q: What is the difference between a column constraint and a table constraint?**
   A: A column constraint is written directly after a single column's data type and applies only to that column (e.g., `name TEXT NOT NULL`). A table constraint is declared as its own separate entry in the `CREATE TABLE` statement and can span or reference multiple columns together (e.g., requiring a combination of two columns to be jointly unique). Both are fully covered in Module 5.

## Summary

- `CREATE TABLE table_name (column definitions...)` defines a new table's structure inside the current database; `IF NOT EXISTS` makes the statement safe to re-run without erroring if the table already exists.
- Designing a table well means translating a real-world requirement into precisely named, precisely typed columns — get the types right using the vocabulary from Module 3.
- Constraints (`NOT NULL`, `PRIMARY KEY`, and others) can be attached to columns or declared at the table level to enforce rules beyond plain data types, but their full behavior is deferred to Module 5 — this topic only establishes that they exist and roughly where they're written.
- Declared column order affects default display only, not what you can `SELECT`; tables live inside a schema (`public` by default), and unquoted identifiers are automatically lowercased.
- Next, Topic 3 covers how to safely reshape a table that already exists — adding, removing, and renaming columns, and changing their types — once real requirements inevitably change.
