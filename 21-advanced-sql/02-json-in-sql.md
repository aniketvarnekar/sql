# JSON in SQL

## Learning Objectives

By the end of this section you should be able to:
- Explain the difference between the `JSON` and `JSONB` types and state why `JSONB` is almost always the better default
- Store semi-structured data in a `JSONB` column alongside ordinary typed columns in the same table
- Query JSON values with `->`, `->>`, and `#>`, and explain what each one returns
- Filter rows using `JSONB`'s containment (`@>`) and existence (`?`) operators
- Create a GIN index on a `JSONB` column and explain why a B-tree index wouldn't serve the same purpose
- Judge, for a given piece of data, whether it belongs in a `JSONB` column or should be normalized into proper tables and rows

## Prerequisites

This topic assumes **Module 3 — Data Types** (see the [Module 3 overview](../03-data-types/00-module-overview.md)), which already established that every column has one declared, enforced type and briefly noted that `JSON`/`JSONB` exist as PostgreSQL types without going deep on them — this topic is that deferred deep dive. It also assumes **Module 13 — Indexes**, specifically [What Is an Index?](../13-indexes/01-what-is-an-index.md) and [B-Tree and Composite Indexes](../13-indexes/02-b-tree-and-composite-indexes.md), since this topic introduces a second, structurally different index type (GIN) and contrasts it directly with the B-tree you already know.

## Motivation

Not all real-world data arrives in a neat, fixed set of columns known in advance. A product catalog spanning shoes, electronics, and furniture has wildly different "attributes" per category; a webhook payload from a third-party service has a shape you don't control and that can change without warning; a user's notification preferences might be an open-ended, evolving set of toggles. Forcing all of this into rigid, individually-declared columns either produces tables with hundreds of mostly-`NULL` columns, or a painful multi-table redesign every time the shape of the data changes slightly. `JSON`/`JSONB` give you a way to keep this kind of data *inside* a normal relational table, sitting right next to strictly-typed columns, without abandoning the relational model or the guarantees (transactions, constraints, a single source of truth) that come with it.

## Problem Statement

Imagine a `products` table where every product needs a name and a price (universal, structured columns), but each product's other attributes vary enormously by category — a pair of shoes has a size and color; a power adapter has a voltage and plug type; a bookshelf has dimensions and a material. Two bad options present themselves without JSON:

- **One column per possible attribute across every category** — `size`, `color`, `voltage`, `plug_type`, `width_cm`, `material`, and dozens more, with the overwhelming majority of columns `NULL` for any given row, and a new `ALTER TABLE` required every time a new product category introduces a new attribute.
- **An Entity-Attribute-Value (EAV) design** — a separate `product_attributes` table with `(product_id, attribute_name, attribute_value)` rows — technically flexible, but every query that wants "this product's size and color together" needs several self-joins or pivoting (Topic 1) just to reassemble one product's attributes back into a readable row, and there's no natural place to enforce that an attribute's value is the right type.

`JSONB` offers a third option: keep the universal, always-present, frequently-filtered attributes (`name`, `price`) as ordinary typed columns, and store the variable, per-category attributes in a single `JSONB` column, queryable directly.

## Concept

### `JSON` vs. `JSONB`

PostgreSQL provides two types for storing JSON documents:

| Aspect | `JSON` | `JSONB` |
|---|---|---|
| Storage | Exact copy of the input text | Decomposed binary format |
| Whitespace, key order | Preserved exactly as entered | Not preserved (keys are reordered/deduplicated) |
| Duplicate keys | All copies kept; only the last is used when queried | Silently deduplicated at input time — only the last value for a repeated key is kept |
| Input speed | Slightly faster (no parsing at insert time) | Slightly slower (parsed and reorganized at insert time) |
| Query/read speed | Slower (re-parsed as text on every access) | Faster (already in a structured, navigable form) |
| Indexable with GIN | No | Yes |
| Supports containment/existence operators (`@>`, `?`) | No | Yes |

