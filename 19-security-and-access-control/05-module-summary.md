# Module 19 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Users and Roles** — `CREATE ROLE`/`CREATE USER`, the `LOGIN` attribute as the only real distinction between a "user" and a group role, role membership via `GRANT role TO role`, inheritance vs. `NOINHERIT`, and `ALTER ROLE`
- [x] **GRANT and REVOKE** — table-, schema-, and database-level privileges, column-level grants, `ALTER DEFAULT PRIVILEGES`, and how to check a role's current privileges with `\dp`, `information_schema`, and `has_table_privilege()`
- [x] **SQL Injection** — a precise definition, a full worked exploit of a vulnerable concatenated query, parameterized queries/prepared statements as the structural fix, and why escaping alone is fragile
- [x] **Principle of Least Privilege** — concrete privilege designs for an application connection role and a read-only reporting role, and the "blast radius" reasoning behind bounding what any single credential can do

## Practical Connections

- **Every production application's database connection is, in effect, a role designed exactly as Topic 4 describes** — scoped `SELECT`/`INSERT`/`UPDATE`/`DELETE` on a known, fixed set of tables, with DDL and DCL explicitly withheld — whether or not the team that set it up ever used the term "least privilege."
- **A reporting dashboard querying millions of rows every day relies on exactly the read-only role pattern from Topic 4** — a `SELECT`-only credential means that dashboard's data source, however widely its connection string ends up copied into config files and scripts, can never be the reason a table gets modified or dropped.
- **Any system that accepts user-entered text and later uses it inside a query — a search box, a login form, a filter — depends on Topic 3's parameterization being applied consistently**, and a single missed spot anywhere in that system is enough to reopen the exact vulnerability class this module worked through.
- **A growing team with regular hiring, offboarding, and shifting responsibilities relies on Topic 1's role-membership model** to keep access management a matter of single `GRANT`/`REVOKE` statements on memberships, rather than an ever-drifting, manually-audited list of individual per-person privileges.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| A PostgreSQL "user" vs. a "role" | There is no separate object type — a user is simply a role with the `LOGIN` attribute set; `CREATE USER` is shorthand for `CREATE ROLE ... LOGIN`. |
| `GRANT` on a table vs. `GRANT` on role membership | The same keyword expresses two different ideas: granting a *privilege* on an *object* (Topic 2) versus granting *membership* of one role in another (Topic 1) — context (an object name vs. another role name) tells you which is meant. |
| Escaping input vs. parameterizing a query | Escaping transforms untrusted text so it's *safe to concatenate* into SQL and must be applied correctly at every point, in the right context, every time. Parameterization removes concatenation of untrusted input from the picture entirely, so there's no escaping step to get right or forget. |
| A privilege vs. a role attribute | Privileges (`SELECT`, `INSERT`, `CREATE` on a schema, etc.) are granted on specific objects via `GRANT`/`REVOKE` (Topic 2). Role attributes (`SUPERUSER`, `CREATEDB`, `CREATEROLE`, `LOGIN`) are properties of the role itself, set via `CREATE ROLE`/`ALTER ROLE` (Topic 1), and are not tied to any single object. |
| `REVOKE`ing a privilege vs. undoing data already changed | `REVOKE` only changes what a role is permitted to do from that point forward; rows already inserted, updated, or deleted while the role held that privilege are unaffected by the revoke itself. |

## What's Next

Module 19 gave you the complete picture of how a relational database governs access to itself: roles as the unit of identity and grouping, `GRANT`/`REVOKE` as the precise mechanism for what each role can touch, SQL injection as a concrete illustration of why that mechanism matters even when application code has bugs, and least privilege as the design discipline that ties it all together. **Module 20 — Performance Tuning** shifts focus from *who can run a query* to *how efficiently the database runs the queries it's allowed to* — reading execution plans with `EXPLAIN ANALYZE`, recognizing common query anti-patterns, and understanding how the query planner (first previewed back in Module 1) actually chooses a strategy for any given statement.
