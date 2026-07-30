# GRANT and REVOKE

## Learning Objectives

By the end of this section you should be able to:
- Grant specific privileges (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, and DDL-related privileges) on specific objects (tables, schemas, databases) to a role
- Use `REVOKE` to precisely remove a privilege you previously granted
- Grant a privilege on only specific columns of a table, and explain when that's preferable to restricting the whole table
- Check exactly what privileges a given role currently holds on a given object

## Prerequisites

- [Users and Roles](01-users-and-roles.md) — you need a role to grant a privilege *to* before any of this topic is actionable; this topic assumes you can already create login and group roles.
- [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md) — `GRANT` and `REVOKE` are this course's canonical examples of the DCL category; this topic is that category's full depth treatment.
- [Module 6 — Modifying Data](../06-modifying-data/00-module-overview.md), especially [INSERT](../06-modifying-data/01-insert.md) and [UPDATE](../06-modifying-data/02-update.md) — the privileges granted here (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) are literally permission to run the exact statements those topics taught; you need to know what each statement actually does to appreciate what granting or withholding it means.

## Motivation

Creating a role (Topic 1) only answers "who." It answers nothing about "what." A freshly created role — even one thoughtfully grouped and named — can do precisely nothing beyond connecting, until specific privileges on specific objects are explicitly granted to it. `GRANT` and `REVOKE` are the two statements that actually wire a role to the data it's allowed (or no longer allowed) to touch, and getting their granularity right is the difference between a database that's genuinely secure and one that's only *organized* to look that way.

## Problem Statement

Continuing directly from Topic 1: you've created a `reporting_team` role and made `priya` a member of it. Right now, if `priya` connects and runs even the simplest query:

```sql
SELECT * FROM orders;
```

PostgreSQL responds:

```
ERROR:  permission denied for table orders
```

This is correct and expected — a newly created role has no privileges on any object by default. `reporting_team` needs to be told, explicitly, exactly which objects it may read, and nothing more. Meanwhile, an application's connection role needs a different, broader set — read *and* write on the specific tables the application touches, but still nothing on tables it has no business seeing, and no ability to alter table structure at all. Two different roles, two different privilege sets, both defined with the same two statements: `GRANT` and `REVOKE`.

## Concept

### The General Shape of GRANT and REVOKE

```sql
GRANT privilege [, ...] ON object TO role [, ...];
REVOKE privilege [, ...] ON object FROM role [, ...];
```

`REVOKE` is the precise mirror image of `GRANT` — whatever `GRANT` adds, the identically-shaped `REVOKE` removes. Nothing about the two statements is asymmetric; if you can grant it, you can revoke exactly that same thing.

### Table-Level Privileges

The privileges most directly tied to the DML/DQL categories (Module 6 and beyond) are granted per table:

```sql
-- reporting_team may only read
GRANT SELECT ON orders TO reporting_team;

-- app_service may read and fully modify data, but not restructure the table
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO app_service;
```

