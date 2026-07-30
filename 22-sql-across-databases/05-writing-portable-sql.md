# Writing Portable SQL

## Learning Objectives

By the end of this section you should be able to:
- List the categories of SQL that are safely portable across PostgreSQL, MySQL, SQL Server, Oracle, and SQLite
- List the specific categories that reliably vary by database and must be isolated or abstracted if portability matters
- Explain the pragmatic argument for why most real applications should commit to one database rather than chase full portability
- Identify the specific circumstances under which writing genuinely portable SQL is actually worth the extra effort

## Prerequisites

- [MySQL Differences](01-mysql-differences.md), [SQL Server (T-SQL) Differences](02-sql-server-differences.md), [Oracle Differences](03-oracle-differences.md), and [SQLite Differences](04-sqlite-differences.md) — this topic is a consolidation of the specific divergences all four prior topics identified; it assumes you've seen the concrete examples, not just the category names.
- Modules 7 through 11 (Querying Basics, Functions & Expressions, Aggregation, Joins & Set Operations, Subqueries) — this topic explicitly claims this core is the portable foundation, so you need to already know what that core actually contains.

## Motivation

You now know a long list of specific ways PostgreSQL, MySQL, SQL Server, Oracle, and SQLite diverge. The natural next question is a practical one: given all of that, how do you actually decide, in a real project, what to write so it keeps working if the database underneath ever changes — and, just as importantly, when should you not even bother trying? This topic turns the previous four topics' specific facts into an actionable decision framework.

## Problem Statement

Imagine you're the lead engineer on a piece of software with a real, non-hypothetical requirement: it must run against whichever database each customer already has installed — some run PostgreSQL, some run MySQL, some run SQL Server. You cannot rewrite a separate version of every query for every customer's database by hand forever; that doesn't scale as the codebase grows. But you also know, from the last four topics, that auto-increment mechanics, upsert syntax, and pagination syntax all genuinely differ. Which parts of your SQL can you write once and trust to work everywhere, and which parts need a deliberate abstraction layer (a different code path per database, or a query-building library that generates the right dialect)? Getting this wrong in either direction is expensive: over-abstracting simple, portable SQL wastes enormous effort on problems that don't exist; under-abstracting the genuinely divergent parts causes production failures the moment a new database target is added.

## Concept

### What Is Safely Portable

The core SQL taught across Modules 7 through 11 of this course — the part of SQL that is closest to the ANSI/ISO standard and has the longest, most stable history across every major relational database — is, in practice, the safest possible foundation:

