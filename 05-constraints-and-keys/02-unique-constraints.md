# UNIQUE Constraints

## Learning Objectives

By the end of this section you should be able to:
- Declare a single-column `UNIQUE` constraint and explain exactly what it prevents
- Declare a multi-column (composite) `UNIQUE` constraint and explain how it differs from uniqueness on each column individually
- Predict, correctly, how PostgreSQL treats `NULL` values under a `UNIQUE` constraint
- Explain the relationship between a `UNIQUE` constraint and the index PostgreSQL creates to enforce it

## Prerequisites

- [NOT NULL and DEFAULT](01-not-null-and-default.md) — you need to already be comfortable with the idea of a constraint rejecting an `INSERT`/`UPDATE`, and with `NULL`'s meaning as "absence of a value," which is central to this topic's discussion of `NULL` under `UNIQUE`.

## Motivation

Some facts about the real world are supposed to never repeat: two customers should never share the same email address if that email is how you identify and contact them; a product's SKU (stock-keeping unit) should never be assigned to two different products. Without a way to enforce this, duplicate rows creep in silently — a customer signs up twice with the same email through a race condition in your application, or two products accidentally get the same SKU during a bulk import — and every piece of code that assumes "one row per email" or "one row per SKU" quietly starts producing wrong results.

## Problem Statement

Continuing the `customers` table from the previous topic:

```sql
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    TEXT NOT NULL,
    email        TEXT NOT NULL
);

INSERT INTO customers (full_name, email) VALUES ('Asha Rao', 'asha@example.com');
INSERT INTO customers (full_name, email) VALUES ('Asha R.',  'asha@example.com');
```

Both inserts succeed. You now have two completely separate customer rows sharing the same email address:

```
 customer_id | full_name |        email
-------------+-----------+----------------------
           1 | Asha Rao  | asha@example.com
           2 | Asha R.   | asha@example.com
(2 rows)
```

If your application logs a customer in "by email," which row does it mean? If it sends a password-reset email, which account does it reset? `NOT NULL` guarantees `email` is never *missing* — it says nothing about whether it's *repeated*. A completely separate constraint is needed to say "this value, across the whole table, must be one-of-a-kind."

## Concept

### Single-Column UNIQUE

```sql
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    TEXT NOT NULL,
    email        TEXT NOT NULL UNIQUE
);

INSERT INTO customers (full_name, email) VALUES ('Asha Rao', 'asha@example.com');
INSERT INTO customers (full_name, email) VALUES ('Asha R.',  'asha@example.com');
```

```
ERROR:  duplicate key value violates unique constraint "customers_email_key"
DETAIL:  Key (email)=(asha@example.com) already exists.
```

The second insert is rejected the moment PostgreSQL detects the value already exists somewhere in the table for that column. `UNIQUE` can also be written as a separate, named table-level constraint rather than inline on the column — functionally identical, but useful when you want control over the constraint's name:

```sql
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    TEXT NOT NULL,
    email        TEXT NOT NULL,
    CONSTRAINT customers_email_unique UNIQUE (email)
);
```

Naming it explicitly (`customers_email_unique`) is useful because that's the name you'll reference if you ever need to drop it: `ALTER TABLE customers DROP CONSTRAINT customers_email_unique;`.

### Multi-Column (Composite) UNIQUE

Sometimes no single column is unique on its own, but a *combination* of columns should never repeat. Consider a `product_reviews` table, where a given customer should only be allowed to leave one review per product (but many different customers can review the same product, and one customer can review many different products):

```sql
CREATE TABLE product_reviews (
    review_id   SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    product_id  INTEGER NOT NULL,
    rating      INTEGER NOT NULL,
    UNIQUE (customer_id, product_id)
);

INSERT INTO product_reviews (customer_id, product_id, rating) VALUES (1, 100, 5);
INSERT INTO product_reviews (customer_id, product_id, rating) VALUES (1, 200, 4);  -- fine: different product
INSERT INTO product_reviews (customer_id, product_id, rating) VALUES (2, 100, 3);  -- fine: different customer
INSERT INTO product_reviews (customer_id, product_id, rating) VALUES (1, 100, 1);  -- violates: same pair repeated
```

```
ERROR:  duplicate key value violates unique constraint "product_reviews_customer_id_product_id_key"
DETAIL:  Key (customer_id, product_id)=(1, 100) already exists.
```

The crucial distinction: `UNIQUE (customer_id, product_id)` does **not** mean `customer_id` alone must be unique, nor that `product_id` alone must be unique — it means the *pair, taken together*, must be unique. Customer 1 can appear in many rows (reviewing many different products), and product 100 can appear in many rows (reviewed by many different customers) — the constraint only blocks the exact same pair from appearing twice.

| Constraint | What repeats freely | What can never repeat |
|---|---|---|
| `UNIQUE (customer_id)` | nothing — each customer can only appear once, period | `customer_id` alone |
| `UNIQUE (product_id)` | nothing — each product can only appear once, period | `product_id` alone |
| `UNIQUE (customer_id, product_id)` | `customer_id` alone (across different products); `product_id` alone (across different customers) | the exact `(customer_id, product_id)` combination |

