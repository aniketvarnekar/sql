# UNION, INTERSECT, and EXCEPT

## Learning Objectives

By the end of this section you should be able to:
- Explain how set operations differ fundamentally from joins — combining result sets vertically instead of horizontally
- Write `UNION` and `UNION ALL`, and explain the deduplication cost that distinguishes them
- Write `INTERSECT` and `EXCEPT`, and explain what each returns
- State and apply the column-count and type-compatibility rule required for all of these operations

## Prerequisites

- **[Joining Multiple Tables](05-multiple-table-joins.md)** — this topic is conceptually distinct from every join covered so far, and understanding *why* it's different requires the join model as a point of contrast.
- **Module 7 — Querying Basics** (`SELECT`, `ORDER BY`) — set operations combine the results of ordinary `SELECT` statements; this topic assumes you can already write those independently.

## Motivation

Every join in this module combines tables **horizontally** — for each matching row, you get *more columns* in the output (a customer's columns plus their order's columns, side by side). But sometimes what you actually want is the opposite: the same *shape* of columns, but from two different sources, stacked **vertically** into one combined list of rows. "Give me every city where we have a customer, or a warehouse, or both" isn't a question about matching rows between two tables at all — it's a question about combining two separate lists into one. That's exactly what `UNION`, `INTERSECT`, and `EXCEPT` are for.

## Problem Statement

Suppose the business also operates a small number of warehouses, tracked in a separate table with its own `city` column, and you're asked: "list every city where we have either a customer or a warehouse." Neither table has a foreign key relationship to the other — a warehouse's city has nothing to do with matching individual customer rows to individual warehouse rows — so there is no meaningful `ON` condition to join them with. A join would try to *pair up* rows between the two tables, which isn't the question being asked at all; what's actually needed is to take the list of cities from one table and the list of cities from the other, and combine them into a single, deduplicated list.

## Concept

### Setting Up the Example

This topic reuses this module's `customers` table and introduces a small `warehouses` table for the same reason `shipping_methods` was introduced in Topic 3 — a small, independent table representing a genuinely unrelated dimension of the business, not something that could sensibly be joined to `customers` on a shared key:

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT NOT NULL,
    city        TEXT
);

INSERT INTO customers (name, email, city) VALUES
    ('Ava Patel',  'ava@example.com',  'Austin'),
    ('Ben Ortiz',  'ben@example.com',  'Denver'),
    ('Chen Wu',    'chen@example.com', 'Austin'),
    ('Dara Singh', 'dara@example.com', 'Seattle');

CREATE TABLE warehouses (
    warehouse_id SERIAL PRIMARY KEY,
    city         TEXT NOT NULL
);

INSERT INTO warehouses (city) VALUES
    ('Austin'),
    ('Chicago');
```

### UNION — Combining Rows Vertically, With Duplicates Removed

```sql
SELECT city FROM customers
UNION
SELECT city FROM warehouses
ORDER BY city;
```

```
  city
---------
 Austin
 Chicago
 Denver
 Seattle
(4 rows)
```

`customers` contributes the cities Austin (twice — Ava and Chen both live there), Denver, and Seattle; `warehouses` contributes Austin and Chicago. `UNION` stacks both `SELECT`s' results into one combined list **and removes duplicate rows**, which is why `'Austin'` appears only once in the final output even though it came from three separate source rows across both tables (two customers and one warehouse). This deduplication is `UNION`'s defining trait, distinguishing it from a join: there was no "matching" happening here at all — `UNION` doesn't care whether a city came from `customers` or `warehouses`, it simply combines both lists and removes exact repeats.

### UNION ALL — Combining Rows Vertically, Keeping Duplicates

```sql
SELECT city FROM customers
UNION ALL
SELECT city FROM warehouses
ORDER BY city;
```

```
  city
---------
 Austin
 Austin
 Austin
 Chicago
 Denver
 Seattle
