# INNER JOIN

## Learning Objectives

By the end of this section you should be able to:
- Explain what a join is and why splitting data across related tables requires one
- Write an `INNER JOIN` using explicit `JOIN...ON` syntax
- Explain why `INNER JOIN` silently excludes rows that have no match on the other side
- Recognize old-style comma/`WHERE` joins and explain why explicit `JOIN...ON` is preferred
- Use a foreign key relationship as the natural, correct join condition

## Prerequisites

- **Module 5 — Constraints & Keys** (foreign keys) — a join condition is, in practice, almost always the exact relationship a foreign key already declares.
- **Module 7 — Querying Basics** (`SELECT`, `WHERE`) — this topic assumes fluent comfort with basic single-table queries; a join is written *inside* that same `SELECT` statement.
- Within this module: none — this is the first topic.

## Motivation

Module 4 and Module 5 taught you to deliberately split data across multiple related tables instead of duplicating everything into one giant table — a customer's name lives once in `customers`, not copied onto every order they've ever placed. That design is excellent for correctness (change a customer's email in exactly one place) but it creates an obvious new problem: almost every real question you want to ask ("which customer placed order #103?", "what's Ava's total spend?") requires data from *more than one table at once*. `INNER JOIN` is the tool that reconnects tables split apart by good design, back into one combined result, using the very foreign key relationship that split them apart in the first place.

## Problem Statement

Suppose you have the `customers` and `orders` tables from this module's running schema, and you're asked: "show me each order's ID, date, and the name of the customer who placed it." The `orders` table has a `customer_id`, but not a customer *name* — that lives only in `customers`. Querying `orders` alone gives you:

```
 order_id | customer_id | order_date
----------+-------------+------------
      101 |           1 | 2026-01-05
      102 |           1 | 2026-02-10
      103 |           2 | 2026-01-20
      104 |           3 | 2026-03-01
      105 |             | 2026-03-15
```

That `customer_id` column is technically correct but useless on its own to a human reading a report — nobody wants to see "2", they want to see "Ben Ortiz." You need a way to look up, for every row in `orders`, the matching row in `customers`, and stitch the two together into one row of output. That is exactly what a join does.

## Concept

### Setting Up the Example

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT NOT NULL,
    city        TEXT
);

CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date  DATE NOT NULL,
    status      TEXT NOT NULL
);

INSERT INTO customers (name, email, city) VALUES
    ('Ava Patel',  'ava@example.com',  'Austin'),
    ('Ben Ortiz',  'ben@example.com',  'Denver'),
    ('Chen Wu',    'chen@example.com', 'Austin'),
    ('Dara Singh', 'dara@example.com', 'Seattle');

INSERT INTO orders (customer_id, order_date, status) VALUES
    (1, '2026-01-05', 'shipped'),
    (1, '2026-02-10', 'shipped'),
    (2, '2026-01-20', 'cancelled'),
    (3, '2026-03-01', 'shipped'),
    (NULL, '2026-03-15', 'shipped');
```

Notice `customer_id` in `orders` is a **foreign key** referencing `customers.customer_id` (Module 5) — this is not a coincidence. It is *because* this relationship is declared and enforced by the database that we know exactly which column to join on. Also notice: Dara Singh (customer 4) has no orders at all, and order 105 has no `customer_id` (`NULL`) — representing a guest checkout with no linked account. Both details matter later in this module (Topic 2 and Topic 3) but are irrelevant to `INNER JOIN`, as you're about to see.

### The Explicit `JOIN...ON` Syntax

```sql
SELECT
    o.order_id,
    o.order_date,
    c.name AS customer_name
FROM orders o
INNER JOIN customers c
    ON o.customer_id = c.customer_id
ORDER BY o.order_id;
```

```
 order_id | order_date | customer_name
----------+------------+---------------
      101 | 2026-01-05 | Ava Patel
      102 | 2026-02-10 | Ava Patel
      103 | 2026-01-20 | Ben Ortiz
      104 | 2026-03-01 | Chen Wu
