# Recursive CTEs

## Learning Objectives

By the end of this section you should be able to:
- Write a `WITH RECURSIVE` query, correctly identifying its anchor member and recursive member
- Explain why `UNION ALL` is the typical choice inside a recursive CTE, and when `UNION` is chosen instead, and at what cost
- Traverse a hierarchy (an org chart) both downward (all reports under a manager) and upward (the chain of command above an employee)
- Traverse a graph that may contain cycles (a small route network) to find all reachable destinations and their paths
- Add explicit guards — a cycle check, a depth cap, or both — to prevent a recursive CTE from running forever

## Prerequisites

- **[The WITH Clause](01-the-with-clause.md)** and **[CTEs vs. Subqueries](02-ctes-vs-subqueries.md)** — you need the basic `WITH` syntax and an understanding that a CTE is a named result set before extending it to the recursive form.
- **Module 10 (Joins & Set Operations)** — every recursive CTE in this topic joins its recursive member back to a base table, using the same explicit `JOIN ... ON` style taught in [Inner Join](../10-joins-and-set-operations/01-inner-join.md).
- **Module 05 (Constraints & Keys)**, specifically a foreign key that references its own table (a self-referencing foreign key), as covered in [Foreign Keys and Referential Integrity](../05-constraints-and-keys/04-foreign-keys-and-referential-integrity.md) — this is exactly how the org-chart example below models "each employee has a manager who is also an employee."

## Motivation

Hierarchical and networked data is everywhere in real systems: an org chart (who reports to whom, arbitrarily many levels deep), a product category tree (categories containing subcategories containing subcategories), a bill of materials (a part made of parts, each possibly made of further parts), or a route network (which cities connect to which others, and how). None of these have a fixed, known depth ahead of time — a company might have three management levels or nine; a category tree might be two levels deep in one place and six in another. A plain `JOIN` connects exactly one level of a relationship at a time; asking "however many levels deep this goes" requires *repeating* that join an unknown number of times. `WITH RECURSIVE` is SQL's answer to exactly this: a CTE that is allowed to refer to its own result, evaluated repeatedly, until there's nothing new left to find.

## Problem Statement

Take an `employees` table where each row stores its own manager's `id` in a `manager_id` column — a self-referencing foreign key:

```sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    title TEXT NOT NULL,
    manager_id INTEGER REFERENCES employees(id)
);

INSERT INTO employees (id, name, title, manager_id) VALUES
    (1, 'Morgan', 'CEO',                   NULL),
    (2, 'Priya',  'VP Engineering',        1),
    (3, 'Sana',   'VP Sales',              1),
    (4, 'Ben',    'Engineering Manager',   2),
    (5, 'Farah',  'Sales Manager',         3),
    (6, 'Diego',  'Engineer',              4),
    (7, 'Elin',   'Engineer',              4),
    (8, 'Gus',    'Sales Rep',             5);
```

Now ask: "Give me everyone who reports up to Priya (id 2), directly or transitively, at any depth." A single `JOIN` gets you Priya's *direct* reports (Ben) but not Ben's reports (Diego and Elin) — those are two joins deep. If the org chart had a tenth level, you'd need a tenth join. You don't know how many levels exist ahead of time, and hand-writing a fixed number of joins is both fragile (breaks the moment the hierarchy grows one level deeper) and ugly. This is precisely the gap `WITH RECURSIVE` fills.

## Concept

