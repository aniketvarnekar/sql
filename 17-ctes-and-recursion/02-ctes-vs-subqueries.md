# CTEs vs. Subqueries

## Learning Objectives

By the end of this section you should be able to:
- Decide when a CTE communicates intent more clearly than a nested subquery, especially when the same logic is needed more than once in a query
- Explain how PostgreSQL's CTE materialization behavior changed in version 12, and what "optimization fence" means
- Force a specific materialization strategy with `MATERIALIZED` or `NOT MATERIALIZED`, and explain what each buys you
- Recognize when a plain derived table (a subquery in `FROM`) is simpler and sufficient for a one-off use

## Prerequisites

- **[The WITH Clause](01-the-with-clause.md)** — this topic assumes you're already comfortable with basic CTE syntax; here we examine what the engine actually does with it.
- **Module 11 (Subqueries)**, specifically derived tables (a subquery used directly in a `FROM` clause) — this topic directly contrasts CTEs against that pattern.

## Motivation

It's tempting to assume a CTE, because it looks like it "computes something first and hands it off," must always be at least as efficient as an equivalent nested subquery — or even that it's always *more* efficient, since it sounds like the database is precomputing a saved result. Neither assumption is safely true in general. PostgreSQL's actual behavior here changed materially in a real version boundary (PostgreSQL 12), and knowing exactly what your version does — and how to override it when needed — is the difference between a query that scales fine and one that mysteriously slows down as data grows, for reasons that look like "the CTE" but are really about a specific execution strategy.

## Problem Statement

Consider a query that needs a department's average salary in two different places: once to join every employee against it, and once more, aggregated further, to compare against a company-wide figure. Written with a subquery, that logic has to be either duplicated or awkwardly restructured:

```sql
-- Without a CTE: the department-average aggregate is effectively needed twice
SELECT
    e.name,
    e.department,
    e.salary,
    (SELECT ROUND(AVG(salary), 2) FROM employees e2 WHERE e2.department = e.department) AS dept_avg,
    (SELECT ROUND(AVG(sub.avg_salary), 2)
       FROM (SELECT department, AVG(salary) AS avg_salary FROM employees GROUP BY department) sub
    ) AS company_avg_of_dept_avgs
FROM employees e
ORDER BY e.department, e.salary DESC;
```

The same `department, AVG(salary)` computation is expressed twice, in two different shapes (once as a correlated subquery, once as a derived table), and nothing in the SQL text signals that they're "the same idea, used twice." A reader has to compare the two subqueries line by line to realize that. This is exactly the situation a CTE is built to clean up — but reaching for one also raises a question worth understanding precisely: does naming this computation once actually make the database *compute* it once, or is that just a visual convenience?

## Concept

### The Same Query, Written With a CTE

```sql
WITH dept_stats AS (
    SELECT department, ROUND(AVG(salary), 2) AS avg_salary, COUNT(*) AS headcount
    FROM employees
    GROUP BY department
)
SELECT
    e.name,
    e.department,
    e.salary,
    d.avg_salary,
    ROUND(e.salary / d.avg_salary, 2) AS pct_of_dept_avg,
    (SELECT ROUND(AVG(avg_salary), 2) FROM dept_stats) AS company_avg_of_dept_avgs
FROM employees e
JOIN dept_stats d ON e.department = d.department
ORDER BY e.department, e.salary DESC;
```

`dept_stats` is defined once and referenced **twice** — once in the main `JOIN`, and again inside the scalar subquery that averages the department averages. Using the same `employees` data as the previous topic:

```
 name  | department  | salary | avg_salary | pct_of_dept_avg | company_avg_of_dept_avgs
-------+-------------+--------+------------+-----------------+---------------------------
 Diego | Engineering | 105000 |   96000.00 |            1.09 |                  83722.22
 Ben   | Engineering |  95000 |   96000.00 |            0.99 |                  83722.22
 Elin  | Engineering |  88000 |   96000.00 |            0.92 |                  83722.22
 Farah | Marketing   |  72000 |   70500.00 |            1.02 |                  83722.22
 Gus   | Marketing   |  69000 |   70500.00 |            0.98 |                  83722.22
 Priya | Sales       |  91000 |   84666.67 |            1.07 |                  83722.22
 Asha  | Sales       |  85000 |   84666.67 |            1.00 |                  83722.22
 Chen  | Sales       |  78000 |   84666.67 |            0.92 |                  83722.22
(8 rows)
```

