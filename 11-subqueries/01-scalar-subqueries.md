# Scalar Subqueries

## Learning Objectives

By the end of this section you should be able to:
- Define precisely what makes a subquery "scalar" (exactly one row, exactly one column)
- Use a scalar subquery inside a `SELECT` list and inside a `WHERE` clause
- Predict and explain the runtime error PostgreSQL raises when a subquery you expected to be scalar returns more than one row
- Apply the two standard techniques (aggregate functions, `ORDER BY … LIMIT 1`) that guarantee a subquery stays scalar

## Prerequisites

This is the first topic in Module 11, so it depends only on earlier modules, not on any other topic in this one: **Module 7 (Querying Basics)** for `WHERE` and comparison operators, and **Module 9 (Aggregation)** for aggregate functions like `AVG` and `MAX`, which most scalar subqueries in this topic wrap.

## Motivation

You already know how to compare a column to a literal value: `WHERE amount > 200`. But real questions are rarely phrased against a number you already know — they're phrased against a number the database itself has to compute first: "orders above *the average* order value," "the customer who placed *the single largest* order." A scalar subquery lets you plug a computed, single value directly into a spot where SQL expects one, without running a separate query, writing down its result by hand, and pasting that number into a second query.

## Problem Statement

Suppose you want to find every order whose amount is above the average order amount. Without a scalar subquery, you'd have to do this in two manual steps: first run `SELECT AVG(amount) FROM orders;`, read off the number, then hand-type that number into a second query's `WHERE amount > 184.55`. This is fragile in an obvious way — the moment a new order is inserted, the average changes, and your hand-typed number is silently stale. What you actually want is to ask the database to compute that number *and* use it in the same statement, every time, guaranteed to be current. That's exactly what a scalar subquery does.

## Concept

### The Running Example Schema

This module reuses and extends the `customers`/`orders` schema introduced in Module 10 (Joins & Set Operations) — including Dara Singh, a customer who has never placed an order, and a guest checkout order with no linked customer at all. Module 10 didn't need a monetary value per order; this module does, since most of its subqueries revolve around comparing amounts. So the running schema here adds one column, `amount`, to `orders`.

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
    status      TEXT NOT NULL,
    amount      NUMERIC(10,2) NOT NULL
);

INSERT INTO customers (customer_id, name, email, city) VALUES
    (1, 'Ava Patel',  'ava@example.com',  'Austin'),
    (2, 'Ben Ortiz',  'ben@example.com',  'Denver'),
    (3, 'Chen Wu',    'chen@example.com', 'Austin'),
    (4, 'Dara Singh', 'dara@example.com', 'Seattle'),
    (5, 'Elin Kask',  'elin@example.com', 'Seattle');

INSERT INTO orders (order_id, customer_id, order_date, status, amount) VALUES
    (101, 1,    '2026-01-05', 'shipped',   250.00),
    (102, 1,    '2026-02-10', 'shipped',   120.00),
    (103, 2,    '2026-01-20', 'cancelled',  75.50),
    (104, 3,    '2026-03-01', 'shipped',   500.00),
    (105, NULL, '2026-03-15', 'shipped',    40.00),
    (106, 2,    '2026-03-18', 'shipped',   160.00),
    (107, 3,    '2026-03-22', 'shipped',    60.00),
    (108, 5,    '2026-02-25', 'shipped',   430.00),
    (109, 1,    '2026-03-27', 'shipped',    60.00),
    (110, 5,    '2026-03-29', 'shipped',   150.00);
