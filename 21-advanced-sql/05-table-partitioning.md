# Table Partitioning

## Learning Objectives

By the end of this section you should be able to:
- Explain what table partitioning is and what specific problem it solves for very large tables
- Create a partitioned table using range, list, and hash partitioning strategies
- Explain query pruning and why it makes queries against a partitioned table faster
- Explain why dropping or detaching a partition is dramatically faster than an equivalent `DELETE`
- Identify partitioning's operational trade-offs, including the requirement that a unique or primary key include the partition column

## Prerequisites

This topic assumes **Module 13 — Indexes** (see the [Module 13 overview](../13-indexes/00-module-overview.md) and [What Is an Index?](../13-indexes/01-what-is-an-index.md)), since partitioning and indexing solve overlapping performance problems for large tables in complementary ways, and this topic contrasts them directly. It also assumes ordinary comfort with `CREATE TABLE` from [Creating Tables](../04-database-and-table-design/02-creating-tables.md) (Module 4) and with primary/unique keys from Module 5 (see [Primary Keys](../05-constraints-and-keys/03-primary-keys.md)), since this topic's most important trade-off concerns exactly those constraints.

## Motivation

An index (Module 13) helps a database find rows within a table quickly, but it doesn't shrink the table itself — a table with 500 million rows still has 500 million rows, an enormous index still spanning all of them, and maintenance operations (statistics updates, index rebuilds) that still have to consider the whole thing, even if the vast majority of queries only ever care about the most recent, small slice of that data. At a certain scale, the table itself becomes the bottleneck, not just the absence of an index. Table partitioning addresses this by physically splitting one enormous logical table into several smaller, self-contained physical pieces — while still letting every query address it as if it were one ordinary table.

## Problem Statement

Imagine an `events` table logging application activity: 500 million rows spanning five years of history. In practice:

- The overwhelming majority of queries only ever ask about the last 30 days of activity, yet an index on `created_at` still has to represent all five years of history, making it far larger — and every maintenance operation on it far slower — than it needs to be for the queries that actually run.
- A data retention policy requires deleting everything older than two years. `DELETE FROM events WHERE created_at < '2024-01-01'` would have to individually find, log, and remove potentially hundreds of millions of rows — a massive, slow, heavily-logged operation that can bloat the table and block other activity for a long time.
- Even routine maintenance (`VACUUM`, statistics collection) has to sweep the entire multi-year table, most of which never changes and is rarely even read.

Partitioning restructures this table so that "the last 30 days" and "everything from 2021" are genuinely separate physical objects, each with its own, much smaller footprint — while every query can still simply say `FROM events` and get correct results.

## Concept

### What Partitioning Is

**Table partitioning** splits one logical table into multiple physical **partitions**, each of which is a real, separate table under the hood, while a query addressed to the parent table transparently sees all of its partitions' rows combined, as if they were one table all along. Rows are routed to the correct partition automatically, based on the value of a designated **partition key** column.

### Partitioning Strategies

| Strategy | Rows are grouped by... | Typical use case |
|---|---|---|
| **RANGE** | A contiguous range of values (e.g., a date range) | Time-series data — the overwhelmingly common case: one partition per month or year |
| **LIST** | An explicit, discrete set of values | Data naturally grouped by a small number of known categories, e.g. region or status |
| **HASH** | A hash of the partition key, spread evenly across a fixed number of partitions | Evenly distributing load/size when there's no natural range or list boundary to partition by |

### Range Partitioning — Worked Example

```sql
CREATE TABLE events (
    id         BIGSERIAL,
    event_type TEXT NOT NULL,
    payload    JSONB,
    created_at DATE NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE events_2026 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

`events` itself is now the partitioned parent; `events_2024`, `events_2025`, and `events_2026` are its physical partitions, each responsible for one bounded range of `created_at` values (each range's lower bound is inclusive, its upper bound exclusive). Inserting into `events` routes each row to the correct partition automatically:

```sql
INSERT INTO events (event_type, payload, created_at) VALUES
    ('login',    '{"user_id": 42}',                  '2025-06-01'),
    ('purchase', '{"user_id": 42, "amount": 99.99}', '2026-03-15');