This is unambiguously clearer than the version with duplicated subquery logic: `dept_stats` is defined exactly once, and both places that need it simply say so by name. This — logic reused multiple times in the same query — is the strongest, least debatable case for reaching for a CTE over a subquery.

### Materialization: What Actually Happens Underneath

Naming something clearly and computing it once are, conceptually, two separate questions. PostgreSQL's answer to the second one has changed across versions:

**Before PostgreSQL 12:** a CTE was always **materialized** — the database always fully evaluated the CTE's inner query first, wrote its result into a temporary in-memory (or on-disk, if large) structure, and only then ran the rest of the statement against that saved result. A CTE also acted as an **optimization fence**: the planner was not allowed to push filter conditions from the outer query down into the CTE's definition, even if doing so would have let it avoid computing rows the outer query was just going to discard anyway.

**PostgreSQL 12 and later:** the planner is allowed to **inline** a CTE — fold its definition directly into the surrounding query, exactly as if it had been written as a plain subquery — as long as it's referenced only once, is non-recursive, and contains no data-modifying statement (an `INSERT`/`UPDATE`/`DELETE` inside a CTE is always materialized regardless of version, since running it more than once would actually change data more than once). By default, PostgreSQL 12+ inlines a CTE under those conditions; if a CTE is referenced more than once, or the planner judges materializing it to be cheaper, it may still be materialized.

Consider:

```sql
EXPLAIN
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 50000
)
SELECT * FROM high_earners WHERE department = 'Engineering';
```

On PostgreSQL 12+, with the default behavior, the planner can inline `high_earners` and combine both filters into a single scan (illustrative, simplified plan):

```
Seq Scan on employees  (cost=0.00..1.10 rows=1 width=40)
  Filter: ((salary > 50000) AND (department = 'Engineering'))
```

Both conditions are evaluated together, in one pass over `employees`, exactly as if the CTE had never existed. Now force the old, pre-12 behavior explicitly with `MATERIALIZED`:

```sql
EXPLAIN
WITH high_earners AS MATERIALIZED (
    SELECT * FROM employees WHERE salary > 50000
)
SELECT * FROM high_earners WHERE department = 'Engineering';
```

```
CTE Scan on high_earners  (cost=0.02..0.08 rows=1 width=40)
  Filter: (department = 'Engineering')
  CTE high_earners
    ->  Seq Scan on employees  (cost=0.00..1.10 rows=4 width=40)
          Filter: (salary > 50000)
```

Here the plan has two distinct steps: first build the full `high_earners` result (every row with `salary > 50000`, regardless of department), then scan *that* intermediate result and filter it down by department. On a table with a handful of rows the difference is invisible, but on a table with millions of rows where only a tiny fraction end up in `Engineering`, materializing every high earner first — instead of letting both conditions narrow the scan together — can mean doing dramatically more work than necessary.

### Forcing the Opposite: `NOT MATERIALIZED`

The reverse keyword forces inlining even when the planner might otherwise choose to materialize (typically relevant when a CTE is referenced more than once, where the default leans toward materializing):

```sql
WITH high_earners AS NOT MATERIALIZED (
    SELECT * FROM employees WHERE salary > 50000
)
SELECT * FROM high_earners WHERE department = 'Engineering';
```

This tells the planner: "treat this exactly like a plain subquery — fold it in and let filters push down," even in situations where it might otherwise decide the safer default is to materialize.

### Choosing Deliberately

| Situation | Reasonable choice |
|---|---|
| CTE referenced once, no reason to force behavior | Leave the default (no keyword) — let the planner decide; on PostgreSQL 12+ it will typically inline. |
| CTE referenced multiple times, and it's cheap to recompute | Default is usually fine, or force `NOT MATERIALIZED` if you specifically want filter pushdown into each reference. |
| CTE referenced multiple times, and it's expensive (a large aggregation, a complex join) | Force `MATERIALIZED` so it's computed exactly once and reused, rather than potentially recomputed per reference. |
| CTE wraps a call to a **volatile** function (e.g., one involving `random()` or a sequence via `nextval()`) that must return the *same* value everywhere it's used | Force `MATERIALIZED` — this guarantees single evaluation, which matters for correctness here, not just performance. |

### When a Derived Table Is Still Simpler

None of the above matters if a computation is only ever needed once, in one place, and isn't part of a bigger multi-step chain. In that case, a plain derived table (a subquery directly in `FROM`, as covered in Module 11) is simpler — it says everything the query needs to say without introducing a name that's only ever used one time:

```sql
SELECT department, avg_salary
FROM (
    SELECT department, ROUND(AVG(salary), 2) AS avg_salary
    FROM employees
    GROUP BY department
) AS dept_avg
WHERE avg_salary > 80000;
```

Wrapping this in a `WITH dept_avg AS (...)` instead would be functionally identical but adds a layer of naming ceremony for zero benefit — there's only one reference to it, no reuse, and no larger chain of steps it's building toward. The rule of thumb: reach for a CTE when a computation is reused more than once in the same query, when it's one step in a longer, multi-stage chain worth naming for readability (Topic 1), or when you specifically need recursion (Topic 3, since only a CTE supports `WITH RECURSIVE` — a derived table cannot reference itself). Otherwise, a derived table is the simpler, equally correct choice for a genuine one-off.

## Internal Working (Deep Dive)

The planner's decision to materialize or inline a CTE, when you haven't forced it explicitly, roughly follows this reasoning:

```
 Does the CTE contain a data-modifying statement (INSERT/UPDATE/DELETE)?
   │
   ├── Yes → always materialize (running it more than once would repeat side effects)
   │
   └── No  → Is it a WITH RECURSIVE CTE?
               │
               ├── Yes → always materialize (the recursive evaluation strategy
               │          inherently needs a stable working result per iteration —
               │          see Topic 3)
               │
               └── No  → Is it referenced exactly once in the outer query?
                           │
                           ├── Yes → eligible to inline (PostgreSQL 12+); planner
                           │          estimates cost of inlined vs. materialized
                           │          plan and picks the cheaper one
                           │
                           └── No (referenced 2+ times) → still eligible to inline,
                                    but materializing is often cheaper for the
                                    planner to choose, since inlining means
                                    re-planning (and potentially re-executing)
                                    the definition at every reference site
```

An explicit `MATERIALIZED` or `NOT MATERIALIZED` short-circuits this reasoning entirely and forces the corresponding strategy, regardless of what the planner would otherwise estimate.

## Real-World Analogy

Materializing a CTE is like computing a subtotal once on a piece of paper and setting that paper aside — every time you need the subtotal again, you glance at the paper instead of redoing the arithmetic. Inlining a CTE is like never writing the subtotal down at all, and instead re-deriving exactly the piece of it you need, fresh, at each point it's used — which can actually be *less* work overall if, at each point, you only ever needed a small piece of the full subtotal (the equivalent of a filter being pushed down into just part of the computation) rather than the whole thing every time.

## Why This Design Exists

Before PostgreSQL 12, treating every CTE as an unconditional optimization fence gave a simple, predictable guarantee: a CTE's defining query always ran exactly as its own isolated step, with no surprises from the planner reaching into it. That predictability was valuable in specific cases (forcing a single evaluation of an expensive or side-effecting computation) but came at a real cost for the far more common case — a CTE used purely for readability, where the author never intended it to act as a barrier to optimization at all. PostgreSQL 12's change to allow inlining aligns the default behavior with what most CTEs are actually for (naming a step, not fencing off the planner), while `MATERIALIZED`/`NOT MATERIALIZED` preserve full explicit control for the cases where the old guarantee — or its opposite — is genuinely what you want. This mirrors a theme from [What Is SQL?](../01-introduction/02-what-is-sql.md): you describe *what* you want (a named intermediate result); the planner, by default, is free to decide *how*, unless you explicitly constrain it.

## Advantages

- **Explicit performance control when you need it** — `MATERIALIZED`/`NOT MATERIALIZED` let you pin down exactly how a CTE is evaluated instead of hoping the planner guesses your intent.
- **Guaranteed single evaluation, when forced** — `MATERIALIZED` is the correct tool when a CTE wraps a volatile computation (e.g., `random()`) that must return an identical value everywhere it's referenced.
- **The default (PostgreSQL 12+) usually matches intent with no extra syntax** — for the common case of a CTE used once purely for readability, the planner already optimizes it as if it were a plain subquery, so you get both readability and performance for free.

## Disadvantages / Limitations

- **Version-dependent behavior** — code written assuming PostgreSQL 12+ inlining semantics can behave measurably differently (materializing everything, always) on an older PostgreSQL instance, which matters if you don't control the deployment target.
- **Easy to assume "named" means "optimized"** — a CTE's readability benefit is completely independent of its performance characteristics; conflating the two is one of the most common misunderstandings developers carry from other tools into SQL.
- **Forcing the wrong strategy can hurt** — forcing `NOT MATERIALIZED` on a CTE that's referenced many times and is expensive to compute can cause the underlying computation to effectively be redone at each reference site; forcing `MATERIALIZED` on something trivial and single-use adds pointless overhead.