(6 rows)
```

`UNION ALL` performs the identical vertical stacking but skips the deduplication step entirely — every row from both `SELECT`s is kept, including the two separate `'Austin'` rows from `customers` and the one from `warehouses`, for **6 rows** total instead of `UNION`'s 4.

### The Deduplication Cost: UNION vs. UNION ALL

The choice between `UNION` and `UNION ALL` is not purely stylistic — it has a real, measurable performance cost, because **removing duplicates requires comparing rows against each other**, which in practice means PostgreSQL must sort or hash the entire combined result set to identify and eliminate repeats before returning it to you. `UNION ALL` skips this step entirely: it simply concatenates both result sets and returns them immediately, with no comparison work at all.

| | `UNION` | `UNION ALL` |
|---|---|---|
| Duplicate rows | Removed | Kept |
| Extra work required | Yes — sorting/hashing the combined set to find and remove duplicates | No — rows are simply concatenated |
| Appropriate when | You know or suspect overlap exists, and only want distinct values | You know the two sources can't overlap, or duplicates are fine/expected/desired |

For small result sets like this topic's examples, the difference is invisible. For large ones — combining results from tables with millions of rows each — deduplication can be a genuinely significant cost. **The practical rule: use `UNION ALL` whenever you know duplicates either can't occur or don't matter for your purpose, and reserve plain `UNION` for when you specifically need a deduplicated combined list.** A very common real mistake is defaulting to `UNION` purely out of habit, paying the deduplication cost on every single run of a query, when `UNION ALL` would have produced an identical result for that specific data.

### INTERSECT — Only Rows Present in Both

```sql
SELECT city FROM customers
INTERSECT
SELECT city FROM warehouses
ORDER BY city;
```

```
  city
--------
 Austin
(1 row)
```

`INTERSECT` returns only the rows that appear in **both** `SELECT` results — here, `'Austin'` is the only city with both a customer and a warehouse. Denver, Seattle (customer-only) and Chicago (warehouse-only) are all excluded, since each appears in only one of the two lists. This directly answers a genuinely useful business question: "in which cities do we have both a customer base and warehouse infrastructure already?"

### EXCEPT — Rows in the First, Not in the Second

```sql
SELECT city FROM customers
EXCEPT
SELECT city FROM warehouses
ORDER BY city;
```

```
  city
---------
 Denver
 Seattle
(2 rows)
```

`EXCEPT` returns rows from the **first** `SELECT` that do **not** appear anywhere in the second `SELECT`'s results — here, cities with a customer but no warehouse presence, directly answering "which customer cities do we not yet have warehouse coverage in?" `EXCEPT` is order-sensitive: swapping the two `SELECT`s produces a different, not merely reordered, result:

```sql
SELECT city FROM warehouses
EXCEPT
SELECT city FROM customers
ORDER BY city;
```

```
  city
---------
 Chicago
(1 row)
```

This asks the mirror-image question instead: "which warehouse cities have no customers at all?" — a completely different (though related) business question, which is why getting the order of the two `SELECT`s right matters for `EXCEPT` in a way it never does for `UNION` or `INTERSECT` (both of which are the same result regardless of which `SELECT` is written first, since "combine" and "appears in both" don't depend on direction).

Note: some other database products use the keyword `MINUS` instead of `EXCEPT` for this same operation — PostgreSQL uses the standard SQL keyword, `EXCEPT`, throughout.

### The Column-Count and Type-Compatibility Rule

All three operations — `UNION`, `INTERSECT`, `EXCEPT` — impose the same structural requirement on the `SELECT` statements being combined:

- **Both `SELECT`s must return the same number of columns.** `SELECT city FROM customers` returns one column; combining it with `SELECT city, warehouse_id FROM warehouses` (two columns) is a straightforward error, since there's no sensible way to stack a one-column row on top of a two-column row.
- **Corresponding columns must have compatible data types.** The first column of the first `SELECT` is matched against the first column of the second `SELECT`, and so on — combining a `TEXT` column with a `NUMERIC` column in the same position, for instance, either errors outright or forces an implicit conversion, neither of which is usually what you want.
- **Column names in the final result come from the first `SELECT` only.** If the two `SELECT`s name a column differently (`city` vs. `warehouse_city`), the combined result uses whatever the *first* `SELECT` called it — the second `SELECT`'s column name is irrelevant to the output, only its position and type matter.

```sql
-- This raises an error — mismatched column counts
SELECT city FROM customers
UNION
SELECT city, warehouse_id FROM warehouses;
```

```
ERROR:  each UNION query must have the same number of columns
```

This rule exists precisely because set operations are combining rows **positionally**, not by column name — unlike a join, which explicitly names the columns and the relationship connecting rows via `ON`, a set operation simply stacks whatever the first column of one `SELECT` produces on top of whatever the first column of the other produces, and so on. There's no matching logic at all here, which is exactly why the shapes must already line up before you even run the query.

## Internal Working (Preview)

For `UNION`, `INTERSECT`, and `EXCEPT`, PostgreSQL typically executes both `SELECT` statements independently first, then combines their results using either a sort-based or hash-based approach to identify duplicates (for `UNION`'s removal, or for determining membership in `INTERSECT`/`EXCEPT`):

```
   SELECT city FROM customers        SELECT city FROM warehouses
              │                                  │
              └────────────────┬─────────────────┘
                                ▼
              Combine positionally (stack columns
              1-to-1), then apply the operation's
              rule: UNION removes duplicate rows,
              UNION ALL keeps everything, INTERSECT
              keeps only rows common to both,
              EXCEPT keeps only rows unique to the
              first SELECT
                                │
                                ▼
                       Final combined result
