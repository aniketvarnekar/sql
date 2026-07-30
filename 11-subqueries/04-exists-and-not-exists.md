# EXISTS and NOT EXISTS

## Learning Objectives

By the end of this section you should be able to:
- Use `EXISTS` and `NOT EXISTS` to test for the presence or absence of matching rows, without caring about their actual values
- Explain why `EXISTS` typically outperforms `IN` with a subquery, in terms of short-circuiting
- Explain, mechanically, why `NOT IN` combined with a subquery result containing a `NULL` silently returns zero rows
- Rewrite a dangerous `NOT IN` query as a safe, equivalent `NOT EXISTS` query

## Prerequisites

- [Subqueries in WHERE and FROM](02-subqueries-in-where-and-from.md) — this topic directly contrasts `EXISTS` with the `IN`-based filter list introduced there.
- [Correlated Subqueries](03-correlated-subqueries.md) — an `EXISTS` subquery is almost always correlated (it needs to ask "does a row related to *this* outer row exist"), so you need the correlated mental model from Topic 3 to follow it.

## Motivation

Sometimes you genuinely don't care what a related row's values are — you only want to know whether at least one exists at all. "Which customers have placed an order?" doesn't need any order's amount or date; it just needs a yes-or-no per customer. `EXISTS` and `NOT EXISTS` are built specifically for this shape of question, and — unlike almost everything else in this module — getting the "not" version wrong here isn't just inefficient, it can be silently, dangerously incorrect.

## Problem Statement

Recall from Module 10 and this module's Topics 1–3 that Dara Singh (`customer_id = 4`) has never placed a single order. A completely reasonable business question is "which customers have never ordered, so we can send them a welcome discount?" The obvious-looking query is:

```sql
SELECT name FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);
```

Run this against the actual data, and it returns **zero rows** — not even Dara Singh, who you can see with your own eyes has never ordered. This isn't a bug in PostgreSQL. It's a well-known, dangerous trap: `orders` contains order 105, whose `customer_id` is `NULL` (a guest checkout), and that single `NULL` silently poisons the entire `NOT IN` comparison for every customer, every time. Understanding exactly why is the core of this topic.

## Concept

### `EXISTS`: A Correlated Presence Test

```sql
SELECT name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
)
ORDER BY name;
```

```
    name
------------
 Ava Patel
 Ben Ortiz
 Chen Wu
 Elin Kask
(4 rows)
```

`EXISTS` takes a subquery and returns `TRUE` if that subquery produces **at least one row**, and `FALSE` if it produces none — it does not care how many rows, or what values are in them. That's why the inner query is conventionally written as `SELECT 1` (a throwaway constant) rather than `SELECT *` or a real column: **the `SELECT` list inside an `EXISTS` subquery is never actually used for anything** — PostgreSQL only checks whether any row exists. `SELECT 1`, `SELECT *`, and `SELECT o.order_id` are all functionally identical inside `EXISTS`; `SELECT 1` is simply the conventional way to communicate to a human reader "we only care about existence here."

Like the correlated subqueries in Topic 3, this `EXISTS` subquery references the outer row (`c.customer_id`), so it's conceptually re-checked once per customer: "does at least one order exist where the order's `customer_id` matches *this* customer's ID?"

### `NOT EXISTS`: An Absence Test

```sql
SELECT name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

```
   name
------------
 Dara Singh