```

`order_id` and `customer_id` are given explicit values here rather than letting `SERIAL` generate them, purely so the IDs stay fixed and predictable across every example in this module. (`SERIAL` columns can be explicitly assigned a value, exactly like any other column — PostgreSQL only auto-generates one when you omit the column entirely, as Module 01's [Your First Query](../01-introduction/05-your-first-query.md) first showed.) Notice two details carried over unchanged from Module 10: **Dara Singh (customer 4) has placed zero orders**, and **order 105 has a `NULL` `customer_id`** — a guest checkout with no linked account. Both details are inert for this topic but become central in Topic 4.

### What Makes a Subquery "Scalar"

> A **scalar subquery** is a subquery, written in parentheses, that is guaranteed (or at least expected) to return **exactly one row and exactly one column** — a single value.

Because it reduces to one value, a scalar subquery can be used anywhere SQL expects a single value: in a `SELECT` list as if it were a computed column, in a `WHERE` clause as if it were a literal, or anywhere else an expression is legal (a `CASE` branch, an argument to a function, and so on).

### Using a Scalar Subquery in a `WHERE` Clause

```sql
SELECT order_id, customer_id, amount
FROM orders
WHERE amount > (SELECT ROUND(AVG(amount), 2) FROM orders)
ORDER BY order_id;
```

```
 order_id | customer_id | amount
----------+-------------+--------
      101 |           1 | 250.00
      104 |           3 | 500.00
      108 |           5 | 430.00
(3 rows)
```

The inner query `SELECT ROUND(AVG(amount), 2) FROM orders` computes a single number — the average of all ten `amount` values, `184.55` — and PostgreSQL substitutes that number in for the subquery, exactly as if you had written `WHERE amount > 184.55`. The difference is that this number is *computed live, every time the query runs*, so it never goes stale as `orders` changes.

### Using a Scalar Subquery to Find "the One With the Extreme Value"

A common pattern combines `ORDER BY` and `LIMIT 1` inside the subquery to pick out a single specific row's value, rather than an aggregate:

```sql
SELECT name
FROM customers
WHERE customer_id = (
    SELECT customer_id
    FROM orders
    ORDER BY amount DESC
    LIMIT 1
);
```

```
   name
-----------
 Chen Wu
(1 row)
```

The inner query sorts every order by `amount`, descending, and `LIMIT 1` keeps only the top row — order 104, Chen Wu's $500.00 order — collapsing what would otherwise be many rows down to exactly one `customer_id` value. Without `LIMIT 1`, this subquery would return all ten `customer_id` values (one per order), and using it as a scalar would fail — which is exactly the failure mode covered next.

### Using a Scalar Subquery in a `SELECT` List

A scalar subquery can also appear as a computed column, evaluated for every row of the outer query:

```sql
SELECT
    order_id,
    amount,
    ROUND(amount - (SELECT AVG(amount) FROM orders), 2) AS diff_from_avg
FROM orders
ORDER BY order_id;
```

```
 order_id | amount  | diff_from_avg
----------+---------+---------------
      101 |  250.00 |          65.45
      102 |  120.00 |         -64.55
      103 |   75.50 |        -109.05
      104 |  500.00 |         315.45
      105 |   40.00 |        -144.55
      106 |  160.00 |         -24.55
      107 |   60.00 |        -124.55
      108 |  430.00 |         245.45
      109 |   60.00 |        -124.55
      110 |  150.00 |         -34.55
(10 rows)
```

The subquery `(SELECT AVG(amount) FROM orders)` computes the same value — `184.55` — for every single output row. This is worth noticing carefully: the subquery does not depend on which row of `orders` is currently being displayed. It is **non-correlated** — it produces one fixed answer, computed independently of the outer query. Topic 3 introduces the opposite case, where the subquery's answer changes per outer row.

### The Danger: When a "Scalar" Subquery Returns More Than One Row

Nothing about the syntax `(SELECT amount FROM orders WHERE customer_id = 1)` guarantees a single row — it's just an ordinary subquery that PostgreSQL will only accept as a scalar value *if, when actually run, it happens to produce one row*. Ava Patel (`customer_id = 1`) has placed **three** orders, so this fails:

```sql
SELECT
    name,
    (SELECT amount FROM orders WHERE customer_id = 1) AS an_order