`JSONB` pays its parsing cost once, at write time, and every subsequent read benefits from already having a structured, indexable representation. `JSON` pays no cost at write time but re-parses the entire text on every single read. In practice, **use `JSONB` by default.** The only real reason to reach for plain `JSON` is when you need to preserve the *exact original text* byte-for-byte — including whitespace, key ordering, or duplicate keys — which is a genuinely rare requirement (occasionally useful for storing a raw webhook payload verbatim for audit purposes, but not for data you intend to query).

### Storing `JSONB` Alongside Ordinary Columns

```sql
CREATE TABLE products (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    price      NUMERIC(10, 2) NOT NULL,
    attributes JSONB
);

INSERT INTO products (name, price, attributes) VALUES
    ('Trail Runner Sneaker', 89.99,
     '{"category": "shoes", "size": 10, "color": "red", "waterproof": true}'),
    ('USB-C Power Adapter', 24.50,
     '{"category": "electronics", "voltage": 240, "plug_type": "USB-C", "warranty_months": 12}'),
    ('Oak Bookshelf', 145.00,
     '{"category": "furniture", "material": "oak", "dimensions_cm": {"width": 80, "height": 180, "depth": 30}}');
```

`id`, `name`, and `price` are ordinary, strictly-typed columns exactly as in every earlier module — they can be `NOT NULL`, uniquely constrained, indexed with a B-tree, and joined against other tables normally. `attributes` holds a genuinely different shape of data per row, and PostgreSQL is completely unbothered by that, because `JSONB` is just another column type — the table is still one relation with a fixed set of columns; only the *contents* of one particular column are semi-structured.

### Querying JSON: `->`, `->>`, and `#>`

Three operators extract data out of a JSON/JSONB value, and the difference between them matters:

| Operator | Input | Returns | Use it when |
|---|---|---|---|
| `->` | key or array index | `json`/`jsonb` (still JSON) | You need to chain further JSON operations on the result |
| `->>` | key or array index | `text` | You need the value as an ordinary SQL text value (for comparison, casting, display) |
| `#>` | an array of keys/indexes (a path) | `json`/`jsonb` | You need to reach into a *nested* object several levels deep |
| `#>>` | an array of keys/indexes (a path) | `text` | Same as `#>` but as text |

```sql
SELECT name, attributes -> 'category' AS category_json,
             attributes ->> 'category' AS category_text
FROM products;
```

```
         name          | category_json |  category_text
------------------------+---------------+------------------
 Trail Runner Sneaker   | "shoes"       | shoes
 USB-C Power Adapter    | "electronics" | electronics
 Oak Bookshelf          | "furniture"   | furniture
(3 rows)
```

Notice `->` returns `"shoes"` — still a JSON value, quoted, because its type is `jsonb`, not `text` — while `->>` returns `shoes` with no quotes, because it's converted straight to a plain SQL `text` value. This distinction matters the moment you want to filter or compare:

```sql
SELECT name FROM products WHERE attributes ->> 'category' = 'electronics';
```

```
         name
----------------------
 USB-C Power Adapter
(1 row)
```

`attributes -> 'category' = 'electronics'` would *not* match here — the left side is a `jsonb` value (`"electronics"`, with quotes) and the right side is a plain text literal, and they're not the same type or value; you almost always want `->>` when comparing against an ordinary SQL literal.

For nested structures — like the bookshelf's `dimensions_cm` object — `#>` and `#>>` take a path (an array of keys) to reach several levels deep in one step:

```sql
SELECT name, attributes #>> '{dimensions_cm,width}' AS width_cm
FROM products
WHERE name = 'Oak Bookshelf';
```

```
     name      | width_cm
----------------+----------
 Oak Bookshelf | 80
(1 row)
```

This is equivalent to, but more concise than, chaining `->` twice and finishing with `->>`: `attributes -> 'dimensions_cm' ->> 'width'`.

