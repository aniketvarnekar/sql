# ROLLUP and CUBE

## Learning Objectives

By the end of this section you should be able to:
- Explain the reporting problem `ROLLUP`, `CUBE`, and `GROUPING SETS` solve, and why manually `UNION ALL`-ing several `GROUP BY` queries is a worse solution to that same problem
- Write a `GROUP BY ROLLUP(...)` query and correctly interpret its subtotal and grand-total rows
- Write a `GROUP BY CUBE(...)` query and explain how its output differs from the equivalent `ROLLUP`
- Use `GROUPING SETS` as the general form underlying both, and use the `GROUPING()` function to distinguish a real `NULL` from a subtotal marker

## Prerequisites

- [GROUP BY](02-group-by.md) — `ROLLUP` and `CUBE` are extensions of `GROUP BY`; you need to already be comfortable with basic and multi-column grouping before adding automatic subtotal generation on top of it.
- [HAVING vs. WHERE](03-having-vs-where.md) — understanding that `GROUP BY` happens before `HAVING`/`SELECT` in the logical order helps place where `ROLLUP`'s extra subtotal rows fit into that same pipeline.

## Motivation

A huge fraction of real-world reporting queries share one shape: "revenue by region, with a subtotal for each region and a grand total at the bottom" — the exact structure of a paper or spreadsheet report a finance team has been building by hand for decades. Plain `GROUP BY` gives you the per-region breakdown but not the subtotal or grand-total rows layered on top of it. You could construct those extra rows yourself with several separate queries glued together — but that approach is exactly the kind of tedious, error-prone, multi-scan solution `GROUP BY` itself was designed to eliminate back in Topic 2, and SQL has a dedicated, declarative answer to it.

## Problem Statement

Suppose a report needs, in one result set: the revenue for every (region, category) combination, a subtotal per region across all its categories, and a single grand total across everything. Using only what Topics 1–3 taught you, you'd have to write three separate queries and glue them together with `UNION ALL`:

```sql
-- Detail: revenue per region and category
SELECT region, category, SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region, category

UNION ALL

-- Subtotal: revenue per region, with category set to NULL as a placeholder
SELECT region, NULL AS category, SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY region

UNION ALL

-- Grand total: revenue across everything, with both columns set to NULL
SELECT NULL AS region, NULL AS category, SUM(quantity * unit_price) AS total_revenue
FROM orders

ORDER BY region NULLS LAST, category NULLS LAST;
```

This works, but look at the cost: the table is logically scanned three separate times (once per `SELECT`), the `NULL` placeholders in the second and third queries have to be manually type-matched to line up with the first query's columns, and every time a new level of subtotal is needed (say, a subtotal per category *regardless* of region), you have to hand-write yet another `SELECT ... UNION ALL` block, get its `NULL` placeholders exactly right, and hope you didn't introduce a copy-paste mistake anywhere in the process. `ROLLUP`, `CUBE`, and `GROUPING SETS` exist to generate exactly this kind of multi-level subtotal report in a single `GROUP BY` clause, computed in one pass over the data.

## Concept

### GROUP BY ROLLUP — Hierarchical Subtotals

`ROLLUP` takes a list of columns and produces the full detail grouping, then progressively "rolls up" by dropping the rightmost column, one at a time, until nothing is left — producing a subtotal at each level plus one grand total:

```sql
SELECT
    region,
    category,
    SUM(quantity * unit_price) AS total_revenue,
    COUNT(*)                   AS order_count
FROM orders
GROUP BY ROLLUP(region, category)
ORDER BY region NULLS LAST, category NULLS LAST;
```

```
 region | category         | total_revenue | order_count
--------+------------------+-----------------+---------------
 East   | Electronics      |        2498.00 |             3
 East   | Furniture        |        1000.00 |             2
 East   | Office Supplies  |         125.00 |             1
 East   |                  |        3623.00 |             6
 North  | Electronics      |         800.00 |             1
 North  | Furniture        |         350.00 |             1
 North  | Office Supplies  |         120.00 |             1
 North  |                  |        1270.00 |             3
 South  | Electronics      |         880.00 |             1
 South  | Furniture        |         600.00 |             1
 South  | Office Supplies  |         100.00 |             1
 South  |                  |        1580.00 |             3
 West   | Electronics      |         750.00 |             2
 West   | Furniture        |         400.00 |             1
 West   | Office Supplies  |         108.00 |             1
 West   |                  |        1258.00 |             4
        |                  |        7731.00 |            16
(17 rows)
```

