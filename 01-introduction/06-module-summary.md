# Module 01 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **What Is a Database and a DBMS?** — the database/DBMS distinction, what makes a database relational, brief context on NoSQL
- [x] **What Is SQL?** — definition, declarative vs. imperative, the SQL standard vs. real-world vendor implementations
- [x] **Categories of SQL Commands** — DDL, DML, DQL, DCL, TCL, with a full classification table
- [x] **Setting Up PostgreSQL** — installation across macOS/Windows/Linux, connecting with `psql`, creating a practice database, useful meta-commands
- [x] **Your First Query** — `CREATE TABLE` → `INSERT` → `SELECT`, dissected statement-by-statement, plus common beginner mistakes

## Practical Connections

Even at this early stage, the concepts in this module underpin everything you'll do with SQL professionally:

- **Every application with a "sign up," "save," or "search" feature** is, underneath, running exactly the same three-step loop you just learned: structure defined once (`CREATE TABLE`), data changed over time (`INSERT`/`UPDATE`/`DELETE`), and data read constantly (`SELECT`) — just at a much larger and more complex scale.
- **The declarative nature of SQL** (Topic 2) is why the same query you write today keeps working correctly even as a table grows from 10 rows to 10 million — the database's query planner adapts its strategy; your SQL doesn't need to change.
- **The DDL/DML/DQL/DCL/TCL categorization** (Topic 3) directly maps onto real-world database permission models — production systems commonly restrict an application's database user to only DML and DQL, denying DDL and DCL, so a bug can't accidentally restructure the database or grant new access.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Database vs. DBMS | The database is the organized data itself; the DBMS is the software that manages and mediates access to it |
| SQL vs. a general-purpose language (Python, etc.) | SQL is declarative (describe *what* you want); general-purpose imperative languages require you to describe *how*, step by step |
| DQL (`SELECT`) vs. DML (`INSERT`/`UPDATE`/`DELETE`) | DQL only reads data and changes nothing; DML actually changes stored data |
| Single quotes vs. double quotes in SQL | Single quotes wrap string literal values (`'Sales'`); double quotes wrap unusually-named identifiers (table/column names) |
| SQL the language vs. PostgreSQL/MySQL/etc. | SQL is the standardized language; PostgreSQL, MySQL, and others are specific DBMS products that implement (and extend) that standard |

## What's Next

Module 01 gave you the conceptual foundation and a working environment: what a database and SQL fundamentally are, how SQL commands are categorized, and a first hands-on loop of defining, inserting, and querying data. **Module 02 — Relational Model** goes deep into the theory this all rests on: what a "relation" formally is, how tables, rows, and schemas are precisely defined, and the reasoning (originating with Edgar Codd) that shaped every relational database you'll ever use.