### `JSONB` Containment and Existence Operators

Beyond extracting values, `JSONB` supports operators that ask questions *about* the document's shape and contents directly:

**Containment (`@>`)** — does the left `JSONB` value contain the right one as a sub-document?

```sql
SELECT name FROM products WHERE attributes @> '{"category": "shoes"}';
```

```
         name
------------------------
 Trail Runner Sneaker
(1 row)
```

```sql
SELECT name FROM products WHERE attributes @> '{"waterproof": true}';
```

```
         name
------------------------
 Trail Runner Sneaker
(1 row)
```

**Existence (`?`)** — does this top-level key exist in the document at all, regardless of its value?

```sql
SELECT name FROM products WHERE attributes ? 'warranty_months';
```

```
        name
----------------------
 USB-C Power Adapter
(1 row)
```

Two related operators check for *multiple* keys at once: `?|` (does at least one of these keys exist) and `?&` (do *all* of these keys exist) — both take an array of text values, e.g. `attributes ?| array['voltage', 'material']`.

### Indexing `JSONB` with GIN

A plain B-tree index (Module 13) is built to accelerate equality and range comparisons on a single, ordinary scalar value — it has no way to efficiently answer "does this document contain this key?" or "does this document contain this sub-structure?" across potentially many keys per row. PostgreSQL instead offers a **GIN index** (Generalized Inverted Index) for exactly this job:

```sql
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
```

A GIN index on a `JSONB` column effectively indexes *every key and value* inside the document, so `@>`, `?`, `?|`, and `?&` queries can jump straight to matching rows instead of decoding and inspecting every row's JSON document one at a time. Once this index exists:

```sql
EXPLAIN SELECT name FROM products WHERE attributes @> '{"category": "electronics"}';
```

```
 Bitmap Heap Scan on products  (cost=12.02..16.04 rows=1 width=36)
   Recheck Cond: (attributes @> '{"category": "electronics"}'::jsonb)
   ->  Bitmap Index Scan on idx_products_attributes  (cost=0.00..12.02 rows=1 width=0)
         Index Cond: (attributes @> '{"category": "electronics"}'::jsonb)
```

PostgreSQL also offers a narrower GIN operator class, `jsonb_path_ops`, which produces a smaller, faster index but supports only `@>` (not `?`/`?|`/`?&`) — worth knowing exists, and a reasonable choice if containment is the only query pattern you need against that column: `CREATE INDEX ... USING GIN (attributes jsonb_path_ops);`.

### When JSON Is Appropriate — and When to Normalize Instead

| Reach for `JSONB` when... | Normalize into real tables/columns when... |
|---|---|
| Attributes genuinely vary by row/category and there's no fixed common shape | Every row has the same fields — that's just a case for ordinary columns |
| The data is a snapshot of an external payload (an API response, a webhook body) you want to retain close to its original shape | The data needs to be joined against, aggregated, or referenced by foreign keys from other tables |
| The nested data is rarely queried on its own — mostly read back wholesale, alongside the row | Individual nested fields are frequently filtered, sorted, summed, or need their own constraints (`NOT NULL`, `CHECK`, uniqueness) |
| The set of possible keys is open-ended or evolves without a schema migration | You need referential integrity on a value inside the document (e.g., it should reference a real row in another table) |

