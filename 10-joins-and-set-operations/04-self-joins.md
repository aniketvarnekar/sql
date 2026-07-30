# Self-Joins

## Learning Objectives

By the end of this section you should be able to:
- Explain what a self-join is and why table aliases are mandatory for writing one
- Write a self-join to navigate a hierarchical (employee/manager-style) relationship within one table
- Write a self-join to find pairs or duplicates within a single table
- Explain why a self-join is not a distinct SQL feature, but an ordinary join applied to one table referenced twice

## Prerequisites

- **[INNER JOIN](01-inner-join.md)** and **[LEFT and RIGHT OUTER JOIN](02-left-and-right-outer-join.md)** — a self-join uses exactly the same `JOIN...ON` syntax already covered; this topic is about *when* and *why* you'd join a table to itself, not new syntax.

## Motivation

Every join so far has combined two visibly different tables — `customers` with `orders`, `customers` with `orders` again for outer joins. But some relationships exist *within* a single table: an employee's manager is also an employee, recorded in the very same `employees` table; two customers might live in the same city, a fact only discoverable by comparing rows of `customers` against other rows of the very same table. SQL doesn't need a separate feature for this — the exact same `JOIN` syntax works perfectly, once you understand the one trick required to make it unambiguous: aliasing the same table twice, as if it were two different tables.

## Problem Statement

Consider a company's `employees` table where each employee optionally has a manager — who is, themselves, just another employee in the same table:

```
 employee_id |  name  | manager_id
-------------+--------+------------
           1 | Farah  |
           2 | Grace  |          1
           3 | Hassan |          1
           4 | Ivy    |          2
```