## Best Practices

- Default to omitting `MATERIALIZED`/`NOT MATERIALIZED` and letting the planner decide, unless you have a specific, measured reason to override it.
- Force `MATERIALIZED` when a CTE contains a volatile function whose value must stay consistent across every reference, or when you've measured (with `EXPLAIN ANALYZE`, covered fully in Module 20) that materializing an expensive, multiply-referenced computation is faster than letting it be re-evaluated per reference.
- Force `NOT MATERIALIZED` when you know the outer query will filter a CTE's result down to a small subset, and you want that filter pushed into the CTE's own scan rather than applied after the CTE is fully computed.
- Never assume performance from syntax alone — always verify with `EXPLAIN` (or `EXPLAIN ANALYZE`) on your actual PostgreSQL version and data volume rather than reasoning about it in the abstract.
- For a computation used exactly once with no reuse, prefer a plain derived table over a CTE — reserve `WITH` for genuine reuse, multi-step readability, or recursion.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a CTE is always faster because it "computes something first" | Whether it's computed once ahead of time (materialized) or folded into the surrounding query (inlined) is a planner decision (or an explicit override) — a CTE carries no inherent performance guarantee either way. |
| Assuming pre-PostgreSQL-12 behavior (always materialized, always an optimization fence) still applies by default on modern PostgreSQL | Since version 12, singly-referenced, non-recursive, non-data-modifying CTEs are eligible for inlining by default — code and mental models built entirely around the old always-materialized behavior can be inaccurate on current versions. |
| Forcing `NOT MATERIALIZED` on a CTE referenced multiple times that wraps a volatile function like `random()` | This can cause the volatile function to be evaluated separately (and differently) at each reference site, silently changing the query's *results*, not just its performance — the opposite of what `MATERIALIZED` exists to guarantee. |
| Reaching for a CTE by default for every single-use calculation | Adds a layer of naming with no reuse or multi-step benefit; a plain derived table communicates the same one-off intent with less ceremony. |

## Interview Questions

1. **Q: What changed about CTE optimization in PostgreSQL 12?**
   A: Before version 12, every CTE was always materialized (fully computed into a temporary result first) and acted as an optimization fence, blocking the planner from pushing outer-query filters into it. From version 12 onward, the planner can inline a CTE — treat it like a plain subquery, allowing filter and join pushdown — by default, as long as it's referenced exactly once, is non-recursive, and contains no data-modifying statement.

2. **Q: When would you deliberately force `MATERIALIZED` even on PostgreSQL 12 or later?**
   A: When the CTE wraps a volatile computation (like `random()` or a sequence call) that must produce the same value at every point it's referenced, or when a CTE is referenced multiple times and you've confirmed computing it once and reusing the result is cheaper than letting the planner potentially re-evaluate it per reference.

3. **Q: Give a concrete scenario where a plain derived table is preferable to a CTE.**
   A: A one-off aggregation used in exactly one place in the query, with no reuse elsewhere and no larger multi-step chain it's building toward — wrapping it in a named `WITH` clause adds indirection without any readability or performance benefit over just nesting it directly in the `FROM` clause.

4. **Q: Does choosing a CTE over an equivalent subquery change a query's result?**
   A: Generally no — the logical result is the same either way; the difference is in how the engine plans and executes the statement. The one caveat is a CTE containing a volatile function, referenced multiple times, with materialization forced off — there, differing evaluation per reference can change the actual output, which is precisely why `MATERIALIZED` exists as an explicit correctness guarantee, not just a performance one.

## Summary

- A CTE reused multiple times in the same query is the clearest, least debatable case for choosing `WITH` over a subquery — it names a computation once instead of duplicating its logic.
- Before PostgreSQL 12, CTEs were always materialized and acted as an unconditional optimization fence; from version 12 onward, the planner can inline a singly-referenced, non-recursive, non-data-modifying CTE by default.
- `MATERIALIZED` forces the CTE to be fully computed once, in isolation; `NOT MATERIALIZED` forces it to be folded into the surrounding query, enabling filter/join pushdown.
- Force `MATERIALIZED` specifically to guarantee single evaluation of a volatile computation, or when you've measured it's faster for an expensive, reused CTE; otherwise, let the planner decide.
- For a genuinely one-off calculation used exactly once, a plain derived table is simpler than a CTE and carries no less clarity.
- The next topic, [Recursive CTEs](03-recursive-ctes.md), covers the one thing only a CTE can do that a subquery fundamentally cannot: reference itself.
