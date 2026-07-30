# Core Conceptual Questions — Ranked & Synthesized

## Learning Objectives

- Recall, from memory and in your own words, the highest-signal SQL conceptual questions across the entire course
- Answer each in interview-appropriate form: a tight, complete answer in 20-45 seconds, not a five-minute lecture
- Know exactly which module to return to for full depth if an interviewer pushes further

## Motivation

Across 22 modules you've built a large, coherent mental model of relational databases. In a real interview, that model isn't tested by asking you to recite it end to end — it's tested by rapid-fire questions that jump between themes, and by an interviewer who is listening for whether you understand *why*, not just *what*. Some questions come up in almost every SQL interview regardless of company or seniority level (joins vs. subqueries, primary vs. foreign keys, `WHERE` vs. `HAVING`, transactions and isolation); others are rarer, deeper cuts that separate a strong candidate from an average one. This topic re-organizes the course's highest-signal questions **by theme**, roughly in the order interviewers actually reach for them, so your review time goes to what matters most first.

**How to use this list:** cover the answer, say the question out loud, answer it out loud in your own words in under a minute, *then* check the model answer. Reading the answer first feels productive but builds almost no real interview recall.

## Theme: The Relational Model & SQL Fundamentals

**Q1. What makes a database "relational," and where does that idea actually come from?**
> A relational database organizes data into relations (tables) — sets of tuples (rows) with named attributes (columns) — and relates data across tables purely through matching attribute values, not through physical pointers. The theoretical foundation is Codd's relational model and his twelve rules for what a system must do to genuinely qualify as relational, including things like guaranteed access to every value via table name, primary key, and column name. (Module 02, The Relational Model)

**Q2. Why is SQL called a declarative language, and how is that different from writing a loop yourself?**
> A declarative query states *what* result you want (`SELECT` these columns `WHERE` this condition holds) and leaves *how* to compute it — join order, algorithm choice, whether to use an index — entirely to the query planner. An imperative loop specifies the exact steps to take. This is why the same SQL query can get faster over time as the optimizer improves, with zero changes to the query itself. (Module 01, What Is SQL?)

**Q3. What are the five categories of SQL commands, and why does the grouping matter?**
> DDL (structure: `CREATE`, `ALTER`, `DROP`), DML (data changes: `INSERT`, `UPDATE`, `DELETE`), DQL (reading data: `SELECT`), DCL (permissions: `GRANT`, `REVOKE`), and TCL (transaction control: `COMMIT`, `ROLLBACK`). The grouping matters because each category has different transactional and locking behavior — for example, DDL in PostgreSQL is transactional and can be rolled back, which surprises people coming from databases where it isn't. (Module 01, Categories of SQL Commands)

**Q4. What does `NULL` actually mean, and why isn't `NULL = NULL` true?**
> `NULL` represents an unknown or missing value, not zero or an empty string. Comparing two unknowns with `=` can't yield `TRUE` or `FALSE` — the honest answer is "unknown," which SQL represents as a third logical value alongside `TRUE`/`FALSE` (three-valued logic). That's why checking for missing values requires `IS NULL`/`IS NOT NULL` instead of `= NULL`, and why a `NOT IN` list containing a `NULL` silently excludes every row. (Module 03, Data Types)

**Q5. `CHAR(n)` vs. `VARCHAR(n)` vs. `TEXT` — what's the real difference in PostgreSQL?**
> `CHAR(n)` pads values with trailing spaces to a fixed length; `VARCHAR(n)` stores variable-length text up to a limit; `TEXT` stores variable-length text with no limit at all. PostgreSQL internally stores `VARCHAR` and `TEXT` almost identically and enforces no meaningful performance difference between them — the practical guidance is to default to `TEXT` unless a length constraint is a genuine business rule you want the database to enforce. (Module 03, Data Types)

## Theme: Constraints & Keys