```

`UNION ALL` is the cheapest of the four operations to execute, precisely because it skips the "identify duplicates" step (sorting or hashing the combined data) that `UNION`, `INTERSECT`, and `EXCEPT` all require in some form — this is the concrete mechanism behind the deduplication cost described above.

## Real-World Analogy

Think of `UNION` like merging two separate mailing lists (say, a newsletter signup list and a loyalty-program list) into one combined contact list, with duplicate email addresses removed so nobody gets mailed twice. `UNION ALL` would be the same merge but keeping every entry exactly as-is, duplicates included — useful if, say, you actually want to count total signups across both programs, duplicates and all. `INTERSECT` would be identifying only the people who appear on *both* lists (perhaps to send them a special "loyal on every front" offer), and `EXCEPT` would be identifying people on the newsletter list who are *not* also loyalty members (perhaps to specifically invite them to join).

## Why Set Operations Were Designed This Way

SQL's `SELECT` statement is built directly on the relational model's mathematical foundation (Module 1, Module 2), and relations correspond closely to mathematical sets of rows. `UNION`, `INTERSECT`, and `EXCEPT` are the direct SQL expression of the classic set-theory operations of the same names (union, intersection, and set difference) applied to two compatible relations. Requiring matching column counts and types isn't an arbitrary restriction — it's the necessary condition for two result sets to represent the *same kind* of thing in the first place, which is the prerequisite for any of these set operations to even be meaningful. `UNION ALL` exists as a deliberate, explicit escape hatch from strict set semantics — because true set union has no concept of "the same element twice," but SQL result sets very often legitimately do need to represent repeated values, and forcing deduplication in every case would both misrepresent the data and impose unnecessary cost.

## Advantages

- **Combines result sets from genuinely unrelated tables cleanly** — no join condition needs to exist (or to be invented) between the two sources, since nothing is being matched.
- **`UNION ALL` is very cheap** — when duplicates are known not to matter, it lets you combine large result sets with essentially no extra computational overhead beyond concatenation.
- **Maps directly onto familiar set-theory operations** — once you know what mathematical union, intersection, and difference mean, `UNION`, `INTERSECT`, and `EXCEPT` require no further conceptual leap.

## Disadvantages / Limitations

- **Rigid structural requirement** — both `SELECT`s must produce the same number of columns with compatible types, which sometimes requires deliberately padding one `SELECT` with a placeholder column (or `NULL`) just to make the shapes line up.
- **`UNION`, `INTERSECT`, and `EXCEPT` all carry a deduplication cost** that grows with the size of the combined result — for very large datasets, this can be a meaningful performance concern, and `UNION ALL`'s indiscriminate duplicate-keeping isn't always an acceptable substitute if duplicates genuinely need to be removed.
- **Column names in the output come only from the first `SELECT`**, which can be a minor but real readability trap if the two `SELECT`s use different column names for what is conceptually the same data.

## Best Practices

- Default to `UNION ALL` unless you specifically need duplicates removed — treat plain `UNION`'s deduplication as a deliberate choice you make for a reason, not a habitual default.
- Alias columns explicitly and identically across both `SELECT`s (e.g., `SELECT city AS location FROM customers UNION SELECT city AS location FROM warehouses`) so the combined result's column naming is unambiguous to future readers, even though only the first `SELECT`'s naming technically matters.
- Add a literal "source" column to each `SELECT` (e.g., `SELECT city, 'customer' AS source FROM customers UNION ALL SELECT city, 'warehouse' AS source FROM warehouses`) whenever it's useful to know, after combining, which original table a given row came from — set operations otherwise erase that information entirely.
- Reach for a join, not a set operation, whenever the actual question involves relating rows between two tables via a shared key (Topics 1–5) — set operations are for combining independent lists of the same shape, not for connecting related data.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `UNION` by default everywhere, including cases where duplicates are known to be impossible or irrelevant | Pays an unnecessary deduplication cost (sorting/hashing the combined result) for no benefit — `UNION ALL` produces an identical result in that case, faster. |
| Combining `SELECT`s with a mismatched number of columns, or incompatible types in the same position | Raises a database error (mismatched count) or produces a nonsensical/implicitly-converted result (mismatched types) — both `SELECT`s must already agree on shape before combining. |
| Using a join when the real goal is combining two independently-listed sets of the same shape | Forces an artificial, meaningless `ON` condition between two tables that have no real relationship, when `UNION`/`UNION ALL` (stacking, not matching) is the operation actually being asked for. |
| Reversing the operands of `EXCEPT` without realizing it changes the result | Unlike `UNION` and `INTERSECT`, `EXCEPT` is direction-sensitive — `A EXCEPT B` and `B EXCEPT A` ask two different, non-interchangeable questions. |

## Interview Questions

1. **Q: What is the fundamental difference between a join and a set operation like `UNION`?**
   A: A join combines two tables **horizontally** — for each matched pair of rows, the result gains more *columns* (data from both tables side by side), based on a matching condition. A set operation combines two result sets **vertically** — the result gains more *rows*, stacking the output of two independent `SELECT` statements of the same shape, with no matching condition involved at all.

2. **Q: What is the practical difference between `UNION` and `UNION ALL`, and when would you prefer each?**
   A: `UNION` combines two result sets and removes duplicate rows, which requires extra work (sorting or hashing the combined data) to detect those duplicates. `UNION ALL` combines them and keeps every row, including duplicates, with no such extra work. Prefer `UNION ALL` whenever duplicates are known to be impossible or don't matter for the purpose at hand, since it's cheaper; use plain `UNION` only when you specifically need a deduplicated result.

3. **Q: What must be true of two `SELECT` statements before they can be combined with `UNION`, `INTERSECT`, or `EXCEPT`?**
   A: They must return the same number of columns, and corresponding columns (matched by position, not name) must have compatible data types. The final result's column names come from the first `SELECT` only.

4. **Q: Does it matter which `SELECT` comes first in an `EXCEPT` query? What about `UNION` or `INTERSECT`?**
   A: Yes for `EXCEPT` — `A EXCEPT B` returns rows in A that are absent from B, while `B EXCEPT A` returns the reverse, and these are generally different result sets entirely. For `UNION` and `INTERSECT`, the order of the two `SELECT`s doesn't change which rows end up in the result (only their default display order, which `ORDER BY` controls regardless), since "combine everything" and "keep only what's common to both" don't depend on which side is listed first.

## Summary

- `UNION`, `INTERSECT`, and `EXCEPT` combine result sets **vertically** (more rows), in contrast to joins, which combine tables **horizontally** (more columns) based on a matching condition.
- `UNION` removes duplicate rows from the combined result, at the cost of extra sorting/hashing work; `UNION ALL` keeps every row, duplicates included, with no such cost.
- `INTERSECT` returns only rows present in both `SELECT` results; `EXCEPT` returns rows present in the first `SELECT` but absent from the second — and unlike `UNION`/`INTERSECT`, `EXCEPT` is sensitive to the order of its two operands.
- All three operations require both `SELECT`s to return the same number of columns, with compatible types in each corresponding position — they combine rows purely positionally, with no matching logic at all.
- This is the final topic-level concept in this module — Topic 7 (the Module Summary) consolidates every join type and set operation covered so far into one recap.