(4 rows)
```

Breaking this down:

| Piece | Meaning |
|---|---|
| `FROM orders o` | Start with the `orders` table, aliased `o` for brevity. |
| `INNER JOIN customers c` | Bring in the `customers` table too, aliased `c`. |
| `ON o.customer_id = c.customer_id` | The **join condition** — for each row in `orders`, find the row(s) in `customers` where this equality holds, and combine them into one output row. |

The keyword `INNER` is actually optional — plain `JOIN` means `INNER JOIN` in every major SQL database, including PostgreSQL. Most style guides (and this course) still write `INNER JOIN` explicitly, because it makes the join's behavior unambiguous to a reader without them needing to know or recall the default.

### Why Order 105 and Dara Singh Are Both Missing

Look closely at the result: it has exactly **4 rows**, even though `orders` has 5 rows and `customers` has 4 rows. Two rows were dropped:

- **Order 105** was dropped because its `customer_id` is `NULL` — there is no row in `customers` where `customer_id = NULL` can ever be true (`NULL` never equals anything, not even another `NULL` — a rule fully explained in Module 3's treatment of `NULL` semantics). No match, so `INNER JOIN` excludes it.
- **Dara Singh** was dropped because she has zero rows in `orders` referencing her `customer_id`. No match, so `INNER JOIN` excludes her too.

This is the single defining trait of `INNER JOIN`: **a row from either table only appears in the output if it has at least one matching row on the other side.** Unmatched rows from *both* tables are silently discarded. This is not a bug or an edge case to work around — it's the precise, intentional definition of what "inner" means in `INNER JOIN`, and it's exactly the right behavior when the question you're asking is "show me orders and their customers" (a guest order with no customer, or a customer with no orders, genuinely has nothing to show in that combined shape). Topic 2 introduces `LEFT`/`RIGHT JOIN` for when you want unmatched rows kept instead.

### Old-Style Comma/`WHERE` Joins

Before the `JOIN...ON` syntax became standard and universally supported, the same result was written using a comma in the `FROM` clause and the join condition moved into `WHERE`:

```sql
-- Old style — avoid this in new code, shown only so you can recognize it
SELECT
    o.order_id,
    o.order_date,
    c.name AS customer_name
FROM orders o, customers c
WHERE o.customer_id = c.customer_id
ORDER BY o.order_id;
```

This produces the **exact same result** as the explicit `JOIN...ON` version above. Mechanically, `FROM orders o, customers c` first forms every possible pairing of an `orders` row with a `customers` row (a Cartesian product — covered fully in Topic 3), and the `WHERE` clause then filters that huge intermediate set down to only the pairs where the customer IDs actually match.

This course recommends **always using explicit `JOIN...ON` syntax** in new code, for concrete reasons, not just style preference:

- **The join condition is visually separated from filtering conditions.** In the old style, `WHERE` does double duty — part of it defines *how tables relate* (`o.customer_id = c.customer_id`) and part of it might define *which rows you want* (e.g., `AND o.status = 'shipped'`), all mixed together with no visual distinction. `JOIN...ON` keeps "how do these tables relate" (`ON`) cleanly separate from "which rows do I want" (`WHERE`).
- **It's much harder to forget a join condition by accident.** Forgetting one clause in a long `WHERE` list with several `AND`s silently turns your query into an accidental cross join (Topic 3) — a serious, easy-to-miss bug. `JOIN...ON` makes each table's join condition explicit and impossible to accidentally omit without the syntax itself looking wrong.
- **Outer joins (Topic 2, Topic 3) have no clean comma-style equivalent.** The comma syntax genuinely cannot express `LEFT JOIN`, `RIGHT JOIN`, or `FULL OUTER JOIN` correctly and portably — so learning explicit `JOIN` syntax now means you already know the syntax you'll need for every other join type in this module.

You will still see the comma/`WHERE` style in older production codebases and in some textbooks, which is why it's worth being able to read — but write explicit `JOIN...ON` in everything you produce.

### Using the Foreign Key as the Natural Join Condition

In this example, `ON o.customer_id = c.customer_id` is not an arbitrary choice — it is *exactly* the relationship `orders.customer_id REFERENCES customers(customer_id)` already declares (Module 5). This is true of the overwhelming majority of joins you will write in real schemas: **the join condition is the foreign key relationship, written out explicitly.** The foreign key is what *guarantees* that every non-`NULL` `customer_id` in `orders` corresponds to a real row in `customers` — without that guarantee, joins could silently produce nonsensical or incomplete results due to orphaned references, and you'd have no structural reason to trust that your join condition covers the data correctly.

### Joining on Multiple Conditions

A join condition isn't limited to a single equality — you can `AND` together multiple conditions in the `ON` clause, exactly as you would in a `WHERE` clause:

```sql
SELECT o.order_id, c.name
FROM orders o
INNER JOIN customers c
    ON o.customer_id = c.customer_id
    AND c.city = 'Austin';
