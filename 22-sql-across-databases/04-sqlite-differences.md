# SQLite Differences

## Learning Objectives

By the end of this section you should be able to:
- Explain SQLite's dynamic typing and "type affinity" model, and contrast it with PostgreSQL's strict, enforced column typing
- Explain why SQLite has no meaningful user/role/permission system, and why `GRANT`/`REVOKE` do not apply to it
- Describe SQLite's historically limited `ALTER TABLE` support and what workaround it requires
- Explain why SQLite does not support true concurrent multi-writer access the way a client-server database does
- Justify why SQLite remains hugely popular despite these apparent limitations

## Prerequisites

- Module 3 (Data Types) — you need PostgreSQL's strict, declared-and-enforced column typing firmly in mind as the baseline this topic contrasts against SQLite's fundamentally different philosophy.
- Module 19 (Security & Access Control), especially [Users and Roles](../19-security-and-access-control/01-users-and-roles.md) — needed to appreciate what SQLite genuinely lacks, since this topic assumes you already know what a role-based permission model provides.
- [Oracle Differences](03-oracle-differences.md) — establishes the pattern, continued here, of comparing an entire database's design philosophy against PostgreSQL's, not just isolated syntax.

## Motivation

Every other database in this module — MySQL, SQL Server, Oracle — is a **client-server** database: a separate server process runs continuously, and applications connect to it over a network (or a local socket) to send SQL and receive results, exactly like PostgreSQL. SQLite breaks this pattern entirely: there is no server process at all. A SQLite "database" is a single ordinary file on disk, and the application linking to SQLite's library talks to that file directly, in-process. This is not a minor implementation detail — it is a completely different architectural philosophy that reshapes nearly every topic in this course, from data types to permissions to concurrency, and it explains both SQLite's limitations and its enormous, genuine popularity.

## Problem Statement

Suppose you take a PostgreSQL table definition you've written many times in this course and create the "same" table in SQLite:

```sql
CREATE TABLE employees (
    id     INTEGER PRIMARY KEY,
    name   TEXT NOT NULL,
    salary NUMERIC CHECK (salary >= 0)
);
```

Then you insert a row that should be rejected by any strictly-typed database:

```sql
INSERT INTO employees (name, salary) VALUES ('Asha', 'not-a-number');
```

In PostgreSQL, this fails immediately — `salary` is declared `NUMERIC`, and the string `'not-a-number'` cannot be converted to one. In SQLite, by default, **this succeeds**, and the literal text `'not-a-number'` is stored in the `salary` column exactly as typed. If your entire mental model of "a column's declared type is enforced by the database" came from PostgreSQL, this SQLite behavior looks like a bug. It is not a bug — it is a deliberate, different design philosophy, and this topic explains it along with SQLite's other structural differences.

## Concept

### Dynamic Typing and Type Affinity

PostgreSQL enforces **strict typing**: a column's declared type (Module 3) is a hard constraint. Every value stored in it must genuinely be, or be losslessly convertible to, that type, and the database rejects anything that isn't at `INSERT`/`UPDATE` time.

SQLite uses a fundamentally different model called **type affinity**. A column's declared type is a *hint* about what kind of data the column *prefers* to store and how it should attempt to convert incoming values — not a hard rule the database enforces. SQLite recognizes five type affinities:

| Affinity | What it means |
|---|---|
| `TEXT` | Stores data as text; numeric values are converted to text before storage |
| `NUMERIC` | Attempts to convert text that looks numeric into an integer or floating-point value; stores genuinely non-numeric text as-is |
| `INTEGER` | Same as `NUMERIC`, but prefers whole-number storage where possible |
| `REAL` | Similar to `NUMERIC`, but prefers floating-point storage |
| `BLOB` (or "no affinity") | Stores exactly whatever byte sequence was given, with no conversion attempt at all |

