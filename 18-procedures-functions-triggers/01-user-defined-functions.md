# User-Defined Functions

## Learning Objectives

By the end of this section you should be able to:
- Write a `CREATE FUNCTION` statement in PL/pgSQL with typed parameters and a typed return value
- Explain the difference between a scalar-returning function and a set-returning function declared with `RETURNS TABLE`
- Call a user-defined function inside a `SELECT` list, a `WHERE` clause, and as a row source in `FROM`
- Choose correctly between a `SQL`-language function and a `PL/pgSQL`-language function for a given task
- Identify when a function is the right tool versus when the logic belongs somewhere else

## Prerequisites

- [Module 6 — Modifying Data](../06-modifying-data/00-module-overview.md), specifically [INSERT](../06-modifying-data/01-insert.md) and [UPDATE](../06-modifying-data/02-update.md) — the worked examples below read and reason about the same `employees`/`departments` schema first introduced there.
- General query-writing fluency from Modules 7–17 (`SELECT`, joins, `CASE` expressions) — function *bodies* are written using exactly these tools, just packaged behind a name.

## Motivation

Every query-writing habit you've built so far lives inside a single `SELECT` statement, written out in full, every time you need it. The moment two or three different reports need the same piece of logic — "what salary band does this person fall into," "how many years has this employee been here," "give me every employee in a department along with their tenure" — you're faced with a choice: retype (and re-maintain, and risk subtly re-diverging) that logic in every query that needs it, or give it a name once and call it like any other expression. A **user-defined function** is how SQL lets you do the latter: teach the database a new word, and then use that word anywhere an expression is valid, exactly like you'd use `UPPER()` or `COALESCE()`.

## Problem Statement

Suppose several different reports against the `employees` table (schema below) all need to classify an employee's tenure into a human-readable band — "New" (under 1 year), "Established" (1–4 years), or "Veteran" (5+ years) — computed from `hire_date`. Without a function, every single query that needs this classification has to repeat the same `CASE` expression against `CURRENT_DATE - hire_date`:

```sql
SELECT
    name,
    CASE
        WHEN CURRENT_DATE - hire_date < 365 THEN 'New'
        WHEN CURRENT_DATE - hire_date < 365 * 5 THEN 'Established'
        ELSE 'Veteran'
    END AS tenure_band
FROM employees;
```

That's fine once. But now imagine this same `CASE` expression copy-pasted into a payroll report, an HR dashboard query, and an ad hoc analyst query — three separate places someone has to remember to update if the tenure thresholds ever change (say, "Veteran" moves from 5 years to 7). One of those three will eventually be missed, and now different reports silently disagree about who counts as a veteran. A function turns this scattered, re-typed logic into a single named definition, updated in exactly one place.

## Concept

### Anatomy of `CREATE FUNCTION`

```sql
CREATE OR REPLACE FUNCTION function_name(parameter_name parameter_type, ...)
RETURNS return_type
LANGUAGE plpgsql
AS $$
BEGIN
    -- procedural logic here
    RETURN some_value;
END;
$$;
```

Breaking this down piece by piece:

| Piece | Meaning |
|---|---|
| `CREATE OR REPLACE FUNCTION` | Defines a new function, or replaces an existing one with the same name and parameter signature. `OR REPLACE` is a common convention so re-running your definition script during development doesn't require a manual `DROP` first. |
| `function_name(parameter_name parameter_type, ...)` | The function's name and its typed parameter list — just like a table column, every parameter has a declared, enforced data type. |
| `RETURNS return_type` | The type of value the function hands back to its caller — a scalar type (`NUMERIC`, `TEXT`, `BOOLEAN`, ...), a whole row type, or a set of rows (`RETURNS TABLE`, `RETURNS SETOF`, covered below). |
| `LANGUAGE plpgsql` | Which procedural language the function body is written in. `plpgsql` (PostgreSQL's own procedural extension of SQL) is by far the most common choice and the one this module focuses on; a plain `LANGUAGE sql` option also exists, covered below. |
| `AS $$ ... $$` | The function body, delimited by **dollar quoting** (`$$`) instead of single quotes — necessary because the body itself may contain single-quoted string literals, and nesting single quotes inside single quotes would be unreadable and error-prone. |
| `BEGIN ... END;` | Every PL/pgSQL function body is a **block**, opened with `BEGIN` and closed with `END;` — this is the same block structure used by procedures and trigger functions later in this module. |
| `RETURN some_value;` | Hands the value back to the caller and ends execution of the function. |

### A Worked Scalar Function

Using the `employees` table (`id`, `name`, `email`, `department_id`, `salary`, `hire_date`) established in Module 6:

```sql
CREATE OR REPLACE FUNCTION tenure_band(p_hire_date DATE)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_years NUMERIC;
BEGIN
    v_years := EXTRACT(YEAR FROM AGE(CURRENT_DATE, p_hire_date));

    IF v_years < 1 THEN
        RETURN 'New';
    ELSIF v_years < 5 THEN
        RETURN 'Established';
    ELSE
        RETURN 'Veteran';
    END IF;
END;
$$;
```

A few new pieces of syntax here, all standard PL/pgSQL:

- `DECLARE ... BEGIN ... END;` — variables local to the function body are declared once, in a `DECLARE` section, before the executable `BEGIN` block starts. `v_years` is given no explicit type constant here; it takes whatever `EXTRACT(...)` returns.
- `:=` — the assignment operator inside PL/pgSQL (as opposed to `=`, which is comparison).
- `IF ... ELSIF ... ELSE ... END IF;` — PL/pgSQL's conditional branching, functionally identical to the `CASE` expression from the Problem Statement, just written as a procedural statement rather than a single expression.

Once created, the function is called exactly like a built-in one, directly inside a `SELECT` list:

```sql
SELECT name, hire_date, tenure_band(hire_date) AS tenure
FROM employees
ORDER BY hire_date;
```

```
    name     | hire_date  |   tenure
-------------+------------+-------------
 Diego Marin | 2025-03-01 | New
 Chen Wei    | 2022-06-15 | Established
 Ben Ochieng | 2019-11-02 | Veteran
 Asha Rao    | 2017-04-20 | Veteran
(4 rows)
```

Every report that needs this classification now calls `tenure_band(hire_date)` instead of re-typing the `CASE` logic — and if the thresholds ever change, there is exactly one definition to update.

### `RETURNS TABLE` — Set-Returning Functions

A scalar function returns one value per call. Often, though, what you actually want is for a function to return an entire *result set* — multiple rows and columns — that can be queried like a table. This is what `RETURNS TABLE` is for:

```sql
CREATE OR REPLACE FUNCTION employees_in_department(p_department_name TEXT)
RETURNS TABLE (
    employee_name TEXT,
    salary        NUMERIC,
    tenure        TEXT
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT
        e.name,
        e.salary,
        tenure_band(e.hire_date)
    FROM employees e
    JOIN departments d ON d.id = e.department_id
    WHERE d.name = p_department_name
    ORDER BY e.salary DESC;
END;
$$;
```

- `RETURNS TABLE (column_name column_type, ...)` declares the shape of the result set — like a miniature table definition attached to the function.
- `RETURN QUERY` is the PL/pgSQL statement used inside a set-returning function: instead of returning one value, it runs a full `SELECT` and streams every row of its result back to the caller.

This function is called in the `FROM` clause, exactly like a table:

```sql
SELECT * FROM employees_in_department('Engineering');
```

```
 employee_name | salary | tenure
---------------+--------+-------------
 Ben Ochieng   |  88000 | Veteran
 Chen Wei      |  76000 | Established
(2 rows)
```

Notice this function reused `tenure_band` from the previous example inside its own body — functions can freely call other functions, exactly like ordinary expressions can be nested inside larger expressions.

### `SQL`-Language Functions

Not every function needs PL/pgSQL's procedural control flow (`IF`, loops, variables). When a function's body is a single query, PostgreSQL also supports plain `LANGUAGE sql`:

```sql
CREATE OR REPLACE FUNCTION average_department_salary(p_department_id INTEGER)
RETURNS NUMERIC
LANGUAGE sql
AS $$
    SELECT AVG(salary) FROM employees WHERE department_id = p_department_id;
$$;
```

There is no `BEGIN`/`END`, no `RETURN` keyword, and no dollar-quoted procedural block — the body is literally the query itself, and its result *is* the return value. Prefer `LANGUAGE sql` whenever a function is a simple, single-query wrapper; reach for `LANGUAGE plpgsql` only once you actually need branching, loops, or local variables, as in `tenure_band` above.

### Function Volatility

PostgreSQL lets (and expects) you to declare how "predictable" a function is, which directly affects how aggressively the query planner can optimize around it:

| Volatility | Meaning | Example |
|---|---|---|
| `IMMUTABLE` | Given the same arguments, always returns the same result, forever, with no dependency on table data. | A pure math function like `fahrenheit_to_celsius(temp)`. |
| `STABLE` | Given the same arguments, returns the same result *within a single statement/scan*, but may depend on table data (which could change between statements). | `average_department_salary` above — it reads from `employees`, so it isn't `IMMUTABLE`, but within one query it won't change. |
| `VOLATILE` (the default if unspecified) | May return a different result on every single call, even with identical arguments within the same statement — e.g., it uses `random()`, `now()`, or modifies data. | A function that inserts a row and returns a generated ID. |

```sql
CREATE OR REPLACE FUNCTION average_department_salary(p_department_id INTEGER)
RETURNS NUMERIC
LANGUAGE sql
STABLE
AS $$
    SELECT AVG(salary) FROM employees WHERE department_id = p_department_id;
$$;
```

Declaring volatility correctly isn't optional decoration — it's covered in depth in the Internal Working section below, because it changes what the planner is allowed to do with calls to your function.

## Internal Working (Preview)

When PostgreSQL first sees a call to a PL/pgSQL function within a session, it compiles the function body once (parsing the procedural block into an internal representation) and caches that compiled form for the rest of the session — subsequent calls in the same session skip re-parsing the body. Each individual SQL statement *inside* the function body (like the `SELECT` in `tenure_band`, or the `RETURN QUERY` in `employees_in_department`) is planned and executed by the exact same query planner that handles any other query you write — a function body is not a separate, lesser execution path; it's ordinary SQL, just invoked from inside a procedural wrapper.

Volatility matters here because of what the planner is willing to do with a function call inside a larger query:

```
 Query calls your_function(col) inside WHERE or SELECT
                     │
                     ▼
        Is your_function IMMUTABLE?
          │                    │
         yes                   no
          │                    │
          ▼                    ▼
 Planner may fold a call   Planner must re-evaluate
 with constant arguments   the function for every row
 once, or even use it in   it's referenced against —
 an index expression.      no shortcuts assumed.
```

Marking a function `IMMUTABLE` when it secretly depends on table data (or the wall clock) is a real correctness bug, not just a missed optimization — the planner may reuse a cached result from an earlier call when it shouldn't, silently returning stale answers. This is exactly why `average_department_salary` above is `STABLE`, not `IMMUTABLE`: its answer depends on the current contents of `employees`, and a future call in a different query needs to see current data.

## Real-World Analogy

A user-defined function is like a labeled, standardized form at a government office — say, a fixed formula for calculating a tax refund from a few input fields. Anyone who needs that calculation done — a clerk, a different office branch, an online portal — fills in the same inputs and gets the same output, computed by the same fixed rule, without needing to understand or re-derive the underlying formula themselves. If the formula changes (a new tax bracket is introduced), it's updated in exactly one place — the official form definition — and every office that uses the form immediately gets the corrected behavior, rather than each office having hand-copied the old formula into its own paperwork and now needing to be tracked down individually.

## Why User-Defined Functions Were Designed This Way

SQL's core strength, established all the way back in Module 1's discussion of the *declarative* nature of the language, is that you describe *what* result you want, and the database figures out *how* to produce it. A user-defined function extends this same declarative philosophy: it lets you extend SQL's own vocabulary, so that `tenure_band(hire_date)` reads and behaves exactly like a built-in expression such as `UPPER(name)` — you're not describing a separate procedural side-quest that the query has to step outside itself to run; you're teaching the query language one new word, which can be composed, nested, and optimized by the same planner as everything else. This is precisely why functions can appear *inside* `SELECT`, `WHERE`, `ORDER BY` — anywhere an expression is valid — rather than being a standalone command you run on its own, the way a stored procedure (next topic) is.

## Advantages

- **Single source of truth.** Business logic like tenure banding or bonus calculation lives in exactly one place, callable from any query, instead of being re-typed (and eventually re-diverging) across many.
- **Composable like any other expression.** A function call can be nested inside larger expressions, joined against, filtered on, and combined with other functions — because syntactically it *is* just an expression.
- **Reduces client-server round trips.** A calculation done inside a function runs where the data already lives, avoiding pulling raw rows across the network just to compute something in application code and (often) send a follow-up query back.
- **Consistent enforcement of derived values.** If every report must compute a value the same way, a function guarantees that — no report can "forget" a branch of the logic the way a copy-pasted `CASE` expression eventually might after being edited in only one location.

## Disadvantages / Limitations

- **Harder to test and debug in isolation.** A function is version-controlled and tested very differently from an ordinary piece of code — there's no simple "run it with these inputs and print the output" workflow unless you build tooling for exactly that around your database.
- **Vendor-specific syntax.** `PL/pgSQL` is PostgreSQL-specific; the same logic written for SQL Server (T-SQL) or Oracle (PL/SQL) uses different syntax entirely (Module 22 — SQL Across Databases covers where these diverge). Logic placed in a function is far less portable than logic placed in a typical application layer written in a general-purpose language.
- **Can obscure where logic actually lives.** If a report's number looks wrong, a function call buried inside a `SELECT` list is easy to overlook while debugging — someone unfamiliar with the schema may not even know a function is involved unless they think to ask.
- **Wrong volatility silently breaks correctness or performance.** As shown in Internal Working, marking a function `IMMUTABLE` when it isn't can cause the planner to return stale results; marking a genuinely constant function `VOLATILE` needlessly prevents optimization.

## Best Practices

- Declare the most restrictive accurate volatility (`IMMUTABLE` > `STABLE` > `VOLATILE`, in order of how much the planner can assume) — never guess; if a function reads a table, it is at best `STABLE`, never `IMMUTABLE`.
- Prefer `LANGUAGE sql` for simple single-query wrappers, and reserve `LANGUAGE plpgsql` for logic that genuinely needs branching, loops, or local variables — a simpler function is easier to read, test, and reason about.
- Give functions verb-first or clearly descriptive names (`tenure_band`, `average_department_salary`) so a call site reads like a sentence, the same naming discipline you'd apply to a well-named column or table.
- Keep functions narrowly scoped to one calculation or one lookup — a function that tries to do five unrelated things is exactly as hard to maintain as a giant, unfocused query would be.
- Document parameters and return shape directly above the `CREATE FUNCTION` statement in your source-controlled schema scripts — a function's signature alone rarely explains *why* the logic exists.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Forgetting `RETURN` (or `RETURN QUERY` for a `RETURNS TABLE` function) on every code path | PL/pgSQL requires an explicit `RETURN` in scalar functions; a code path that falls through without one raises a runtime error ("control reached end of function without RETURN"). |
| Marking a function that reads a table as `IMMUTABLE` | As covered in Internal Working, this can cause the planner to reuse a stale cached result — a correctness bug, not just an optimization miss. Only mark a function `IMMUTABLE` if its result depends *purely* on its arguments. |
| Using single quotes instead of dollar quoting (`$$`) for the function body | A body containing its own single-quoted string literals becomes an unreadable mess of escaped quotes; dollar quoting sidesteps this entirely and is the near-universal convention. |
| Treating a `RETURNS TABLE` function like a scalar value (e.g., trying to assign its result directly to a variable) | A set-returning function must be queried with `SELECT ... FROM function_name(...)`, treating it as a row source, not called like `variable := function_name(...)`. |

## Interview Questions

1. **Q: What is the difference between `RETURNS NUMERIC` and `RETURNS TABLE (...)` on a function?**
   A: `RETURNS NUMERIC` (or any other scalar type) means the function produces exactly one value per call, usable anywhere a single expression is valid. `RETURNS TABLE (column_name type, ...)` means the function produces a full result set of zero, one, or many rows, and must be queried as a row source (`SELECT * FROM function_name(...)`) rather than treated as a single value.

2. **Q: Why does PostgreSQL let you choose between `LANGUAGE sql` and `LANGUAGE plpgsql` for a function body?**
   A: Many functions are simply a single query wrapped in a name — `LANGUAGE sql` handles that case directly with no extra ceremony. `LANGUAGE plpgsql` exists for logic that needs procedural control flow — variables, `IF`/`ELSIF` branching, loops — which a single SQL statement can't express on its own.

3. **Q: What does marking a function `STABLE` communicate to the query planner, and how does it differ from `IMMUTABLE`?**
   A: `STABLE` tells the planner the function will return the same result for the same arguments *within a single statement*, but its result can depend on table data that might change between statements — so it's safe to avoid re-evaluating it redundantly within one query, but not safe to cache indefinitely or use in certain index expressions. `IMMUTABLE` is a stronger promise: the result depends purely on the arguments, never on table data or the current time, so the planner can fold or cache it far more aggressively, including inside index expressions.

4. **Q: If you need to run five separate `INSERT` statements as one unit of work, with the ability to `COMMIT` partway through, should you write a function?**
   A: No — a function executes inside the transaction of whatever statement called it and cannot issue its own `COMMIT` or `ROLLBACK`. That capability belongs to a stored procedure, covered in the next topic.

## Summary

- A user-defined function packages reusable logic behind a name, callable anywhere an expression is valid — in a `SELECT` list, a `WHERE` clause, or as a row source in `FROM` when declared `RETURNS TABLE`.
- `LANGUAGE sql` functions are single-query wrappers; `LANGUAGE plpgsql` functions add procedural control flow (`DECLARE`, `IF`/`ELSIF`, local variables) for logic a single query can't express.
- `RETURNS TABLE` combined with `RETURN QUERY` lets a function return an entire result set, queryable like a table.
- Volatility (`IMMUTABLE`/`STABLE`/`VOLATILE`) tells the planner how safely it can optimize around a function call — getting it wrong is a correctness risk, not just a missed performance opportunity.
- A function is the right tool exactly when logic is reusable, side-effect-free (or nearly so), and needs to be composed inside a larger query as an expression — once you need to manage a transaction's boundaries yourself, you need a procedure instead.