### Syntax Structure

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor member: runs once, produces the starting row(s)
    SELECT ...
    FROM some_table
    WHERE ...

    UNION ALL

    -- Recursive member: references cte_name itself; runs repeatedly
    SELECT ...
    FROM some_table t
    JOIN cte_name c ON ...
)
SELECT * FROM cte_name;
```

A recursive CTE has exactly two parts, combined with `UNION` or `UNION ALL`:

- The **anchor member** — an ordinary, non-recursive query that runs exactly once and seeds the starting row(s). It must not reference `cte_name` itself.
- The **recursive member** — a query that *does* reference `cte_name`, joining it against a base table to find "the next step outward" from whatever the previous iteration found. It runs repeatedly: each time, `cte_name` (as referenced inside the recursive member) refers only to the rows produced by the *previous* iteration, not the entire accumulated result so far. Iteration stops the moment the recursive member produces zero new rows.

### UNION ALL vs. UNION

`UNION ALL` is the typical choice, and for a good reason: it does no duplicate-elimination — every row produced by every iteration is kept, with no comparison against previously produced rows. If the underlying relationship is a genuine tree or DAG with no cycles (an employee cannot be their own transitive manager, a category cannot contain itself), no true duplicate rows will ever be produced anyway, so paying the cost of deduplication buys you nothing.

`UNION`, by contrast, performs a deduplication pass after every iteration, comparing each newly produced row against the entire result accumulated so far, and discarding exact duplicates. This is more expensive — but it provides a form of automatic cycle protection: if the underlying data actually contains a cycle (as a general graph might, even though a management hierarchy shouldn't), `UNION` guarantees a row that has already been produced won't be re-added, which prevents the *same exact row* from being recomputed forever. The important caveat: `UNION`'s protection only catches **exact duplicate rows**. The moment a recursive member accumulates any per-row state that differs between visits to the same node — like a growing path or running total, as in the graph example below — two visits to the same node no longer produce identical rows, so `UNION`'s deduplication does nothing to stop a cycle, and an explicit guard is required regardless. This is why the general-purpose, reliable technique for cycle safety is an explicit guard in the recursive member (shown below), not relying on plain `UNION`.

### Worked Example 1 — Hierarchy, Traversing Downward

Find every employee who reports, directly or transitively, up to Priya (id 2):

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: start at Priya herself
    SELECT id, name, title, manager_id, 1 AS depth
    FROM employees
    WHERE id = 2

    UNION ALL

    -- Recursive: find employees whose manager_id matches someone already found
    SELECT e.id, e.name, e.title, e.manager_id, oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT id, name, title, depth
FROM org_chart
ORDER BY depth, id;
```

Result:

```
 id | name  |        title         | depth
----+-------+-----------------------+-------
  2 | Priya | VP Engineering        |     1
  4 | Ben   | Engineering Manager   |     2
  6 | Diego | Engineer              |     3
  7 | Elin  | Engineer              |     3
(4 rows)
```

Tracing the iterations by hand makes the mechanics concrete:

```
Iteration 1 (anchor):     { Priya (id 2), depth 1 }
Iteration 2 (recursive):  employees where manager_id = 2  →  { Ben (id 4), depth 2 }
Iteration 3 (recursive):  employees where manager_id = 4  →  { Diego (id 6), Elin (id 7), depth 3 }
Iteration 4 (recursive):  employees where manager_id IN (6, 7)  →  (no rows)  →  STOP
```

Recursion stops the moment an iteration finds nothing new — nobody reports to Diego or Elin, so iteration 4 produces zero rows, and the final result is the union of every row produced across iterations 1 through 3.

### Worked Example 2 — Hierarchy, Traversing Upward

Now the reverse direction: given Diego (id 6), find his full chain of command up to the CEO.

```sql
WITH RECURSIVE chain_of_command AS (
    -- Anchor: start at Diego himself
    SELECT id, name, title, manager_id, 1 AS level
    FROM employees
    WHERE id = 6

    UNION ALL

    -- Recursive: find the manager of whoever was just found
    SELECT e.id, e.name, e.title, e.manager_id, cc.level + 1
    FROM employees e
    JOIN chain_of_command cc ON e.id = cc.manager_id
)
SELECT id, name, title, level
FROM chain_of_command
ORDER BY level;
```

Result:

```
 id | name  |        title        | level
----+-------+----------------------+-------
  6 | Diego | Engineer             |     1
  4 | Ben   | Engineering Manager  |     2
  2 | Priya | VP Engineering       |     3
  1 | Morgan| CEO                  |     4
(4 rows)
```