(1 row)
```

`NOT EXISTS` simply negates the check: keep the outer row only if the inner subquery finds **zero** matching rows. This correctly identifies Dara Singh as the one customer with no orders at all — the exact answer the `NOT IN` version at the top of this topic failed to produce.

### Why `EXISTS` Is Often More Efficient Than `IN`

Conceptually, `EXISTS` can **short-circuit**: for a given outer row, the database can stop looking the moment it finds a single matching inner row — it never needs to find every matching row, just confirm that one exists. An `IN`-based query, in the most naive execution strategy, first computes the *entire* subquery result (every `customer_id` that has ever placed an order) before checking membership. For a large inner table, `EXISTS` in principle only needs to do enough work to find one match per outer row, whereas `IN` in principle needs to materialize the whole inner result set up front.

In practice, PostgreSQL's planner frequently rewrites a straightforward `IN` subquery into a plan that behaves very similarly to `EXISTS` (a "semi-join"), so the two aren't always as different in measured performance as the naive picture above suggests — as with the correlated-subquery-vs-join comparison in Topic 3, the only way to know for certain on your actual data is to check with `EXPLAIN` (Module 20). What's true unconditionally, though, and what matters more in practice, is the `NULL`-safety difference covered next — and that difference has nothing to do with performance at all.

### The `NOT IN` + `NULL` Trap

Return to the failing query from the Problem Statement:

```sql
SELECT name FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);
```

The inner subquery `SELECT customer_id FROM orders` returns the list `1, 1, 2, 3, NULL, 2, 3, 5, 1, 5` — ten values, one per order, including the `NULL` from order 105's guest checkout. `NOT IN` conceptually expands to a chain of `AND`-ed inequality checks:

```
customer_id <> 1 AND customer_id <> 1 AND customer_id <> 2 AND customer_id <> 3
AND customer_id <> NULL AND customer_id <> 2 AND customer_id <> 3 AND customer_id <> 5
AND customer_id <> 1 AND customer_id <> 5
```

Every single comparison against `NULL` (`customer_id <> NULL`) evaluates to `UNKNOWN` — not `TRUE`, not `FALSE` — because SQL's three-valued logic says any direct comparison to `NULL` is always `UNKNOWN`, no matter what `customer_id` actually is. And `TRUE AND UNKNOWN` is `UNKNOWN`, for every single customer, no matter how many of the other comparisons were `TRUE`. A `WHERE` clause only keeps rows where the overall condition evaluates to `TRUE` — `UNKNOWN` is treated the same as `FALSE` for that purpose (the row is excluded), which is why even Dara Singh, none of whose comparisons involve her own ID, still gets silently excluded: the single `NULL` in the list makes the *entire* `NOT IN` expression `UNKNOWN` for absolutely every row, not just for rows related to that `NULL`.

### `NOT EXISTS`: The Safe Alternative

`NOT EXISTS` never runs into this problem, because it never compares the outer value against the inner subquery's individual values at all — it only asks whether any matching row was found:

```sql
SELECT name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

```
   name
------------
 Dara Singh
(1 row)
```

Order 105's `NULL` `customer_id` is completely irrelevant here: it simply never matches any real customer's `c.customer_id` in the correlated condition (`NULL = anything` is never `TRUE`), so it neither helps nor poisons the check for any customer. This is the single most important reason this course recommends `NOT EXISTS` over `NOT IN` by default.

### If You Must Use `NOT IN`: Filter Out `NULL`s Defensively

If a `NOT IN` query is unavoidable (for style or legacy-code reasons), the trap can be defused by explicitly excluding `NULL` from the subquery's result:

```sql
SELECT name FROM customers c
WHERE c.customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);
```

```
   name
------------
 Dara Singh
(1 row)
```

Now the subquery's list is `1, 1, 2, 3, 2, 3, 5, 1, 5` — no `NULL` — so every comparison resolves cleanly to `TRUE` or `FALSE`, and the query produces the correct result. This fix works, but it depends on the query author remembering to add it, every single time, on every column that could ever contain a `NULL` — which is precisely why `NOT EXISTS` is the safer default rather than a defensive habit you must remember to apply correctly each time.

## Internal Working (Preview or Deep Dive)

`EXISTS`/`NOT EXISTS` are typically executed as a **semi-join** (`EXISTS`) or **anti-join** (`NOT EXISTS`) — plan shapes purpose-built for existence checks, distinct from a regular join, because they never need to produce more than one matched row per outer row even if several inner rows match:

```
 EXISTS (semi-join):
   for each outer row:
       scan inner rows looking for ANY match
       found one?  →  keep outer row, STOP scanning inner rows for this outer row
       found none? →  discard outer row

 NOT EXISTS (anti-join):
   for each outer row:
       scan inner rows looking for ANY match
       found one?  →  discard outer row
       found none? →  keep outer row
```

