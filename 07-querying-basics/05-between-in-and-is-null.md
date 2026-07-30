# BETWEEN, IN, and IS NULL

## Learning Objectives

By the end of this section you should be able to:
- Use `BETWEEN` to test an inclusive range, and recognize its most common pitfalls
- Use `IN` and `NOT IN` to test membership against a literal list, and know that a subquery can appear in the same position
- Use `IS NULL`/`IS NOT NULL` correctly, and explain why `= NULL` never works
- Explain why `NOT IN` can silently return zero rows when its list contains a `NULL`

## Prerequisites

- [Comparison and Logical Operators](03-comparison-and-logical-operators.md) — `BETWEEN` and `IN` are convenient shorthand for chains of the comparison/logical operators already covered.
- Module 03's [NULL and Three-Valued Logic](../03-data-types/05-null-and-three-valued-logic.md) — this topic's `IS NULL` and `NOT IN` discussions are direct, concrete applications of the three-valued logic established there.

## Motivation

Some filtering questions are naturally about *ranges* ("priced between $10 and $50"), *lists* ("in Kitchen, Furniture, or Office Supplies"), or *absence* ("has no recorded stock count"). All three could technically be written as chains of `AND`/`OR`/comparison from the previous two topics — but doing so is verbose, easy to get subtly wrong, and hides the actual intent of the query behind mechanical repetition. `BETWEEN`, `IN`, and `IS NULL` exist specifically to express these three extremely common patterns directly and unambiguously.

## Concept

### `BETWEEN` — Inclusive Range Checks

`BETWEEN low AND high` is shorthand for `column >= low AND column <= high` — **both endpoints are included**:

```sql
SELECT name, price
FROM products
WHERE price BETWEEN 10 AND 50;
```

```
        name         | price
----------------------+--------
 Wireless Mouse       |  24.99
 Desk Lamp            |  34.95
 Stapler              |  12.00
 Coffee Mug           |  11.25
 Electric Kettle      |  45.00
 100% Cotton Towel    |  15.00
(6 rows)
```

Electric Kettle, priced at exactly `45.00`, is included — and if a product were priced at exactly `10.00` or `50.00`, it would be included too. This inclusivity is the single most important fact to remember about `BETWEEN`: it is never a strict, exclusive range.

`NOT BETWEEN` inverts it, excluding both endpoints along with everything in between:

```sql
SELECT name, price FROM products WHERE price NOT BETWEEN 10 AND 50;
```

```
         name           | price
-------------------------+--------
 Mechanical Keyboard     |  89.99
 Standing Desk           | 349.00
 Office Chair            | 199.50
 Notebook Pack of 5      |   6.49
 Whiteboard Marker Set   |   8.75
 Blender_Pro 2000        |  79.99
 Uncategorized Widget    |   3.50
(7 rows)
```

**A common pitfall: reversed bounds.** `BETWEEN` expects its low bound first and high bound second — writing them backward doesn't raise an error, it silently matches nothing:

```sql
SELECT name FROM products WHERE price BETWEEN 50 AND 10;
```

```
(0 rows)
```

No row can simultaneously be `>= 50` and `<= 10` — the query is syntactically valid and runs without complaint, but is logically empty by construction. This is exactly the kind of mistake that's easy to introduce when the bounds come from variables computed elsewhere (a form's "min price" and "max price" fields swapped, for instance) rather than typed as literals.

**A second pitfall: date/time boundaries.** `BETWEEN`'s inclusive upper bound is especially easy to get wrong with timestamps rather than plain dates — `BETWEEN '2024-01-01' AND '2024-01-31'` against a `DATE` column includes all of January 31st, but the same range against a `TIMESTAMP` column would only include *exactly midnight* on January 31st, silently excluding the rest of that day. Module 8 (Functions & Expressions) covers date arithmetic in depth; for now, the takeaway is to double-check what "inclusive" actually means for the specific data type you're filtering.

### `IN` — Membership Against a List

`IN (v1, v2, v3, ...)` is shorthand for a chain of `OR`-ed equality checks — `column = v1 OR column = v2 OR column = v3`:

```sql
SELECT name, category
FROM products
WHERE category IN ('Kitchen', 'Office Supplies');
```

```
          name           |     category
--------------------------+------------------
 Notebook Pack of 5       | Office Supplies
 Stapler                  | Office Supplies
 Whiteboard Marker Set    | Office Supplies
 Coffee Mug               | Kitchen
 Electric Kettle          | Kitchen
 Blender_Pro 2000         | Kitchen
(6 rows)
```

Far more readable than writing `category = 'Kitchen' OR category = 'Office Supplies'` — and the readability advantage only grows as the list gets longer. `NOT IN` inverts it, keeping only rows that match *none* of the listed values:

```sql
SELECT name, category
FROM products
WHERE category NOT IN ('Kitchen', 'Office Supplies');
```

```
         name         |   category
-----------------------+--------------
 Wireless Mouse        | Electronics
 USB-C Cable           | Electronics
 Mechanical Keyboard   | Electronics
 Standing Desk         | Furniture
 Office Chair          | Furniture
 Desk Lamp             | Furniture
 100% Cotton Towel     | Home Goods
(7 rows)
```

Notice `Uncategorized Widget` (`category` is `NULL`) doesn't appear in either result — not in the `IN` list (its category isn't literally `'Kitchen'` or `'Office Supplies'`) and, perhaps more surprisingly, not in the `NOT IN` result either, for reasons explained below.

### `IN` With a Subquery — A Preview

The list inside `IN` doesn't have to be written out literally — it can be produced by a subquery instead, as long as that subquery returns a single column:

```sql
SELECT name
FROM products
WHERE category IN (
    SELECT category FROM products WHERE price > 100
);
```

```
        name
---------------------
 Standing Desk
 Office Chair
```

This finds every product sharing a category with something priced over $100 (here, Furniture — because Standing Desk is $349.00). This is only a preview: subqueries — queries nested inside other queries — are a large enough topic to earn their own full treatment in Module 11 (Subqueries), including correlated subqueries and `EXISTS`. For now, the important thing to recognize is that `IN`'s right-hand side can be either a literal list or a query that produces one.

### `IS NULL` / `IS NOT NULL` — Testing for Missing Values

Module 3's [NULL and Three-Valued Logic](../03-data-types/05-null-and-three-valued-logic.md) already established the core rule this topic leans on directly: `NULL` means "unknown," so `= NULL` and `<> NULL` both always evaluate to `UNKNOWN`, never a usable `TRUE`/`FALSE` — they can never match any row, no matter what the data actually contains. `IS NULL` and `IS NOT NULL` are dedicated predicates, not comparisons, built specifically to test for `NULL` and always return a definite answer:

```sql
SELECT name, category FROM products WHERE category IS NULL;
```

```
         name           | category
-------------------------+-----------
 Uncategorized Widget    |
(1 row)
```

```sql
SELECT name, discontinued_on FROM products WHERE discontinued_on IS NOT NULL;
```

```
       name        | discontinued_on
--------------------+-------------------
 Desk Lamp          | 2024-11-01
 Electric Kettle    | 2023-06-15
(2 rows)
```

```sql
SELECT name FROM products WHERE discontinued_on = NULL;
```

```
(0 rows)
```

The last query matches zero rows, *always*, regardless of what `discontinued_on` actually contains anywhere in the table — `= NULL` is not "equal to nothing," it's a comparison that can never resolve to `TRUE`, which is precisely the point already established in Module 3 and worth restating here as a direct warning: never write `= NULL` or `<> NULL` in a `WHERE` clause; always use `IS NULL`/`IS NOT NULL`.

### The `NOT IN` / `NULL` Trap, Applied to This Table

This is one of the most consequential `NULL` pitfalls in real SQL, and it applies directly to `IN`/`NOT IN`. Suppose someone runs:

```sql
SELECT name FROM products
WHERE category NOT IN ('Kitchen', 'Furniture', NULL);
```

```
(0 rows)
```

