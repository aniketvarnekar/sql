# The WITH Clause

## Learning Objectives

By the end of this section you should be able to:
- Write a single Common Table Expression (CTE) using `WITH name AS (...)` and reference it in the query that follows
- Define multiple CTEs in one `WITH` clause, where a later CTE references an earlier one
- Explain, in your own words, why naming an intermediate result set improves readability over nesting subqueries
- Rewrite a query built from nested subqueries as an equivalent, flatter chain of CTEs

## Prerequisites

- **Module 11 (Subqueries)** — a CTE is conceptually a named subquery; you need to already understand what a subquery and a derived table are before "a subquery you can name and reuse" means anything.
- **Module 09 (Aggregation)** — this topic's worked examples group rows with `GROUP BY` and compute `AVG`/`COUNT`; you should already be comfortable with both.
- **Module 10 (Joins & Set Operations)**, specifically [Inner Join](../10-joins-and-set-operations/01-inner-join.md) — the examples below join a CTE's result back to the base table using the same explicit `JOIN ... ON` syntax taught there.

## Motivation

Real reporting questions are rarely answerable in a single flat step. "Which employees earn more than their department's average salary?" already requires two logical steps: first compute each department's average, then compare each employee against it. As soon as a query needs three or four such steps chained together, writing it as deeply nested subqueries becomes something you have to unravel from the inside out just to read it — and the person reading your query six months from now (quite possibly you) pays that tax every single time. The `WITH` clause exists to let you name each logical step once, in the order you actually think about the problem, and then simply refer to those names.

## Problem Statement

Suppose you want each department's average salary, then only the employees earning above their own department's average, ordered by department and salary. Without a way to name an intermediate result, you're forced to nest:

```sql
SELECT e.name, e.department, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department = e.department
)
ORDER BY e.department, e.salary DESC;
```

This works, but the department-average logic is buried inside a correlated subquery in the `WHERE` clause — you have to read the whole statement and mentally track that `e2` is "the same table again, filtered per outer row" before the query's intent becomes clear. Now imagine needing that same department-average number in two more places in the same query (say, also displaying it as a column, and also using it in a second filter) — you'd either repeat that subquery text three times, or fall back to an awkward self-join. Neither option reads as clearly as "first compute department averages, then compare against them."

## Concept

### Basic Syntax

A **Common Table Expression (CTE)** is a named, temporary result set defined at the top of a query using the `WITH` keyword, which the rest of that single statement can then refer to by name — as if it were a real table.

```sql
WITH dept_avg AS (
    SELECT department, ROUND(AVG(salary), 2) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.department, e.salary, d.avg_salary
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_salary
ORDER BY e.department, e.salary DESC;
```

Reading this top to bottom in the order it's written now mirrors the order you'd actually explain the problem out loud: "First, get each department's average salary (`dept_avg`). Then, join every employee to their department's average, and keep only the ones above it."

Assume a familiar `employees` table (extending the one first introduced in [Your First Query](../01-introduction/05-your-first-query.md)):

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    department TEXT NOT NULL,
    salary NUMERIC NOT NULL
);

INSERT INTO employees (name, department, salary) VALUES
    ('Asha',  'Sales',       85000),
    ('Ben',   'Engineering', 95000),
    ('Chen',  'Sales',       78000),
    ('Diego', 'Engineering', 105000),
    ('Elin',  'Engineering', 88000),
    ('Farah', 'Marketing',   72000),
    ('Gus',   'Marketing',   69000),
    ('Priya', 'Sales',       91000);
```

Running the `WITH` query above against this data produces:

```
 name  | department  | salary |  avg_salary
-------+-------------+--------+--------------
 Diego | Engineering | 105000 |     96000.00
 Farah | Marketing   |  72000 |     70500.00
 Priya | Sales       |  91000 |     84666.67
 Asha  | Sales       |  85000 |     84666.67
