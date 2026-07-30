# Users and Roles

## Learning Objectives

By the end of this section you should be able to:
- Create a role with `CREATE ROLE`, and explain why PostgreSQL treats "a user" as just a special case of a role
- Distinguish a login-capable role from a pure group role, and explain what the `LOGIN` attribute controls
- Grant one role membership in another, and explain the difference between `INHERIT` and `NOINHERIT` membership
- Modify an existing role's attributes with `ALTER ROLE`
- Explain, with a concrete scenario, why granting permissions to roles (and adding people to those roles) scales far better than granting permissions to individuals directly

## Prerequisites

- [Categories of SQL Commands](../01-introduction/03-categories-of-sql-commands.md) — this entire module lives inside the DCL category introduced there; you should already know that DCL is the category concerned with *permissions*, distinct from DDL, DML, DQL, and TCL.
- [Creating and Dropping Databases](../04-database-and-table-design/01-creating-and-dropping-databases.md) — roles are meaningless without something to grant access *to*; you should already be comfortable with the idea of a database as a container you can create and connect to.

## Motivation

The instant more than one person — or more than one application — connects to the same database, "who is allowed to do what" stops being an implicit assumption and becomes something the database itself must track, check, and enforce on every single statement. Before you can grant or restrict any permission (Topic 2), the database needs a first-class object to attach that permission to. That object is a **role**, and getting comfortable with how roles work — and, critically, how to *group* them — is what makes real-world permission management manageable instead of a constant, error-prone chore.

## Problem Statement

Imagine a small company's database supports: five backend developers who need full read/write access while building features, one data analyst who should only ever read data for reports, and one application server that connects automatically every time a customer places an order. That's seven distinct connections into the same database, each needing a different, specific level of access.

A naive approach is to create seven individual logins and, for each one, individually run every `GRANT` statement it needs on every table it needs. This has real, compounding problems:
- When an eighth developer joins, you must remember and re-run the *entire* list of grants a developer needs, by hand, hoping you don't miss one.
- When a developer leaves, you must remember every individual grant they were ever given, across every table, and revoke each one — and if you miss even one, they retain access after leaving.
- If the company later decides "developers should also be able to read a new `audit_log` table," you must go back and repeat one `GRANT` statement per developer, instead of expressing the idea once.

None of this is because `GRANT` and `REVOKE` (Topic 2) are badly designed — it's because individual accounts were used as the unit of permission management, instead of a role representing "what a developer is allowed to do."

## Concept

### There Is No Separate "User" Object in PostgreSQL

In PostgreSQL, every one of these — a person's login, an application's connection identity, and a permission-holding group with nobody able to log in as it directly — is internally the exact same kind of object: a **role**. The commands `CREATE USER` and `CREATE ROLE` create the identical underlying object; `CREATE USER` is simply shorthand that also sets the `LOGIN` attribute:

```sql
-- These two statements produce an equivalent role:
CREATE USER alice WITH PASSWORD 'a-strong-password';
CREATE ROLE alice WITH LOGIN PASSWORD 'a-strong-password';
```

The only thing that makes a role usable as a login is its `LOGIN` attribute. A role created without `LOGIN` still fully exists, can still be granted privileges, and can still have other roles as members — it just cannot itself be used to open a database connection. This single design choice (one object type, `ROLE`, with `LOGIN` as just one of its attributes) is what makes role-based grouping possible at all, as you'll see below.

### Creating Roles

```sql
-- A role that can log in (a "user" in the everyday sense)
CREATE ROLE alice LOGIN PASSWORD 'a-strong-password';

-- A role that exists purely to group permissions — nobody logs in as this directly
CREATE ROLE developers;

-- A role that can log in AND has extra attributes
CREATE ROLE app_service LOGIN PASSWORD 'another-strong-password' CONNECTION LIMIT 20;
```

Common role attributes you'll encounter:

