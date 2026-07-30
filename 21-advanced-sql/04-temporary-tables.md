# Temporary Tables

## Learning Objectives

By the end of this section you should be able to:
- Create a session-scoped temporary table with `CREATE TEMPORARY TABLE`
- Explain a temporary table's default lifetime and how `ON COMMIT DROP`/`ON COMMIT DELETE ROWS` change it
- Identify realistic situations where staging intermediate results in a temporary table is the right tool
- Distinguish a temporary table from a CTE (`WITH` clause) and from a view along the axes of persistence, materialization, and scope

## Prerequisites

This topic assumes **Module 17 — CTEs & Recursion** (see the [Module 17 overview](../17-ctes-and-recursion/00-module-overview.md)), since the whole point of this topic is to precisely contrast a temporary table against the `WITH` clause you learned there. It also assumes a general familiarity with views from **Module 12 — Views** (see the [Module 12 overview](../12-views/00-module-overview.md) and [Creating and Using Views](../12-views/01-creating-and-using-views.md)), for the same reason — this topic is as much about *distinguishing* three similar-sounding tools as it is about introducing a new one.

## Motivation

Some reporting and data-processing tasks are naturally a sequence of steps, where each step's output feeds the next — "first compute this quarter's high-value customers, then compute their average order size, then join that back against a product-preferences table to build a targeted mailing list." Writing this as one giant, deeply nested query is possible but often unreadable, and a CTE (Module 17), while great for a single query's readability, disappears the moment that one query finishes — it can't be reused across several separate statements, and it can't be indexed. Temporary tables exist for exactly this gap: real, disk-backed, indexable storage for an intermediate result, scoped automatically to your session so it never leaks into anyone else's, and never needs manual cleanup.

## Problem Statement

Imagine building a multi-step monthly reconciliation report: first identify customers whose total spend this quarter exceeds a threshold, then compute each of their average order sizes, then join that back against product category preferences, then produce a final ranked summary. You want to:

- Break this into clearly separated, individually-checkable steps (rather than one unreadable nested query).
- Reuse the first step's result in several later, *separate* statements — something a CTE can't do, since a CTE only lives for the duration of the one query it's attached to.
- Avoid leaving a permanent, shared table cluttering the schema once the report is done — you don't want every analyst's ad hoc report tables piling up forever, and you don't want two analysts running this report at the same time to collide on the same table name.

A temporary table solves all three at once.

## Concept

### `CREATE TEMPORARY TABLE` Syntax

```sql
CREATE TEMPORARY TABLE high_value_customers AS
SELECT customer_id, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 10000;
```

(`TEMP` is an accepted abbreviation for `TEMPORARY`.) This behaves, from this point forward, exactly like an ordinary table — it can be queried, joined, indexed, updated, and reused across as many subsequent statements as you like, within the same session:

```sql
SELECT * FROM high_value_customers ORDER BY total_spent DESC LIMIT 3;
```

```
 customer_id | total_spent
-------------+-------------
          14 |    24500.00
           7 |    19800.00
          22 |    17250.00
(3 rows)
```

You can now build the next step directly on top of it, in a separate statement, in the same session:

```sql
SELECT h.customer_id, h.total_spent, COUNT(o.order_id) AS order_count,
       ROUND(h.total_spent / COUNT(o.order_id), 2) AS avg_order_size
FROM high_value_customers h
JOIN orders o ON o.customer_id = h.customer_id
GROUP BY h.customer_id, h.total_spent
ORDER BY h.total_spent DESC;
```

Each of these is its own separate, readable statement — a multi-step process staged through real, queryable, intermediate tables, rather than one unreadable nested expression.

### Session Scope and Default Lifetime

A temporary table is visible **only to the database session (connection) that created it.** Two different sessions can each create a temp table named `high_value_customers` at the same time without any conflict at all — each session's temporary tables live in their own private, session-specific namespace, invisible to every other session, even one connected as the same database user. By default, a temporary table (and its data) is automatically dropped the moment its creating session ends (disconnects) — no manual `DROP TABLE` is ever required.

### Controlling Lifetime with `ON COMMIT`

By default (`ON COMMIT PRESERVE ROWS`), a temporary table's rows survive every `COMMIT` within the session and are only removed when the session itself ends. Two other options narrow that lifetime further:

```sql
CREATE TEMPORARY TABLE step_results (
    customer_id INTEGER,
    metric      NUMERIC
) ON COMMIT DELETE ROWS;
```

`ON COMMIT DELETE ROWS` empties the table's rows every time the enclosing transaction commits, while the table definition itself remains for the rest of the session — useful for a scratch table reused, freshly emptied, across many separate transactions in a long-running batch job.