Critically, SQLite determines a column's affinity from the *declared type name you wrote*, using pattern-matching rules (a declared type containing `"INT"` gets `INTEGER` affinity, one containing `"CHAR"`, `"CLOB"`, or `"TEXT"` gets `TEXT` affinity, and so on) — but affinity only *steers* storage; it never *rejects* a value. This is why the earlier example succeeded: `salary NUMERIC` gives the column `NUMERIC` affinity, which *attempts* to convert `'not-a-number'` into a number, fails to do so, and — rather than raising an error the way PostgreSQL would — simply stores the original text value unchanged, in whatever underlying storage class (`TEXT`, `INTEGER`, `REAL`, `BLOB`, or `NULL`) the value actually is.

```sql
-- SQLite: every row below is perfectly legal, even in the same "salary" column
CREATE TABLE employees (id INTEGER PRIMARY KEY, salary NUMERIC);
INSERT INTO employees (salary) VALUES (85000);
INSERT INTO employees (salary) VALUES (85000.50);
INSERT INTO employees (salary) VALUES ('not-a-number');
INSERT INTO employees (salary) VALUES (NULL);

SELECT salary, typeof(salary) FROM employees;
```
```
    salary     | typeof(salary)
---------------+----------------
 85000         | integer
 85000.5       | real
 not-a-number  | text
               | null
(4 rows)
```

The built-in `typeof()` function above reveals what PostgreSQL would consider alarming: the *same column*, across different rows, is genuinely storing four different underlying storage classes. In PostgreSQL, this table definition would guarantee every non-null `salary` value is a valid number, full stop, enforced by the engine itself; in SQLite, that guarantee simply does not exist unless you add it yourself (SQLite does support `CHECK` constraints, covered in Module 5, which — unlike bare column typing — *are* actually enforced, and are the correct way to get PostgreSQL-like guarantees in SQLite if you need them).

Modern SQLite versions also offer an opt-in `STRICT` table mode (`CREATE TABLE ... (...) STRICT;`) that does enforce declared types much closer to how PostgreSQL always does — but this is opt-in, not the historical default, and dynamic typing remains SQLite's defining, original behavior.

### No Real User/Role/Permission System

Module 19 covered PostgreSQL's `GRANT`/`REVOKE` model in depth: roles, privileges scoped to specific tables and columns, and the principle of least privilege, all enforced by the database server as different client connections authenticate as different roles. **None of this exists in any meaningful form in SQLite.**

This isn't a smaller or simplified version of a permission system — it's the direct, structural consequence of SQLite having no server and no concept of a "connection" authenticating as a distinct identity in the first place. A SQLite database is just a file; whatever operating-system process opens that file has exactly the access the *operating system's own file permissions* grant it (read, write, or neither) — and if a process can open the file at all, it can do absolutely anything to any data inside it, because there is no database-level layer sitting between the file and the code reading it. There is no `CREATE ROLE`, no `GRANT SELECT ON table TO some_role`, and no way to say "this connection can read table A but not table B" — that entire class of control simply has no equivalent to reach for.

| Concept from Module 19 | PostgreSQL | SQLite |
|---|---|---|
| Authenticating as a distinct database user | Yes — roles with passwords/authentication methods | No concept of a database-level user at all |
| Table-level `GRANT`/`REVOKE` | Yes | Not applicable — no permission layer exists |
| Column-level privilege granularity | Yes | Not applicable |
| Access control that does exist | Database-enforced, independent of the OS | Entirely delegated to the operating system's file permissions on the `.sqlite` file itself |

### Limited `ALTER TABLE` Support, Historically

PostgreSQL's `ALTER TABLE` (Module 4) supports a very wide range of structural changes directly: adding/dropping columns, changing a column's type, adding/dropping constraints, renaming things, and more. SQLite's `ALTER TABLE` has historically supported only a narrow subset — for a long time, essentially just renaming a table, renaming a column, and adding a new column (with restrictions on what that new column can look like). Operations PostgreSQL treats as a single, direct statement — dropping a column, changing a column's type, adding certain constraints — have traditionally required an entirely different, manual approach in SQLite:

```sql
-- The classic SQLite workaround for "unsupported" ALTER TABLE operations
BEGIN TRANSACTION;

CREATE TABLE employees_new (
    id     INTEGER PRIMARY KEY,
    name   TEXT NOT NULL
    -- the "salary" column is deliberately omitted here, achieving a drop
);

INSERT INTO employees_new (id, name)
SELECT id, name FROM employees;

DROP TABLE employees;
ALTER TABLE employees_new RENAME TO employees;

COMMIT;
```

That is: create a new table shaped exactly the way you want the final table to look, copy the surviving data into it, drop the old table, and rename the new one into its place. Newer SQLite versions have gradually added direct support for a few more operations (including `DROP COLUMN` in recent releases), narrowing this gap somewhat — but the create-copy-drop-rename pattern remains a commonly seen, defensible technique for anything still unsupported, and is worth recognizing on sight in existing SQLite migration scripts.

### No True Concurrent Multi-Writer Support

PostgreSQL, as a client-server database, is built from the ground up to let many separate processes read and write the same database simultaneously, coordinated by the DBMS's concurrency-control machinery (Module 14). SQLite's architecture — no server, the application linking directly to the database file — makes this fundamentally harder to offer in the same way. SQLite's traditional locking model allows **many simultaneous readers, but only one writer at a time**; a second process attempting to write while another write is in progress must wait (or receive a "database is locked" error, depending on configuration) rather than proceeding concurrently the way two simultaneous PostgreSQL `UPDATE` statements against different rows normally can.

Modern SQLite's **WAL (Write-Ahead Logging) mode** substantially improves on this — it allows readers to continue running concurrently with a single in-progress writer, rather than blocking them — but even in WAL mode, SQLite still fundamentally serializes writers to one at a time. This is an intentional, architectural ceiling, not a bug to be patched: a single-file, serverless database has no separate coordinating process capable of interleaving multiple simultaneous write transactions the way a full client-server DBMS does.

```
 PostgreSQL (client-server):        Client A ─┐
                                     Client B ─┼──▶  Server process
                                     Client C ─┘     coordinates concurrent
                                                     reads AND writes

 SQLite (serverless, WAL mode):     Process A ──▶  employees.sqlite  ◀── Process B (write, one at a time)
                                                        ▲
                                                        │ (many concurrent readers, always)
                                                    Process C, D, E... (reads)
```

### Why SQLite Is Still Hugely Popular

Given the apparent limitations above, it's fair to ask why SQLite is, by some measures, the single most widely deployed database engine in the world. The answer is that SQLite was never designed to compete with PostgreSQL, MySQL, SQL Server, or Oracle at their own game (large, shared, multi-writer, server-based data stores) — it targets a different, enormous niche entirely: the **embedded, single-file, single-application database**.

