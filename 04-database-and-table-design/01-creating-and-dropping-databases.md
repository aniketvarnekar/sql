# Creating and Dropping Databases

## Learning Objectives

By the end of this section you should be able to:
- Create a new database with `CREATE DATABASE`, including common options like `OWNER` and `ENCODING`
- List the databases that exist on a PostgreSQL server
- Remove a database with `DROP DATABASE`, including the `IF EXISTS` safeguard
- Explain why a database generally cannot be dropped while you (or anyone else) are connected to it, and how to work around that

## Prerequisites

- [Setting Up PostgreSQL](../01-introduction/04-setting-up-postgresql.md) — you already created a database (`sql_course`) there without a full explanation of the statement; this topic gives you that explanation and the rest of the picture (options, listing, dropping).

## Motivation

Every table you will ever create has to live *inside* something — a database. Before you can design tables in depth (this module's remaining topics), you need to be comfortable creating, inspecting, and removing the databases those tables live in. This is also usually the very first DDL statement anyone runs against a fresh PostgreSQL server, and it's the first place beginners get tripped up by a surprisingly counter-intuitive rule: you cannot drop the database you're currently sitting inside.

## Problem Statement

Suppose you're setting up a new project. You need a clean, isolated place for its tables to live — separate from any other project's data on the same server, so a mistake in one project (dropping a table, running a bad migration) can never touch another project's data by accident. You also need the ability to throw the whole thing away and start over cleanly if an experiment goes wrong. Both needs point to the same feature: the ability to create and destroy entire databases, not just tables within one.

You might try to just reuse one big database for everything, giving every project's tables slightly different names to avoid collisions (`project_a_users`, `project_b_users`). This avoids learning `CREATE DATABASE`, but it means every project shares the same connection, the same permission boundary, and the same namespace — a naming collision or a careless `DROP TABLE` in one project can now affect another. Separate databases exist specifically to give each project (or environment, or tenant) a hard isolation boundary.

## Concept

### What a Database Means in PostgreSQL

A PostgreSQL **server** (one running `postgres` process, listening on a port — usually `5432`) can host many independent **databases**. Each database has its own set of tables, its own set of other objects (views, indexes, sequences), and — critically — a client can only ever be connected to *one* database at a time. There is no SQL syntax to query "across" two databases in a single `SELECT` the way you can join two tables within the same database.

```
 PostgreSQL server (one process, one port)
  ├── database: postgres     (default administrative database)
  ├── database: sql_course   (yours, from Module 1)
  ├── database: template1    (template used when creating new databases)
  └── database: template0    (pristine, untouched template — rarely used directly)
```

### Creating a Database

The basic form, which you already used in Module 1, is:

```sql
CREATE DATABASE sql_course;
```

The full syntax accepts several optional clauses. The ones you'll actually reach for:

```sql
CREATE DATABASE store
    WITH
    OWNER = akshay
    ENCODING = 'UTF8'
    CONNECTION LIMIT = 50;
```

| Option | Meaning |
|---|---|
| `OWNER` | The database role (user) that owns this database. The owner has full rights over it by default. If omitted, the role that ran `CREATE DATABASE` becomes the owner. |
| `ENCODING` | The character encoding used to store text in this database — `'UTF8'` is almost always the right choice today, since it can represent virtually any text (accented letters, emoji, non-Latin scripts). It's fixed at creation time and cannot be changed afterward without recreating the database. |
| `TEMPLATE` | Which existing database to copy as the starting point for the new one. By default this is `template1`, an empty-but-customizable template database that ships with every PostgreSQL install. `template0` is a pristine, never-modified fallback template, useful in rare cases where `template1` has been altered in a way that conflicts with the new database's settings (for example, a different encoding). |
| `CONNECTION LIMIT` | The maximum number of simultaneous connections allowed to this database; `-1` (the default) means unlimited. Useful for capping how much of a shared server one database can monopolize. |
| `LC_COLLATE` / `LC_CTYPE` | Locale settings controlling text sort order and character classification (e.g., whether `'a' < 'B'` in a sort). Like `ENCODING`, these are fixed at creation time. |

For everyday learning and most small-to-medium real applications, `CREATE DATABASE name;` with no options — accepting the server's defaults — is perfectly normal. The options above matter most when you need explicit control (a specific owner for permission separation, a specific encoding for a legacy data source).

### Listing Databases

Two ways to see what databases exist on the server you're connected to:

Using the `psql` meta-command (fastest, interactive-only):

```
\l
```

```
                                  List of databases
     Name     |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
--------------+----------+----------+---------+---------+-----------------------
 postgres     | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 sql_course   | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0    | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
 template1    | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
(4 rows)
```

Using pure SQL, by querying PostgreSQL's system catalog (its internal metadata tables, briefly previewed in Module 1 and covered properly later in this course):

```sql
SELECT datname, datcollate, encoding_name
FROM pg_database, pg_encoding_to_char(encoding) AS encoding_name
WHERE datistemplate = false;
```

```
  datname   | datcollate | encoding_name
------------+------------+---------------
 postgres   | C.UTF-8    | UTF8
 sql_course | C.UTF-8    | UTF8
(2 rows)
```

`\l` is a `psql` convenience wrapper around a query very much like this one — it is not a separate SQL feature, just a shortcut (as established in Module 1's coverage of `psql` meta-commands).

### Dropping a Database

To remove a database entirely — deleting every table, every row, every object inside it, irreversibly:

```sql
DROP DATABASE store;
```

If the database might not exist, and you don't want an error in that case (for example, in a script that's meant to be safely re-runnable), add `IF EXISTS`:

```sql
DROP DATABASE IF EXISTS store;
```

Without `IF EXISTS`, dropping a database that doesn't exist raises an error:

```
ERROR:  database "store" does not exist
```

With `IF EXISTS`, PostgreSQL silently does nothing instead of erroring — useful for idempotent setup/teardown scripts, but be careful: it also means a typo in the database name will fail *silently*, quietly doing nothing rather than warning you the name was wrong.

### Why You Can't Drop the Database You're Connected To

Try this from inside `sql_course`:

```
sql_course=# DROP DATABASE sql_course;
ERROR:  cannot drop the currently open database
```

PostgreSQL refuses. This isn't a permissions issue — it's structural. A database being actively used by a connection has open resources (temporary files, cached state, an active transaction context) tied to that connection; dropping the underlying database out from under a live connection would leave that connection in an undefined, unsafe state. More generally, PostgreSQL also refuses to drop a database that has *any* other active connections, not just your own:

```
ERROR:  database "store" is being accessed by other users
DETAIL:  There is 1 other session using the database.
```

To actually drop a database, you must first disconnect from it (or make sure no one else is connected to it) and connect to a *different* database instead — typically the always-present `postgres` administrative database:

```
\c postgres
DROP DATABASE sql_course;
```

If other sessions are stubbornly still connected (for example, a forgotten open terminal), PostgreSQL 13 and later offers a forceful option that terminates other connections for you as part of the drop:

```sql
DROP DATABASE store WITH (FORCE);
```

Use `WITH (FORCE)` deliberately and rarely — it forcibly disconnects other sessions, which could interrupt someone else's in-progress work.

## Internal Working (Preview)

`CREATE DATABASE` is a heavier operation than it might look:

```
CREATE DATABASE store
        │
        ▼
 PostgreSQL copies the entire template database (template1 by default)
 file-by-file at the filesystem level
        │
        ▼
 A new entry is added to the server's pg_database catalog,
 recording the new database's name, owner, encoding, and settings
```

This is why `TEMPLATE` matters: any tables, extensions, or settings already present in `template1` when you run `CREATE DATABASE` get copied into your new database automatically. Most installs leave `template1` empty, so this is invisible day to day, but it explains why some teams pre-load `template1` with commonly needed extensions so every new database gets them "for free."

`DROP DATABASE` is essentially the reverse: PostgreSQL removes the catalog entry and deletes the database's underlying files from disk. This is precisely why it cannot happen while the database is in use — you cannot safely delete files a running process still has open.

## Real-World Analogy

Think of a PostgreSQL server as an office building with several independently locked departments (databases). `CREATE DATABASE` is renting and fitting out a brand-new department — you can specify who holds the master key (`OWNER`) and which "starter kit" of furniture to copy in (`TEMPLATE`). `DROP DATABASE` is demolishing that department's space entirely. You cannot demolish a room while people are still standing inside working — everyone has to leave first (disconnect), or building security has to forcibly clear the room (`WITH (FORCE)`), before demolition can safely proceed.

## Why This Was Designed This Way

A database is the top-level isolation and administrative boundary in PostgreSQL — separate databases cannot be joined together in one query, and (mostly) cannot share a single transaction. Because so much depends on a database's existence remaining stable for the duration of anyone's active work inside it, PostgreSQL treats "in use" as a hard block against deletion rather than a soft warning: this protects against a class of catastrophic, hard-to-debug corruption that would occur if a database's files vanished mid-transaction for some other connected session. This mirrors a theme from Module 1 — the DBMS exists precisely to prevent exactly this kind of unsafe, uncoordinated access to shared data.

## Advantages

- **Strong isolation between projects/environments** — separate databases on the same server share no tables, so mistakes in one cannot directly corrupt another.
- **Independent configuration per database** — encoding, connection limits, and ownership can all be tuned per database, which matters when one server hosts multiple applications with different needs.
- **Simple mental model for cleanup** — an entire project's data can be discarded with a single `DROP DATABASE`, without having to remember and drop dozens of individual tables.

## Disadvantages / Limitations

- **No cross-database queries** — you cannot write a single `SELECT` that joins a table in one database with a table in another; if two projects' data genuinely need to be queried together, they generally need to live in the same database (or use more advanced cross-server features outside this course's scope).
- **Coarse-grained operation** — `CREATE DATABASE` and `DROP DATABASE` act on an entire database at once; there's no partial version of either. You cannot "drop half a database."
- **The connection restriction can be mildly inconvenient during scripted teardown** — an automated script that both uses and later drops the same database must explicitly reconnect elsewhere first, which is an easy step to forget.