```sql
BEGIN;
CREATE TEMPORARY TABLE step1_staging (
    customer_id INTEGER,
    total_spent NUMERIC
) ON COMMIT DROP;

INSERT INTO step1_staging
SELECT customer_id, SUM(amount) FROM orders GROUP BY customer_id;

-- use step1_staging for further processing within this same transaction

COMMIT;
-- step1_staging no longer exists at all — dropped automatically at commit
```

`ON COMMIT DROP` drops the entire table — not just its rows — the instant the transaction that created it commits (or rolls back). This is the right choice for a scratch table meaningful only within a single transaction/batch job, with no need to persist even for the rest of the session.

### Temporary Tables vs. CTEs vs. Views

| Aspect | Temporary Table | CTE (`WITH ...`, Module 17) | View (Module 12) |
|---|---|---|---|
| Persistence | Session- or transaction-scoped; dropped automatically | Exists only for the single query it's part of | Permanent, until explicitly `DROP`ped |
| Physically materialized | Yes — real, disk-backed storage, like an ordinary table | Not durable storage — PostgreSQL may inline or materialize it internally for that one query's execution, but nothing persists afterward | No, by default — its defining query re-runs every time it's selected from, unless it's a materialized view |
| Reusable across separate statements | Yes — any later statement in the same session can query it | No — scoped strictly to the one statement that defines it | Yes — permanently, by any session with appropriate permission |
| Visible to other sessions | No — private to the creating session | No — not a stored object at all | Yes — a shared, named database object |
| Can have its own index | Yes | No — there's no persistent structure to attach an index to | No (the view itself can't be indexed; its underlying base tables can) |

The practical decision rule: reach for a **CTE** when you're naming an intermediate step purely for one query's readability and don't need to reuse it elsewhere; reach for a **view** when you want a permanently reusable, shared, named query that many sessions will use over time; reach for a **temporary table** when you need a real, indexable, disk-backed intermediate result that must be reused across *multiple separate statements* within one session, without permanently existing in the shared schema.

## Internal Working (Preview)

Every session that creates a temporary table is transparently assigned its own private schema, internally named something like `pg_temp_3` (the exact number varies per session) — this is *why* two sessions can each have a table called `high_value_customers` without collision: they're actually `pg_temp_3.high_value_customers` and `pg_temp_7.high_value_customers`, distinct objects that happen to share a short display name. A temp table's rows are backed by real storage, exactly like an ordinary table's, which is why they can be indexed, analyzed, and queried efficiently — the only difference from a permanent table is the automatic cleanup tied to session or transaction lifetime.

```
 Session connects
        │
        ▼
 CREATE TEMPORARY TABLE t  →  created inside this session's private pg_temp_N schema
        │
        ▼
 Used across several statements, indexed, joined, updated — behaves like a real table
        │
        ▼
 Session disconnects (or, with ON COMMIT DROP, the transaction commits)
        │
        ▼
 Table (and its private schema entry) is automatically dropped — no manual cleanup needed
```

## Real-World Analogy

A temporary table is like a whiteboard in a private meeting room booked just for you. You can write on it, erase parts, and add to it throughout your entire meeting (session) — it behaves like any other whiteboard while you're using it. The moment your meeting ends and you leave the room (disconnect), the building staff wipe it clean automatically for the next booking, with no action required from you. Critically, nobody in a different room can see your whiteboard, or be confused by what's written on it — it's private to your booking, the same way a temp table is private to your session, even if someone else, in a different room, is using a whiteboard with the exact same notes written on it.

## Why Temporary Tables Were Designed This Way

A CTE (Module 17) is convenient precisely because it's *not* durable — it's scoped and often optimized within a single query, which is exactly right for improving that one query's readability, but it means it can never be indexed and never outlives the statement it's part of. A view (Module 12) is convenient because it's a permanently reusable, shared, named query — but "permanent and shared" is the opposite of what a throwaway, work-in-progress intermediate result needs, and a plain view isn't materialized storage anyway. Temporary tables fill the specific, genuine gap between these two: real, indexable, disk-backed storage, but scoped automatically to a session or transaction so it never needs a manual `DROP` and never risks colliding with, or leaking into, another session's work. This mirrors the relational model's general principle (Module 2) that a table is just a named, structured store of rows — a temporary table applies that same idea, deliberately narrowed to a private, self-cleaning lifetime.

## Advantages