FROM customers
WHERE customer_id = 1;
```

```
ERROR:  more than one row returned by a subquery used as an expression
```

PostgreSQL cannot collapse three rows (`250.00`, `120.00`, `60.00`) into the single value the surrounding expression demands, so it raises a runtime error rather than silently picking one of them. This is not a syntax error caught before the query runs — the parser has no way to know in advance how many rows a subquery will produce, since that depends on the actual data. The check happens at execution time, and it will pass or fail depending on the data currently in the table, which is exactly why it's easy to write a scalar subquery that works fine in testing (against a customer with one order) and then breaks in production the moment a customer places a second order.

### Guaranteeing a Subquery Stays Scalar

Two techniques reliably prevent this error:

1. **Wrap the subquery in an aggregate function** (`AVG`, `MAX`, `MIN`, `SUM`, `COUNT`) — an aggregate always collapses any number of rows (including zero) down to exactly one row, by definition (Module 9).
2. **Add a deterministic `ORDER BY` plus `LIMIT 1`** — this explicitly picks a single row out of a set that might otherwise contain many, as shown in the "customer with the largest order" example above.

A subquery filtered only by an equality condition on a non-unique column (like `customer_id = 1` above) guarantees neither — it happens to return one row only when that particular value is associated with exactly one row, which is a fact about the data, not about the query.

## Internal Working (Preview)

For a non-correlated scalar subquery like `(SELECT AVG(amount) FROM orders)`, PostgreSQL's planner typically evaluates it once, independently of the outer query, and treats the result as a fixed value for the rest of execution — internally, this is often planned as an "InitPlan," a one-time subplan computed before the main query runs:

```
 Outer query execution
        │
        ▼
 Scalar subquery (InitPlan): run once, produce ONE value
        │
        ▼
 That single value is substituted into the outer WHERE/SELECT
        │
        ▼
 Outer query proceeds using that fixed value, like a constant