Zero rows — even though plenty of products (Electronics, Office Supplies) are obviously neither Kitchen nor Furniture. `category NOT IN ('Kitchen', 'Furniture', NULL)` expands to `category <> 'Kitchen' AND category <> 'Furniture' AND category <> NULL`. That last term, `category <> NULL`, evaluates to `UNKNOWN` for *every single row*, no matter what `category` actually is — and an `AND` chain containing even one `UNKNOWN` alongside true conditions can never resolve to a definite `TRUE` (Module 3's truth tables: `TRUE AND UNKNOWN` is `UNKNOWN`, not `TRUE`). So every row's overall condition becomes `UNKNOWN`, and `WHERE` excludes all of them, silently, with no error. This exact failure mode is why Module 3 recommends preferring `NOT EXISTS` (Module 11) over `NOT IN` whenever the list — especially a subquery-produced one — might contain a `NULL`.

## Internal Working (Preview)

Conceptually, the parser rewrites `BETWEEN` and `IN` into their equivalent longhand comparison chains before evaluation — the two forms are truly interchangeable in meaning:

```
price BETWEEN 10 AND 50
        │  rewritten as
        ▼
price >= 10 AND price <= 50

category IN ('Kitchen', 'Office Supplies')
        │  rewritten as
        ▼
category = 'Kitchen' OR category = 'Office Supplies'
```

For a large literal list, the query planner is free to implement the equivalent check more efficiently than a literal chain of `OR`s (for example, using a sorted lookup or hash check instead of testing each value one at a time) — but the *result* is guaranteed identical to the longhand `OR` chain, which is exactly why the `NOT IN`/`NULL` trap above behaves the way it does: it's not a special case, it's the ordinary three-valued `AND`/`OR` behavior from Topic 3, just reached through `IN`'s shorthand instead of a hand-written chain.

## Real-World Analogy

`BETWEEN` is like a venue's age policy printed as "ages 13 to 19 admitted" — both 13 and 19 are let in, not just the ages strictly in between. `IN` is like checking a name against a guest list: one lookup against the whole list, rather than asking "are you Alex? Are you Sam? Are you Jordan?" one exhausting name at a time. `IS NULL` is like a form field that was physically left blank — checking "was anything at all written here?" is a different, more basic question than checking "does what's written here match some specific value," which is exactly why it needs its own dedicated check rather than an ordinary comparison.

## Why These Operators Were Designed This Way

`BETWEEN` and `IN` are syntactic sugar — they don't add any new expressive power beyond what `AND`/`OR`/comparison operators (Topic 3) already provide, but they make extremely common patterns (a range, a membership list) dramatically more readable and less error-prone to write by hand. This is consistent with SQL's broader declarative design philosophy: say what you mean ("in this range," "one of these values") directly, rather than mechanically spelling out the equivalent boolean chain every time. `IS NULL` exists as a dedicated predicate, rather than an overload of `=`, precisely because Codd's Rule 3 (systematic `NULL` handling, covered in [The Relational Model and Codd's Rules](../02-relational-model/03-the-relational-model-and-codds-rules.md)) requires a clean, consistent way to test for "unknown" that isn't tangled up with three-valued comparison semantics — a job `=` fundamentally cannot do, since `= NULL` always evaluates to `UNKNOWN` by the very definition of what `NULL` means.

## Advantages

- **Far more readable than the equivalent long-hand chains** — `price BETWEEN 10 AND 50` and `category IN (...)` communicate intent immediately, where a long `AND`/`OR` chain requires a reader to reconstruct that intent themselves.
- **`IN` scales gracefully to long lists** — adding a tenth value to an `IN` list is one more item; adding it to a manually written `OR` chain is one more repetitive clause.
- **`IS NULL`/`IS NOT NULL` give a reliable, always-correct way to test for missing data**, something no comparison operator can honestly provide.

## Disadvantages / Limitations

