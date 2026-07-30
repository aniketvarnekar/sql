# Materialized Views

## Learning Objectives

By the end of this section you should be able to:
- State the key difference between a regular view and a materialized view in terms of *what is stored and when it's computed*
- Create a materialized view with `CREATE MATERIALIZED VIEW` and populate it with data
- Refresh a materialized view with `REFRESH MATERIALIZED VIEW`, and explain what `CONCURRENTLY` changes and what it requires
- Explain the staleness trade-off a materialized view introduces, in your own words, with a concrete example
- Decide, for a given scenario, whether a regular view, a materialized view, or a plain table is the right tool

## Prerequisites

- [Creating and Using Views](01-creating-and-using-views.md) — this topic assumes you already understand that a regular view is a saved query with no stored data of its own; materialized views exist specifically as the deliberate exception to that.
- Module 9 (Aggregation) — the motivating example for this topic, like Topic 1's, is an aggregation over a join, since expensive aggregations are the most common reason to reach for a materialized view.
- [Updatable Views](02-updatable-views.md) is not required for this topic — materialized views are read-only from the perspective of ordinary `INSERT`/`UPDATE`/`DELETE` and don't participate in the automatic-updatability discussion at all.

## Motivation

Some queries are just expensive — a multi-table join with a `GROUP BY` over a large volume of historical data, run to power a dashboard that gets refreshed by dozens of people throughout the day. A regular view, as Topic 1 established, gives you a convenient name for that query, but it doesn't make it any cheaper — the full join and aggregation still runs on every single reference. If the underlying data genuinely only changes once a day (say, from an overnight batch load), recomputing that same expensive result dozens of times between loads is pure waste. You want the convenience of a named, `SELECT`-able object, but with the cost paid once, not on every read.

## Problem Statement

Suppose a monthly revenue-by-category report aggregates `orders`, `order_items`, and `products` — potentially across millions of historical rows — and is queried by a dashboard refreshed constantly throughout the business day:

```sql
SELECT
    date_trunc('month', o.order_date)::date AS month,
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.status = 'completed'
GROUP BY 1, 2;
```

If order data is only loaded in nightly batches, this exact result is guaranteed to be identical every time it's queried between one night's load and the next — yet a regular view over this query would recompute the full join and aggregation from scratch on every single dashboard refresh, all day, for an answer that hasn't actually changed since the last batch load finished.

## Concept

### CREATE MATERIALIZED VIEW

A **materialized view** looks almost identical to a regular view to create, but behaves fundamentally differently once it exists:

```sql
CREATE MATERIALIZED VIEW monthly_category_revenue AS
SELECT
    date_trunc('month', o.order_date)::date AS month,
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.status = 'completed'
GROUP BY 1, 2
WITH DATA;
```

`WITH DATA` (the default if omitted) tells PostgreSQL to run the query immediately and physically store its result. Querying it afterward reads that stored result directly — no join, no aggregation, no recomputation:

```sql
SELECT * FROM monthly_category_revenue ORDER BY month, category;
```

```
   month    |  category   | revenue
------------+-------------+---------
 2026-01-01 | Electronics |    1598
 2026-01-01 | Stationery  |    1245
 2026-02-01 | Electronics |    3499
 2026-03-01 | Electronics |     799
 2026-03-01 | Stationery  |     747
(5 rows)
```

### The Key Difference from a Regular View