SELECT tableoid::regclass AS physical_partition, id, event_type, created_at
FROM events
ORDER BY created_at;
```

```
 physical_partition | id | event_type | created_at
---------------------+----+------------+------------
 events_2025         |  1 | login      | 2025-06-01
 events_2026         |  2 | purchase   | 2026-03-15
(2 rows)
```

`tableoid::regclass` reveals which physical partition actually stores each row — the query itself never had to mention a partition by name; it addressed `events`, and PostgreSQL routed and retrieved rows across the correct partitions transparently.

### List Partitioning — Worked Example

```sql
CREATE TABLE orders (
    id      BIGSERIAL,
    region  TEXT NOT NULL,
    amount  NUMERIC(10, 2) NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE orders_us    PARTITION OF orders FOR VALUES IN ('US');
CREATE TABLE orders_eu    PARTITION OF orders FOR VALUES IN ('DE', 'FR', 'ES');
CREATE TABLE orders_other PARTITION OF orders DEFAULT;
```

`orders_other`, declared `DEFAULT`, catches any row whose `region` doesn't match any explicitly listed partition — without a `DEFAULT` partition, an insert with an unmatched value would simply fail.

### Hash Partitioning — Worked Example

```sql
CREATE TABLE sessions (
    id      BIGSERIAL,
    user_id INTEGER NOT NULL,
    data    JSONB
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Here, `user_id`'s hash value modulo 4 decides which of the four partitions a row lands in, spreading rows evenly across all four regardless of the actual distribution of `user_id` values. Hash partitioning is used purely to distribute *size and load* evenly — unlike range or list partitioning, a hash reveals nothing about the ordering of values, so a query like `WHERE user_id = 42` cannot be pruned down to "obviously partition 2" without the planner computing that hash itself (which it does automatically), but a query like `WHERE user_id BETWEEN 1 AND 100` gains no benefit at all, since consecutive ids are deliberately scattered across every partition.

### Query Pruning

The single biggest performance benefit of range/list partitioning: when a query's `WHERE` clause constrains the partition key, the planner can determine, before touching any data at all, which partitions could not possibly contain a matching row — and skip opening them entirely. This is called **partition pruning**.

```sql
EXPLAIN SELECT * FROM events WHERE created_at >= '2026-01-01';
```

```
 Append  (cost=0.00..35.50 rows=820 width=72)
   ->  Seq Scan on events_2026  (cost=0.00..35.50 rows=820 width=72)
         Filter: (created_at >= '2026-01-01'::date)
```

Notice the plan only mentions `events_2026` — `events_2024` and `events_2025` are never scanned at all, because the planner can statically prove, from the partition bounds declared in `CREATE TABLE ... PARTITION OF ... FOR VALUES FROM ... TO ...`, that no row in either of those partitions could possibly satisfy `created_at >= '2026-01-01'`. Compare this to an equivalent, unpartitioned table with only a B-tree index (Module 13): an index scan there would still need to consult the index structure itself (much faster than a full scan, but not "provably zero relevant partitions," the way pruning achieves).

### Easier Bulk-Delete of Old Data

Instead of:

```sql
DELETE FROM events WHERE created_at < '2024-01-01';
```

— which has to individually locate, log, and remove every matching row, however many hundreds of millions there might be — dropping an entire partition is a near-instant, largely metadata-only operation:

```sql
DROP TABLE events_2023;
```

If you'd rather keep the old partition around (for archival, or to double-check before permanently discarding it) without it being part of `events` any longer:

```sql
ALTER TABLE events DETACH PARTITION events_2023;
```

`DETACH PARTITION` disconnects the partition from its parent, leaving it as a completely ordinary, standalone table you can inspect, back up, or drop later, without it appearing in any query against `events` from that point on.

## Internal Working (Preview)

In PostgreSQL's modern (declarative) partitioning, the parent table (`events`) holds essentially no rows or storage of its own — it exists as a logical umbrella and routing definition. Each partition is a genuinely distinct physical table with its own storage files, and (though you typically define indexes once on the parent, and PostgreSQL propagates them) effectively its own index structures and its own planner statistics.

```
                events  (logical parent — no storage of its own)
                   │
      ┌────────────┼────────────┐
      ▼            ▼            ▼
 events_2024   events_2025   events_2026
 (physical)    (physical)    (physical)
```

On `INSERT`, PostgreSQL evaluates the partition key's value against each partition's declared bounds and routes the row to the correct physical partition automatically — this routing, and the pruning shown above, both happen using the same declared bounds metadata, checked once at plan time (for pruning) or once per row (for insert routing), rather than by scanning data.

## Real-World Analogy

Picture a large physical records room, organized as separate labeled filing cabinets by year, all inside one room labeled "Records" at the door. A clerk asked to retrieve "everything from 2026" walks straight to the 2026 cabinet and never opens the others at all — that's partition pruning. When 2020's records are no longer needed under the retention policy, the entire 2020 cabinet is wheeled out and disposed of as one unit, rather than a clerk pulling and shredding a hundred thousand individual folders one at a time — that's dropping a partition instead of running a row-by-row `DELETE`.

## Why Partitioning Was Designed This Way

An index (Module 13) makes finding rows within a table faster, but the index itself, and every maintenance operation touching the table, still scales with the *entire* table's size — a natural limit once a table's size and access pattern become lopsided (mostly old, rarely-touched data, alongside a small, heavily-queried recent slice). Partitioning addresses this at the physical storage layer directly, mirroring how real-world archival systems separate active, frequently-accessed records from historical ones physically, rather than keeping everything in one undifferentiated pile and relying entirely on a lookup aid (an index) to compensate. It's a complementary technique to indexing, not a replacement for it — a well-partitioned, time-series table typically still has ordinary indexes defined on each of its (much smaller) individual partitions.

## Advantages

- **Query pruning** dramatically reduces the amount of data scanned for queries that filter on the partition key, especially common time-range queries.
- **Near-instant bulk removal of old data** — dropping or detaching a partition is a metadata-level operation, not a logged, row-by-row `DELETE`.
- **Scoped maintenance** — operations like `VACUUM` or statistics collection can be limited to a single, much smaller partition instead of an entire enormous table.
- **Smaller, faster-to-maintain indexes per partition** — a new partition starts fresh and small, rather than inheriting the bulk of years of accumulated history.

## Disadvantages / Limitations

- **Added schema complexity** — partitions generally must be created ahead of time (or via an automated process) before data needing them arrives; a query that doesn't filter on the partition key gains no pruning benefit and may see a small amount of added overhead from the parent's routing logic.
- **A `UNIQUE` or `PRIMARY KEY` constraint on a partitioned table must include the partition key as part of the key** — PostgreSQL cannot cheaply enforce uniqueness *across* separate physical partitions the way it can within a single table's own index, so the partition column is required to be part of any such constraint. This is a real, sometimes surprising restriction to design around, not an oversight.
- **Choosing the wrong partition key or granularity can make things worse, not better** — partitioning by a column queries never filter on gains nothing, and creating far too many small partitions (say, one per day for years of history) adds real per-partition planner and catalog overhead that can outweigh the benefit.

## Best Practices

- Partition by the column your queries overwhelmingly filter on — for time-series data, this is almost always a timestamp or date column, partitioned by month or year depending on data volume.
- Keep the number of partitions reasonable (generally dozens to a few hundred, not tens of thousands) — the planner still incurs some per-partition overhead regardless of pruning.
- Automate the creation of future partitions ahead of the data that will need them (e.g., a scheduled job that creates next month's partition before month-end), rather than relying on someone to remember.
- Always include the partition key column in any `UNIQUE`/`PRIMARY KEY` constraint on a partitioned table.
- Don't partition a table purely because it's large — the complexity is only worth it when your actual query and deletion patterns benefit from pruning or bulk-partition-removal; a large table that's always queried by a non-partition-key column, in full, gains little from partitioning alone.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Partitioning by a column queries rarely filter on | Gains none of pruning's performance benefit, while still adding the schema and operational complexity of managing partitions. |
| Defining a `PRIMARY KEY` on a partitioned table without including the partition column | PostgreSQL rejects it — a partitioned table's unique/primary key must include every partition key column, since uniqueness can only be enforced within a single partition's own index, not centrally across all partitions at once. |
| Forgetting to create a partition for incoming data ahead of time | An insert whose partition-key value doesn't fall inside any existing partition's bounds fails outright, unless a `DEFAULT` partition exists to catch it. |
| Creating far too many, overly fine-grained partitions (e.g., one per day, kept for years) | Each partition carries its own catalog and planner overhead; excessive partitioning multiplies that overhead well past the point where pruning's benefit outweighs it. |

## Interview Questions

1. **Q: What specific problem does table partitioning solve that indexing alone does not?**
   A: An index makes finding rows within a table faster, but the index and every maintenance operation still scale with the entire table's size. Partitioning physically splits the table itself, so queries that filter on the partition key can skip entire partitions outright (pruning), and old data can be removed by dropping a whole partition instead of a slow, row-by-row `DELETE` — neither benefit comes from indexing alone.

2. **Q: What's the difference between range, list, and hash partitioning, and when would you use each?**
   A: Range partitioning groups rows by a contiguous range of values (most commonly dates, for time-series data). List partitioning groups rows by an explicit, discrete set of values (like region codes). Hash partitioning spreads rows evenly across a fixed number of partitions using a hash of the key, used purely to balance size/load when there's no natural range or list boundary to partition by.

3. **Q: Why must a `UNIQUE` or `PRIMARY KEY` constraint on a partitioned table include the partition key column?**
   A: Because uniqueness is enforced per-partition, using each partition's own index — PostgreSQL has no cheap way to check for a duplicate value across every separate physical partition the way it can within one table's single index. Requiring the partition key as part of the constraint guarantees that any two rows with the same key values are necessarily routed to (and checkable within) the same partition.

4. **Q: Why is dropping a partition so much faster than running an equivalent `DELETE` statement?**
   A: `DROP TABLE` (or `ALTER TABLE ... DETACH PARTITION`) on a partition is a metadata-level operation — it removes (or detaches) the whole physical table at once. A `DELETE` has to individually locate, log, and remove every matching row, which for hundreds of millions of rows is a massive, slow, heavily-logged operation by comparison.

## Summary

- Table partitioning splits one logical table into several physical partitions, while queries continue to address it as a single table.
- **Range** partitioning suits contiguous values (especially dates), **list** partitioning suits discrete known categories, and **hash** partitioning suits evenly spreading load when no natural range or list boundary exists.
- **Partition pruning** lets the query planner skip partitions that provably cannot contain matching rows, based purely on declared partition bounds — no data scanning required to rule them out.
- Dropping or detaching an entire partition is a near-instant, metadata-level operation, dramatically faster than an equivalent row-by-row `DELETE` for bulk-removing old data.
- A `UNIQUE`/`PRIMARY KEY` constraint on a partitioned table must include the partition key column, since uniqueness can only be enforced within a single partition.
- Partitioning is a complementary technique to indexing (Module 13), not a replacement for it, and is worth its added complexity only when your actual query and retention patterns genuinely benefit from pruning or bulk-partition removal.
- This is the final topic in Module 21 — the [Module Summary](06-module-summary.md) consolidates everything covered across all five topics.