The "STOP scanning" step is the short-circuit behavior described above — the moment one match is found, there's no need to look for a second one, since the answer ("at least one exists") is already settled. Contrast this with the correlated subquery in Topic 3, which needed to look at *every* matching row to compute an `AVG` — that kind of subquery cannot short-circuit, because the aggregate genuinely depends on every matching value, not just the first one found.

## Real-World Analogy

`EXISTS` is like a bouncer checking a guest list for "is there anyone here named Priya?" — the moment the bouncer spots one Priya on the list, they can stop reading immediately; it doesn't matter if there are two, five, or fifty Priyas, or what else is written next to her name. `NOT EXISTS` is the same bouncer confirming "nobody on this list is named Priya" by reading the whole list and finding no match. Now imagine the guest list has one entry that's illegible — smudged ink, impossible to read. A careless bouncer using a `NOT IN`-style process ("check this name against every name on the list, one by one, and only admit them if every single comparison says 'different'") gets stuck the moment they reach the smudged entry: they cannot honestly say "different" about an entry they can't read, so they can't complete the check for anyone — and refuse to certify *any* name as absent, even names that obviously aren't Priya. The `EXISTS`-based bouncer never has this problem, because they're only ever looking for a positive match, and an illegible entry simply can never be that match.

## Why EXISTS and NOT EXISTS Were Designed This Way

`EXISTS` directly implements the existential quantifier (∃, "there exists") from formal logic — "does there exist a row such that..." — as a first-class SQL construct, rather than forcing every existence question to be awkwardly rephrased as a value-comparison (`IN`) that happens to work correctly only when no `NULL`s are involved. SQL's `NULL` represents "unknown," and SQL deliberately uses three-valued logic (`TRUE`/`FALSE`/`UNKNOWN`) so that comparisons against unknown values don't silently masquerade as `TRUE` or `FALSE` — the trade-off is that value-comparison operators like `<>` inherit this caution (an unknown value can't be proven unequal to something), while a presence check like `EXISTS` sidesteps the question of comparing values altogether, asking only "did the search find a row," which is a question `NULL` cannot make ambiguous.

## Advantages

- **`NULL`-safe by construction.** `EXISTS`/`NOT EXISTS` never individually compare the outer value against the subquery's values, so a stray `NULL` in the subquery's result can never silently corrupt the entire check, unlike `NOT IN`.
- **Reads naturally as "a matching row exists."** For questions that are genuinely about presence/absence rather than a specific value, `EXISTS` states the intent directly.
- **Can short-circuit.** Conceptually (and often literally, depending on the plan chosen), the check stops at the first match rather than needing to examine every related row.

## Disadvantages / Limitations

- **Slightly less intuitive to a first-time reader than `IN`.** `WHERE customer_id IN (subquery)` reads a little more directly as English than the correlated `EXISTS` form, for someone encountering both for the first time.
- **Still requires a reasonable execution plan to be fast.** If the correlated condition inside `EXISTS` has no supporting index (Module 13) on a very large inner table, the "short-circuit" savings can still mean scanning a meaningful number of rows per outer row before finding (or ruling out) a match.
- **`NOT EXISTS` doesn't eliminate the need to think about `NULL`s everywhere** — it solves this *specific* trap cleanly, but `NULL` comparison pitfalls can still appear elsewhere in a query (for example, directly in a `WHERE` clause's own conditions).

## Best Practices