| | Regular view | Materialized view |
|---|---|---|
| What's stored | Only the query definition | The query's actual result, physically on disk |
| When the query runs | Every time the view is referenced | Only at `CREATE MATERIALIZED VIEW` time and at each explicit `REFRESH` |
| Always reflects current base-table data? | Yes, always | No — only as of the last refresh |
| Cost of reading it | Full cost of the underlying query, every time | Cost of reading stored rows, like an ordinary table |
| Can be indexed? | No (there's nothing of its own to index) | Yes — exactly like a real table |

This is the entire trade-off in one sentence: **a materialized view exchanges guaranteed freshness for guaranteed cheap reads.**

### REFRESH MATERIALIZED VIEW

Because a materialized view's contents are a snapshot, they must be explicitly told to catch up with the base tables:

```sql
REFRESH MATERIALIZED VIEW monthly_category_revenue;
```

By default, this takes an `ACCESS EXCLUSIVE` lock on the materialized view for the duration of the refresh — meaning **queries against the materialized view are blocked until the refresh finishes.** For a large, frequently-queried materialized view, that's often unacceptable; PostgreSQL offers `CONCURRENTLY` to avoid it:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_category_revenue;
```

`CONCURRENTLY` computes the new result into a temporary working area, compares it row-by-row against the existing stored contents, and applies only the differences — all without holding a lock that blocks concurrent readers. It has one hard requirement: **the materialized view must have at least one `UNIQUE` index**, so PostgreSQL can identify which stored rows correspond to which newly-computed rows:

```sql
CREATE UNIQUE INDEX ON monthly_category_revenue (month, category);

REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_category_revenue;
```

Without that unique index, `CONCURRENTLY` fails outright:

```
ERROR:  cannot refresh materialized view "monthly_category_revenue" concurrently
HINT:  Create a unique index with no WHERE clause on one or more columns of the materialized view.
```

If a materialized view is created with `WITH NO DATA` instead of `WITH DATA`, it exists as an object but holds nothing yet, and querying it before the first `REFRESH` fails:

```sql
CREATE MATERIALIZED VIEW monthly_category_revenue AS
SELECT ... 
WITH NO DATA;

SELECT * FROM monthly_category_revenue;
```

```
ERROR:  materialized view "monthly_category_revenue" has not been populated
HINT:  Use the REFRESH MATERIALIZED VIEW command.
```

### The Staleness Trade-off

Between one `REFRESH` and the next, `monthly_category_revenue` reflects whatever `orders`, `order_items`, and `products` looked like *at the time of that refresh* — not whatever they look like right now. If a new completed order is recorded five minutes after the last refresh, it simply won't appear in `monthly_category_revenue` until the next refresh runs. PostgreSQL has no built-in mechanism to automatically refresh a materialized view the moment its underlying data changes — refreshing is always something you (or a scheduled job you set up) trigger explicitly. This is a deliberate, visible trade-off rather than a hidden one: you always know exactly how stale the data can be, because it's bounded by whatever refresh schedule you've chosen — every hour, every night, every five minutes — rather than some opaque, unpredictable delay.

### When to Use a Materialized View vs. a Regular View vs. a Plain Table

| Situation | Best choice |
|---|---|
| The query is cheap, or the result must always reflect the very latest data | Regular view |
| The query is expensive (large joins/aggregations), it's queried often, and some staleness (minutes to a day) is acceptable | Materialized view |
| The data is genuinely its own independent dataset — written to directly, not derived by aggregating or joining other tables | Plain table |
| An expensive query is run rarely, or by very few consumers | Regular view (or just running the query directly) — the maintenance overhead of scheduling refreshes usually isn't worth it for something queried occasionally |

The deciding question is almost always: *"Can this data tolerate being slightly out of date, in exchange for being fast to read?"* If yes, and the query is genuinely expensive, a materialized view is likely the right call. If the answer is "no, it must be exact, right now" — a payment balance, an inventory count someone is about to check out against — a materialized view is the wrong tool, regardless of how expensive the query is.

## Internal Working (Preview)

Internally, a materialized view is stored as a real heap of pages on disk, tagged in the system catalog (`pg_class.relkind = 'm'`) to distinguish it from an ordinary table (`'r'`) or a regular view (`'v'`). `REFRESH` without `CONCURRENTLY` behaves roughly like an internal `TRUNCATE` followed by re-running the defining query and inserting its output — done under an exclusive lock so no reader can see a half-refreshed, inconsistent state:

```
REFRESH MATERIALIZED VIEW monthly_category_revenue
        │
        ▼
Take ACCESS EXCLUSIVE lock (blocks concurrent reads of this materialized view)
        │
        ▼
Empty existing stored rows, run the defining query fresh, store its output
        │
        ▼
Release lock — readers see either the old data or the new data, never a mix
```

`REFRESH ... CONCURRENTLY` instead computes the new result into a separate transient location, then diffs it against the currently-stored rows (matching them up via the required unique index) and applies only row-level inserts/updates/deletes needed to reconcile the two — all inside a transaction, without ever taking a lock that blocks ordinary readers:

```
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_category_revenue
        │
        ▼
Run the defining query fresh, into a separate transient result set
        │
        ▼
Diff transient result against currently stored rows (matched via unique index)
        │
        ▼
Apply only the changed rows, transactionally — concurrent readers
see either fully-old or fully-new data, and are never blocked
```

This extra diffing work is why `CONCURRENTLY` is not simply a strictly-better default: it costs more CPU and I/O than a plain `REFRESH`, in exchange for not blocking readers.

## Real-World Analogy

A regular view is like logging into your bank account online — the balance shown is computed live, this instant, guaranteed current. A materialized view is like the paper statement mailed to you once a month: it's a physically real document, instantly readable with no computation needed, but it's a snapshot frozen at the moment it was printed. Any transaction that happens after the statement was generated simply isn't reflected in it — not because anything is broken, but because a printed statement was never meant to update itself; it only "refreshes" the next time a new one is generated. Choosing between a live account login and a mailed statement is exactly the choice between a regular view and a materialized view: immediate freshness at full computational cost, versus a physically stored, instantly-readable snapshot that's only as current as its last refresh.

## Why Materialized Views Were Designed This Way

The tension here is a very general one in database (and computing) systems: correctness/freshness versus cost. Recomputing a query from scratch on every read guarantees the answer is always exactly correct as of right now, at the cost of doing that work every single time. Storing a computed result guarantees cheap reads, at the cost of that result potentially being out of date. PostgreSQL's materialized views make this trade-off completely explicit and developer-controlled rather than hiding it behind implicit "eventually consistent" magic: you decide exactly when a refresh happens, so you always know the maximum possible staleness of the data you're reading, instead of being surprised by it. This mirrors the same "prefer an explicit, predictable behavior over an implicit, surprising one" philosophy seen in Module 5's constraints and Module 6's insistence on explicit `WHERE` clauses — PostgreSQL would rather you consciously choose a trade-off than stumble into one.

## Advantages

- **Large, predictable performance win for expensive, frequently-read queries** — the expensive join/aggregation cost is paid once per refresh, not once per read, which can turn a multi-second query into a millisecond one for readers.
- **Queried exactly like a table or regular view** — consumers don't need special syntax or awareness that they're reading a materialized result rather than a live one.
- **Supports indexes, exactly like a real table** — you can add indexes on a materialized view's columns (Module 13) to speed up filtered or sorted reads against the stored result even further.
- **`CONCURRENTLY` avoids blocking readers during refresh** — a large materialized view can be kept up to date without a maintenance window that locks out queries.
- **The staleness window is fully visible and controllable** — you decide the refresh cadence, so you always know the maximum possible age of the data, rather than guessing.

## Disadvantages / Limitations

- **Data can be stale between refreshes** — it is the wrong tool whenever a reader genuinely needs the exact, current state of the underlying data (an account balance about to be charged, an inventory count at the moment of checkout).
- **Consumes real, additional storage** — unlike a regular view, the result is physically duplicated on disk alongside the base tables it was derived from.
- **Refreshing is entirely your responsibility** — PostgreSQL will not automatically refresh a materialized view when its underlying data changes; you must schedule refreshes yourself (an external scheduler, a cron-style job, or the `pg_cron` extension are common approaches).
- **`CONCURRENTLY` requires upkeep of its own** — it needs a unique index to function, and it does more work (computing a full diff) than a plain refresh, so it isn't a free upgrade in every case.

## Best Practices

- Confirm the underlying query is actually expensive (using `EXPLAIN ANALYZE`, covered fully in Module 20) before reaching for a materialized view — don't add the staleness trade-off and refresh maintenance burden for a query that wasn't a real bottleneck to begin with.
- If a materialized view will be refreshed with `CONCURRENTLY`, create the required unique index up front, on whatever column(s) uniquely identify each row of its result (here, `(month, category)`).
- Schedule refreshes automatically rather than relying on someone to remember to run `REFRESH` by hand — an unrefreshed materialized view silently becomes more and more stale with no warning.
- Document the expected staleness window for anyone querying the materialized view (e.g., "refreshed nightly at 2 AM; do not use for same-day figures") — the staleness is invisible from the query itself, so it has to be communicated some other way.
- Prefer a regular view (or direct query tuning/indexing, Module 13) over a materialized view whenever the data must always be current — don't default to a materialized view just because a query involves a join and an aggregation.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a materialized view automatically updates when base tables change | It does not — PostgreSQL never auto-refreshes a materialized view; every update to it requires an explicit `REFRESH MATERIALIZED VIEW`. |
| Running `REFRESH MATERIALIZED VIEW CONCURRENTLY` without a unique index | PostgreSQL requires at least one unique index on the materialized view for `CONCURRENTLY` to have a way to match old and new rows; without one, the command fails with an explicit error. |
| Using a materialized view for data that must always be exactly current | This defeats the purpose of choosing it — if perfect real-time correctness is required, use a regular view (always current, at full recompute cost) or query the base tables directly. |
| Not realizing a plain `REFRESH` blocks reads | Without `CONCURRENTLY`, `REFRESH MATERIALIZED VIEW` takes a lock that blocks queries against it until the refresh completes — a large materialized view refreshed this way during business hours can cause a real, visible outage for its readers. |

## Interview Questions

1. **Q: What is the fundamental difference between a regular view and a materialized view?**
   A: A regular view stores only a query definition and is re-executed against current data every time it's referenced. A materialized view physically stores the query's result on disk; reading it reads that stored snapshot, and the snapshot only changes when `REFRESH MATERIALIZED VIEW` is explicitly run.

2. **Q: How do you refresh a materialized view without blocking concurrent readers, and what does that require?**
   A: `REFRESH MATERIALIZED VIEW CONCURRENTLY viewname`, which requires the materialized view to have at least one unique index so PostgreSQL can match up old and new rows while diffing, rather than needing to lock out readers during the refresh.

3. **Q: Describe a realistic scenario where a materialized view is clearly the right choice over a regular view.**
   A: A dashboard aggregating a large volume of historical order data (a multi-table join with `GROUP BY`) that's queried dozens of times a day but whose underlying data only changes with a nightly batch load — recomputing the full aggregation on every read would be wasted work for an answer that's identical between loads, so storing it once and refreshing nightly is strictly cheaper with no meaningful loss of correctness.

4. **Q: Does a PostgreSQL materialized view automatically stay in sync with its underlying tables?**
   A: No — PostgreSQL requires an explicit `REFRESH MATERIALIZED VIEW` (optionally `CONCURRENTLY`) to update it; there is no automatic invalidation-on-write. (Some other database systems offer materialized/indexed views with automatic incremental refresh behavior — Module 22, SQL Across Databases, covers where vendors diverge on this.)

## Summary

- A materialized view physically stores its query's result, unlike a regular view, which stores only the query itself and recomputes on every read.
- `CREATE MATERIALIZED VIEW ... AS ... WITH DATA` populates it immediately; `WITH NO DATA` leaves it empty until an explicit `REFRESH`.
- `REFRESH MATERIALIZED VIEW` recomputes and replaces the stored result, blocking readers by default; `REFRESH MATERIALIZED VIEW CONCURRENTLY` avoids that block but requires a unique index on the materialized view.
- The core trade-off is staleness for speed: a materialized view is only ever as current as its last refresh, in exchange for reads that cost nothing beyond reading stored rows.
- Choose a materialized view when a query is genuinely expensive, queried often, and can tolerate some bounded staleness; choose a regular view when correctness must always be current; choose a plain table when the data is its own independent dataset rather than a derived result.
- This closes out the module's progression: Topic 1 established what a view fundamentally is, Topic 2 covered writing through one, and this topic covered making an expensive one fast to read.