Farah has no manager (she's the top of the hierarchy). Grace and Hassan both report to Farah. Ivy reports to Grace. If you're asked "list every employee together with their manager's name," querying `employees` alone only gives you `manager_id` — a number, not a name. You need to look up, for every employee's `manager_id`, the matching row in the very same `employees` table to find that manager's name. There's no second table involved at all — you need to join `employees` to itself.

## Concept

### Setting Up the Example

This topic introduces a small `employees` table specifically for demonstrating hierarchical self-joins — distinct from this module's running `customers`/`orders`/`order_items` schema, but built the same way, with `manager_id` as a foreign key referencing the very same table's primary key:

```sql
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    manager_id  INTEGER REFERENCES employees(employee_id)
);

INSERT INTO employees (name, manager_id) VALUES
    ('Farah',  NULL),
    ('Grace',  1),
    ('Hassan', 1),
    ('Ivy',    2);
```

`manager_id INTEGER REFERENCES employees(employee_id)` is a foreign key that references its *own* table — perfectly valid, and exactly the mechanism that makes this a self-join scenario rather than an ordinary two-table one. Farah's `manager_id` is `NULL` because she has no manager.

### Aliasing the Same Table Twice

You cannot write `FROM employees JOIN employees ON ...` — PostgreSQL has no way to tell which `employees` reference in the `SELECT` list or `ON` clause means what, since both sides are the literal same table name. The fix is to alias the same table twice, as if they were two separate tables:

```sql
SELECT
    e.name        AS employee_name,
    m.name        AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.name;
```

```
 employee_name | manager_name
---------------+---------------
 Farah         |
 Grace         | Farah
 Hassan        | Farah
 Ivy           | Grace
(4 rows)
```

Here, `e` and `m` both refer to `employees`, but they represent two different *roles* in this query: `e` is "the employee," `m` is "that employee's manager." `LEFT JOIN` is used (not `INNER JOIN`) specifically because Farah has no manager (`manager_id IS NULL`) — an `INNER JOIN` would silently drop her from the result entirely, exactly as covered in Topic 1 and Topic 2, since this is, mechanically, no different from any other join you've already learned; the only new idea is that both sides happen to come from the same underlying table.

### Use Case: Hierarchical Data

The employee/manager example above is the canonical case for a self-join: any table that models a hierarchy by having each row optionally point back to another row *in the same table* (an employee's manager, a category's parent category, a comment's parent comment) needs a self-join to "flatten" one level of that hierarchy into a single readable row.

You can extend this to ask more targeted hierarchical questions, such as "who are Farah's direct reports?":

```sql
SELECT e.name AS direct_report
FROM employees e
INNER JOIN employees m ON e.manager_id = m.employee_id
WHERE m.name = 'Farah';
```

```
 direct_report
----------------
 Grace
 Hassan
(2 rows)
```

Note this only finds *direct* reports (one level down). Finding an entire multi-level reporting chain of arbitrary depth (e.g., "everyone who ultimately reports up to Farah, at any level") requires a **recursive** query, since a fixed self-join can only walk exactly one level of a hierarchy per join — this is covered in full in Module 17 (CTEs & Recursion).

### Use Case: Finding Pairs Within One Table

A second, equally common self-join use case is comparing rows of a table *against each other* — for example, finding pairs of customers who happen to live in the same city, using this module's running `customers` table:

```sql
SELECT
    c1.name AS customer_a,
    c2.name AS customer_b,
    c1.city
FROM customers c1
INNER JOIN customers c2
    ON c1.city = c2.city
    AND c1.customer_id < c2.customer_id
ORDER BY c1.city;
```

```
 customer_a | customer_b |  city
------------+------------+--------
 Ava Patel  | Chen Wu    | Austin
(1 row)
```

Two details here are essential, not optional style choices:

- **`c1.city = c2.city`** is the actual condition of interest — "these two rows share a city."
- **`c1.customer_id < c2.customer_id`** exists purely to avoid two problems at once: without it, every customer would trivially match themselves (`c1.customer_id = c2.customer_id`, comparing a row to itself, which is always true for `city = city`), and every genuine pair would appear **twice**, once as "(Ava, Chen)" and once again as "(Chen, Ava)." Using `<` instead of `!=` solves both problems in one condition: it excludes a row matching itself, and it keeps only one direction of each pair (the one where the first customer's ID is smaller), rather than both.

This same pattern — self-join plus an inequality on a unique column — is the standard way to find duplicate values, near-duplicate records, or any other "compare this row to other rows in the same table" question in SQL.

## Internal Working (Preview)

There is nothing mechanically special about a self-join from the database engine's perspective. Once you write `FROM employees e LEFT JOIN employees m ON ...`, PostgreSQL's planner treats `e` and `m` as two independent row sources to be joined — internally, it doesn't matter to the join algorithm (nested loop, hash join, or merge join — Module 20) whether those two row sources happen to be scans of the same underlying table or two entirely different tables. The alias is purely a SQL-language-level bookkeeping device to let *you* (and the parser) refer to the two "roles" unambiguously; by the time the query planner is choosing a join strategy, it's working with two row streams like any other join, full stop.

## Real-World Analogy

Think of a self-join like a family reunion sign-in sheet used to trace lineage. Everyone signs in with their own name and their parent's name, on the same single sheet — there's no separate "parents sheet" and "children sheet." To answer "who is Priya's parent?", you look up Priya's row to find the parent's name, then search the *same* sheet again for a row where that name is the person who signed in — effectively treating one copy of the sheet as "the child view" and another look at the same sheet as "the parent view," even though it's physically one sheet the whole time.

## Why Self-Joins Work This Way

SQL doesn't provide a separate "self-join" keyword or distinct syntax, and that is a deliberate consequence of the relational model's design: a table is just a table, referenced by name, and a `JOIN` combines any two row sources based on a condition — nothing in that definition requires the two row sources to be *different* tables. Rather than inventing special-case syntax for "joining a table to itself," the existing general-purpose `JOIN...ON` mechanism, combined with the pre-existing ability to alias any table reference, already covers this case completely. This is a good example of relational query design favoring a small number of general, composable primitives (tables, aliases, joins, conditions) over many narrow, special-purpose features — the same principle you'll see again with recursive CTEs (Module 17) handling arbitrary-depth hierarchies that a single self-join cannot.

## Advantages

- **No new syntax to learn** — a self-join is exactly the `JOIN...ON` syntax from Topics 1–3, applied with both sides aliased to the same underlying table; there's nothing conceptually new to internalize beyond the aliasing trick.
- **Cleanly expresses one level of hierarchy or in-table comparison in a single query** — no need for application-level code to loop through rows and cross-reference them manually.
- **Composable with everything else in this module** — a self-join can be `LEFT` or `INNER`, chained with other tables (Topic 5), or filtered with ordinary `WHERE` conditions, exactly like any other join.

## Disadvantages / Limitations

- **Only walks one level of a hierarchy per join** — finding an employee's manager's manager requires either a second self-join (quickly getting unwieldy for arbitrary depth) or a recursive query (Module 17); a single self-join cannot express "however many levels deep" on its own.
- **Aliases are mandatory and must be tracked carefully** — every column reference in a self-join must be qualified with the correct alias (`e.name` vs. `m.name`), and mixing them up produces a query that runs without error but returns a subtly wrong result, since PostgreSQL has no way to know you meant the "other" role.
- **Pair-finding self-joins can be easy to get subtly wrong** — forgetting the inequality condition (using `!=` instead of `<`, or omitting it entirely) either produces duplicate mirrored pairs or lets every row match itself, both of which are easy to overlook if you don't specifically check for them.

## Best Practices

- Choose clear, meaningful aliases that describe each row's *role* in the relationship (`e`/`m` for "employee"/"manager", or more explicit still, `employee`/`manager`) rather than generic `a`/`b`, especially in a hierarchical self-join — it makes the query's intent immediately legible to someone reading it later.
- When finding pairs within a table, always include an inequality on a unique column (typically `<` on the primary key) to avoid both self-matches and mirrored duplicate pairs — treat this as a required part of the pattern, not an optional refinement.
- Use `LEFT JOIN` rather than `INNER JOIN` for hierarchical self-joins whenever the "parent" side can legitimately be absent (a top-level employee with no manager, a top-level category with no parent category) — otherwise you'll silently lose exactly the rows at the top of the hierarchy, the same trap covered in Topic 1 and Topic 2.
- Reach for a recursive query (Module 17) instead of a self-join the moment you need "any number of levels," rather than trying to chain several self-joins together to simulate it.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Forgetting to alias the table, writing `FROM employees JOIN employees ON ...` | PostgreSQL raises an error (or, without care, is simply ambiguous) — the same table name can't refer to two different roles without distinct aliases. |
| Using `!=` instead of `<` when finding pairs within a table | `!=` excludes self-matches but still produces every genuine pair twice (once in each direction); `<` correctly excludes self-matches *and* keeps only one direction per pair. |
| Using `INNER JOIN` for a hierarchical self-join when top-level rows (no manager/no parent) exist | Silently drops every row at the top of the hierarchy from the result, exactly as an `INNER JOIN` drops any other unmatched row (Topic 1) — use `LEFT JOIN` when the "parent side" can legitimately be absent. |
| Expecting a single self-join to walk an arbitrary number of hierarchy levels | A self-join only ever walks exactly one level (row → its direct match) per join written; multi-level hierarchies of unknown depth require a recursive query (Module 17), not more self-joins. |

## Interview Questions

1. **Q: What is a self-join, and why is aliasing required to write one?**
   A: A self-join is an ordinary join where both sides of the `JOIN` reference the same underlying table, used to relate rows of a table to other rows of that same table (e.g., an employee to their manager, who is also a row in the employees table). Aliasing is required because SQL needs two distinct names to refer to the table's two different "roles" in the query (e.g., `e` for the employee row, `m` for the manager row) — without aliases, every column reference would be ambiguous.

2. **Q: How would you find every pair of customers who share the same city, without duplicate or self-matched pairs?**
   A: Self-join the `customers` table to itself (e.g., aliased `c1` and `c2`) on `c1.city = c2.city`, and add `c1.customer_id < c2.customer_id` to exclude a row matching itself and to keep only one direction of each pair rather than both mirrored copies.

3. **Q: Why would you use `LEFT JOIN` rather than `INNER JOIN` for an employee/manager self-join?**
   A: Because some employees (typically those at the top of the organization) have no manager (`manager_id IS NULL`). An `INNER JOIN` would exclude those employees entirely from the result, since they have no matching row on the "manager" side; `LEFT JOIN` preserves them, showing a `NULL` manager name instead.

4. **Q: Can a self-join alone answer "who are all of this employee's indirect reports, at any depth"?**
   A: No — a single self-join only relates a row to rows exactly one level away (its direct manager or direct reports). Answering a question involving an arbitrary, unknown number of hierarchy levels requires a recursive query (a recursive common table expression), covered in Module 17, not repeated self-joins.

## Summary

- A self-join is not a distinct SQL feature — it's an ordinary `JOIN` where both sides reference the same table, made unambiguous by giving each side a different alias.
- The two canonical use cases are: navigating one level of a hierarchical relationship stored within a single table (employee/manager, category/parent-category), and finding pairs or duplicates by comparing a table's rows against its own other rows.
- Use `LEFT JOIN` for hierarchical self-joins whenever the "parent" side can legitimately be missing, to avoid silently dropping top-level rows.
- When finding pairs, always add an inequality (typically `<` on a unique column) to exclude self-matches and avoid counting each pair twice.
- A self-join only walks exactly one level per join — arbitrary-depth hierarchies require a recursive query, covered later in Module 17.