| Privilege | Allows |
|---|---|
| `SELECT` | Reading rows (DQL) |
| `INSERT` | Adding new rows (DML) |
| `UPDATE` | Modifying existing rows (DML) |
| `DELETE` | Removing rows (DML) |
| `TRUNCATE` | Emptying the table via `TRUNCATE` (distinct from `DELETE`'s privilege, since `TRUNCATE` behaves more like a DDL-adjacent operation, as Module 1 Topic 3 noted) |
| `REFERENCES` | Creating a foreign key that points *at* this table |
| `TRIGGER` | Creating a trigger on this table |

You can grant several at once, as shown above, or grant `ALL PRIVILEGES` to hand over everything table-level in one statement (used sparingly — see Best Practices):

```sql
GRANT ALL PRIVILEGES ON orders TO app_service;
```

Revoking mirrors this exactly:

```sql
-- app_service should never be able to empty the table wholesale
REVOKE TRUNCATE ON orders FROM app_service;

-- reporting_team's access is being shut off entirely for this table
REVOKE SELECT ON orders FROM reporting_team;
```

### Applying Privileges to Every Table in a Schema

Rather than repeating a `GRANT` per table, you can target every table currently in a schema in one statement:

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting_team;
```

This is a one-time convenience for *existing* tables only — it does **not** automatically apply to tables created afterward. For that, PostgreSQL provides a separate mechanism, `ALTER DEFAULT PRIVILEGES`, which changes what privileges get granted automatically the moment a *future* table is created:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO reporting_team;
```

After running this once, any table created later in `public` automatically grants `SELECT` to `reporting_team`, with no further action needed.

### Schema-Level and Database-Level Privileges

Tables don't exist in isolation — they live inside a schema, which lives inside a database (Module 4). Both of those containers have their own privileges, and a role typically needs a small amount of container-level access before its table-level grants even matter:

```sql
-- Permission to even connect to this database at all
GRANT CONNECT ON DATABASE storefront TO reporting_team;

-- Permission to "see into" this schema at all (a prerequisite for table access to matter)
GRANT USAGE ON SCHEMA public TO reporting_team;

-- Permission to create new objects (tables, etc.) inside this schema — a DDL-related privilege
GRANT CREATE ON SCHEMA public TO app_migrations;
```

| Container | Key privileges |
|---|---|
| Database | `CONNECT` (may open a connection at all), `CREATE` (may create new schemas), `TEMP` (may create temporary tables) |
| Schema | `USAGE` (may reference objects inside it at all), `CREATE` (may create new objects, such as tables, inside it — a DDL-related privilege) |

Notice `USAGE` on a schema is a quiet but essential prerequisite: granting `SELECT` on a table inside a schema a role has no `USAGE` on still results in `permission denied` — the role can't even "see" its way to the table.

### Column-Level Privileges

Sometimes an entire table shouldn't be uniformly readable — for example, `reporting_team` should see order totals and dates, but never a stored payment token. PostgreSQL lets you grant `SELECT` (and `UPDATE`) on a specific subset of a table's columns:

```sql
GRANT SELECT (order_id, customer_id, order_date, total_amount)
ON orders
TO reporting_team;
```

With only this grant, `reporting_team` can run:

```sql
SELECT order_id, total_amount FROM orders;
```

but attempting to include a column it wasn't granted:

```sql
SELECT order_id, payment_token FROM orders;
```

fails with:

```
ERROR:  permission denied for table orders
```

even though the same role can already read *other* columns of the exact same row. Column-level grants are used sparingly in practice — a view that exposes only the safe columns (a later module's topic) is often preferred for anything beyond a very small, stable set of restricted columns — but the underlying column-level privilege mechanism is what makes such fine-grained control possible at all.

### Checking a Role's Current Privileges

In `psql`, `\dp table_name` (or its full form `\z`) shows the access privileges currently set on a table:

```
                              Access privileges
 Schema |  Name  | Type  |         Access privileges         | Column privileges
--------+--------+-------+------------------------------------+--------------------
 public | orders | table | app_service=arwd/postgres         +| payment_token:     +
        |        |       | reporting_team=r/postgres          |   app_service=arwd/postgres
(1 row)
```

Reading the shorthand: `arwd` stands for `INSERT` (a), `SELECT` (r, "read"), `UPDATE` (w, "write"), `DELETE` (d) — so `app_service=arwd/postgres` means the role `app_service` holds all four, granted by `postgres`. `reporting_team=r/postgres` means `reporting_team` holds only `SELECT`.

For programmatic or cross-database-portable checks, the standard `information_schema` views work identically in spirit across most SQL databases:

```sql
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'orders';
```

```
   grantee      | table_name |  privilege_type
-----------------+------------+------------------
 app_service     | orders     | SELECT
 app_service     | orders     | INSERT
 app_service     | orders     | UPDATE
 app_service     | orders     | DELETE
 reporting_team  | orders     | SELECT
(5 rows)
```

You can also ask, from inside a specific role's own session, "can I actually do X?" using the `has_table_privilege` function — useful for scripting a check without parsing the full grant list yourself:

```sql
SELECT has_table_privilege('reporting_team', 'orders', 'DELETE');
```

```
 has_table_privilege
----------------------
 f
(1 row)
```

## Internal Working (Preview)

Every grantable object in PostgreSQL (tables, schemas, databases, and more) carries an internal **Access Control List (ACL)** — stored, for a table, in a system catalog column called `relacl` on `pg_class`. Each `GRANT` appends an entry to that list; each `REVOKE` removes the matching entry. Conceptually:

```
 Statement arrives (e.g. INSERT INTO orders ...)
        │
        ▼
 Parser identifies the target object (orders) and required privilege (INSERT)
        │
        ▼
 Resolve the calling role's full privilege set (own grants + inherited role
 memberships from Topic 1, walked transitively)
        │
        ▼
 Is the required privilege present in that resolved set for this object?
    ├─ Yes → proceed to planning/execution
    └─ No  → reject immediately: "permission denied for table orders"
```

This check happens before the statement is ever planned or executed — a role without `SELECT` on a table isn't merely denied *results*, it's denied the ability to run the query against that table at all. This is also why the DCL category was described back in Module 1 as being checked "before any other statement from a given user is allowed to execute" — permission checking sits structurally in front of every other subsystem (parser, planner, executor) a statement passes through.

## Real-World Analogy

Extend Topic 1's keycard analogy one step further: creating a role was defining a badge type. `GRANT` is the act of walking up to a specific door (a table) and programming it to accept that badge type — and, with column-level grants, it's like a door that only unlocks certain drawers inside the room rather than the whole room, letting the same badge type see the filing cabinet's "public reports" drawer but not its "payroll" drawer. `REVOKE` is removing that badge type from the door's accepted list — the door itself, and every other door, is completely unaffected; only that one specific door/badge-type pairing changes.

## Why GRANT and REVOKE Were Designed This Way

`GRANT` and `REVOKE` are deliberately symmetric, object-specific, and privilege-specific rather than an all-or-nothing switch, because SQL's declarative nature (Module 1) applies just as much to permissions as it does to querying data: you state the *end result* you want ("this role should be able to `SELECT` on this table") and the database enforces it on every future statement, rather than you writing procedural logic that checks permissions yourself before every query. The granularity down to individual privileges (not just "access" vs. "no access") and even individual columns exists because real-world access needs are rarely all-or-nothing — a role frequently needs to read one table but not write it, or read most of a table's columns but not a sensitive one, and the DBMS's grant system is built to express exactly that shape of requirement natively, rather than forcing it to be worked around in application code.

## Advantages

- **Fine-grained by design** — privileges are independently controllable per statement type (`SELECT` vs. `INSERT` vs. `DELETE`, etc.) and per object, so access can be shaped exactly to a role's real needs rather than approximated.
- **Column-level grants protect specific sensitive fields** without requiring an entirely separate table or duplicated data.
- **Perfectly symmetric revocation** — anything grantable is precisely revocable with the identically-shaped statement, so undoing a mistake or tightening access later is never more complex than the original grant.
- **Enforced by the database itself, not application code** — every connection, from every application and every person, is subject to the same checks, so there's no way to "forget" to apply a permission check in one code path.

## Disadvantages / Limitations

- **Many individual grants, without the role-grouping from Topic 1, become hard to audit** — a table with a long, scattered list of individual per-user grants is much harder to reason about than one with a handful of grants to well-named group roles.
- **`ALTER DEFAULT PRIVILEGES` is easy to forget** — a plain `GRANT ... ON ALL TABLES IN SCHEMA` only affects tables that already exist; teams that add new tables regularly and forget the "default privileges" step end up with inconsistent, surprising access on newly created tables.
- **Revoking a privilege from a role doesn't retroactively undo the effects of prior actions** — if a role used a privilege to insert rows before that privilege was revoked, those rows remain; `REVOKE` only affects what the role can do going forward, not what already happened (Module 14's transaction guarantees govern what "already happened" precisely means).

## Best Practices

- **Grant to roles, not to individual login accounts** — combine this topic directly with Topic 1: define the policy once on a group role, and manage who has that policy purely through role membership.
- **Avoid `GRANT ALL PRIVILEGES` out of convenience** — grant exactly the statements a role needs (`SELECT`, or `SELECT, INSERT, UPDATE, DELETE`), since "all privileges" almost always includes more than any single role actually requires (Topic 4 makes this a formal principle).
- **Remember `ALTER DEFAULT PRIVILEGES` whenever you grant on "all tables" in an actively growing schema** — otherwise every new table silently starts with no grants for roles that should have had them automatically.
- **Periodically audit real grants against `information_schema.role_table_grants`** rather than relying on memory of what was set up when the role was first created — schemas and requirements both drift over time.
- **Use column-level grants sparingly and only for a small, stable set of sensitive columns** — for broader column subsetting needs, a dedicated view is usually more maintainable.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Granting `SELECT` on a table but forgetting `USAGE` on its containing schema | The role still gets `permission denied` — `USAGE` on the schema is a silent prerequisite for any privilege on objects inside it to take effect. |
| Assuming `GRANT ... ON ALL TABLES IN SCHEMA public` covers tables created afterward | It only affects tables that exist at the moment the statement runs; new tables need `ALTER DEFAULT PRIVILEGES` to inherit the same grants automatically. |
| Using `GRANT ALL PRIVILEGES` as a shortcut during development and forgetting to narrow it before production | Leaves a role able to do far more than it needs (including, depending on the object, structural changes), directly undermining the least-privilege reasoning covered in Topic 4. |
| Believing `REVOKE` undoes data changes a role already made while it held a privilege | `REVOKE` only changes what is permitted from that point forward; rows already inserted, updated, or deleted before the revoke remain exactly as they were left. |

## Interview Questions

1. **Q: What is the relationship between `GRANT` and `REVOKE`?**
   A: They are exact mirror-image statements over the same privilege/object/role shape — anything `GRANT` can add to a role's permitted actions, the identically-structured `REVOKE` can remove. Neither statement affects data or table structure directly; both only affect the permission catalog.

2. **Q: Why might granting `SELECT` on a table still result in "permission denied" for a role?**
   A: Most commonly because the role lacks `USAGE` on the schema that table lives in — schema-level `USAGE` is a prerequisite for any privilege on an object inside that schema to actually take effect, and it's easy to grant the table-level privilege while forgetting the schema-level one.

3. **Q: When would you use a column-level `GRANT` instead of just granting `SELECT` on the whole table?**
   A: When a table mixes generally-readable columns with a small number of sensitive ones (e.g., a stored token or an internal-only flag) that a particular role should never see, while still needing to read the rest of that same row's columns.

4. **Q: What's the difference between `GRANT SELECT ON ALL TABLES IN SCHEMA public TO role;` and `ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO role;`?**
   A: The first grants `SELECT` on every table that exists in the schema *at the moment the statement runs*, and has no effect on tables created afterward. The second changes the *default* — any table created in that schema from then on will automatically have `SELECT` granted to the role, without needing a repeated manual `GRANT`.

## Summary

- `GRANT privilege ON object TO role;` and its exact mirror `REVOKE privilege ON object FROM role;` are how permissions are actually attached to and removed from roles.
- Table-level privileges (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER`) map directly onto the DML/DQL statement categories from Module 1 and Module 6.
- Databases and schemas have their own container-level privileges (`CONNECT`, `CREATE`, `USAGE`) that act as prerequisites before table-level grants matter at all.
- Column-level `GRANT` lets you expose most of a table's columns to a role while withholding specific sensitive ones, without a separate table.
- `\dp` in `psql`, `information_schema.role_table_grants`, and `has_table_privilege()` are the standard ways to check what a role can currently do.
- `ALTER DEFAULT PRIVILEGES` is the mechanism that makes grants apply automatically to tables created in the future, separate from a one-time grant on tables that exist right now.
- Next, Topic 3 turns to a very different kind of risk — SQL injection — and Topic 4 shows how everything learned here comes together as a single, deliberate design principle: least privilege.