- **Real, indexable, disk-backed storage** — unlike a CTE, a temp table can be indexed (Module 13) and queried efficiently across many separate subsequent statements.
- **Automatic cleanup** — no manual `DROP TABLE` required; the table disappears with the session (or transaction, with `ON COMMIT DROP`), leaving no schema clutter behind.
- **Private per session** — no naming collisions between concurrent users or concurrent runs of the same report, since each session's temp tables are invisible to every other session.

## Disadvantages / Limitations

- **Not shared across sessions** — each session must (re)build its own temp table if it needs the same intermediate result; there's no automatic caching or sharing of a temp table's contents across different users or connections.
- **Still consumes real storage and I/O while it exists** — a very large temp table used heavily by many concurrent sessions can add real load to the database, the same as any other table would.
- **Easy to be confused by connection pooling** — if your application uses a connection pool, a "next" query issued by your application may be handed to a *different* physical database session than the one that created the temp table, and that session will not see it at all — this is a common source of "the temp table I just created suddenly isn't there" confusion.

## Best Practices

- Use `ON COMMIT DROP` for scratch tables meant to live only for a single transaction or batch job step.
- Add an index to a temporary table exactly as you would a normal one (Module 13) if it's large and queried repeatedly within the session — this is one of the main reasons to reach for a temp table over a CTE in the first place.
- Never rely on a temporary table's contents surviving across separate application connections, especially under connection pooling — treat it as valid only within the single session/transaction that created it.
- Prefer a CTE for a single, one-off query where the intermediate result doesn't need to be reused elsewhere or indexed — reach for a temporary table only once you genuinely need reuse across multiple statements or indexing.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a temp table persists across different connections or a connection-pooled application | Temp tables are strictly session-scoped by design; a different physical connection (which a pooled application may hand you at any time) never sees another session's temp table. |
| Reaching for a temporary table where a CTE would cleanly express the same logic in one query | Adds unnecessary schema objects and a persistence lifetime to manage, for logic a single readable multi-CTE query (Module 17) already solves without any of that overhead. |
| Forgetting `ON COMMIT DROP` on a scratch table meant to last only one transaction, in a long-lived, pooled session | Leaves the (now-empty or stale) temp table lingering for the rest of the session instead of being cleaned up immediately, quietly accumulating clutter. |
| Naming a temp table identically to an existing permanent table | A session's temp table of the same name shadows (takes priority over) the permanent table for unqualified references within that session — queries may silently operate on the temp data instead of the "real" table without any error being raised. |

## Interview Questions

1. **Q: What is the default lifetime of a `CREATE TEMPORARY TABLE`, and how do you shorten it to a single transaction?**
   A: By default, a temporary table (and its rows) persists for the entire session and is dropped automatically only when the session disconnects. Adding `ON COMMIT DROP` to the `CREATE TEMPORARY TABLE` statement instead drops the entire table the moment the transaction that created it commits.

2. **Q: What's the core difference between a temporary table and a CTE?**
   A: A CTE exists only for the duration of the single query it's attached to, is not durable storage, and cannot be indexed. A temporary table is real, disk-backed storage that persists across multiple separate statements within a session (or transaction), and can be indexed like an ordinary table.

3. **Q: What's the core difference between a temporary table and a view?**
   A: A view is a permanently stored, named query, shared and visible to any session with permission, re-executed each time it's selected from (unless materialized). A temporary table is private to one session, physically materialized once, and automatically dropped when its session (or, with `ON COMMIT DROP`, its transaction) ends.

4. **Q: Why can two different sessions each create a temporary table with the exact same name without any conflict?**
   A: Each session is transparently given its own private, session-specific schema (internally something like `pg_temp_N`) to hold its temporary tables. A temp table named `staging` in one session is actually a distinct object from a temp table named `staging` in another session — they only appear to share a name because each session sees its own private schema without needing to qualify it.

## Summary

- `CREATE TEMPORARY TABLE` (or `CREATE TEMP TABLE`) creates a real, disk-backed table visible only to the session that created it.
- By default, a temp table persists for the whole session; `ON COMMIT DELETE ROWS` empties it at each commit, and `ON COMMIT DROP` removes the entire table at the commit of the transaction that created it.
- Temporary tables are the right tool for staging intermediate results that need to be reused across multiple separate statements, or that benefit from being indexed — something a CTE cannot offer, since a CTE is scoped to a single query and isn't durable storage.
- Compared to a view, a temporary table is private and session-scoped rather than permanent and shared, and is physically materialized rather than re-run on every reference.
- Never rely on a temp table's contents surviving a different connection — this is a frequent, confusing failure mode under connection pooling.
- Next, Topic 5 closes out this module with table partitioning — splitting one enormous logical table into smaller, more manageable physical pieces.