(4 rows)
```

Breaking down the pieces:

| Piece | Meaning |
|---|---|
| `WITH dept_avg AS ( ... )` | Defines a CTE named `dept_avg`, whose result is the `SELECT department, ROUND(AVG(salary), 2) AS avg_salary ... GROUP BY department` query. |
| `dept_avg` | From here down, `dept_avg` can be used anywhere a table name could be used — in a `FROM`, in a `JOIN`, in a subquery — for the rest of this one statement only. |
| `JOIN dept_avg d ON e.department = d.department` | Joins each employee row to the one `dept_avg` row matching its department, exactly like joining two real tables. |
| `WHERE e.salary > d.avg_salary` | Keeps only employees whose own salary exceeds their department's computed average. |

Notice the query now reads as two clearly separated, named steps — compute `dept_avg`, then use it — rather than one subquery nested inside a `WHERE` clause.

### Multiple CTEs in One Query

A single `WITH` clause can define more than one CTE, separated by commas. Each subsequent CTE may reference any CTE defined **before** it (but not one defined after it — order matters, top to bottom):

```sql
WITH dept_avg AS (
    SELECT department, ROUND(AVG(salary), 2) AS avg_salary
    FROM employees
    GROUP BY department
),
above_avg AS (
    SELECT e.id, e.name, e.department, e.salary, d.avg_salary
    FROM employees e
    JOIN dept_avg d ON e.department = d.department
    WHERE e.salary > d.avg_salary
)
SELECT department, COUNT(*) AS above_avg_count
FROM above_avg
GROUP BY department
ORDER BY department;
```

Here `above_avg` is itself built on top of `dept_avg` — it couldn't exist without it — and the final `SELECT` is built on top of `above_avg`. Running this against the same data:

```
 department  | above_avg_count
-------------+-----------------
 Engineering |               1
 Marketing   |               1
 Sales       |               2
(3 rows)
```

Each CTE is a clean, self-contained, named step:

```
dept_avg      →  one row per department, its average salary
     │
     ▼
above_avg     →  every employee earning more than their department's average
     │
     ▼
final SELECT  →  how many such employees per department
```

This is the central benefit of `WITH`: a query that would otherwise be one large, deeply nested expression becomes a readable sequence of named, single-purpose steps — much like giving each intermediate value in a calculation its own clearly labeled name instead of writing one enormous unlabeled expression.

### Column Naming

By default, a CTE's output columns take their names from the `SELECT` inside it (via aliases, as `avg_salary` was aliased above). You can also name them explicitly right after the CTE's name, which is useful when the inner query's column expressions don't have (or don't deserve) their own aliases:

```sql
WITH dept_avg (department, avg_salary) AS (
    SELECT department, ROUND(AVG(salary), 2)
    FROM employees
    GROUP BY department
)
SELECT * FROM dept_avg ORDER BY avg_salary DESC;
```

This produces the identical result as before, just with the column names declared up front instead of inferred from the inner `SELECT`'s aliases.

## Internal Working (Preview)

Conceptually, when PostgreSQL executes a query with a `WITH` clause, each CTE's defining query is a distinct, isolated unit of work whose result is handed to the rest of the statement under the given name:

```
 WITH dept_avg AS (...)     →  [1] evaluate this query, producing a named result set
 WITH above_avg AS (...)    →  [2] evaluate this query, which may reference dept_avg's result
 final SELECT ...           →  [3] evaluate this last, which may reference dept_avg and/or above_avg