- **`BETWEEN`'s inclusive-both-ends semantics can silently misalign with intent**, especially with date/time boundaries, as shown above.
- **Reversed `BETWEEN` bounds produce no error, just an always-empty result** — a subtle bug that's easy to miss in code review.
- **`NOT IN` with a `NULL`-containing list (literal or subquery) silently returns zero rows for every outer row** — a well-known, dangerous correctness trap with no warning at query time.
- **Very large `IN` lists can hurt readability**, and in extreme cases (many thousands of literal values), planning time.

## Best Practices

- Double-check the direction of `BETWEEN`'s bounds, especially when they come from variables rather than literals typed directly into the query.
- Be explicit about `BETWEEN`'s inclusivity when working with dates/timestamps — verify what "the end date" is actually supposed to include.
- Prefer `NOT EXISTS` (Module 11) over `NOT IN` whenever the compared list is produced by a subquery that could plausibly contain a `NULL` — or filter `NULL`s out of the list explicitly first (`WHERE category NOT IN (SELECT category FROM ... WHERE category IS NOT NULL)`).
- Always use `IS NULL`/`IS NOT NULL` for `NULL` checks — never `=`/`<>`.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing `WHERE price = NULL` or `WHERE price <> NULL` | Both always evaluate to `UNKNOWN` and can never match a row — use `IS NULL`/`IS NOT NULL` instead. |
| Assuming `NOT IN` with a `NULL` in the list simply skips that one value | It doesn't skip it — a `NULL` anywhere in the list poisons the entire `NOT IN` comparison to `UNKNOWN` for every row, silently returning zero rows overall. |
| Writing `BETWEEN` bounds in the wrong order (`BETWEEN high AND low`) | No error is raised; the condition becomes impossible to satisfy and the query silently returns nothing. |
| Assuming `BETWEEN` on a date range excludes the end date | `BETWEEN` is always inclusive of both bounds — it never excludes either endpoint. |

## Interview Questions

1. **Q: Is `BETWEEN` inclusive or exclusive of its bounds?**
   A: Inclusive of both — `BETWEEN 10 AND 50` matches values equal to 10 and 50 as well as everything strictly in between; it's equivalent to `>= 10 AND <= 50`.

2. **Q: Why does `= NULL` never return `TRUE`, and what should you use instead?**
   A: `NULL` represents an unknown value, and comparing anything to an unknown value can't be truthfully resolved to `TRUE` or `FALSE` — it evaluates to `UNKNOWN`, which `WHERE` treats the same as `FALSE` (excluded). `IS NULL`/`IS NOT NULL` are dedicated predicates that always return a definite `TRUE`/`FALSE`.

3. **Q: Why can `NOT IN` silently return zero rows when its list contains a `NULL`, and how would you avoid it?**
   A: `NOT IN (list)` expands to an `AND` chain of `<>` comparisons against every value in the list. A `NULL` in that list makes one of those comparisons evaluate to `UNKNOWN` for every row, and since an `AND` chain with any `UNKNOWN` term can never resolve to a definite `TRUE`, every row is silently excluded. Prefer `NOT EXISTS`, or filter `NULL`s out of the list explicitly, to avoid this.

4. **Q: What is the equivalent longhand for `col IN (1, 2, 3)` using `OR`?**
   A: `col = 1 OR col = 2 OR col = 3` — `IN` is shorthand for exactly this chain of equality checks.

## Summary

- `BETWEEN low AND high` is inclusive of both bounds; reversed bounds silently produce zero rows rather than an error.
- `IN (...)` is shorthand for a chain of `OR`-ed equality checks, and its list can be a literal set of values or the result of a subquery (Module 11 covers subqueries in depth).
- `IS NULL`/`IS NOT NULL` are the only correct way to test for `NULL` — `= NULL` and `<> NULL` always evaluate to `UNKNOWN` and never match any row.
- `NOT IN` against a list containing even one `NULL` silently returns zero rows for every outer row, because the `NULL` poisons the underlying `AND` chain to `UNKNOWN` — prefer `NOT EXISTS` when this risk exists.
- The next topic, [Sorting with ORDER BY](06-sorting-with-order-by.md), moves from *which* rows qualify to *what order* they're presented in.