The only structural difference from Example 1 is the join direction: `e.id = cc.manager_id` (find the row whose `id` matches the previous row's manager) instead of `e.manager_id = cc.id` (find rows whose manager is the previous row). Recursion stops after level 4 because Morgan's `manager_id` is `NULL`, and no employee's `id` can ever equal `NULL` — so the recursive member's join produces zero rows on the next attempt, and iteration halts.

### Worked Example 3 — Graph Traversal With a Cycle

Hierarchies are trees — no cycles possible by construction. Many real networks aren't trees at all. Consider a small directed route network between cities, where a return route creates an actual cycle:

```sql
CREATE TABLE routes (
    origin      TEXT NOT NULL,
    destination TEXT NOT NULL,
    distance    INTEGER NOT NULL
);

INSERT INTO routes (origin, destination, distance) VALUES
    ('NYC', 'CHI', 800),
    ('CHI', 'DEN', 900),
    ('CHI', 'LAX', 2000),
    ('DEN', 'SEA', 1300),
    ('LAX', 'SEA', 1100),
    ('SEA', 'NYC', 2400);
```

Notice `SEA -> NYC` closes a loop back to the start — `NYC -> CHI -> DEN -> SEA -> NYC -> CHI -> ...` could continue forever if nothing stops it. The goal: find every city reachable from `NYC`, the path taken to reach it, and the total distance — without looping forever on the cycle.

```sql
WITH RECURSIVE reachable AS (
    -- Anchor: every route leaving NYC directly
    SELECT
        origin,
        destination,
        distance AS total_distance,
        ARRAY[origin, destination] AS path
    FROM routes
    WHERE origin = 'NYC'

    UNION ALL

    -- Recursive: extend a known path by one more route, unless that would revisit a city already in the path
    SELECT
        r.origin,
        r.destination,
        reach.total_distance + r.distance,
        reach.path || r.destination
    FROM routes r
    JOIN reachable reach ON r.origin = reach.destination
    WHERE NOT (r.destination = ANY(reach.path))
)
SELECT destination, total_distance, path
FROM reachable
ORDER BY total_distance;
```

Result:

```
 destination | total_distance |         path
-------------+-----------------+------------------------
 CHI         |             800 | {NYC,CHI}
 DEN         |            1700 | {NYC,CHI,DEN}
 LAX         |            2800 | {NYC,CHI,LAX}
 SEA         |            3000 | {NYC,CHI,DEN,SEA}
 SEA         |            3900 | {NYC,CHI,LAX,SEA}
(5 rows)
```

The `path` column (a PostgreSQL array, built up with `||`, array concatenation) accumulates every city visited so far on that particular route. The guard `WHERE NOT (r.destination = ANY(reach.path))` is what actually breaks the cycle: once a path has reached `SEA`, the route `SEA -> NYC` is a candidate for the next iteration, but `NYC` is already present in that path's array, so the guard excludes it before it's ever added — recursion simply stops extending that branch instead of looping back to `NYC` and starting the whole traversal over again. Note that two different, equally valid paths reach `SEA` (`NYC -> CHI -> DEN -> SEA` and `NYC -> CHI -> LAX -> SEA`) — a recursive CTE finds every distinct path, not just one, which is exactly what makes the cycle guard (rather than plain `UNION`'s whole-row deduplication) the right tool here: these two `SEA` rows are not duplicates of each other at all, so `UNION` would never have collapsed them, and it certainly wouldn't have stopped the `SEA -> NYC` cycle either, since a row with a longer, different path array is never an exact duplicate of an earlier row.

### Guarding Against Infinite Recursion

Two complementary techniques, and when to use each:

- **Cycle guard (shown above)** — track visited nodes in an accumulating array (or similar structure) and exclude any candidate that would revisit one. This is the general-purpose technique for any graph that isn't provably acyclic, and it's a correctness guard, not just a performance one — without it, Example 3 would never terminate.
- **Depth/hop cap, as a safety net** — add a counter column and stop extending past a fixed limit, even in structures that are *supposed* to be acyclic (like an org chart), as insurance against bad data (a manager_id that accidentally points to a subordinate, for instance) or a bug in the query itself:

```sql
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees
    WHERE id = 2

    UNION ALL

    SELECT e.id, e.name, e.manager_id, oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
    WHERE oc.depth < 50   -- safety cap: no real org chart should be this deep
)
SELECT * FROM org_chart;
```

PostgreSQL does not impose an automatic limit on how many iterations a recursive CTE may run — left unguarded against genuinely cyclic data, it will keep recursing, consuming increasing memory and time, until it either exhausts available resources, hits a configured `statement_timeout` (Module 14 and Module 20 cover session-level safeguards like this), or is manually canceled. There is no substitute for testing a new recursive query against a small, known dataset first and confirming it terminates and produces the rows you expect, before ever running it against full-scale data.

## Internal Working (Deep Dive)

PostgreSQL evaluates a recursive CTE via an iterative, fixed-point algorithm, not by making repeated function-style calls the way recursion works in a general-purpose programming language:

```
 1. Evaluate the anchor member.
    → Its rows become the initial "working table," and are also appended
      to the final accumulated result.

 2. Repeat:
      a. Evaluate the recursive member, but with `cte_name` (as referenced
         inside it) bound ONLY to the current working table — the rows
         produced by the previous iteration, not the entire result so far.
      b. The rows this produces become the NEW working table, and are
         appended to the final accumulated result.
      c. If step (a) produced zero rows, stop.

 3. The final result is every row accumulated across all iterations.
```

For Worked Example 1, this looks like:

```
 working table (iter 1): { Priya }
 working table (iter 2): { Ben }              (found via Priya)
 working table (iter 3): { Diego, Elin }      (found via Ben)
 working table (iter 4): {}  (empty)          →  STOP
 final result: union of iterations 1, 2, and 3
```

This is exactly why `UNION ALL` is efficient: each iteration only ever needs to look at the *previous* iteration's new rows to compute the *next* iteration's new rows — it never needs to re-scan or compare against the entire accumulated result so far, the way `UNION`'s deduplication does. The process closely resembles a breadth-first traversal, expanding one "frontier" of rows at a time until a frontier comes back empty.

## Real-World Analogy

A recursive CTE behaves like an emergency phone tree: the anchor member is the single person who receives the very first call. In round one, that person calls everyone who reports directly to them. In round two, *each* of those people calls everyone who reports to *them* — but only the people reached in round one make calls in round two; someone who was already called in round one doesn't get re-called or re-triggered to call again in round three. The tree keeps expanding, one round at a time, using only the newest round's people to trigger the next round, until a round produces nobody new to call — at which point the whole notification process is naturally complete.

## Why Recursive CTEs Were Designed This Way

SQL's relational model (established in [What Is a Database and a DBMS?](../01-introduction/01-what-is-a-database-and-a-dbms.md)) is fundamentally set-based and declarative — there is no native "loop" construct for an ordinary query, the way a general-purpose language has a `for` or `while` loop. `WITH RECURSIVE` doesn't quietly break that model by smuggling in a procedural loop; instead, it's defined in terms of **fixed-point iteration** over sets — repeatedly applying the recursive member to the newest set of rows until applying it again adds nothing new — which is a set-based, declarative notion of recursion drawn from relational query theory, not a call-stack notion borrowed from procedural programming. This is also precisely why the recursive member has real restrictions (no aggregate or window functions over the recursive reference, no `DISTINCT`, `GROUP BY`, or `ORDER BY`/`LIMIT` inside the recursive term itself, and the recursive CTE may be referenced at most once, only as a direct table reference): each iteration must remain a straightforward, well-defined set-producing step for the fixed-point process to have clear, predictable termination behavior.

## Advantages

- **Expresses arbitrary, unknown depth in pure SQL** — an org chart nine levels deep and one three levels deep are handled by the identical query, with no code change, unlike hand-written joins which would need a different number of joins for each.
- **Composable with the rest of SQL** — the final result of a recursive CTE is just a normal result set: it can be filtered, joined, or aggregated like any other query source, as shown by simply adding `ORDER BY` at the end of every example above.
- **A single, atomic statement** — the entire traversal happens within one SQL statement, without needing external procedural code to loop and issue repeated queries.

## Disadvantages / Limitations

- **Genuinely harder to read** — tracing what a recursive CTE actually does requires mentally simulating iterations, which is a materially higher bar than reading a plain `SELECT`; this is reflected in how often it's specifically tested in interviews.
- **Real risk of runaway execution** — without a cycle guard or depth cap, recursion over genuinely cyclic or malformed data can run indefinitely, consuming growing memory, until a timeout or manual cancellation intervenes.
- **Performance can degrade on deep or wide structures** — each iteration is effectively a fresh join pass over the working table; a graph with a very large branching factor or very deep chains can mean substantial cumulative work across many iterations.
- **Dialect support varies** — `WITH RECURSIVE` syntax and behavior differ somewhat across database vendors (for example, some databases historically lacked recursive CTE support entirely, or added it much later than PostgreSQL); Module 22 (SQL Across Databases) covers these gaps for anyone working across multiple database products.

## Best Practices

- Always include a cycle guard (an accumulating visited-nodes array, as in Example 3) for any hierarchy or graph that isn't *provably* acyclic by its own structure — don't assume "it's supposed to be a tree" is the same as "it's guaranteed to be a tree" in real data.
- Add a depth cap as a cheap safety net even on structures you believe are acyclic (like an org chart), since it protects you against a single bad row (a `manager_id` pointing the wrong way) without changing correct behavior on good data.
- Test every new recursive query against a small, fully-understood dataset first, and manually verify the result against what you'd expect, before pointing it at production-scale hierarchical or graph data.
- Prefer `UNION ALL` with an explicit cycle guard over plain `UNION` whenever the recursive member accumulates any per-row state (a path, a running total, a depth) — `UNION`'s whole-row deduplication cannot recognize "same node, different path" as something needing protection, so it silently fails to prevent the kind of cycle that matters most in graph traversals.
- Name the accumulating columns (`depth`, `level`, `path`, `total_distance`) descriptively — they carry the entire meaning of what a given iteration represents.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using plain `UNION` and assuming it fully protects against infinite recursion | `UNION` only discards exact duplicate rows; the moment the recursive member accumulates any per-row state (a path, a running total) two visits to the same node are no longer identical rows, so `UNION` does nothing to stop the cycle — an explicit guard is still required. |
| Having the anchor member reference the CTE's own name | The anchor member must be a plain, non-recursive query; only the recursive member is allowed to reference the CTE itself. |
| Trying to use an aggregate function, `DISTINCT`, or `ORDER BY`/`LIMIT` directly inside the recursive member | PostgreSQL disallows these inside the recursive term itself (the final, outer `SELECT` over the finished CTE may still use them freely) — the recursive member must remain a simple, well-defined row-producing step for each iteration. |
| Omitting any cycle guard or depth cap on data that isn't provably acyclic | Leads to a query that may never terminate on its own, consuming increasing memory until a timeout or manual cancellation — always guard graph traversals explicitly, as shown in Example 3. |
| Confusing `WITH RECURSIVE`'s set-based, fixed-point recursion with function-call recursion in a general-purpose language | They're different mechanisms solving different problems; genuine recursive function calls within the database are a topic for stored procedures and functions (Module 18), not for `WITH RECURSIVE`, which is fundamentally an iterative, set-at-a-time process. |

## Interview Questions

1. **Q: What are the two parts of a `WITH RECURSIVE` CTE, and what does each do?**
   A: The anchor member is a plain, non-recursive query that runs exactly once and seeds the starting rows. The recursive member references the CTE itself, joined against a base table, and runs repeatedly — each time using only the previous iteration's rows — until it produces zero new rows, at which point recursion stops. The two are combined with `UNION` or `UNION ALL`.

2. **Q: Why is `UNION ALL` typically preferred over `UNION` inside a recursive CTE?**
   A: `UNION ALL` skips deduplication entirely, which is cheaper, and is safe whenever the underlying structure can't produce true duplicate rows (a tree or DAG). `UNION` adds a deduplication pass that compares each new row against the entire accumulated result, which is more expensive, and only meaningfully protects against cycles when a repeated visit to the same node would produce an exact duplicate row — which stops being true the moment any per-row state, like a path, is accumulated.

3. **Q: How would you find all descendants of a node in a tree using a recursive CTE, and how would that differ from finding all its ancestors?**
   A: For descendants (downward), anchor on the starting node and have the recursive member join child rows where `child.parent_id = cte.id`. For ancestors (upward), anchor on the starting node and have the recursive member join the parent row where `parent.id = cte.parent_id`. The only structural difference is which side of the join condition references the CTE.

4. **Q: How do you prevent infinite recursion when traversing data that may contain cycles?**
   A: Track visited nodes in an accumulating column (commonly an array built with `path || next_node`), and add a `WHERE NOT (candidate = ANY(path))` guard in the recursive member to exclude any node already present in that row's path. As an additional safety net, a depth counter with a hard cap (`WHERE depth < N`) protects against runaway recursion even on data believed to be acyclic.

## Summary

- `WITH RECURSIVE` has an anchor member (runs once, seeds the starting rows) and a recursive member (references the CTE itself, runs repeatedly against the previous iteration's rows, stops when it produces nothing new).
- `UNION ALL` is the typical, efficient choice; plain `UNION` trades performance for automatic protection against exact-duplicate-row cycles, which doesn't help once per-row state like a path is being accumulated.
- Downward hierarchy traversal joins children to parents already found; upward traversal joins parents to children already found — the direction of the join condition is the only structural difference.
- Graph traversal over data that may contain cycles requires an explicit guard (typically an accumulating visited-nodes array) — this is a correctness requirement, not an optional optimization.
- PostgreSQL will not automatically stop a runaway recursive query on genuinely cyclic, unguarded data — always test against small, known data first, and add a depth cap as a cheap safety net even on data you believe is acyclic.
- This closes out the module: the next file, [Module Summary](04-module-summary.md), consolidates everything covered across `WITH`, materialization, and recursion.