| Feature | Portable? | Notes |
|---|---|---|
| `SELECT`, `FROM`, `WHERE` | Yes | Core to every relational database covered in this module, essentially unchanged since the earliest SQL standard. |
| Standard comparison/logical operators (`=`, `<>`, `AND`, `OR`, `NOT`, `IN`, `BETWEEN`, `LIKE`) | Yes | Universally supported; minor wildcard/escaping differences exist but the operators themselves are universal. |
| `ORDER BY`, `GROUP BY`, `HAVING` | Yes | Standard clauses, behave consistently everywhere. |
| Standard aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) | Yes | Universal, from Module 9. |
| `INNER JOIN`, `LEFT JOIN` (`LEFT OUTER JOIN`), `RIGHT JOIN` | Yes | Universally supported, from Module 10 — only `FULL OUTER JOIN` (Topic 1 of this module) is the specific exception among joins. |
| `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT` (or `MINUS` on Oracle) | Mostly | The operations themselves are standard; Oracle spells `EXCEPT` as `MINUS` — a naming exception worth remembering, but not a structural gap. |
| Subqueries, `EXISTS`, `IN` with a subquery (Module 11) | Yes | Standard and consistently supported. |
| Basic `CREATE TABLE` with standard types (`INTEGER`, `TEXT`/`VARCHAR`, `DATE`) and `NOT NULL`/`UNIQUE`/`CHECK`/foreign key constraints (Module 5) | Mostly | The constraint *concepts* are universal; exact type names and enforcement strictness vary at the edges (Topic 4 of this module covered SQLite's type affinity as the sharpest exception). |
| `INSERT`, `UPDATE`, `DELETE` in their plain forms (Module 6) | Yes | The everyday, non-upsert forms of all three are standard and portable. |
| Standard-SQL `OFFSET ... FETCH NEXT ... ROWS ONLY` | Increasingly | Supported by PostgreSQL, modern Oracle (12c+), and SQL Server (2012+); notably not MySQL, which needs its own `LIMIT` syntax instead. |

If a query only uses features from this list, it is a very safe bet that it will run — with, at most, minor type-name adjustments — on any of the five databases covered in this module.

### What Reliably Varies and Must Be Isolated

The following categories, all covered in concrete detail across Topics 1–4 of this module, are exactly the parts of SQL that do **not** transfer as-is, and should be deliberately isolated behind an abstraction (a per-database code branch, a query-building library, or an object-relational mapping layer) if your application genuinely needs to run against more than one database:

| Category | Why it varies | Where covered |
|---|---|---|
| Auto-increment / identity mechanics | `SERIAL`/`GENERATED ALWAYS AS IDENTITY` (PostgreSQL), `AUTO_INCREMENT` (MySQL), `IDENTITY(seed, increment)` (SQL Server), sequences with `NEXTVAL` or `GENERATED ALWAYS AS IDENTITY` (Oracle), and SQLite's own `INTEGER PRIMARY KEY` rowid behavior are all different keywords and, in places, different underlying mechanisms. | Topics 1–3 |
| Upsert syntax | `ON CONFLICT` (PostgreSQL), `ON DUPLICATE KEY UPDATE` (MySQL), `MERGE` (SQL Server, Oracle) are structurally different statements, not just different keywords for the same shape. | Topics 1–3 |
| Pagination/row-limiting syntax | `LIMIT`/`OFFSET` (PostgreSQL, MySQL, SQLite) vs. `TOP` (SQL Server, legacy) vs. `ROWNUM` (Oracle, legacy) vs. the increasingly-shared `OFFSET`/`FETCH` standard form. | Topics 1–3 |
| Procedural code (functions, procedures, triggers) | PL/pgSQL (PostgreSQL, Module 18), T-SQL (SQL Server), PL/SQL (Oracle) are three genuinely different procedural languages with different syntax for variables, control flow, and error handling — none of this transfers at all between databases. | Topics 2–3 |
| Identifier quoting | Double quotes (PostgreSQL), backticks (MySQL), square brackets (SQL Server) — a trivial but real syntax incompatibility if any identifier ever needs quoting. | Topics 1–2 |
| String concatenation | `||` (PostgreSQL, Oracle, standard SQL) vs. `+` (SQL Server) vs. `CONCAT()` (works as a function on most, including MySQL). | Topic 2 |
| `FULL OUTER JOIN` | Missing entirely from MySQL; must be rebuilt from `LEFT JOIN`/`RIGHT JOIN`/`UNION`. | Topic 1 |
| Permission/role model | PostgreSQL's `GRANT`/`REVOKE` role system (Module 19) has no equivalent at all in SQLite. | Topic 4 |
| Type strictness | PostgreSQL's strictly enforced types vs. SQLite's type affinity model are a difference in fundamental philosophy, not just syntax — code relying on strict rejection of bad data will behave differently on SQLite by default. | Topic 4 |

### The Pragmatic Reality: Most Applications Should Commit to One Database

Despite the checklist above, the honest, experience-backed guidance is this: **most real-world applications should pick one database and commit to it fully**, using that database's specific features without apology, rather than trying to write SQL that runs unmodified everywhere. The reasoning:

- **Full portability has a real, ongoing cost** — every query must be written (or generated) in the lowest-common-denominator subset, or maintained in multiple dialect-specific versions, forever, for every future change. This tax is paid on every feature, indefinitely, whether or not the application ever actually runs on a second database.
- **Committing to one database lets you use its best features fully** — PostgreSQL's `ON CONFLICT`, its rich indexing options (Module 13), window functions (Module 16), and JSON support (Module 21) are all more capable, and simpler to write, than a lowest-common-denominator abstraction could express. Abstracting them away to preserve theoretical portability sacrifices real, immediate capability for a hypothetical future need.
- **In practice, applications almost never actually migrate databases** — most software is written for one database, deployed against that one database, and never ports to another for its entire operational lifetime; the effort spent preemptively guarding against a switch that never happens is effort not spent on the application's actual requirements.

### When Portability Genuinely Matters

Full portability is a legitimate, worthwhile goal in a much narrower set of real circumstances — recognizing them is the actual skill this topic is teaching, not "always abstract everything" or "never abstract anything":

- **You are building a library, framework, or tool meant to be installed against a database its own users choose** — an object-relational mapper, a general-purpose migration tool, a reporting product sold to customers who each already run their own (differing) database — where you genuinely do not control, and cannot assume, which database will be underneath at runtime.
- **You are building software explicitly required to support multiple specific databases as a stated product requirement** — e.g., an on-premises enterprise product that must support both a customer's existing Oracle installation and a customer's existing SQL Server installation, because the sales requirement genuinely demands both.
- **You are deliberately avoiding vendor lock-in for strategic/organizational reasons** — some organizations mandate database independence as an explicit architectural policy, independent of any single project's immediate technical need, usually to preserve future negotiating leverage or flexibility.

In every one of these cases, the isolation checklist above (auto-increment, upsert, pagination, procedural code) is exactly the set of things to push behind a dedicated abstraction layer — most commonly, a query-building or object-relational mapping tool that already knows how to emit the correct dialect-specific syntax for whichever database it's configured against, so your application code expresses intent once ("insert or update this row") and the tool handles the `ON CONFLICT`-vs-`ON DUPLICATE KEY UPDATE`-vs-`MERGE` translation.

## Internal Working

Think of the decision as a simple filter every piece of SQL in a portable-by-requirement codebase should pass through:

```
              New SQL statement to write
                        │
                        ▼
     Does it use ONLY features from the
     "safely portable" table above?
          │                    │
         Yes                   No
          │                    │
          ▼                    ▼
   Write it directly    Isolate it behind a
   in plain SQL,        per-database abstraction
   no abstraction        (dialect-specific code path,
   needed                query builder, or ORM feature)
```

The practical value of the two checklists earlier in this topic is that they let you run this filter *before* writing the query, rather than discovering a portability gap only after a second database target actually shows up in production.

## Real-World Analogy

Writing portable SQL is like designing a shipping container versus designing a custom-built truck bed. A standardized shipping container (the portable-core SQL) can move seamlessly between a ship, a train, and a truck, because everyone agreed on its exact dimensions in advance — but it also can't be shaped perfectly to any one vehicle's unique capabilities. A custom-built truck bed (database-specific features like `ON CONFLICT` or window functions) fits its one truck far better and can carry things a generic container never could — but it cannot be lifted onto a train or a ship without modification. Most cargo (most applications) only ever travels by one specific vehicle its entire life, so building a perfectly custom bed for it is the right call; only cargo that genuinely needs to move between multiple transport types justifies paying the standardization cost of the shipping-container approach.

## Why This Guidance Is Shaped This Way

This connects directly back to the relational model's own design philosophy from Module 1 and Module 2: SQL is declarative specifically so that *what* you ask for can remain stable even as *how* it's computed changes underneath. Vendor-specific features are, in effect, an extension of that same declarative promise, scoped to one particular vendor rather than the whole standard — `ON CONFLICT` still lets you declare "avoid this exact conflict scenario" without you writing the check-then-insert-or-update logic by hand. The trade-off portability introduces is real, not imagined: a lowest-common-denominator subset of SQL is, by construction, giving up access to every vendor's newest and most powerful extensions in exchange for that subset running unmodified everywhere. Recognizing when that trade-off is actually worth making — rather than defaulting to either extreme — is the entire point of this topic and, in a real sense, of this whole module.

## Advantages

- **The portable core is genuinely large and genuinely stable** — Modules 7 through 11's entire content transfers to any of the four other databases in this module with little to no change, which is a strong, durable foundation to build judgment on top of.
- **A clear checklist beats vague instinct** — knowing specifically which categories (auto-increment, upsert, pagination, procedural code) are the recurring trouble spots means you can audit a codebase for portability risk quickly and concretely, rather than guessing.
- **The "commit to one database" guidance saves real engineering effort** for the large majority of projects that will, in practice, never actually need to run on a second database.

## Disadvantages / Limitations

- **The portable-core/isolate-the-rest split is not perfectly crisp** — some features sit in a gray zone (e.g., `OFFSET`/`FETCH` is portable to three of the four other databases in this module but not MySQL, which needs its own `LIMIT` form instead), so judgment is still required at the edges.
- **Committing fully to one database's features is itself a real, if usually acceptable, form of lock-in** — the guidance in this topic to embrace it is a deliberate trade-off, not a free lunch; a genuine future need to migrate databases will still cost real rework, this topic simply argues that cost is usually smaller than the ongoing cost of avoiding it preemptively.
- **Query-building/ORM abstraction layers, when genuine portability is required, are not magic** — they still cannot make every vendor-specific feature disappear cleanly; some dialect-specific edge cases inevitably still leak through and need direct handling.

## Best Practices

- Default to writing plain, standard SQL using PostgreSQL's full feature set (this course's reference dialect) unless you have a concrete, present reason to do otherwise — don't pre-emptively cripple your SQL to a lowest-common-denominator subset for a portability need that may never materialize.
- If you are building something that genuinely must support multiple databases (a library, a multi-database product), isolate exactly the categories in this topic's second table behind a single abstraction point each — one place that knows "this is an upsert, dispatch to the right dialect" — rather than scattering dialect-specific `IF` branches throughout the codebase.
- When you do need portability, write automated tests that actually run the same logical operation against every target database you claim to support — a portability claim that has never been executed against a second real database is not a verified claim.
- Revisit the "does this actually need to be portable" question periodically as a project's requirements evolve, rather than assuming an early architectural decision (either direction) is permanent.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Writing every query in a maximally generic, lowest-common-denominator style "just in case" a second database is ever needed | Pays the ongoing cost of forgoing a chosen database's best features (upserts, window functions, JSON support) for a hypothetical future requirement that, for most applications, never actually arrives. |
| Assuming "I only used `SELECT`/`WHERE`/`JOIN`, so my whole application is portable" | The portable core covers query-reading SQL well, but real applications also create schemas, generate keys, and perform upserts — exactly the categories that are *not* portable — so a portability claim based only on query syntax is incomplete. |
| Building a full cross-database abstraction layer for a project with a single, fixed, known database target and no stated requirement to ever change it | Solves a problem that does not exist for that project, at real ongoing engineering cost, when simply writing directly against the one real target database would have been both simpler and faster to build. |
| Assuming a query-building/ORM tool automatically makes an application "fully portable" without ever testing against the second database | Abstraction tools reduce, but do not eliminate, dialect-specific edge cases — a portability claim needs to be verified by actually running against every claimed target, not merely asserted because a tool was used. |