### How `NULL` Interacts With `UNIQUE` in PostgreSQL

This is the single most surprising behavior in this topic for newcomers. Add an optional `phone` column and make it `UNIQUE`:

```sql
CREATE TABLE customers (
    customer_id  SERIAL PRIMARY KEY,
    full_name    TEXT NOT NULL,
    email        TEXT NOT NULL UNIQUE,
    phone        TEXT UNIQUE
);

INSERT INTO customers (full_name, email, phone) VALUES ('Asha Rao',   'asha@example.com',  NULL);
INSERT INTO customers (full_name, email, phone) VALUES ('Ben Ochieng','ben@example.com',   NULL);
INSERT INTO customers (full_name, email, phone) VALUES ('Chen Wei',   'chen@example.com',  '555-0100');
```

All three inserts succeed:

```
 customer_id |  full_name  |        email        |  phone
-------------+-------------+----------------------+----------
           1 | Asha Rao    | asha@example.com     |
           2 | Ben Ochieng | ben@example.com      |
           3 | Chen Wei    | chen@example.com     | 555-0100
(3 rows)
```

Two rows have `NULL` in `phone`, and the `UNIQUE` constraint did not object. **In the SQL standard and in PostgreSQL, `NULL` is never considered equal to another `NULL` for the purposes of a `UNIQUE` constraint — every `NULL` is treated as distinct from every other `NULL`.** This follows directly from `NULL`'s meaning: it represents an *unknown or absent* value, and it would be logically incoherent to declare two unknowns "equal" to each other just because neither is known. As a direct consequence, a `UNIQUE` column that also allows `NULL` (i.e., not additionally marked `NOT NULL`) can have any number of rows with `NULL` in it — the constraint only ever blocks a *second occurrence of the same known, non-`NULL` value*.