**Q6. What's the difference between a `PRIMARY KEY` and a `UNIQUE` constraint?**
> Both enforce uniqueness, but a table can have only one `PRIMARY KEY` (which also forbids `NULL`), while it can have many `UNIQUE` constraints, and a `UNIQUE` column can hold `NULL` — in fact PostgreSQL treats multiple `NULL`s in a `UNIQUE` column as *not* duplicates of each other, since `NULL` never equals `NULL`. The primary key is the one column (or set of columns) chosen as the table's canonical identifier. (Module 05, Constraints & Keys)

**Q7. What does a foreign key actually guarantee, and what happens on `ON DELETE`?**
> A foreign key guarantees referential integrity: every non-null value in the referencing column must match an existing value in the referenced table's key column — no "orphaned" rows pointing at something that doesn't exist. `ON DELETE` (and `ON UPDATE`) control what happens when the referenced row disappears: `RESTRICT`/`NO ACTION` block the delete, `CASCADE` deletes the dependent rows too, `SET NULL` nulls out the reference, and `SET DEFAULT` resets it to a default value. (Module 05, Constraints & Keys)

**Q8. Natural key vs. surrogate key — what's the trade-off?**
> A natural key is a real-world attribute that's already unique (an email address, a national ID number); a surrogate key is a database-generated identifier with no business meaning (a `SERIAL` integer or a `UUID`). Natural keys avoid a redundant column but can turn out not to be as unique or stable as assumed (people change emails), forcing painful key changes later. Surrogate keys are stable and simple to index but require an extra `UNIQUE` constraint on the natural attribute if you still need to enforce its real-world uniqueness. Most production schemas default to surrogate keys for exactly this stability reason. (Module 05, Constraints & Keys)

**Q9. What's the difference between `NOT NULL`, `DEFAULT`, and a `CHECK` constraint?**
> `NOT NULL` forbids missing values outright. `DEFAULT` supplies a fallback value when an `INSERT` doesn't specify one — it doesn't forbid `NULL` if `NULL` is explicitly inserted. `CHECK` attaches an arbitrary boolean expression that every row must satisfy (e.g. `CHECK (salary > 0)`), enforcing business rules a data type or uniqueness constraint alone can't express. All three are enforced by the database itself, not application code, so they hold regardless of which application or script writes the data. (Module 05, Constraints & Keys)

## Theme: Querying, Joins & Aggregation

**Q10. What is the logical order in which a `SELECT` query is actually processed?**
> Conceptually: `FROM`/`JOIN` (assemble the rows) → `WHERE` (filter individual rows) → `GROUP BY` (collapse into groups) → `HAVING` (filter groups) → `SELECT` (compute the output columns) → `ORDER BY` (sort) → `LIMIT`/`OFFSET` (cap and paginate). This explains why you can't reference a `SELECT`-list alias in `WHERE` (it doesn't exist yet at that stage) but you can in `ORDER BY` (it does by then). (Module 07, Querying Basics / Module 09, Aggregation)

**Q11. `INNER JOIN` vs. `LEFT JOIN` — when do they return a different number of rows?**
> `INNER JOIN` returns only rows with a match on both sides; any row on either side without a match is dropped. `LEFT JOIN` keeps every row from the left table regardless of a match, filling unmatched right-side columns with `NULL`. They return the same rows only when every left-side row is guaranteed to have at least one match — the moment that's not true, `LEFT JOIN` returns more rows (or the same left-side rows padded with `NULL`s) than `INNER JOIN`. (Module 10, Joins & Set Operations)

**Q12. Why can't you filter an aggregate result with `WHERE` — why does SQL need `HAVING`?**
> `WHERE` filters individual rows *before* grouping happens; at that point in query evaluation, an aggregate like `SUM(salary)` doesn't exist yet — there's nothing to group yet. `HAVING` filters *after* `GROUP BY` has collapsed rows into groups, so it can reference aggregate expressions directly. Using `WHERE` where you meant `HAVING` is a very common early mistake that produces a "column must appear in GROUP BY" or aggregate-related error. (Module 09, Aggregation)

