# Module 10 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **INNER JOIN** — combining rows from two tables on a matching condition using explicit `JOIN...ON` syntax, why unmatched rows on either side are excluded, and using foreign keys as the natural join condition.
- [x] **LEFT and RIGHT OUTER JOIN** — preserving every row from one side with `NULL`s filling in for non-matches, why `LEFT JOIN` dominates in practice, and the `LEFT JOIN` + `IS NULL` "find rows with no match" pattern.
- [x] **FULL OUTER JOIN and CROSS JOIN** — preserving unmatched rows from both sides at once, generating a Cartesian product deliberately with `CROSS JOIN`, and the danger of an accidental cross join from a forgotten join condition.
- [x] **Self-Joins** — joining a table to itself with aliases, for hierarchical data (employee/manager) and for finding pairs or duplicates within one table.
- [x] **Joining Multiple Tables** — chaining three or more tables, consistent aliasing conventions, and why the query planner (not your written order) decides actual join execution order.
- [x] **UNION, INTERSECT, and EXCEPT** — combining result sets vertically instead of horizontally, `UNION` vs. `UNION ALL`'s deduplication cost, and the column-count/type-compatibility rule.

## Practical Connections

- **Every report or dashboard that shows "customer name next to their order history"** is running some variant of the joins covered in this module — pulling related data that was deliberately split across multiple tables (Module 4, Module 5) back into one readable result.
- **Data-quality auditing in any system with foreign key relationships** relies directly on the `LEFT JOIN`/`FULL OUTER JOIN` + `IS NULL` patterns from this module — finding orphaned records, unlinked accounts, or reconciliation gaps between two systems is exactly this technique applied to real production data.
- **A reporting system merging results from two independently-collected sources** (say, two regional sales databases, or two different signup channels feeding one combined contact list) reaches for `UNION`/`UNION ALL` rather than a join, because the goal is combining comparable lists, not relating rows through a shared key.
- **Every multi-level org chart, category tree, or comment thread display** in a real application is built on the self-join technique from this module for a single level, extended with recursive queries (Module 17) when arbitrary depth is required.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| `INNER JOIN` vs. `LEFT JOIN` | `INNER JOIN` keeps only rows matched on both sides; `LEFT JOIN` keeps everything `INNER JOIN` would, plus every unmatched row from the left (first-named) table, with `NULL`s filling the right side. |
| `LEFT JOIN` vs. `RIGHT JOIN` | Mirror images of each other — `LEFT JOIN` preserves the first-named table's unmatched rows, `RIGHT JOIN` preserves the second-named table's; any `RIGHT JOIN` can be rewritten as an equivalent `LEFT JOIN` by swapping table order, which is why `LEFT JOIN` dominates by convention. |
| `FULL OUTER JOIN` vs. `CROSS JOIN` | `FULL OUTER JOIN` still matches rows via an `ON` condition and additionally preserves unmatched rows from both sides; `CROSS JOIN` has no `ON` condition at all and pairs every row of one table with every row of the other, regardless of any relationship. |
| A filter in `ON` vs. the same filter in `WHERE` on an outer join | A condition in `ON` only restricts which rows are eligible to match, while every row from the preserved side is still kept; the identical condition in `WHERE` is applied after the join and discards `NULL`-filled unmatched rows, silently canceling the outer join's effect for that condition. |
| Joins vs. set operations (`UNION`/`INTERSECT`/`EXCEPT`) | Joins combine tables horizontally — matched rows gain more columns via an `ON` condition. Set operations combine result sets vertically — no matching condition exists; rows from two same-shaped `SELECT`s are simply stacked, deduplicated, intersected, or subtracted. |
| `UNION` vs. `UNION ALL` | `UNION` removes duplicate rows from the combined result, at the cost of extra sorting/hashing work; `UNION ALL` keeps every row, including duplicates, with no such cost — default to `UNION ALL` unless deduplication is specifically needed. |
| A self-join vs. an ordinary two-table join | Mechanically identical `JOIN...ON` syntax — the only difference is both sides reference the same underlying table, requiring two distinct aliases to disambiguate the table's two "roles" in the query. |

## What's Next

This module gave you the complete toolkit for combining data across tables — every join variant for combining rows horizontally, and every set operation for combining result sets vertically — which unblocks the overwhelming majority of realistic, multi-table questions you'll ever need to ask of a relational database. **Module 11 — Subqueries** builds directly on top of this: instead of always joining two tables side by side, you'll learn to nest one query inside another (as a filtering condition, a computed column, or a stand-in for a table), including correlated subqueries and the `EXISTS`/`ANY`/`ALL` operators — often solving problems that would otherwise require awkward or inefficient joins, and setting up the reasoning you'll need later for window functions (Module 16) and common table expressions (Module 17).