- Default to `EXISTS`/`NOT EXISTS` over `IN`/`NOT IN` whenever the subquery's column could plausibly contain a `NULL` — in practice, this means defaulting to it almost always, since it's rarely obvious from a query alone that a column is guaranteed `NOT NULL` at every call site.
- Reserve `NOT IN` for subqueries you can prove return a column that is genuinely `NOT NULL` (for example, a primary key column, which can never be `NULL` by definition, Module 5) — and even then, `NOT EXISTS` costs nothing to use instead.
- Write `SELECT 1` (not `SELECT *` or a specific column) inside an `EXISTS`/`NOT EXISTS` subquery, by convention, to signal clearly to readers that the selected values are irrelevant.
- If you inherit a `NOT IN` query you can't immediately rewrite, at minimum add a defensive `... IS NOT NULL` filter to the subquery's column, as shown above — but treat this as a stopgap, not a substitute for switching to `NOT EXISTS`.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `NOT IN` with a subquery on a column that can contain `NULL` | If even one row in the subquery's result has a `NULL` in that column, the entire `NOT IN` condition evaluates to `UNKNOWN` (excluded) for every outer row — silently returning far fewer rows than expected, often zero, with no error raised. |
| Assuming `IN` has the same `NULL` risk as `NOT IN` | Plain `IN` degrades gracefully with a `NULL` in the list — it can still correctly return `TRUE` for a genuine match; only the *negated* form (`NOT IN`) is catastrophically broken by a `NULL`, because proving "not equal to everything, including something unknown" is impossible. |
| Forgetting to correlate an `EXISTS` subquery to the outer row | An `EXISTS` subquery with no reference to the outer query (e.g., `EXISTS (SELECT 1 FROM orders)`) just checks "does `orders` have any rows at all" once, for every outer row identically — almost never the intended question. |
| Believing `SELECT *` inside `EXISTS` is slower than `SELECT 1` | In PostgreSQL, the `SELECT` list inside an `EXISTS` subquery is not evaluated for its column values at all — `SELECT 1`, `SELECT *`, and `SELECT some_column` perform identically; the convention of `SELECT 1` is purely a readability signal, not a performance technique. |

## Interview Questions

1. **Q: Why does `NOT IN (SELECT ... )` sometimes return zero rows when you clearly expect some?**
   A: If the subquery's result contains even one `NULL`, every comparison in the implicit chain of `<>` checks against that `NULL` evaluates to `UNKNOWN` rather than `TRUE` or `FALSE`. Because `UNKNOWN` combined with `AND` can never become `TRUE`, the entire `NOT IN` condition becomes `UNKNOWN` — treated as excluded — for every single outer row, regardless of whether that row's own value has anything to do with the `NULL` entry.

2. **Q: Why is `NOT EXISTS` considered the safe replacement for `NOT IN`?**
   A: `NOT EXISTS` never performs a value-by-value comparison against the subquery's individual results — it only checks whether a correlated match was found. A `NULL` in the related table can never itself "match" an outer row's non-`NULL` value, so it cannot poison the check for any other row the way it does with `NOT IN`.

3. **Q: Why is `EXISTS` often more efficient than `IN` with a subquery?**
   A: Conceptually, `EXISTS` can short-circuit — stop searching as soon as one matching row is found — while a naive `IN` execution needs to materialize the subquery's entire result set before checking membership. In practice, PostgreSQL's planner often rewrites simple `IN` queries into an equivalent semi-join plan, so the real-world performance gap depends on the specific query and should be verified with `EXPLAIN` (Module 20) rather than assumed.

4. **Q: What does `SELECT 1` mean inside an `EXISTS` subquery, and does it matter what you select there?**
   A: `EXISTS` only checks whether the subquery produces at least one row — it never uses the selected columns' actual values. `SELECT 1` is a conventional placeholder signaling "we only care about row existence here"; `SELECT *` or any real column would behave identically.

## Summary

- `EXISTS` and `NOT EXISTS` test purely for the presence or absence of at least one matching row, ignoring the matched row's actual values.
- `EXISTS` can short-circuit — stopping at the first match — which is why it's conceptually more efficient than `IN`, though real-world query planners can narrow this gap; verify with `EXPLAIN` (Module 20) rather than assuming.
- `NOT IN` combined with a subquery whose result contains even one `NULL` silently evaluates to `UNKNOWN` (excluded) for every single outer row — a classic, dangerous trap that returns too few rows (often zero) with no error.
- `NOT EXISTS` avoids this trap entirely, because it never performs a value comparison against the subquery's rows — only a presence check — making it the safe default over `NOT IN`.
- If `NOT IN` must be used, defensively filter `NULL` out of the subquery's column (`WHERE column IS NOT NULL`) — but treat this as a stopgap, not a substitute for `NOT EXISTS`.