```

```
 order_id |   name
----------+-----------
      101 | Ava Patel
      102 | Ava Patel
      104 | Chen Wu
(3 rows)
```

Note the difference between putting `c.city = 'Austin'` in the `ON` clause (as above) versus a separate `WHERE c.city = 'Austin'` after the join — for an `INNER JOIN`, both produce identical results, because there's no unmatched row for the extra condition to accidentally "protect." That distinction becomes important with outer joins, and is revisited in Topic 2.

## Internal Working (Preview)

Conceptually, an `INNER JOIN` between two tables works like this:

```
 orders (5 rows)              customers (4 rows)
      │                              │
      └──────────────┬───────────────┘
                      ▼
        For every combination of an orders row
        and a customers row, keep it only if
        the ON condition evaluates to true
                      │
                      ▼
              Matching pairs only
           (unmatched rows from either
             side are discarded)
```

Conceptually, this is "try every pairing, keep the ones that match" — but that is *not* what the database actually does mechanically, and this is a good first place to preview why the declarative/imperative distinction from Module 1 matters. Literally trying every pairing (a nested loop over both tables) would be correct but extremely slow for large tables — for two tables of a million rows each, that's a trillion comparisons. Instead, PostgreSQL's query planner chooses from several join algorithms (nested loop, hash join, merge join) based on table sizes and available indexes, and picks whichever produces the identical correct result fastest. You describe *what* rows should match (the `ON` condition); the planner decides *how* to find them efficiently. Module 13 (Indexes) and Module 20 (Performance Tuning) cover these algorithms and how to read `EXPLAIN` output to see which one PostgreSQL actually chose for a given query.

## Real-World Analogy

Think of `INNER JOIN` like matching event tickets to a guest list at a door. You have a stack of tickets (one table) and a printed guest list (the other table), and a bouncer's job is to only let through people whose ticket number appears on the guest list *and* whose name on the guest list has a corresponding ticket. A ticket with no matching name on the list gets turned away; a name on the list with no ticket in hand also gets turned away. Only the pairs that match on both sides get combined into "this person, holding this ticket, is let in." That's precisely `INNER JOIN`'s "keep only matched pairs, discard everything else" behavior.

## Why INNER JOIN Was Designed This Way

The relational model (Module 1, Module 2) deliberately keeps each table representing one *kind* of thing, related to others only through key values, not physical nesting or duplication. `INNER JOIN` is the operation that restores the "natural" combined view of related data — but it must have a precise, unambiguous rule for what to do when a relationship is incomplete (a `NULL` foreign key, an orphaned reference before it existed, or simply a row that hasn't been related to anything yet). Choosing "only produce a combined row when both sides genuinely have something to combine" is the mathematically simplest, most predictable rule available, and it maps directly onto formal relational algebra's *natural join* operation. Every other join type in this module (`LEFT`, `RIGHT`, `FULL OUTER`) is explicitly defined as "start from `INNER JOIN`'s result, then also keep some additional unmatched rows" — `INNER JOIN` is the conceptual baseline every other join variation is built on top of.

## Advantages

- **Predictable, minimal result set** — you only ever see rows that have genuine data on both sides, so you never have to immediately filter out or special-case incomplete rows.
- **Matches the most common real-world question** — "show me X together with its related Y" almost always implicitly means "...for the X that actually has a Y," which is exactly `INNER JOIN`'s behavior.
- **The query planner has the most freedom to optimize** — because `INNER JOIN` has no requirement to "preserve" any particular side's rows, PostgreSQL's optimizer can reorder and choose algorithms for multi-table inner joins more freely than for outer joins (Topic 5 revisits this).

## Disadvantages / Limitations

- **Silently drops data you might not expect to lose** — a customer with no orders yet, or a data-entry gap, disappears from the result with no warning; if you actually needed to see "every customer, whether or not they've ordered," `INNER JOIN` is the wrong tool (Topic 2).
- **A missing or wrong `ON` condition is a serious, easy-to-miss bug** — get the join condition wrong (e.g., join on the wrong column, or omit a condition needed to disambiguate a multi-column key) and you can silently produce duplicated or nonsensical rows rather than an obvious error.

## Best Practices

- Always use explicit `JOIN...ON` syntax, never the old comma/`WHERE` style, in any new query you write.
- Always alias your tables (`orders o`, `customers c`) once a query involves more than one table, and use those aliases consistently on every column reference — it disambiguates columns that exist in both tables and makes the query dramatically easier to read (Topic 5 covers this in more depth for 3+ table joins).
- Base your `ON` condition on the actual foreign key relationship declared in your schema (Module 5) rather than inventing a new column-matching rule — if you find yourself joining on a condition that *isn't* an existing foreign key, pause and double check it's actually correct, since it may indicate a missing constraint in your schema design.
- Qualify every column name with its table alias (`o.order_id`, not just `order_id`) once you're joining tables, even for columns that only exist in one of them — it's a small habit that prevents future breakage if a column of the same name is ever added to the other table.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Forgetting the `ON` clause entirely (`FROM orders o INNER JOIN customers c;` with no `ON`) | PostgreSQL will raise a syntax error for `INNER JOIN` specifically (unlike the comma-style, which silently produces a cross join) — but the underlying misunderstanding, forgetting to specify how tables relate, is the same dangerous mistake as Topic 3's accidental cross join. |
| Joining on a column that looks similar but isn't the actual foreign key (e.g., matching on `name` instead of `customer_id`) | Names and other non-key text can collide or vary in casing/spelling across real data; a foreign key column exists specifically because it's guaranteed unique and stable — always prefer it as the join condition. |
| Expecting `INNER JOIN` to show unmatched rows "just this once" for a report | `INNER JOIN` unconditionally excludes unmatched rows — there's no flag or option to make it behave otherwise; if you need unmatched rows preserved, you need `LEFT`, `RIGHT`, or `FULL OUTER JOIN` (Topics 2 and 3), not `INNER JOIN`. |

## Interview Questions

1. **Q: What does `INNER JOIN` do, in one sentence, and what happens to rows with no match?**
   A: `INNER JOIN` combines rows from two tables based on a matching condition in the `ON` clause, and it includes a row in the output only if it has at least one matching row on the other side — rows without a match on either side are excluded entirely from the result.

2. **Q: Why is explicit `JOIN...ON` syntax preferred over the old comma-style join in `FROM`/`WHERE`?**
   A: `JOIN...ON` visually separates the join condition (how tables relate) from filtering conditions (which rows you want), makes it much harder to accidentally omit a join condition and produce an unintended cross join, and is the only syntax that cleanly supports outer joins — the comma style has no clean equivalent for `LEFT`/`RIGHT`/`FULL OUTER JOIN`.

3. **Q: If table A has 10 rows and table B has 10 rows, and you `INNER JOIN` them on a foreign key, how many rows can the result have?**
   A: Anywhere from 0 up to 10 (assuming a typical one-to-many or one-to-one style key relationship) — it depends entirely on how many actual matches exist. It is not necessarily 10, 20, or 100; the row count is determined purely by how many pairs satisfy the `ON` condition, not by the sizes of the input tables.

4. **Q: Why does a foreign key make a natural, trustworthy join condition?**
   A: A foreign key constraint (Module 5) guarantees that every non-`NULL` value in the referencing column corresponds to an existing value in the referenced table's key column. That guarantee is exactly what makes it a reliable basis for a join — you know the relationship the join condition expresses is enforced by the database itself, not just assumed to be true by convention.

## Summary

- A **join** combines columns from two (or more) tables into one result, row by row, based on a matching condition — necessary because well-designed schemas deliberately split related data across multiple tables (Module 4, Module 5).
- `INNER JOIN ... ON <condition>` is the standard, explicit syntax: it keeps only the rows that have a genuine match on both sides, and silently discards everything else.
- The old comma-style join (`FROM a, b WHERE ...`) produces identical results for inner joins but should be avoided in new code — it conflates join conditions with filters and cannot express outer joins.
- A foreign key relationship (Module 5) is, in the vast majority of real queries, exactly the join condition you should use — it's the very relationship the constraint was declared to guarantee.
- `INNER JOIN`'s "only matched rows" behavior is the conceptual baseline every other join type in this module modifies — Topic 2 introduces joins that preserve unmatched rows from one or both sides.