If you actually want "at most one row may have `NULL` here" (a stricter rule than standard SQL's default), that requires a different tool entirely — a partial unique index — which is outside standard `UNIQUE` constraint syntax and is covered as part of Module 13 (Indexes).

### UNIQUE Constraints vs. Unique Indexes

A `UNIQUE` constraint is not a separate enforcement mechanism from an index — PostgreSQL implements every `UNIQUE` constraint by automatically creating a **unique index** behind the scenes on the constrained column(s). You can see this directly:

```sql
\d customers
```

```
...
Indexes:
    "customers_pkey" PRIMARY KEY, btree (customer_id)
    "customers_email_key" UNIQUE CONSTRAINT, btree (email)
    "customers_phone_key" UNIQUE CONSTRAINT, btree (phone)
```

Each `UNIQUE` constraint shows up as a `btree` index in the table's index listing. This means every `UNIQUE` constraint gives you two things simultaneously, for free: the uniqueness guarantee itself, *and* fast lookups on that column, since the underlying index can be used by the query planner to accelerate `WHERE email = '...'` just as effectively as an index you created by hand. It also means a `UNIQUE` constraint has the same storage and write-overhead cost as any other index — every `INSERT`/`UPDATE` touching that column must also update the index. Module 13 (Indexes) covers how indexes work internally, how to create them independently of a constraint, and how to read the query planner's use of them in detail — for now, the important takeaway is simply that "unique constraint" and "unique index" are two names circling the same underlying mechanism, not two independent features.

## Internal Working (Preview)

```
INSERT/UPDATE row with value V into unique-constrained column
        │
        ▼
Look up V in the constraint's underlying B-tree index
        │
        ▼
  Does a non-NULL entry equal to V already exist?
        │
   ┌────┴────┐
  yes        no
   │          │
 REJECT    Insert V into the index, write the row
```

Because the check is an index lookup rather than a full table scan, verifying uniqueness stays fast even as a table grows to millions of rows — this is precisely why the constraint is implemented as an index rather than, say, re-scanning the whole table on every write.

## Real-World Analogy

Think of a coat check at a theater. Each ticket stub number given out must be unique — no two coats can be claimed with the same stub, or the wrong person walks off with someone else's coat. That's a single-column `UNIQUE`. Now imagine a shared office parking garage where a "unique" resource is really the *pair* (parking spot number, day of the week) — spot 12 can be used by different people on different days, and the same day can see many different spots occupied, but spot 12 on a Tuesday can only ever be assigned to one person. That's composite `UNIQUE`. And the `NULL` behavior is like a coat check that has both "cataloged" coats (with a stub) and unclaimed hooks with no stub at all yet — any number of coats can sit on hooks with "no stub assigned," because the rule "no duplicate stub numbers" simply doesn't apply to coats that don't have one.

## Why UNIQUE Was Designed This Way

`UNIQUE`, like `NOT NULL`, exists to move an integrity rule out of application code and into the one place guaranteed to see every write to the table. Its specific `NULL` behavior follows directly and necessarily from what `NULL` already means in the relational model (Module 3): if `NULL` represents "value unknown/not applicable," then two rows both lacking a value aren't making a *conflicting claim* about the world the way two rows both claiming `'asha@example.com'` are — there's nothing to contradict. Rather than inventing a special case, SQL treats `NULL` consistently with its meaning everywhere else in the language: it is never considered equal to anything, including another `NULL` (the same reason `NULL = NULL` evaluates to `NULL`, not `TRUE`, which Module 3 and later modules on `WHERE`/`JOIN` behavior return to repeatedly).

## Advantages

- **Prevents silent duplication** of values that must identify or represent something one-of-a-kind (emails, SKUs, usernames) without any application-level checking.
- **Composite `UNIQUE` expresses real-world business rules directly** (e.g., "one review per customer per product") that a single-column constraint can't capture.
- **Comes with a free index** — the uniqueness check is fast even at scale, and the same index accelerates ordinary lookups and joins on that column.
- **Enforced atomically at the database level**, so it's immune to race conditions between two near-simultaneous inserts from different application instances — a check-then-insert pattern written in application code can lose this race; the database constraint cannot.

## Disadvantages / Limitations

- **Every unique constraint adds index-maintenance overhead** to every `INSERT`/`UPDATE` touching that column — a table with many `UNIQUE` constraints pays a small write-performance cost on each of them.
- **Multiple `NULL`s being allowed is sometimes not what's actually wanted** — if the true business rule is "at most one row may ever have no value here," a plain `UNIQUE` constraint alone can't express that; it requires a partial unique index (Module 13).
- **A composite `UNIQUE` constraint says nothing about the individual columns** — a common mistake is assuming `UNIQUE (a, b)` also implies `a` alone or `b` alone is unique, which it does not.

## Best Practices

- Add `UNIQUE` to any column whose entire purpose is to identify something to the outside world (emails, usernames, SKUs, ISBNs) — don't rely on application-level "check if it exists first, then insert" logic alone, since that pattern is vulnerable to race conditions the database constraint isn't.
- Give multi-column `UNIQUE` constraints explicit, meaningful names (`CONSTRAINT one_review_per_customer_per_product UNIQUE (customer_id, product_id)`) so that a future constraint-violation error message is immediately self-explanatory rather than a generic auto-generated name.
- If you need "unique, but multiple `NULL`s are not okay," don't fight the standard `UNIQUE` constraint — reach for a partial unique index instead (Module 13), which can express exactly that.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `UNIQUE (a, b)` means `a` and `b` are each individually unique | It only constrains the *combination* — either column alone can repeat freely across different values of the other. |
| Assuming a `UNIQUE` column can only ever have one `NULL` row | PostgreSQL (and standard SQL) treats every `NULL` as distinct from every other `NULL`, so any number of rows can have `NULL` in a `UNIQUE` column unless it's also `NOT NULL`. |
| Believing `UNIQUE` and an index are two separate things you might need to create both of | A `UNIQUE` constraint *is* implemented as a unique index — creating the constraint already creates the index; you don't need a separate `CREATE INDEX` for the same columns. |
| Relying only on an application-level existence check ("look up the email, insert only if not found") instead of a database `UNIQUE` constraint | Two near-simultaneous requests can both pass the "not found" check before either has inserted, both then insert, and you end up with a duplicate — a race condition a database-level `UNIQUE` constraint closes entirely. |

## Interview Questions

1. **Q: Does a `UNIQUE` constraint allow more than one row with `NULL` in that column?**
   A: Yes, in standard SQL and in PostgreSQL. `NULL` represents an unknown/absent value and is never considered equal to another `NULL`, so a `UNIQUE` constraint (without an accompanying `NOT NULL`) permits any number of rows with `NULL` — it only rejects a second occurrence of the same known, non-`NULL` value.

2. **Q: What's the difference between `UNIQUE (a, b)` and having separate `UNIQUE (a)` and `UNIQUE (b)` constraints?**
   A: `UNIQUE (a, b)` only forbids the exact pair of values from repeating together — `a` alone and `b` alone can each repeat across different rows. Separate `UNIQUE (a)` and `UNIQUE (b)` constraints instead forbid *either* column from ever repeating on its own, which is a much stronger and different guarantee.

3. **Q: How does PostgreSQL actually enforce a `UNIQUE` constraint internally?**
   A: By automatically creating a unique B-tree index on the constrained column(s). Every `INSERT`/`UPDATE` checks the new value against that index; if a matching non-`NULL` entry already exists, the operation is rejected. This is why a `UNIQUE` constraint gives you fast lookups on that column as a side effect, at the cost of index-maintenance overhead on every write.

## Summary

- `UNIQUE` guarantees a column (or set of columns) never holds the same non-`NULL` value twice across the whole table.
- A composite `UNIQUE (a, b)` constrains only the *combination* of `a` and `b`, not either column individually.
- PostgreSQL treats every `NULL` as distinct from every other `NULL` under `UNIQUE`, so multiple rows can have `NULL` in a unique column unless it's also `NOT NULL`.
- A `UNIQUE` constraint is implemented internally as a unique index — you get uniqueness enforcement and fast lookups on that column together, automatically. Module 13 covers indexes, including partial unique indexes, in full depth.
- Prefer a database-level `UNIQUE` constraint over an application-level "check then insert" pattern, since the constraint is immune to race conditions that the application-level check is not.