## Best Practices

- Use a dedicated database per distinct project or application, rather than cramming unrelated projects' tables into one shared database with prefixed names — this is exactly what `CREATE DATABASE` exists to make cheap and easy.
- Always connect to a neutral database (commonly `postgres`) before running `DROP DATABASE` on the one you were just working in — get in the habit of `\c postgres` before any drop, so you never hit the "currently open database" error mid-workflow.
- Prefer `DROP DATABASE IF EXISTS` in setup/teardown scripts you intend to re-run, but never rely on `IF EXISTS` to mask an actual typo — double check the name.
- Treat `WITH (FORCE)` as a deliberate, rare administrative action, not a routine habit — forcibly disconnecting other sessions can interrupt someone else's real work.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Trying to `DROP DATABASE` the database you're currently connected to. | PostgreSQL always refuses this, regardless of permissions — you must connect to a different database first. |
| Assuming `DROP DATABASE IF EXISTS wrong_name;` "worked" because it ran without error. | If `wrong_name` was a typo, `IF EXISTS` causes PostgreSQL to silently do nothing rather than error — always double-check the target name before relying on `IF EXISTS`. |
| Expecting to `SELECT` across two databases in one query. | A single connection is bound to a single database; querying data that lives in another database on the same server is not directly possible with plain SQL the way joining two tables in the same database is. |

