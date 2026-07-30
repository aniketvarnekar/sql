# Module 22 — SQL Across Databases

## Module Goal

By the end of this module, you will understand that everything you have learned in this course so far — while genuinely standard, portable SQL for the most part — was demonstrated on **one specific database product**, PostgreSQL, and that the other major relational databases in real-world use (MySQL, SQL Server, Oracle, SQLite) each diverge from it in specific, learnable ways. You will be able to look at a piece of MySQL, T-SQL, PL/SQL, or SQLite code and recognize what it is doing even though the keywords differ from what you have practiced, translate a PostgreSQL idiom (an auto-incrementing key, an upsert, a paginated query) into its equivalent on another engine, and make an informed, deliberate decision about when writing genuinely portable SQL is worth the extra discipline it requires — versus when committing fully to one database's dialect is the pragmatic choice. This module is the payoff of a promise made all the way back in Module 1: that PostgreSQL was chosen as a *teaching* reference, not because SQL only exists in one form.

## Topics Covered in This Module

1. **[MySQL Differences](01-mysql-differences.md)** — auto-increment columns, the missing `FULL OUTER JOIN` and its workaround, `LIMIT` syntax, case-sensitivity defaults, backtick identifier quoting, `ON DUPLICATE KEY UPDATE` upserts, and storage engines.
2. **[SQL Server (T-SQL) Differences](02-sql-server-differences.md)** — `TOP` vs. `LIMIT`, `IDENTITY` columns, square-bracket quoting, `GETDATE()`, `+` string concatenation, T-SQL's procedural extensions, and `MERGE` upserts.
3. **[Oracle Differences](03-oracle-differences.md)** — `ROWNUM`/`FETCH FIRST`, sequences and `NEXTVAL`, the `DUAL` table, `VARCHAR2`, PL/SQL, and `MERGE` upserts.
4. **[SQLite Differences](04-sqlite-differences.md)** — dynamic typing and type affinity, the absence of a user/role/permission system, limited `ALTER TABLE` support, single-writer concurrency, and why SQLite thrives anyway.
5. **[Writing Portable SQL](05-writing-portable-sql.md)** — a consolidated checklist of what is safely portable, what to isolate when portability matters, and a pragmatic framework for deciding whether to chase portability at all.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

This module assumes you have completed **Modules 1 through 21, in order** — it is the one module in this entire course that is fundamentally comparative, and every comparison in it is stated as "PostgreSQL does X (which you already know); database Y does Z instead." A few specific earlier touchpoints matter more than the rest:

- **Module 1 ([What Is SQL?](../01-introduction/02-what-is-sql.md))** — this is where the course first told you PostgreSQL was chosen as its reference database precisely *because* other databases diverge from the standard, and promised this module as the deep dive into where. Everything here fulfills that promise.
- **Module 5 ([Primary Keys](../05-constraints-and-keys/03-primary-keys.md))** and **Module 4 ([Creating Tables](../04-database-and-table-design/02-creating-tables.md))** — you need to already be completely comfortable with `SERIAL` and `GENERATED ALWAYS AS IDENTITY` as PostgreSQL's auto-incrementing key mechanisms before you can meaningfully compare them to `AUTO_INCREMENT`, `IDENTITY`, and Oracle sequences.
- **Module 6 ([INSERT](../06-modifying-data/01-insert.md))** — PostgreSQL's `INSERT ... ON CONFLICT` upsert syntax is the baseline every other database's upsert syntax in this module is measured against.
- **Module 10 ([FULL OUTER JOIN and CROSS JOIN](../10-joins-and-set-operations/03-full-outer-join-and-cross-join.md))** — you need a solid working understanding of what a `FULL OUTER JOIN` actually returns before you can understand why MySQL's lack of one is a real gap, and how to rebuild the same result set from `LEFT JOIN` and `RIGHT JOIN`.
- **Module 3 (Data Types)** — PostgreSQL's strict, declared-and-enforced column typing is the baseline this module contrasts against SQLite's radically different type affinity model.
- **Module 18 (Procedures, Functions & Triggers)** and **Module 19 (Security & Access Control)** — PostgreSQL's PL/pgSQL and its `GRANT`/`REVOKE` privilege model are the baselines this module contrasts against Oracle's PL/SQL and SQLite's near-total absence of a permission system, respectively.

## How to Study This Module

Read Topics 1 through 4 in the order given — this is a deliberate "distance from PostgreSQL" ordering, not an arbitrary one. MySQL (Topic 1) is the closest relative: same general relational model, mostly overlapping standard SQL, and a small, learnable list of concrete divergences. SQL Server (Topic 2) and Oracle (Topic 3) each add a full procedural language of their own (T-SQL, PL/SQL) and their own historical quirks (`IDENTITY`, sequences and `DUAL`), so by the time you reach them you should already be comfortable with the *pattern* of "same concept, different keyword" from the MySQL topic. SQLite (Topic 4) is the outlier — it is not just "PostgreSQL with different keywords" but a database built on a genuinely different philosophy (an embedded, single-file, dynamically-typed engine rather than a client-server, strictly-typed one), so read it slowly and resist the urge to map every SQLite behavior onto a PostgreSQL equivalent; sometimes there isn't one. Topic 5 (Writing Portable SQL) is the practical capstone of the whole module — read it last, after the specific divergences are fresh, so the checklist of "what's safe" and "what to isolate" actually means something concrete rather than being an abstract list of rules.