**Q13. `UNION` vs. `UNION ALL` — what's the actual cost difference?**
> Both stack two compatible result sets vertically (same column count, compatible types). `UNION` additionally removes duplicate rows, which requires an internal sort or hash step to detect them; `UNION ALL` keeps every row from both sides with no dedup step at all, and is therefore cheaper. Default to `UNION ALL` unless you specifically know duplicates are possible and unwanted. (Module 10, Joins & Set Operations)

**Q14. How would you find all customers who have never placed an order?**
> Two equivalent patterns: a `LEFT JOIN` from `customers` to `orders` filtered with `WHERE orders.id IS NULL` (the "anti-join" pattern), or `WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)`. `NOT EXISTS` is generally preferred over `NOT IN` for this because `NOT IN` silently returns zero rows if the subquery's list contains even one `NULL` — a classic, hard-to-spot bug. (Module 10, Joins & Set Operations / Module 11, Subqueries)

## Theme: Subqueries & CTEs

**Q15. What's the difference between a correlated and a non-correlated subquery?**
> A non-correlated subquery is fully self-contained — it can run on its own and produces one result used by the outer query. A correlated subquery references a column from the outer query's current row, so conceptually it's re-evaluated once per outer row (though the planner may rewrite this internally for efficiency). Correlated subqueries are how you express "compare this row against something computed specifically for its own group," like "this employee's salary vs. their own department's average." (Module 11, Subqueries)

**Q16. Why does `EXISTS` typically outperform `IN`, and what is the `NOT IN`/`NULL` trap?**
> `EXISTS` only needs to find *one* matching row and can stop as soon as it does; `IN` conceptually needs the full candidate list materialized for comparison, though on many rows the planner optimizes both similarly. The real danger is `NOT IN`: if the subquery's result list contains even a single `NULL`, the entire `NOT IN` comparison evaluates to unknown for every row, silently returning zero results instead of an error — `NOT EXISTS` doesn't have this failure mode at all. (Module 11, Subqueries)

**Q17. What is a CTE, and when does it communicate intent better than a nested subquery?**
> A CTE (`WITH name AS (...)`) names an intermediate result at the top of a query so the main query can reference it like a table. It's preferred over a deeply nested subquery when the same intermediate logic is used more than once in the outer query, or when nesting subqueries three or four levels deep would make the query's structure hard to read top-to-bottom. It's the same underlying capability as a derived table in `FROM`, but named and readable instead of buried inline. (Module 17, CTEs & Recursion)

**Q18. How does a recursive CTE actually work?**
> It has two parts joined by `UNION ALL`: an anchor member (the base case — e.g. the top-level manager with no manager of their own) and a recursive member that references the CTE's own name to repeatedly join back against the previous iteration's results, building up the hierarchy one level at a time until a round produces no new rows. It's the standard tool for hierarchical data (org charts) and graph traversal (route networks), and needs a termination condition to avoid infinite recursion. (Module 17, CTEs & Recursion)

**Q19. Window functions vs. `GROUP BY` — what's the essential difference?**
> `GROUP BY` collapses many rows into one row per group, discarding the individual rows. A window function (`... OVER (PARTITION BY ... ORDER BY ...)`) computes an aggregate-like value *per row* while keeping every row in the output — so you can show each employee's salary alongside their department's average salary in the same row, something `GROUP BY` alone cannot do without a self-join or subquery. (Module 16, Window Functions)

## Theme: Indexes & Performance

**Q20. What is an index, and why isn't adding one always a free win?**
> An index is a separate, ordered data structure (a B-tree by default in PostgreSQL) that lets the database find matching rows without scanning the whole table. It's not free because every `INSERT`/`UPDATE`/`DELETE` must also update every index on that table, and each index consumes additional disk space — over-indexing a write-heavy table can slow writes noticeably for a benefit that only pays off on reads. (Module 13, Indexes)

**Q21. What is the leftmost-prefix rule for composite indexes?**
> A composite index on `(a, b, c)` is usable for queries filtering on `a` alone, `a` and `b` together, or all three — but not for a query filtering on `b` or `c` alone, because the index is physically sorted by `a` first, then `b` within each `a`, then `c` within each `b`. Column order in a composite index is a deliberate design decision, not an arbitrary list. (Module 13, Indexes)