| Attribute | Meaning |
|---|---|
| `LOGIN` / `NOLOGIN` | Whether this role can be used to open a database connection directly. `NOLOGIN` is the default for `CREATE ROLE`; `LOGIN` is the default for `CREATE USER`. |
| `PASSWORD 'x'` | The password required when connecting as this role (only meaningful alongside `LOGIN`). |
| `SUPERUSER` / `NOSUPERUSER` | Whether this role bypasses every permission check entirely. `NOSUPERUSER` is the default and should stay the default for essentially every role except the database administrator's own account. |
| `CREATEDB` / `NOCREATEDB` | Whether this role is allowed to create new databases. |
| `CREATEROLE` / `NOCREATEROLE` | Whether this role is allowed to create and manage other roles. |
| `CONNECTION LIMIT n` | Caps how many simultaneous connections this role may hold open. |
| `VALID UNTIL 'timestamp'` | An expiration timestamp after which this role's password stops being accepted — useful for temporary contractor access. |

### Role Membership and Inheritance

The real power of roles isn't creating them individually — it's that **one role can be a member of another**, and a member automatically picks up whatever the group role can do. This is done with `GRANT`, the same keyword used for privileges (Topic 2) — granting *role membership* is just another thing `GRANT` can express:

```sql
CREATE ROLE developers;
CREATE ROLE alice LOGIN PASSWORD 'x';
CREATE ROLE bob   LOGIN PASSWORD 'y';

-- Make alice and bob members of the developers role
GRANT developers TO alice;
GRANT developers TO bob;
```

Once `developers` is granted a set of table privileges (Topic 2 shows exactly how), both `alice` and `bob` automatically gain those same privileges — without a single `GRANT` statement ever mentioning `alice` or `bob` by name.

By default, membership is **inherited**: a member role automatically exercises the privileges of every role it belongs to, with no extra step. PostgreSQL also supports `NOINHERIT` membership, where a member must explicitly switch into the group role (using `SET ROLE`) before that role's privileges take effect — used far less often, typically when you want a login role to be able to *become* an administrative role deliberately, rather than always silently carrying that power.

```sql
-- Default: alice automatically exercises developers' privileges at all times
GRANT developers TO alice;

-- alice must explicitly SET ROLE developers to use its privileges
GRANT developers TO alice WITH INHERIT FALSE;
```

### Altering an Existing Role

`ALTER ROLE` changes attributes on a role that already exists — you don't need to drop and recreate it:

```sql
-- Change a password
ALTER ROLE alice WITH PASSWORD 'a-new-strong-password';

-- Give a role the ability to create databases
ALTER ROLE alice WITH CREATEDB;

-- Set an expiration for temporary access
ALTER ROLE contractor_jane VALID UNTIL '2026-09-30';

-- Rename a role
ALTER ROLE bob RENAME TO robert;
```

### Listing Roles

In `psql`, the meta-command `\du` (short for "describe users") lists every role and its attributes:

```
                                   List of roles
   Role name    |                         Attributes
-----------------+------------------------------------------------------------
 alice           |
 app_service     | Connection limit: 20
 bob             |
 developers      | Cannot login
 postgres        | Superuser, Create role, Create DB, Replication, Bypass RLS
(5 rows)
```

Notice `developers` shows `Cannot login` — confirming it's a pure group role, exactly as designed.

### Why Grouping Into Roles Scales

Revisit the Problem Statement with roles in place:

```sql
CREATE ROLE developers;
CREATE ROLE reporting_team;

-- (Topic 2 shows the actual privilege grants on tables — assume they're done)

CREATE ROLE alice LOGIN PASSWORD 'x'; GRANT developers TO alice;
CREATE ROLE bob   LOGIN PASSWORD 'y'; GRANT developers TO bob;
-- ... three more developers, same one-line pattern
CREATE ROLE priya LOGIN PASSWORD 'z'; GRANT reporting_team TO priya;
```

Now:
- **A new developer joins:** `CREATE ROLE`, then one `GRANT developers TO new_dev;` — done. They inherit every privilege ever granted to `developers`, including ones granted last year, without you having to remember or re-list them.
- **A developer leaves:** `REVOKE developers FROM alice;` (or drop the role entirely) — one statement removes every privilege they had via that membership, instantly and completely.
- **Developers need access to a new table:** one `GRANT ... TO developers;` statement, and every current *and future* member of `developers` gains it automatically — you never touch individual accounts again.