- **Zero setup or administration** — there is no server to install, configure, patch, or keep running; a SQLite database is created the instant a file is opened, by any application with the SQLite library linked in.
- **A single portable file** — an entire database is one file that can be copied, emailed, backed up, or bundled directly inside an application's installer, with no export/import step needed.
- **It is the de facto standard embedded database** — it ships inside every major web browser, every Android and iOS device, and countless desktop applications, for exactly the local-storage use case it was designed for (a browser's local cache, a mobile app's offline data store, a single-user desktop tool's settings and data).
- **It is genuinely excellent at what it targets** — for a single application reading and writing its own local data, SQLite's lack of a server, permission system, and multi-writer concurrency aren't missing features; they're irrelevant overhead this specific use case never needed in the first place.

The lesson generalizes beyond SQLite specifically: "different from PostgreSQL" and "worse than PostgreSQL" are not the same statement — SQLite is a deliberately, successfully different tool for a deliberately different job.

## Internal Working

SQLite's storage model assigns every individual value one of five underlying **storage classes** — `NULL`, `INTEGER`, `REAL`, `TEXT`, or `BLOB` — largely independent of the column's declared affinity. A column's affinity only influences the conversion SQLite *attempts* the moment a value is inserted:

```
 Value arrives for insertion
        │
        ▼
 Column's declared affinity checked (TEXT / NUMERIC / INTEGER / REAL / BLOB)
        │
        ▼
 SQLite attempts a conversion matching that affinity
        │
   ┌────┴─────┐
   ▼          ▼
Succeeds   Fails
   │          │
   ▼          ▼
Stored as   Stored as whatever storage
the target  class the original value
type        actually was, unconverted
```

This is the internal mechanism behind the earlier example: `'not-a-number'` inserted into a `NUMERIC`-affinity column triggers a conversion attempt, that attempt fails (the text isn't numeric), and SQLite falls back to storing it as its original `TEXT` storage class rather than rejecting the statement — a design decision PostgreSQL's stricter engine (Module 3) never makes.

## Real-World Analogy

Think of PostgreSQL's strict typing like a shipping company that weighs and inspects every package against its declared category before accepting it — a box labeled "fragile glass" that turns out to contain bricks is rejected at the counter, full stop. SQLite's type affinity is like a shipping company that reads the label, *tries* to route the package accordingly, but if the contents genuinely don't match the label, it ships the package anyway, exactly as it actually is, with a note about what it really contained. Neither company is "wrong" — one is built for strict, uniform cargo where a mismatch signals a real problem worth halting for; the other is built for flexible, fast throughput where accepting the occasional oddly-labeled package is an acceptable trade-off for never turning shipments away.

## Why SQLite Was Designed This Way

SQLite's creator built it explicitly to solve a different problem than server-based databases were solving: applications (originally, specifically, embedded systems and simple desktop tools) that needed *some* structured, queryable, transactional storage, but for which running and administering a full separate database server was pure overhead — no separate process to install, no network port to secure, no multiple simultaneous users to coordinate, because there typically was only one application, one process, reading its own local file. Dynamic typing traces to SQLite's design goal of being maximally flexible and forgiving for exactly this single-application use case, where a rigid schema mismatch failing an entire operation is far less useful than simply storing what was given and letting the single controlling application decide how strict to be (via `CHECK` constraints, or the newer opt-in `STRICT` mode, if it wants PostgreSQL-like guarantees after all). The absent permission system and single-writer concurrency model are not oversights — they are the direct, necessary consequence of removing the server process that a permission system and true multi-writer coordination would otherwise both depend on.

## Advantages

- **Zero administration overhead** — no server process to install, configure, secure, or keep running, which is precisely the right trade-off for a single embedded application's local data.
- **Maximal portability** — an entire database genuinely is one file, trivially copyable, backupable, and bundleable, with none of PostgreSQL's separate data-directory/server-process/network-configuration considerations.
- **Type affinity's flexibility can genuinely be convenient** for rapid prototyping or loosely-structured local data, where PostgreSQL's strict rejection of a mismatched value would otherwise interrupt a quick, low-stakes script.
- **Massive real-world adoption for its target niche** proves the design succeeded at what it set out to do — it is not a "lesser" relational database, it is a correctly-scoped one.

## Disadvantages / Limitations

- **Dynamic typing removes a genuine safety net** — a typo or an application bug that inserts the wrong kind of value into a column will not be caught by the database the way PostgreSQL's strict typing catches it automatically, unless you've added explicit `CHECK` constraints or opted into `STRICT` mode.
- **No database-level permission system at all** — any process able to open the file can do anything to any data in it; access control must be entirely reimplemented at the operating-system or application layer, with none of Module 19's fine-grained, role-based tooling available.
- **Single-writer-at-a-time concurrency** makes SQLite a poor fit for any workload with genuinely simultaneous multi-user writes (the exact scenario Module 14's concurrency-control machinery was built for) — it is not the right choice for a busy multi-user server application's primary data store.
- **Historically limited `ALTER TABLE`** requires the create-copy-drop-rename workaround for structural changes many other databases handle with a single direct statement, though modern versions have narrowed this gap.

## Best Practices

- Add explicit `CHECK` constraints (Module 5) to any SQLite column where you genuinely need PostgreSQL-like type enforcement, since bare column types alone will not provide it — or use `STRICT` table mode on modern SQLite versions if you want that guarantee across an entire table by default.
- Never assume a SQLite deployment can substitute for a client-server database in any application with genuinely concurrent multi-user writers — reach for PostgreSQL, MySQL, or SQL Server instead the moment "many separate processes writing to the same data at the same time" is a real requirement, not a hypothetical one.
- If you need to run structural changes SQLite's `ALTER TABLE` doesn't support directly, always wrap the create-copy-drop-rename sequence in an explicit transaction (as shown above), so a failure partway through doesn't leave the database in a half-migrated state.
- Treat SQLite's file-level access exactly like any other sensitive file on disk — since there is no database-level permission system, operating-system file permissions are the *entire* access-control story, and must be set deliberately.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a SQLite column's declared type guarantees the same enforcement PostgreSQL provides | SQLite's declared types express *affinity* (a storage preference/conversion attempt), not a hard constraint — a value that doesn't match can still be stored as-is, unless `CHECK` constraints or `STRICT` mode are explicitly added. |
| Looking for a `GRANT`/`REVOKE` equivalent in SQLite | There is no database-level permission system to grant or revoke privileges within — access is governed entirely by the operating system's file permissions on the database file itself. |
| Choosing SQLite as the primary data store for an application expecting many simultaneous writers | SQLite serializes writers to one at a time (even in WAL mode); an application with genuinely concurrent multi-user write traffic needs a client-server database like PostgreSQL instead. |
| Treating SQLite's limitations as evidence it is a "worse" or incomplete database | SQLite was designed for a different use case (embedded, single-file, single-application storage) than server-based databases, and is enormously successful and appropriate within that scope — the comparison is apples-to-oranges, not better-to-worse. |

## Interview Questions

1. **Q: What is "type affinity" in SQLite, and how does it differ from PostgreSQL's column typing?**
   A: Type affinity is SQLite's practice of treating a column's declared type as a preference/conversion hint rather than an enforced rule — SQLite attempts to convert an incoming value to match the column's affinity, but if that conversion fails, it stores the value as its original type anyway rather than rejecting the statement. PostgreSQL, by contrast, enforces a column's declared type strictly: a value that cannot be validly converted is rejected at insert/update time.

2. **Q: Why doesn't SQLite have a `GRANT`/`REVOKE` permission system like PostgreSQL's?**
   A: SQLite has no server process and no concept of a database-level user or connection identity to grant privileges to in the first place — a SQLite database is simply a file, and whatever operating-system process can open that file has whatever access the OS's own file permissions allow, with no additional database-level layer of control possible.

3. **Q: Given SQLite's limitations around typing, permissions, and concurrency, why is it one of the most widely deployed databases in the world?**
   A: SQLite targets a fundamentally different use case than server-based databases: embedded, single-file, typically single-application local storage, where there is no server to administer, no multiple simultaneous users to coordinate permissions for, and often no genuinely concurrent multi-writer workload. In that niche — mobile apps, browsers, desktop application local storage — its lack of a server, permission system, and multi-writer concurrency are not missing features but simply irrelevant to the problem it solves, which is why it succeeds enormously there rather than being a lesser competitor to PostgreSQL.

## Summary

- SQLite uses **dynamic typing with type affinity**: a column's declared type is a conversion hint, not an enforced rule, unlike PostgreSQL's strict, always-enforced column typing (Module 3) — `CHECK` constraints or opt-in `STRICT` mode are needed to recover PostgreSQL-like guarantees.
- SQLite has **no meaningful user/role/permission system** — there is no server or connection identity for `GRANT`/`REVOKE` (Module 19) to apply to; access control is entirely delegated to the operating system's file permissions.
- SQLite's `ALTER TABLE` has historically supported only a narrow set of operations directly, requiring a create-new-table, copy-data, drop-old-table, rename workaround for anything else.
- SQLite allows many concurrent readers but fundamentally serializes writers to one at a time, even with WAL mode's improvements — it is not built for genuinely concurrent multi-writer workloads the way a client-server database (Module 14) is.
- SQLite's popularity comes from targeting an entirely different niche than PostgreSQL, MySQL, SQL Server, or Oracle — embedded, single-file, typically single-application storage — where its apparent limitations are simply irrelevant to the problem being solved.