```

Whether step [1]'s result is physically computed once into a temporary structure ("materialized") before step [2] runs, or whether the planner instead folds the CTE's definition directly into the surrounding query as it would a plain subquery, is a planner decision with real performance consequences — that decision, and how to control it explicitly, is the entire subject of the next topic, [CTEs vs. Subqueries](02-ctes-vs-subqueries.md). For now, treat a CTE simply as "a named, temporary result usable for the rest of this one statement" — the mental model in this topic is correct regardless of which strategy the planner picks underneath it.

## Real-World Analogy

Writing a query with several CTEs is like following a recipe that lists prepared components before assembly — "the marinade," "the sauce," "the filling" — each defined once, by name, in its own short paragraph, rather than one unbroken wall of text that interleaves every ingredient and step for the entire dish. When the final assembly step says "add the sauce," you don't have to re-read how the sauce was made; you trust the name. A deeply nested subquery is the unbroken wall of text — technically it contains the same information, but you have to hold the whole thing in your head at once to follow it.

## Why the WITH Clause Was Designed This Way

SQL is a declarative language (established in [What Is SQL?](../01-introduction/02-what-is-sql.md)) — you describe *what* result you want, not the step-by-step procedure to compute it. But "what you want" can still be a genuinely multi-stage idea, and forcing every multi-stage idea into a single nested expression conflates *the logic of the query* with *the physical shape of the SQL text*. The `WITH` clause separates these: it lets you express a multi-step declarative idea as multiple named declarative steps, still within one statement, still with no procedural loops or variables — just a sequence of named results, each one allowed to build on the results named before it. This is why a CTE is scoped to exactly one statement and disappears afterward (unlike a view, covered in Module 12, which persists as a named object in the database) — it exists purely to organize the *reasoning* of a single query, not to create a durable, independently reusable object.

## Advantages

- **Readability** — a query built from several named CTEs can be read top-to-bottom in the same order you'd explain the logic out loud, instead of inside-out as nested subqueries require.
- **Single place to define reused logic** — if a calculation is needed more than once in the same query, defining it once as a CTE avoids repeating (and risking a typo or inconsistency between copies of) the same subquery text.
- **Breaks a complex problem into named, checkable steps** — you can mentally (or literally, with `SELECT * FROM dept_avg;` while developing) inspect what an individual CTE produces in isolation, which makes debugging a multi-step query far easier than debugging one giant nested expression.
- **Foundation for recursion** — the `WITH RECURSIVE` form (Topic 3 of this module) is only possible because a CTE can, in that specific form, refer back to itself — a capability a plain subquery does not have.

## Disadvantages / Limitations

- **Scoped to a single statement** — unlike a view (Module 12), a CTE cannot be reused across different queries; it must be redefined every time, in every statement that needs it.
- **Adds ceremony for genuinely trivial, one-off calculations** — naming a CTE that's used exactly once, for a very simple filter, can be more verbose than just inlining a small subquery directly (this trade-off is explored in full in the next topic).
- **Order-dependent** — a CTE can only reference CTEs defined earlier in the same `WITH` clause (except in the recursive form), so restructuring the order of your CTEs can require moving dependent definitions around.

## Best Practices

- Give each CTE a descriptive, purpose-revealing name (`dept_avg`, `above_avg`) rather than a generic one (`t1`, `cte2`) — the name is the whole point of readability.
- Keep each CTE focused on a single logical transformation step; if a CTE's inner query is trying to do three unrelated things at once, split it into multiple CTEs instead.
- Use multiple, smaller CTEs chained together rather than one CTE with a sprawling `SELECT` — it mirrors how you'd actually explain the query's logic in steps.
- Reach for a CTE specifically when a subquery would otherwise be reused more than once in the same query, or when naming an intermediate step measurably improves how the query reads — not automatically for every query as a matter of style.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a CTE is visible in a later, separate query | A CTE only exists for the duration of the single statement it's defined in — it is not a persistent object like a view (Module 12) and cannot be referenced from a different `SELECT` run afterward. |
| Referencing a later CTE from an earlier one | CTEs (in the non-recursive form) can only reference CTEs defined *before* them in the same `WITH` clause; a forward reference is an error. |
| Forgetting the comma between multiple CTE definitions | `WITH a AS (...) b AS (...)` is invalid — each CTE definition after the first must be separated by a comma: `WITH a AS (...), b AS (...)`. |
| Using a CTE purely out of habit for a query that's used once and isn't reused anywhere else | Adds a layer of naming and indirection with no readability or reuse benefit over a plain subquery — covered in depth in the next topic. |

## Interview Questions

1. **Q: What is a Common Table Expression, and what is the basic syntax to define one?**
   A: A CTE is a named, temporary result set defined at the start of a query using `WITH name AS (subquery)`, which the rest of that same statement can then reference by name as if it were a table. It exists only for the duration of that one statement.

2. **Q: Can one CTE in a `WITH` clause reference another CTE defined in the same clause? Are there restrictions on the order?**
   A: Yes — a CTE may reference any CTE defined earlier in the same `WITH` clause (comma-separated definitions are evaluated top to bottom for this purpose), but it cannot reference one defined later. The exception is `WITH RECURSIVE`, where a CTE is specifically allowed to reference itself (Topic 3 of this module).

3. **Q: Is a CTE stored anywhere in the database after its query finishes running?**
   A: No. Unlike a view, a CTE is not a persistent database object — it exists only within the single statement that defines it and disappears once that statement completes.

4. **Q: What's the main readability advantage of a CTE over a deeply nested subquery?**
   A: A CTE lets you name and separate each logical step of a multi-stage query, so the statement can be read top-to-bottom in the same order you'd explain its logic out loud, rather than requiring the reader to unravel a nested expression from the inside out.

## Summary

- A CTE is defined with `WITH name AS (subquery)` and can be referenced, by name, anywhere in the rest of that single statement — in a `FROM`, a `JOIN`, or a subquery.
- Multiple CTEs can be chained in one `WITH` clause, comma-separated, with later ones allowed to reference earlier ones.
- A CTE exists only for the duration of the statement that defines it — it is not a persistent, reusable database object like a view.
- The core benefit is readability and avoiding repeated subquery logic: naming an intermediate result lets a complex query be read as a sequence of clear, purposeful steps.
- Whether naming an intermediate step also changes performance — and it can — is a separate question from readability, covered next in [CTEs vs. Subqueries](02-ctes-vs-subqueries.md).