## Interview Questions

1. **Q: Why does PostgreSQL refuse to drop a database you're currently connected to?**
   A: Because a database in active use has open resources tied to the connection (transaction state, temporary files, cached data); deleting its underlying files while a session still depends on them would leave that session in an unsafe, undefined state. You must disconnect (typically by connecting to a different database, like `postgres`) before the drop can proceed, or force-disconnect other sessions explicitly.

2. **Q: What is the purpose of the `TEMPLATE` option in `CREATE DATABASE`, and what does PostgreSQL use by default?**
   A: `TEMPLATE` specifies which existing database to copy as the starting point for the new database — any objects or settings already in the template are copied into the new database. By default, PostgreSQL uses `template1`. `template0` is a pristine fallback for cases where `template1` has been customized in a way that conflicts with the new database's desired settings.

3. **Q: Can you join a table in one PostgreSQL database with a table in a different database in a single query?**
   A: No — a connection operates against exactly one database at a time, and standard SQL joins require both tables to be visible within that same connection's database. If related data must be queried together, it typically needs to live in the same database.

## Summary

- A PostgreSQL server can host multiple independent databases; each is a hard isolation boundary — no cross-database joins, and (mostly) no shared transactions.
- `CREATE DATABASE name;` creates a database using defaults; `OWNER`, `ENCODING`, `TEMPLATE`, and `CONNECTION LIMIT` are the options you'll reach for most often.
- `\l` (a `psql` meta-command) or querying the `pg_database` system catalog both list existing databases.
- `DROP DATABASE name;` (optionally with `IF EXISTS`) permanently deletes a database and everything in it; PostgreSQL refuses to drop a database that is currently connected to, by you or anyone else, unless you explicitly force it with `WITH (FORCE)` (PostgreSQL 13+).
- Next, Topic 2 moves inside a database to the real focus of this module: designing and creating the tables that live within it.