## Interview Questions

1. **Q: What categories of SQL should you assume are NOT portable across major relational databases, and why?**
   A: Auto-increment/identity mechanics, upsert syntax, pagination/row-limiting syntax, and any procedural code (functions, procedures, triggers) — because each major database implements these with genuinely different keywords and, in several cases, different underlying mechanisms (a separate sequence object vs. a table attribute; a single upsert clause vs. a general-purpose `MERGE`; `LIMIT` vs. `TOP` vs. `ROWNUM`). Everything covered in Modules 7 through 11 (core querying, joins, subqueries, aggregation) is, by contrast, close to universally portable.

2. **Q: Why would you recommend most applications commit to one database rather than write fully portable SQL?**
   A: Because full portability imposes an ongoing tax — every feature must be written in a lowest-common-denominator style or maintained in multiple dialect-specific versions — paid indefinitely regardless of whether the application ever actually changes databases, and in practice, most applications never do. Committing to one database also allows full, simpler use of that database's best features (e.g., PostgreSQL's `ON CONFLICT` or window functions) rather than abstracting them away to preserve a hypothetical, often-unrealized future need.

3. **Q: Give a concrete example of a project where writing genuinely portable SQL is the right call, and explain why.**
   A: A general-purpose object-relational mapping library or a database migration tool intended to be installed by users who may already run any of PostgreSQL, MySQL, SQL Server, or Oracle. Here the developer does not control, and cannot assume, which database will be underneath at runtime — portability isn't a hypothetical future concern, it's the explicit, present product requirement, which justifies isolating auto-increment, upsert, and pagination logic behind a dialect-aware abstraction from the start.

## Summary

- Standard `SELECT`/`WHERE`/`JOIN`/`GROUP BY`/aggregate/subquery SQL (Modules 7–11) is the safely portable core across PostgreSQL, MySQL, SQL Server, Oracle, and SQLite.
- Auto-increment mechanics, upsert syntax, pagination syntax, and procedural code are the categories that reliably diverge and should be isolated behind a deliberate abstraction if true portability is required.
- Most real applications should commit fully to one database rather than pay the ongoing cost of writing lowest-common-denominator SQL for a portability need that, in practice, rarely materializes.
- Genuine portability is worth its cost specifically when building a library/tool/framework meant to run against a database its own users choose, when a product explicitly must support multiple named databases, or when an organization has a deliberate anti-lock-in policy.
- Deciding "does this specific piece of SQL need to be portable" is a judgment call best made using the two checklists in this topic, not by defaulting to either extreme.