The general rule: `JSONB` is a good fit for data that is **read and written as a whole document** and whose internal shape is genuinely variable. The moment you find yourself writing frequent, performance-sensitive queries that filter, join, or aggregate on one specific nested field, that field is telling you it wants to be a real column — pull it out with `->>` (and a `GENERATED` column or a normal `ALTER TABLE ... ADD COLUMN`, if it's important enough), rather than continuing to reach into JSON for it on every query.

## Internal Working (Preview)

`JSON` stores exactly the text you inserted; every operator (`->`, `->>`, etc.) must **re-parse that entire text string from scratch** on every single access, every single time the row is read.

`JSONB` instead decomposes the document, at insert time, into a binary tree structure with keys sorted and indexed internally, so that later lookups can jump directly to a key rather than scanning character by character:

```
 JSON column:    '{"category": "shoes", "size": 10}'   (raw text, stored as-is)
                        │
              every read: re-parse the whole string
                        │
                        ▼
                 (slow, repeated work)

 JSONB column:   decomposed at INSERT time into a binary, key-sorted structure
                        │
              every read: binary search/lookup directly into the structure
                        │
                        ▼
                 (fast, done once)
```

A large `JSONB` (or `TEXT`) value that exceeds a size threshold (a few kilobytes) is transparently moved by PostgreSQL into a side storage mechanism called TOAST ("The Oversized-Attribute Storage Technique"), out of the main table's row storage, and fetched only when actually needed — this is invisible to your queries but explains why very large JSON documents impose more I/O cost than small ones even with an index in place.

## Real-World Analogy

A `JSON` column is like a handwritten paragraph you must re-read from the beginning every single time someone asks you a question about it — even a simple one like "what color is mentioned?" requires scanning the whole paragraph again. A `JSONB` column is like a filled-out form with labeled fields already sorted and tabbed — asking "what's in the color field?" means flipping directly to that tab, without rereading anything else on the form. Both documents contain the same information; only one of them was organized, once, to make repeated lookups fast.

## Why JSON Support Was Designed This Way

Module 1's discussion of the relational model's disadvantages noted that a strictly tabular structure can be awkward for data that doesn't naturally fit rows and fixed columns. Rather than forcing you to abandon the relational database entirely for this kind of data (and lose transactions, constraints, and joins in the process), PostgreSQL added `JSON`/`JSONB` as native column types that participate fully in the relational model — a `JSONB` column has a declared type just like any other (Module 3), lives inside an ordinary table alongside strictly-typed columns, is covered by the same transactional guarantees as every other column (Module 14), and can be indexed (just with a structurally different index, GIN, suited to its shape). This is a deliberate design choice to bring *bounded* schema flexibility into the relational model, rather than treating "some columns are variable" as a reason to leave it altogether.

## Advantages

- **Schema flexibility exactly where you need it** — new attributes can appear in the data without an `ALTER TABLE` migration.
- **Still fully relational** — `JSONB` columns participate in transactions, can sit in a table with foreign keys and constraints on its other columns, and can be joined against normally.
- **Indexable** — GIN indexes make containment and key-existence queries fast even at scale, unlike naively storing JSON as plain text.
- **Keeps related structured and semi-structured data physically together** — one row, one product, with both its fixed and variable attributes in the same place, rather than split across a separate flexible store entirely.

## Disadvantages / Limitations

- **No enforced internal structure** — nothing stops one row's `attributes` from having a `size` as a number and another's as a string, unless you add explicit `CHECK` constraints using JSON functions to validate shape, which is more work than a normal column's built-in type enforcement.
- **Harder to enforce per-field integrity** — you cannot easily attach a foreign key or a `NOT NULL` constraint to one specific key inside a JSON document the way you can to a real column.
- **Encourages denormalization if overused** — it's tempting to dump entire related record sets (like a list of order line items) into a JSON array instead of a proper child table, which then makes summing, filtering, or joining on that nested data far more awkward than ordinary rows would be.
- **Updates rewrite the whole value** — there's no in-place partial update at the storage level; changing one nested field (even with the `jsonb_set()` function) still writes an entirely new `JSONB` value for that row.

## Best Practices

- Default to `JSONB`, not `JSON`, unless you have a specific, rare reason to preserve exact original text.
- Keep any field you filter, sort, join on, or aggregate frequently as a real, separate column — reserve `JSONB` for attributes that are genuinely variable or read as a whole.
- Add a GIN index as soon as a `JSONB` column is queried with `@>` or `?` at any meaningful scale — without one, every such query performs a full scan decoding every row's document.
- Don't use `JSONB` as a way to avoid schema design — if every row's document actually has the same shape, that's a sign the fields belong in ordinary columns, not a JSON blob.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Defaulting to `JSON` instead of `JSONB` | `JSON` re-parses the full text on every read and cannot be indexed with GIN or used with containment operators — there is almost never a reason to prefer it. |
| Comparing `attributes -> 'key'` to a plain text literal | `->` returns a `jsonb` value (quoted), not `text` — comparing it to `'value'` will not match; use `->>` when you need an ordinary SQL text value. |
| Storing data that needs to be joined or aggregated (like order line items) inside a JSON array instead of a proper child table | Defeats the relational model's join and aggregation tools — summing quantities or joining against a product table becomes far more awkward than plain rows in a normal table would be. |
| Querying a `JSONB` column with `@>` or `?` at scale without a GIN index | Forces PostgreSQL to decode and inspect every row's document individually — exactly the full-scan problem indexes exist to solve (Module 13), just applied to JSON instead of a scalar column. |

## Interview Questions

1. **Q: What's the difference between `JSON` and `JSONB`, and which should you use by default?**
   A: `JSON` stores the exact input text and re-parses it on every read; `JSONB` stores a decomposed binary form, parsed once at write time, which makes reads faster and enables GIN indexing and containment/existence operators. `JSONB` is the correct default for nearly every use case; `JSON` is only worth it when you must preserve the exact original text (whitespace, key order, duplicate keys).

2. **Q: What's the difference between `->` and `->>`?**
   A: `->` extracts a value and returns it still as `json`/`jsonb`; `->>` extracts the same value but returns it as plain SQL `text`. Use `->>` when comparing against an ordinary literal or when you need the value as text/for casting; use `->` when you need to chain further JSON operations on the result.

3. **Q: How would you find all rows whose `JSONB` column contains a specific key/value pair, and how would you make that query fast at scale?**
   A: Use the containment operator, e.g. `WHERE attributes @> '{"category": "electronics"}'`. To make it fast at scale, create a GIN index on the column (`CREATE INDEX ... ON table USING GIN (column)`), which lets PostgreSQL look the condition up directly instead of scanning and decoding every row's document.

4. **Q: You're designing a `products` table and a colleague suggests putting every product attribute into one `JSONB` column, including `price`, for maximum flexibility. What's your objection?**
   A: `price` is a universal, always-present attribute that is almost certainly filtered, sorted, and aggregated on (e.g. `WHERE price > 100`, `ORDER BY price`, `SUM(price)`) — keeping it as an ordinary `NUMERIC` column gives it proper type enforcement, efficient B-tree indexing, and simple arithmetic, none of which a JSON-embedded value gives you as cleanly. `JSONB` should hold only the attributes that genuinely vary by row; universal, heavily-queried fields belong in real columns.

## Summary

- `JSONB` (binary, decomposed, indexable) is almost always preferred over `JSON` (raw text, re-parsed on every read); default to `JSONB` unless exact text preservation is required.
- `->` and `#>` return JSON values; `->>` and `#>>` return plain SQL text — use the text-returning forms when comparing against ordinary literals.
- `@>` (containment) and `?`/`?|`/`?&` (key existence) are `JSONB`-only operators that ask questions about a document's shape and contents.
- A GIN index makes `@>`/`?` queries against a `JSONB` column fast at scale, the same way a B-tree index makes equality/range queries fast against an ordinary scalar column (Module 13) — just structured differently to suit JSON's shape.
- Use `JSONB` for genuinely variable, whole-document data; keep universal, frequently filtered/joined/aggregated fields as ordinary typed columns.
- Next, Topic 3 steps away from JSON entirely and looks at sequences — the mechanism quietly generating every `SERIAL` primary key value you've used since Module 1.