```

The "more than one row" check is a safety net enforced at the moment the subquery's actual result is consumed: if execution produces a second row where only one value was expected, PostgreSQL raises the error immediately rather than silently discarding the extra rows. Topic 3 covers a fundamentally different execution shape — a **correlated** subquery, which cannot be evaluated once up front like this, because its answer depends on the specific outer row currently being processed.

## Real-World Analogy

A scalar subquery is like asking one specific, trusted expert a single yes/no or one-number question: "what's the average order size this month?" You expect one number back, and you build your next sentence around it ("orders bigger than that number..."). If, instead of one expert, you accidentally sent that question to an entire committee and each member shouted back a different number, you'd have no coherent way to finish your sentence — that's precisely the "more than one row" error: PostgreSQL refuses to arbitrarily pick one voice out of a crowd when your query's grammar demanded exactly one answer.

## Why Scalar Subqueries Were Designed This Way

SQL is a declarative, expression-based language (Module 01 — [What Is SQL?](../01-introduction/02-what-is-sql.md)): a `WHERE` clause is built from expressions that must ultimately resolve to a single value per row being tested, and a `SELECT` list column is a single expression per output row. A scalar subquery lets the full expressive power of a `SELECT` statement — joins, aggregation, filtering — appear anywhere a single value is expected, instead of restricting you to literal constants or values you already know ahead of time. Enforcing the single-row/single-column rule at runtime (rather than trying to guess it in advance) preserves SQL's declarative nature: you describe *what* value you want, and the database is responsible for actually producing it — and for telling you clearly if your description didn't, in fact, pin down exactly one.

## Advantages

- **No manual two-step process.** The value you need is computed inside the same statement that uses it, so there's no "run one query, copy the number, paste it into a second query" workflow to keep in sync.
- **Always current.** Because the subquery runs every time the outer query runs, the value it plugs in reflects the data at query time — no risk of a stale hand-typed constant.
- **Composable with the full power of `SELECT`.** A scalar subquery can itself contain joins, `WHERE` clauses, and aggregates — anything a normal query can do, as long as the final result collapses to one value.

## Disadvantages / Limitations

- **Runtime risk, not compile-time safety.** As shown above, a subquery that isn't provably scalar can pass code review and testing, then fail in production the instant the underlying data changes shape.
- **Can be evaluated repeatedly if not truly independent of the outer row.** A scalar subquery that references an outer-query column (a correlated scalar subquery, Topic 3) may be conceptually re-run once per outer row, which can be a real performance cost on large tables (Module 20 covers measuring this precisely with `EXPLAIN`).
- **Harder to read for a query with many nested scalar subqueries.** Several scalar subqueries stacked in one `SELECT` list can make a query harder to scan than an equivalent join or CTE (Module 17) that computes the same values once, named, up front.

## Best Practices

- Default to wrapping a scalar subquery in an aggregate function (`MAX`, `AVG`, `COUNT`, `SUM`) whenever the underlying condition could plausibly match more than one row — this guarantees safety rather than hoping the data stays that way.
- If you must use `ORDER BY … LIMIT 1` to pick "the one," make the `ORDER BY` fully deterministic (add a tiebreaker column, such as `order_id`, if two rows could tie on the primary sort column) — otherwise which row "wins" can vary between runs.
- Always give a `SELECT`-list scalar subquery a column alias (`AS diff_from_avg` above) — an unaliased subquery column produces an unreadable, PostgreSQL-generated name in the result set.
- If you find yourself writing more than one or two scalar subqueries in the same `SELECT` list, consider whether a join to a derived table (Topic 2) would compute the same values once and read more clearly.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Filtering a scalar subquery with an equality on a non-unique column (e.g., `customer_id = 1`) and assuming it will always return one row | It only returns one row *if that value happens to correspond to exactly one row in the current data*. The moment a customer places a second order, the identical query starts erroring — the mistake was assuming a fact about today's data was a guarantee. |
| Forgetting to alias a `SELECT`-list scalar subquery | The result set gets an auto-generated, unreadable column name (often literally `?column?` in `psql`) instead of a meaningful one. |
| Using `MAX()`/`AVG()` inside a subquery but forgetting it still needs to be scalar overall | Wrapping in an aggregate guarantees one row *for that subquery's own `GROUP BY` grouping* — if the subquery has a `GROUP BY` clause producing multiple groups, it can still return multiple rows even though it contains an aggregate function. |
| Assuming a scalar subquery error is a syntax error you can catch by reading the query | It's a runtime error dependent on the data — the same query text can succeed today and fail tomorrow as rows are added. |

## Interview Questions

1. **Q: What is a scalar subquery, and where can it legally appear in a SQL statement?**
   A: A scalar subquery is a subquery that returns exactly one row and exactly one column — a single value. Because it resolves to one value, it can appear anywhere SQL expects a single value: in a `SELECT` list as a computed column, in a `WHERE` or `HAVING` clause compared against a column, or as an argument to a function.

2. **Q: What happens if a subquery you intended to be scalar actually returns more than one row at runtime?**
   A: PostgreSQL raises a runtime error — `more than one row returned by a subquery used as an expression` — rather than silently picking one row. This is a data-dependent runtime check, not something caught at parse time, so a query can work today and fail later if the underlying data changes.

3. **Q: How do you guarantee a subquery always stays scalar?**
   A: Wrap it in an aggregate function (`AVG`, `MAX`, `MIN`, `SUM`, `COUNT`), which always collapses any number of matching rows down to one, or add a deterministic `ORDER BY` combined with `LIMIT 1` to explicitly pick a single row out of a potentially larger result set.

4. **Q: What's the difference in behavior between a scalar subquery like `(SELECT AVG(amount) FROM orders)` and a correlated scalar subquery?**
   A: The non-correlated version above computes one fixed value, independent of the outer query, and can conceptually be evaluated once. A correlated scalar subquery references a column from the outer query, so its answer can differ for every outer row and must be conceptually re-evaluated per row — covered in full in Topic 3.

## Summary

- A **scalar subquery** returns exactly one row and one column, so it can be used anywhere SQL expects a single value — in a `WHERE` clause, a `SELECT` list, or any other expression position.
- Scalar subqueries let you compute a derived value (an average, a maximum, "the row with the extreme value") live, inside the same statement that uses it, instead of a fragile two-step manual process.
- If a subquery you're treating as scalar actually returns more than one row, PostgreSQL raises a runtime error rather than guessing — this is a data-dependent failure, not a syntax error.
- Wrapping the subquery in an aggregate function, or adding a deterministic `ORDER BY … LIMIT 1`, are the two standard techniques that guarantee scalar behavior.
- A non-correlated scalar subquery (like the ones in this topic) produces one fixed value regardless of which outer row is being processed — Topic 3 introduces correlated subqueries, whose answer changes per outer row.