The role is doing exactly what a well-designed abstraction should: it lets you state a policy once ("developers can do X") and have every person who fits that description automatically comply, rather than repeating the policy per person and hoping you remember to keep every copy in sync.

## Internal Working (Preview)

PostgreSQL stores every role as a row in the system catalog `pg_roles` (a view over the lower-level `pg_authid`), and every membership relationship as a row in `pg_auth_members` (who is a member of whom, and whether that membership is inherited). Conceptually:

```
 Login attempt as "alice"
        │
        ▼
 Look up "alice" in pg_roles → LOGIN? password/auth method correct?
        │
        ▼
 Resolve alice's full privilege set:
   alice's own directly-granted privileges
        +
   privileges of every role alice is an (inherited) member of
        +
   privileges of every role THOSE roles are members of, transitively
        │
        ▼
 This combined set is what every later permission check (Topic 2) is tested against
```

Membership can be nested arbitrarily deep — `alice` can be a member of `developers`, which is itself a member of `staff` — and, as long as every step is `INHERIT`, PostgreSQL walks the whole chain automatically when deciding what `alice` can do.

## Real-World Analogy

Think of an office building's electronic keycard system. You don't program the door to the server room to recognize each of forty individual employees' keycards one by one. Instead, you define a badge *group* — "Infrastructure Team" — program the server room door to accept that group once, and then simply add or remove each employee's keycard from the group as they join or leave the team. Adding a new infrastructure engineer is "add this badge to the Infrastructure Team group," not "reprogram every door in the building to also recognize this new person's badge." A pure group role with `NOLOGIN` is like a badge-group definition that has no physical badge of its own — it exists purely to hold a set of door permissions that real badges (login roles) can be added to.

## Why Roles Were Designed This Way

Collapsing "user" and "group" into a single `ROLE` concept — distinguished only by the `LOGIN` attribute — is a deliberate simplification. If PostgreSQL instead had two structurally different object types (a rigid "USER" type and a separate "GROUP" type, as some older database systems do), you'd need entirely separate rules for "can a user belong to another user," "can a group belong to a group," and "can a group log in." By making membership just another relationship between two instances of the *same* object type, PostgreSQL gets nested groups, groups-of-groups, and even a login role temporarily acting as another login role (`SET ROLE`) all from one consistent mechanism, with one set of rules — a direct application of good relational design's habit of preferring one clean, general structure over several special-cased ones.

## Advantages

- **Permissions are declared once per policy, not once per person** — a role represents "what this kind of account is allowed to do," and any member automatically complies.
- **Onboarding and offboarding become single statements** — `GRANT role TO new_person;` and `REVOKE role FROM leaving_person;` instead of replaying or hunting down a long, easily-incomplete list of individual grants.
- **Nested roles mirror real organizational structure** — a `senior_developers` role can itself be a member of `developers`, layering additional privileges on top of a shared baseline without duplicating the baseline's grants.
- **Auditing is far simpler** — "what can Alice do?" reduces to "what roles is Alice a member of, and what is each of those roles granted?" rather than manually reconstructing a scattered history of individual grants.

## Disadvantages / Limitations

- **Deep or tangled membership chains become hard to reason about** — if role memberships are nested many levels deep, or if the same person is a member of several overlapping group roles, it can become genuinely difficult to answer "what, exactly, can this account do?" without careful tooling or querying the catalogs directly.
- **`NOINHERIT` membership is easy to forget about** — since inherited membership (the default) is silent and automatic, a `NOINHERIT` grant that requires an explicit `SET ROLE` can surprise someone who expects all memberships to behave the same way.
- **Role design still requires upfront thought** — grouping only pays off if the groups reflect real, stable job functions; overly fine-grained or poorly named roles (e.g., one role per person, defeating the entire point) recreate the original problem under a different name.

## Best Practices