Read this as three tiers, all produced by one query:
- **12 detail rows** — one per (region, category) combination, identical to plain `GROUP BY region, category` from Topic 2.
- **4 subtotal rows** — one per region, with `category` blank (`NULL`), summing across all of that region's categories (East's subtotal, $3,623.00, matches Topic 2's region-only breakdown exactly).
- **1 grand-total row** — both `region` and `category` blank, summing across the entire table ($7,731.00, matching the module's running total from Topic 1).

`ROLLUP` is **order-sensitive**: `ROLLUP(region, category)` rolls up category first, then region — producing per-region subtotals as shown above. `ROLLUP(category, region)` would instead produce per-category subtotals (rolling up region within each category first), which is a genuinely different report, not just a cosmetic reordering of the same rows.

### Telling a Subtotal Row Apart from a Real NULL: GROUPING()

The blank cells above are `NULL` — but they're a different *kind* of `NULL` than "this order genuinely had no discount code" from Topic 1. They mean "this row is a subtotal across all values of this column," not "the value is unknown." If a table's `category` column could itself legitimately contain `NULL` for some other reason, you'd have no way to tell the two apart just by looking at a blank cell. PostgreSQL's `GROUPING()` function resolves this ambiguity:

```sql
SELECT
    region,
    category,
    SUM(quantity * unit_price) AS total_revenue,
    GROUPING(region)   AS region_is_subtotal,
    GROUPING(category) AS category_is_subtotal
FROM orders
GROUP BY ROLLUP(region, category)
ORDER BY region NULLS LAST, category NULLS LAST
LIMIT 5;
```

```
 region | category    | total_revenue | region_is_subtotal | category_is_subtotal
--------+--------------+-----------------+-----------------------+-------------------------
 East   | Electronics |        2498.00 |                     0 |                       0
 East   | Furniture   |        1000.00 |                     0 |                       0
 East   | Office Supplies |     125.00 |                     0 |                       0
 East   |             |        3623.00 |                     0 |                       1
 North  | Electronics |         800.00 |                     0 |                       0
(5 rows)
```

`GROUPING(column)` returns `1` when that column has been "rolled up" (aggregated over) to produce the current row, and `0` when the row still reflects an actual value for that column. The East subtotal row shows `category_is_subtotal = 1`, unambiguously marking it as a subtotal rather than a real `(East, NULL)` combination — this is the mechanism a reporting layer would use to bold a subtotal row or label it "Total" instead of printing a blank.

### GROUP BY CUBE — Every Possible Combination of Subtotals

`ROLLUP` only produces subtotals along one hierarchy (detail → per-region subtotal → grand total). `CUBE` goes further: it produces subtotals for **every possible combination** of the listed columns, including ones `ROLLUP` never generates — in this case, a subtotal per *category*, regardless of region:

```sql
SELECT
    region,
    category,
    SUM(quantity * unit_price) AS total_revenue,
    COUNT(*)                   AS order_count
FROM orders
GROUP BY CUBE(region, category)
ORDER BY region NULLS LAST, category NULLS LAST;
```

```
 region | category         | total_revenue | order_count
--------+------------------+-----------------+---------------
 East   | Electronics      |        2498.00 |             3
 East   | Furniture        |        1000.00 |             2
 East   | Office Supplies  |         125.00 |             1
 East   |                  |        3623.00 |             6
 North  | Electronics      |         800.00 |             1
 North  | Furniture        |         350.00 |             1
 North  | Office Supplies  |         120.00 |             1
 North  |                  |        1270.00 |             3
 South  | Electronics      |         880.00 |             1
 South  | Furniture        |         600.00 |             1
 South  | Office Supplies  |         100.00 |             1
 South  |                  |        1580.00 |             3
 West   | Electronics      |         750.00 |             2
 West   | Furniture        |         400.00 |             1
 West   | Office Supplies  |         108.00 |             1
 West   |                  |        1258.00 |             4
        | Electronics      |        4928.00 |             7
        | Furniture        |        2350.00 |             5
        | Office Supplies  |         453.00 |             4
        |                  |        7731.00 |            16
(20 rows)
```

The first 16 rows are identical to the `ROLLUP` result — every detail row and every region subtotal. The three new rows near the bottom (`Electronics` / `Furniture` / `Office Supplies`, each with `region` blank) are subtotals **across all regions, per category** — a breakdown `ROLLUP(region, category)` never produces, because rolling up strictly drops columns right-to-left (region, then category), never leaving category alone with region dropped. `CUBE` computes subtotals for every subset of the grouping columns: `{region, category}`, `{region}`, `{category}`, and `{}` (the grand total) — four subsets for two columns, which is why `CUBE` on N columns produces up to 2ᴺ distinct grouping combinations, growing far faster than `ROLLUP`'s N+1 combinations as more columns are added.

### GROUPING SETS — the General Form

`ROLLUP` and `CUBE` are both convenient shorthand for the same, more general underlying feature: `GROUPING SETS`, which lets you specify *exactly* which grouping combinations you want, with no built-in hierarchy or combinatorial expansion assumed:

```sql
-- ROLLUP(region, category) is exactly equivalent to:
GROUP BY GROUPING SETS ((region, category), (region), ())

-- CUBE(region, category) is exactly equivalent to:
GROUP BY GROUPING SETS ((region, category), (region), (category), ())
```

`GROUPING SETS` becomes genuinely useful (rather than just an equivalent way to spell `ROLLUP`/`CUBE`) when you want a set of breakdowns that matches *neither* built-in pattern — for example, a report that wants region subtotals and category subtotals side by side, but has no use for the full (region, category) detail rows or a grand total:

```sql
SELECT
    region,
    category,
    SUM(quantity * unit_price) AS total_revenue
FROM orders
GROUP BY GROUPING SETS ((region), (category))
ORDER BY region NULLS LAST, category NULLS LAST;
```

```
 region | category         | total_revenue
--------+------------------+-----------------
 East   |                  |        3623.00
 North  |                  |        1270.00
 South  |                  |        1580.00
 West   |                  |        1258.00
        | Electronics      |        4928.00
        | Furniture        |        2350.00
        | Office Supplies  |         453.00
(7 rows)
```

Seven rows — four region totals and three category totals — with none of the twelve detailed (region, category) rows and no grand total, because none of `(region, category)`, `()` were included in the `GROUPING SETS` list. This exact combination has no `ROLLUP`/`CUBE` shorthand; `GROUPING SETS` is the only way to express it declaratively, in a single pass, without resorting to the manual `UNION ALL` approach from the Problem Statement.

## Internal Working (Preview)

A naive execution of the manual `UNION ALL` version from the Problem Statement would scan the `orders` table three separate times — once per `SELECT`. PostgreSQL's query planner, when it sees `ROLLUP`, `CUBE`, or `GROUPING SETS`, instead computes every required grouping combination in a **single pass** over the data, typically using a plan node you'll see labeled `MixedAggregate` if you run `EXPLAIN` on one of these queries (a tool covered fully in Module 13 — Indexes and Module 20 — Performance Tuning):

```
 All qualifying rows (after WHERE)
        │
        ▼
 One scan, feeding EVERY requested grouping combination at once:
   ┌───────────────────┬───────────────────┬───────────────────┐
   │ (region, category) │      (region)      │        ()          │
   │  12 group buckets   │    4 group buckets  │   1 grand total    │
   └───────────────────┴───────────────────┴───────────────────┘
        │                       │                      │
        ▼                       ▼                      ▼
   detail aggregates      subtotal aggregates      grand total
        │                       │                      │
        └───────────────────────┴──────────────────────┘
                              │
                              ▼
                 all rows combined into ONE result set
```

Every row that streams past is simultaneously fed into all of the requested grouping combinations' running accumulators in that single pass — the database never re-reads the table once per combination the way the manual `UNION ALL` approach logically requires. This is the concrete efficiency argument for reaching for `ROLLUP`/`CUBE`/`GROUPING SETS` over hand-written `UNION ALL` queries: fewer scans of the underlying data for the same reporting result.

## Real-World Analogy

Think of a store receipt at checkout. The register doesn't hand you a separate slip for every single item, another slip listing category subtotals, and a third slip with just the grand total — it prints **one receipt**, itemizing every product, with a subtotal line under each category as it goes, and a single grand total at the very bottom, all generated from one pass through your cart at the register. `ROLLUP`/`CUBE` are the database equivalent of that single, structured receipt — detail lines, subtotal lines, and a grand-total line, produced together from one register "run," rather than being reconstructed afterward by taping together three separately printed slips.

## Why ROLLUP, CUBE, and GROUPING SETS Were Designed This Way

Subtotal-and-grand-total reporting is common enough, across virtually every business domain, that the SQL standard added these as first-class, declarative extensions to `GROUP BY` rather than leaving every database user to reinvent the `UNION ALL` pattern from the Problem Statement by hand, every time. This is the same declarative philosophy from Module 1 applied one level higher: instead of specifying *how* to assemble a multi-level subtotal report (three separate queries, hand-aligned `NULL` placeholders, unioned together), you specify *what* levels of subtotal you want (`ROLLUP(region, category)`, or a specific `GROUPING SETS` list), and the query planner determines the most efficient single-pass way to compute all of them together. `GROUPING()` exists specifically to preserve correctness in the face of this convenience — since the mechanism reuses `NULL` as a subtotal marker, a dedicated function is needed to keep that marker distinguishable from a genuine `NULL` value in the underlying data.

## Advantages

- **Single scan, multiple grouping levels** — one query computes detail rows and every requested subtotal level together, rather than the database re-scanning the table once per level.
- **Guaranteed internal consistency** — because every subtotal is computed from the exact same single read of the data, there's no risk of a detail query and a separately run subtotal query disagreeing due to data changing between the two (a real risk with the manual `UNION ALL` approach if the table is being written to concurrently).
- **Far less hand-written SQL** — no manually aligned `NULL` placeholders across several `UNION ALL`-ed `SELECT`s, and no risk of a copy-paste mistake in one of those placeholder queries.
- **`GROUPING SETS` covers any combination a report needs** — not limited to the strict hierarchy `ROLLUP` assumes or the full combinatorial expansion `CUBE` produces.

## Disadvantages / Limitations

- **Mixed granularity in one result set requires care** — detail rows, subtotal rows, and the grand-total row are all mixed together in the same columns, and a consumer of the raw result (a report renderer, a downstream query) must use `GROUPING()` to tell them apart correctly rather than assuming every blank is a genuine `NULL`.
- **`CUBE` grows combinatorially** — `CUBE` on N columns produces up to 2ᴺ grouping combinations; with more than a handful of columns, this can generate an enormous, mostly-uninteresting result set, most of which no report actually needs.
- **Order of columns in `ROLLUP` changes the report's meaning** — `ROLLUP(region, category)` and `ROLLUP(category, region)` produce different subtotal hierarchies, not the same rows in a different order; picking the wrong order silently produces the wrong subtotal breakdown.
- **Vendor support and syntax vary** — `ROLLUP`, `CUBE`, and `GROUPING SETS` are part of the SQL standard and PostgreSQL supports all three exactly as shown here, but older versions of some other database products historically supported only a subset, or used different syntax; see Module 22 (SQL Across Databases) if you need to write one of these reports against a non-PostgreSQL system.

## Best Practices

- Reach for `ROLLUP` when your report has a natural hierarchy (region, then region-and-category) and you want subtotals along that hierarchy plus a grand total.
- Reach for `CUBE` only when you genuinely need subtotals for *every* combination of the grouping columns — for two columns this is a modest four-way expansion, but don't reach for it out of habit on wider column lists without considering the resulting row count.
- Reach for `GROUPING SETS` directly whenever the exact combination of breakdowns you need doesn't match `ROLLUP`'s hierarchy or `CUBE`'s full combinatorial expansion — it's the general form both are built from, and stating your needed combinations explicitly is often clearer than fighting for special-case output from `ROLLUP`/`CUBE`.
- Always use `GROUPING(column)` (rather than an `IS NULL` check on the column itself) to detect subtotal and grand-total rows in application or reporting code — an `IS NULL` check cannot distinguish a genuine `NULL` value in the data from a `ROLLUP`/`CUBE`-generated subtotal marker.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Treating a `NULL` in a `ROLLUP`/`CUBE` result as if it means "no value recorded" | That `NULL` is a subtotal/grand-total marker meaning "aggregated across all values of this column," not a record of missing data — use `GROUPING(column)` to tell the two apart reliably. |
| Assuming `ROLLUP(a, b)` and `ROLLUP(b, a)` produce the same report | They produce different subtotal hierarchies: `ROLLUP(a, b)` subtotals by `a` (rolling up `b`), while `ROLLUP(b, a)` subtotals by `b` (rolling up `a`) — column order determines which level gets the intermediate subtotal row. |
| Reaching for `CUBE` when only a simple region-then-category hierarchy is needed | `CUBE` additionally computes subtotals for every other combination (here, per-category-regardless-of-region), producing extra rows a hierarchical `ROLLUP` report has no use for and would have to filter back out. |
| Rebuilding a `ROLLUP`/`CUBE`-style report by hand with several `UNION ALL`-ed `GROUP BY` queries | This is exactly the tedious, multi-scan, manually-`NULL`-padded approach `ROLLUP`/`CUBE`/`GROUPING SETS` exist to replace — it requires more code, risks column-alignment mistakes, and typically requires scanning the underlying table once per unioned query instead of once overall. |

## Interview Questions

1. **Q: What does `GROUP BY ROLLUP(a, b)` produce that plain `GROUP BY a, b` does not?**
   A: In addition to the full detail rows (one per unique combination of `a` and `b`), `ROLLUP` adds subtotal rows for each distinct value of `a` (summed across all its `b` values, with `b` shown as `NULL`), plus one grand-total row (both `a` and `b` shown as `NULL`).

2. **Q: What is the difference between `ROLLUP(a, b)` and `CUBE(a, b)`?**
   A: `ROLLUP` produces a hierarchical set of subtotals — full detail, subtotal by `a`, and grand total — following a fixed left-to-right roll-up order. `CUBE` produces subtotals for every possible combination of the columns: full detail, subtotal by `a`, subtotal by `b` alone, and grand total — `CUBE`'s output is a strict superset of `ROLLUP`'s for the same two columns.

3. **Q: How does `GROUPING SETS` relate to `ROLLUP` and `CUBE`?**
   A: `GROUPING SETS` is the general-purpose mechanism both are shorthand for. `ROLLUP(a, b)` is equivalent to `GROUPING SETS ((a, b), (a), ())`, and `CUBE(a, b)` is equivalent to `GROUPING SETS ((a, b), (a), (b), ())`. `GROUPING SETS` lets you specify any arbitrary list of grouping combinations directly, including ones that match neither built-in pattern.

4. **Q: How do you distinguish a subtotal row's `NULL` from a genuine `NULL` value stored in the underlying data, when using `ROLLUP` or `CUBE`?**
   A: Use the `GROUPING(column)` function in the `SELECT` list. It returns `1` for a row where that column has been aggregated over (a subtotal or grand-total row) and `0` where the column reflects an actual grouped value — including a genuine stored `NULL`, which `GROUPING()` would still report as `0` since it wasn't rolled up.

## Summary

- `ROLLUP`, `CUBE`, and `GROUPING SETS` extend `GROUP BY` to produce detail rows plus one or more levels of subtotal and grand-total rows, all in a single query and a single scan of the data.
- `ROLLUP(a, b)` produces a hierarchy: full detail, subtotals by `a`, and one grand total — and is order-sensitive (`ROLLUP(a, b)` ≠ `ROLLUP(b, a)`).
- `CUBE(a, b)` produces subtotals for every combination of the listed columns — a strict superset of the equivalent `ROLLUP`'s rows — growing combinatorially (up to 2ᴺ combinations) as more columns are added.
- `GROUPING SETS` is the general form underlying both, letting you specify an arbitrary, exact list of grouping combinations when neither `ROLLUP`'s hierarchy nor `CUBE`'s full expansion matches the report you need.
- `GROUPING(column)` reliably distinguishes a subtotal/grand-total row's placeholder `NULL` from a genuine `NULL` value stored in the data — never assume a blank cell in `ROLLUP`/`CUBE` output means "no data."
- All three replace what would otherwise require several hand-written `GROUP BY` queries stitched together with `UNION ALL` and manually aligned `NULL` placeholders — a slower, more error-prone, multi-scan approach to the exact same report.
