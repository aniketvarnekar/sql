# Module 22 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **MySQL Differences** — `AUTO_INCREMENT` vs. `SERIAL`/`IDENTITY`, the missing `FULL OUTER JOIN` and its `UNION`-based workaround, `LIMIT` syntax overlap, default case-insensitive string comparison, backtick identifier quoting, `ON DUPLICATE KEY UPDATE` upserts, and InnoDB as the default storage engine.
- [x] **SQL Server (T-SQL) Differences** — `TOP` and standard-SQL `OFFSET`/`FETCH` vs. `LIMIT`, `IDENTITY(seed, increment)` columns, square-bracket identifier quoting, `GETDATE()` vs. `NOW()`, `+` vs. `||` string concatenation, T-SQL as SQL Server's procedural extension, and `MERGE` upserts.
- [x] **Oracle Differences** — `ROWNUM`'s pre-sort numbering trap and modern `FETCH FIRST`, sequences with explicit `NEXTVAL` vs. auto-generated identity, the `DUAL` table, `VARCHAR2`, PL/SQL as Oracle's procedural extension, and `MERGE` (which Oracle originated).
- [x] **SQLite Differences** — dynamic typing and type affinity vs. PostgreSQL's strict typing, the total absence of a `GRANT`/`REVOKE`-style permission system, historically limited `ALTER TABLE` support and its create-copy-drop-rename workaround, single-writer-at-a-time concurrency, and why SQLite thrives in its embedded, single-file niche regardless.
- [x] **Writing Portable SQL** — a consolidated checklist of the portable core (Modules 7–11) versus the categories that must be isolated (auto-increment, upsert, pagination, procedural code), the pragmatic case for committing to one database, and the specific circumstances where genuine portability is worth its cost.

## Practical Connections

- **A software vendor selling an on-premises product to enterprise customers** routinely has to support whichever database a given customer already operates — commonly PostgreSQL for smaller or cloud-native customers and Oracle or SQL Server for large, established enterprises — and the auto-increment, upsert, and pagination translation tables in this module are exactly what an engineer maintaining that product needs on hand.
- **A developer reading unfamiliar legacy code during a migration or acquisition** will routinely encounter Oracle's `ROWNUM` subquery pattern, T-SQL's `+`-based string concatenation, or MySQL's `ON DUPLICATE KEY UPDATE` — recognizing these on sight, rather than needing to look each one up from scratch, is a direct, practical skill this module built.
- **A mobile or desktop application storing local data** almost always reaches for SQLite specifically because of, not despite, its serverless, single-file, embedded design — understanding *why* that trade-off makes sense there (Topic 4) is what separates "SQLite is a limited database" from "SQLite is the correctly chosen tool for this particular job."
- **A team designing a new internal system's data layer** benefits directly from Topic 5's framework: most such teams should simply commit to one database and use its full feature set, reserving the isolation-and-abstraction effort of true portability for the much narrower set of situations (a shared library, an explicit multi-database product requirement) where it actually earns its cost.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| MySQL's `AUTO_INCREMENT` vs. PostgreSQL's `SERIAL` | Both auto-generate unique keys, but `AUTO_INCREMENT` is a table-metadata attribute while `SERIAL` creates a genuinely separate sequence database object — same concept, different mechanism. |
| SQL Server's `TOP` vs. `OFFSET`/`FETCH` | `TOP` limits row count but has no built-in row-skipping and predates the standard; `OFFSET`/`FETCH` is the standard-SQL form, requires `ORDER BY`, and supports true pagination — prefer it. |
| Oracle's `ROWNUM` vs. `FETCH FIRST` | `ROWNUM` numbers rows *before* `ORDER BY` sorts them at the same query level (a classic bug source requiring a subquery); `FETCH FIRST` applies after sorting, correctly, by construction. |
| SQLite's type affinity vs. PostgreSQL's strict typing | Affinity is a storage/conversion *preference* SQLite attempts but does not enforce; PostgreSQL's declared types are hard, always-enforced constraints — the same-looking `CREATE TABLE` statement means genuinely different guarantees on each database. |
| `ON CONFLICT` / `ON DUPLICATE KEY UPDATE` / `MERGE` | All three solve "insert, or update if it already exists," but `ON CONFLICT` (PostgreSQL) and `ON DUPLICATE KEY UPDATE` (MySQL) are purpose-built upsert clauses, while `MERGE` (SQL Server, Oracle) is a more general insert/update/delete reconciliation statement that happens to also cover upsert. |
| "Portable SQL" vs. "SQL that happens to run on one other database by luck" | Genuine portability means deliberately restricting to (or abstracting around) the categories this module identified as divergent; a query merely happening to also run on a second database, untested, is not the same guarantee. |

## What's Next

Module 22 completed the "how PostgreSQL fits into the wider world of relational databases" picture: you now know not just PostgreSQL's own syntax and behavior, built up across Modules 1 through 21, but precisely where and why MySQL, SQL Server, Oracle, and SQLite diverge from it, and how to reason about when portability across them is worth pursuing. **Module 23 — Interview Preparation** is the final module of this course: it consolidates interview-style questions and query-writing drills spanning everything learned from Module 1 onward — including the cross-database awareness this module just gave you, which is a frequent, realistic interview topic in its own right — into a single, structured review pass before you consider your SQL foundation complete.