**Q22. What is a covering index / an index-only scan?**
> A covering index includes every column a specific query needs (via the indexed columns themselves or PostgreSQL's `INCLUDE` clause), so the query planner can satisfy the query by reading the index alone — an index-only scan — without ever touching the underlying table's heap pages, which is significantly faster than an index scan that still has to fetch each matching row from the table. (Module 13, Indexes)

**Q23. Why might the query planner reject a perfectly good index in favor of a sequential scan?**
> The planner is cost-based, not rule-based: if a `WHERE` condition matches a large fraction of the table's rows (low selectivity — e.g. a boolean column that's `TRUE` in 80% of rows), reading the index and then randomly jumping to each matching row in the table can cost more than simply reading the whole table sequentially. This is a genuinely correct decision by the planner, not a bug, and is exactly why `EXPLAIN` is needed instead of assuming an index will always be used. (Module 20, Performance Tuning)

**Q24. What does "sargable" mean, and why does wrapping an indexed column in a function break index usage?**
> Sargable ("Search ARGument ABLE") means a condition can directly use an index range scan. Wrapping an indexed column in a function — `WHERE UPPER(name) = 'ASHA'` instead of `WHERE name = 'Asha'` — forces the database to compute that function for every row before it can compare, defeating a plain index on `name` entirely (unless a matching expression index exists). The fix is either to avoid transforming the column, or to build an expression index that matches the transformation. (Module 20, Performance Tuning)

## Theme: Transactions & Concurrency

**Q25. Explain ACID, ideally with a concrete example of what breaks without each property.**
> Atomicity: a transaction's statements all succeed or all fail together — without it, a bank transfer could debit one account and crash before crediting the other. Consistency: a transaction always moves the database from one valid state to another, respecting all constraints. Isolation: concurrent transactions don't see each other's uncommitted intermediate state. Durability: once committed, data survives a crash immediately after. (Module 14, Transactions & Concurrency)

**Q26. What are dirty reads, non-repeatable reads, and phantom reads?**
> A dirty read sees another transaction's *uncommitted* change (which might later roll back). A non-repeatable read re-reads the same row twice within one transaction and gets a different value because another transaction committed a change in between. A phantom read re-runs the same filtering query twice and gets a different *set of rows* because another transaction inserted or deleted matching rows in between. Each stricter isolation level rules out one or more of these anomalies at the cost of more blocking/conflict overhead. (Module 14, Transactions & Concurrency)

**Q27. What is a deadlock, and how does PostgreSQL handle one?**
> Two (or more) transactions each hold a lock the other is waiting for, so neither can ever proceed — a circular wait. PostgreSQL runs periodic deadlock detection and, on finding one, forcibly aborts one of the transactions (rolling it back) so the other can continue, rather than letting both hang forever. The practical defense is to always acquire locks on multiple resources in a consistent order across your application. (Module 14, Transactions & Concurrency)