- **Name roles after function, not after a person or team roster** — `reporting_team` or `app_service`, not `q3_2026_analysts`, so the role's purpose stays meaningful even as the people (or applications) behind it change.
- **Make pure group roles `NOLOGIN`** — if nobody should ever connect *as* `developers` directly, don't give it a password or `LOGIN` at all; only the individual login roles that are members of it should be connectable.
- **Grant privileges to the group role, then grant membership to individuals** — never grant a privilege directly to a person's login role if a suitable group role already exists or should exist; this is what makes onboarding/offboarding a single statement (see Advantages).
- **Reserve `SUPERUSER` and `CREATEROLE` for a small number of true administrator accounts** — every other role, including every application connection, should never carry these attributes (Topic 4 covers this reasoning in full).
- **Use `VALID UNTIL` for temporary access** (contractors, short-term audits) so expired access doesn't rely on someone remembering to manually revoke it later.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `CREATE USER` and `CREATE ROLE` create fundamentally different kinds of objects | They create the exact same object type; `CREATE USER` is purely a convenience alias that also sets `LOGIN`. There is no separate "user" table or object type internally. |
| Granting every privilege directly to each person's individual login role | Works at first, but every new hire, departure, or policy change then requires editing every individual account by hand instead of a single group-role grant — this is exactly the scaling problem this topic exists to solve. |
| Giving a group role `LOGIN` "just in case someone needs to use it directly" | Defeats the purpose of a group role and creates a shared, harder-to-audit credential; if someone needs elevated access temporarily, use `SET ROLE` from their own login role instead. |
| Forgetting that membership can be `NOINHERIT` and assuming every `GRANT role TO x` behaves identically | `NOINHERIT` members must explicitly `SET ROLE` before that role's privileges take effect — checking `\du` or the catalogs is the only way to be sure which kind of membership you're looking at. |

## Interview Questions

1. **Q: In PostgreSQL, what is the actual difference between `CREATE USER` and `CREATE ROLE`?**
   A: None, structurally — both create a `ROLE` object. `CREATE USER` is shorthand for `CREATE ROLE ... LOGIN`; a "user" is simply a role that has the `LOGIN` attribute set, allowing it to be used to open a database connection.

2. **Q: Why would you create a role that has `NOLOGIN`?**
   A: To use it as a pure permission-grouping mechanism — a role that privileges are granted to, and that other login-capable roles are made members of, without that role ever being connected to directly. This lets you define a policy ("what a developer can do") once and have every current and future member automatically comply.

3. **Q: What does it mean for role membership to be "inherited," and what's the alternative?**
   A: With inherited membership (the default), a member role automatically exercises the privileges of every role it belongs to, with no extra action required. The alternative, `NOINHERIT` membership, requires the member to explicitly run `SET ROLE` to temporarily "become" the group role before its privileges take effect.

4. **Q: A company has ten developers who all need the same table privileges. What's the disadvantage of granting those privileges to each developer's login role individually, instead of creating a shared `developers` role?**
   A: Every future change — a new developer joining, one leaving, or the privilege set itself changing — requires repeating or manually re-auditing the same set of `GRANT`/`REVOKE` statements across every individual account, which is slow and highly error-prone. A shared role lets you state the policy once and manage membership (adding/removing a single line) instead.

## Summary

- PostgreSQL has exactly one kind of permission-holding object, the **role** — "user" is not a separate object type, just informal language for a role that has `LOGIN`.
- `CREATE ROLE` (optionally with `LOGIN`) creates a role; `CREATE USER` is shorthand that always sets `LOGIN`; `ALTER ROLE` changes an existing role's attributes without recreating it.
- `GRANT group_role TO member_role;` makes one role a member of another, and by default the member automatically inherits everything the group role can do.
- Group roles are typically created with `NOLOGIN` — they exist purely to hold a policy's worth of privileges, which real login roles are then added to as members.
- Granting permissions to roles rather than individuals turns onboarding, offboarding, and policy changes into single statements, instead of manually replaying or auditing a growing, error-prone list of individual grants.
- Next, Topic 2 covers exactly how to grant those privileges in the first place — `GRANT` and `REVOKE` on tables, schemas, and databases.