**Q28. What's the practical difference between `READ COMMITTED` and `SERIALIZABLE` isolation?**
> `READ COMMITTED` (PostgreSQL's default) only prevents dirty reads — each statement within a transaction sees a fresh snapshot of committed data, so non-repeatable and phantom reads are still possible across statements in the same transaction. `SERIALIZABLE` guarantees the outcome of concurrently running transactions is equivalent to some serial (one-at-a-time) ordering of them, ruling out every anomaly, at the cost of the database sometimes forcing a transaction to retry when a true conflict is detected. (Module 14, Transactions & Concurrency)

## Theme: Normalization & Design

**Q29. What is a functional dependency, and why does it underlie every normal form?**
> A functional dependency `A → B` means that for any given value of `A`, there is exactly one corresponding value of `B` (e.g. `employee_id → department_name` if each employee belongs to exactly one department). Every normal form is really just a formal statement about which functional dependencies are and aren't allowed to exist in a well-designed table — 2NF, 3NF, and BCNF each rule out a specific category of "bad" dependency that causes update anomalies. (Module 15, Normalization & Design)

**Q30. What's the practical difference between 2NF and 3NF?**
> 2NF eliminates partial dependencies — relevant only when a table has a composite primary key — where a non-key column depends on just *part* of that composite key rather than the whole thing. 3NF goes further and eliminates transitive dependencies, where a non-key column depends on another non-key column rather than directly on the key (e.g. `zip_code` determining `city`, inside a table keyed by `employee_id`). Both are fixed the same way: split the offending column(s) into their own table. (Module 15, Normalization & Design)

**Q31. When is denormalization actually justified?**
> When read performance for a specific, high-frequency query pattern matters more than eliminating redundancy, and the update-anomaly risk that redundancy reintroduces is either rare in practice or explicitly managed (e.g. via a trigger keeping the redundant copy in sync). A common real example is storing a denormalized `order_total` column on an `orders` table instead of always summing `order_items` on every read — it trades a small, controlled duplication risk for avoiding a join and aggregation on every single query. (Module 15, Normalization & Design)

## Theme: Security

**Q32. What is SQL injection, and how do parameterized queries actually prevent it?**
> SQL injection happens when untrusted input is concatenated directly into a query string, letting an attacker inject their own SQL logic (e.g. a login form field containing `' OR '1'='1`). Parameterized queries (prepared statements) send the query structure and the user-supplied values as separate channels to the database — the value is never parsed as SQL syntax at all, only ever bound as a literal value, which closes off the injection vector structurally rather than relying on carefully escaping every special character. (Module 19, Security & Access Control)

**Q33. What's the principle of least privilege, applied to database roles?**
> Every role (a human user or an application's connection credential) should be granted only the specific privileges it actually needs to do its job, and nothing more — an application's connection user gets `SELECT`/`INSERT`/`UPDATE` on the tables it touches, not blanket superuser access, and a reporting user gets read-only `SELECT` and nothing else. This bounds the blast radius if any single credential is ever compromised. (Module 19, Security & Access Control)

**Q34. What's the difference between a PostgreSQL role and a user, and what do `GRANT`/`REVOKE` actually do?**
> In PostgreSQL, a "user" is just a role created with the `LOGIN` privilege — there's no separate underlying concept. Roles can be granted membership in other roles to inherit their privileges, which is how permissions scale cleanly across many people instead of granting each privilege to each person individually. `GRANT` attaches a specific privilege (`SELECT`, `INSERT`, DDL rights, etc.) on a specific object (table, schema, database) to a role; `REVOKE` removes it. (Module 19, Security & Access Control)

## Best Practices for Using This List

- Time yourself: a strong 30-45 second answer beats a rambling three-minute one — interviewers read excessive length as uncertainty, not thoroughness.
- Always be ready to go one level deeper than the model answer if asked "why" again — every question here points back to a full module topic for exactly that purpose.
- Practice explaining these out loud, not just recognizing them silently on the page — the skill an interview actually tests is verbal, real-time explanation, not passive recall.

## Common Mistakes

- Reciting a memorized definition without connecting it to *why* it matters — interviewers consistently rate "explains the reasoning" answers higher than "recites the fact" answers, even when both are technically correct.
- Scope-creeping an answer — turning "what's the difference between `WHERE` and `HAVING`" into an unprompted tour of the entire query pipeline. Answer what was asked; let the interviewer ask a follow-up if they want more.
- Freezing on a question you genuinely don't remember instead of reasoning toward it out loud — interviewers routinely value visible, structured reasoning under uncertainty over a memorized-but-silent struggle.

## Summary

- This topic re-organized the course's highest-signal conceptual questions **by cross-cutting theme** — relational fundamentals, constraints/keys, querying/joins/aggregation, subqueries/CTEs, indexes/performance, transactions/concurrency, normalization/design, and security — rather than by module order, prioritizing what interviewers ask most.
- Every answer is deliberately interview-length (20-45 seconds spoken), with a pointer back to the full module for further depth.
- The full, exhaustive Q&A set for every individual topic remains available inside each module's own topic files, under their own "Interview Questions" sections.
