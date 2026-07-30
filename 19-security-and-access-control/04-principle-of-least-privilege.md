# Principle of Least Privilege

## Learning Objectives

By the end of this section you should be able to:
- State the principle of least privilege precisely and explain what "minimum necessary" means in a database context
- Design a realistic privilege set for an application's own database connection role
- Design a realistic privilege set for a read-only reporting role
- Explain, concretely, how least privilege reduces the "blast radius" of a compromised credential or an unforeseen bug

## Prerequisites

- [Users and Roles](01-users-and-roles.md) and [GRANT and REVOKE](02-grant-and-revoke.md) — this topic is an applied design discipline built entirely out of the mechanisms those two topics already taught; nothing here is a new SQL statement, only a deliberate way of using the ones you already know.
- [SQL Injection](03-sql-injection.md) — this topic's central argument (bounding damage *in case* something goes wrong) makes the most sense once you've seen a concrete way something can go wrong despite careful application code.

## Motivation

Topics 1 and 2 gave you every tool needed to create roles and grant or withhold any privilege with total precision. This topic answers the question those tools raise but don't answer on their own: **how much should any given role actually get?** The principle of least privilege is the single design rule that turns "I know how to use GRANT and REVOKE" into "I know how to set up a database that stays reasonably safe even when something eventually goes wrong" — and something eventually going wrong (a bug, a leaked password, a misconfigured service) is not a hypothetical; it is a routine, expected part of operating any real system over time.

## Problem Statement

Consider a small storefront application whose database connection was set up during early development by simply reusing the database's own superuser/owner role, because it was the fastest way to get the application working. Months later, a subtle bug in how the application builds one particular query turns out to be exploitable roughly the way Topic 3 described — an attacker manages to get arbitrary SQL executed under that same connection.

Because that connection's role can do *everything* — read and write every table, alter or drop any table, create and drop entire schemas, even create new roles with elevated privileges — the resulting damage is total: customer data can be exfiltrated, the schema itself can be destroyed, and the attacker can potentially create a new, persistent role of their own to regain access later, even after the original bug is fixed.

Now consider the same breach, but where that connection's role had only ever been granted `SELECT`, `INSERT`, `UPDATE`, and `DELETE` on the three specific tables the application actually uses — nothing else, no `DROP`, no `ALTER`, no `CREATE`, no ability to touch any other table, and certainly no ability to create new roles. The exact same bug, exploited the exact same way, can now only read and modify data within those three tables. The schema is intact, other tables are untouched, and the attacker cannot grant themselves anything further. The vulnerability existed in both scenarios — the *outcome* was entirely different, because of a decision made purely with `GRANT`/`REVOKE` (Topic 2), long before any bug was ever exploited.

## Concept

### The Principle, Precisely

**The principle of least privilege states that a role should be granted only the minimum set of privileges necessary to perform its intended function — never more, "just in case" a future need might arise.** It is not a specific SQL feature; it is a design discipline applied on top of the exact `CREATE ROLE` and `GRANT`/`REVOKE` mechanisms already covered in Topics 1 and 2. Applying it well means, for every role, being able to answer: "what, specifically, does this role's job require, and have I granted anything beyond that?"

### Concrete Example: An Application's Connection Role

An application server needs to read and modify data as part of normal operation, but it should never need to restructure the schema, manage other roles, or touch tables outside its own domain:

```sql
CREATE ROLE app_service LOGIN PASSWORD 'a-strong-generated-password';

GRANT CONNECT ON DATABASE storefront TO app_service;
GRANT USAGE ON SCHEMA public TO app_service;

GRANT SELECT, INSERT, UPDATE, DELETE
ON orders, customers, order_items
TO app_service;
```

Just as important as what's granted is what's deliberately withheld — none of the following are ever given to `app_service`:

```sql
-- Explicitly NOT granted to app_service:
--   CREATE on the schema or database   (no ability to make new tables)
--   Any privilege on tables outside its own domain (e.g. an internal audit_log table)
--   CREATEROLE / SUPERUSER attributes   (no ability to manage roles or bypass checks)
--   GRANT OPTION on anything            (no ability to hand its own access to others)
```

If the default `public` schema grants `CREATE` to every role by default in your PostgreSQL setup (a common default worth checking), it should be explicitly revoked:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

This one role can now do exactly what the application's normal operation requires — read and write rows in three named tables — and structurally nothing more.

### Concrete Example: A Reporting Role

A business intelligence tool or an analyst only ever needs to read data, never to change it:

```sql
CREATE ROLE reporting_user LOGIN PASSWORD 'a-different-strong-password';

GRANT CONNECT ON DATABASE storefront TO reporting_user;
GRANT USAGE ON SCHEMA public TO reporting_user;

GRANT SELECT
ON orders, customers, order_items
TO reporting_user;
```

Notice there is no `INSERT`, `UPDATE`, or `DELETE` at all — not even on the same three tables the application role can modify. If this role's password is ever embedded, unencrypted, in a BI tool's configuration file and later leaks (a genuinely common way credentials leak in practice), the most an attacker can do with it is **read** data. They cannot alter an order, delete a customer record, or write anything at all — the credential's entire capability is bounded by a single privilege, granted deliberately.

### Blast Radius: The Core Idea

"Blast radius" is the practical question this whole topic is really about: **if this specific credential is compromised right now, what is the total possible damage?** Least privilege doesn't try to prevent every possible bug or leak from ever happening — that's not realistic for any system operated over a long period of time. Instead, it accepts that compromises will eventually happen, to *some* credential, at *some* point, and asks: given that, how do we make sure any single compromise is as contained as possible?

| Role | If compromised, attacker can... | If compromised, attacker cannot... |
|---|---|---|
| `app_service` (scoped to 3 tables, DML+DQL only) | Read/modify rows in `orders`, `customers`, `order_items` | Touch any other table, alter/drop any table, create new roles, grant itself more access |
| `reporting_user` (`SELECT` only, same 3 tables) | Read rows in those 3 tables | Modify or delete any row, touch any other table, alter structure at all |
| The original superuser-reused connection | Everything — every table, every schema, role creation, structural changes | Nothing is withheld — this is precisely the scenario this principle exists to avoid |

### Least Privilege Applies to DCL Itself, Not Just DML

A subtle but important extension of the principle: a role should almost never be able to grant privileges to *other* roles, even if it's fine for that role to hold those privileges itself. PostgreSQL's `WITH GRANT OPTION` and the `CREATEROLE` attribute are exactly the kind of thing that should be withheld from an application's own connection role — otherwise, a compromised `app_service` credential wouldn't just be able to misuse its own access, it could actively *expand* access for a role of the attacker's choosing. Least privilege isn't only about restricting DML/DQL; it applies with equal force to DCL (Module 1 Topic 3's category for `GRANT`/`REVOKE` itself).

## Internal Working (Preview)

There is no separate enforcement mechanism for "least privilege" inside PostgreSQL — it is enforced by the exact same ACL check described in Topic 2's Internal Working section, applied to a role whose granted set was simply kept smaller on purpose:

```
 Statement arrives under role "app_service"
        │
        ▼
 Resolve app_service's full privilege set
   (only what was explicitly GRANTed — Topics 1 & 2)
        │
        ▼
 Required privilege present in that (deliberately small) set?
    ├─ Yes → proceed
    └─ No  → permission denied
```

Least privilege is not a new feature to learn — it's a deliberate, disciplined choice about what goes into the "resolve privilege set" step above, made in advance, before any bug or credential leak ever puts that choice to the test.

## Real-World Analogy

Think of hotel staff keycards. A housekeeper's keycard reliably opens every guest room on their assigned floor and the supply closet — everything their job genuinely requires — but it does not open the manager's office safe, the security control room, or the floors they don't service, even though the housekeeper is fully trusted to do their job well. This isn't distrust of the housekeeper; it's a recognition that keycards get lost, cloned, or borrowed, and when that happens, the hotel wants the *scope* of what a lost card can access to match the job it was issued for — not the entire building. A lost housekeeping card is an inconvenience contained to a few rooms; a lost master key is a building-wide incident.

## Why This Principle Matters at the Database Layer Specifically

This connects directly back to Module 1 Topic 3's original framing of DCL: "a production application's database user is commonly granted only DML and DQL permissions... but explicitly denied DDL... and denied DCL." Least privilege is the reasoning *behind* that specific, concrete recommendation — it isn't an arbitrary convention. Since Topic 3 established that vulnerabilities like SQL injection can, despite an application team's best efforts, eventually let untrusted input reach the database as an executed statement, the database's own permission system (Topics 1 and 2) is the last remaining, independent layer of defense once that happens. Relying solely on "the application code will never have a bug" is optimistic in a way that decades of real-world breaches have not rewarded; bounding what a compromised credential can *possibly* do, at the database layer itself, works even when every other layer of defense has already failed.

## Advantages

- **Bounds the blast radius of any single compromised credential** — a leaked or misused password can only do what its role was actually granted, no matter how it was obtained or exploited.
- **Makes audits far simpler** — a role's granted privileges directly describe its job function, so "why does this role have this access?" almost always has an obvious, defensible answer.
- **Catches accidental over-privileging early** — deliberately designing a minimal grant set for a new role's specific job often surfaces "wait, does this actually need that?" questions during setup, rather than after an incident.
- **Reduces the chance of an accidental, non-malicious catastrophe** — an application bug that tries to run an unintended `DROP TABLE` or a stray schema change simply fails with `permission denied` if the role was never granted DDL, rather than silently succeeding.

## Disadvantages / Limitations

- **Requires more upfront design discipline** than "grant everything and move on" — every new role means genuinely thinking through what it needs, rather than reusing a broad, already-working credential.
- **Legitimate new needs require an explicit grant, which can create friction** if a role's job function grows — though in practice this is a single, fast `GRANT` statement, not a redesign, and that friction is a reasonable cost for the containment it buys.
- **Over-restricting without planning for legitimate edge cases can cause confusing failures** — for example, if a runtime process occasionally needs a one-off administrative action and nobody anticipated it, the resulting `permission denied` can be surprising if the restriction wasn't clearly documented as intentional.

## Best Practices

- **Use separate roles for separate functions** — a runtime application role (DML + DQL only), a migrations/schema-management role (DDL, used rarely and typically not by the running application itself), a reporting role (`SELECT` only), and a small number of genuine administrator roles, rather than one broad role covering all of them.
- **Never let an application's runtime connection role hold DDL or DCL privileges** — schema changes and permission management should happen through a separate, more tightly controlled role and process, not the same credential handling live traffic.
- **Give read-only tools (BI dashboards, reporting, ad hoc analyst access) a role with `SELECT` only** — never grant write access "just in case a report someday needs to write back," since that "someday" need can always be granted later, precisely, when it actually arises.
- **Periodically audit granted privileges against actual usage** (Topic 2's `information_schema.role_table_grants` is the right tool for this) — roles accumulate unnecessary grants over time as requirements shift, and a stale, over-broad grant is exactly the risk this principle exists to prevent.
- **Combine this with role membership from Topic 1** — define the minimal privilege set once on a well-named group role, and manage who/what holds that role through membership, so the discipline scales as the number of applications and people grows.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using a database's superuser or table-owner role as an application's everyday connection credential | Removes every possible boundary on what a bug or leaked credential can do — the exact scenario this topic's Problem Statement walks through. |
| Granting broad or "all privileges" access early in development and never revisiting it before production | Convenient short-term, but ships an unnecessarily large blast radius into production; access should be narrowed deliberately, not left at whatever was easiest while prototyping. |
| Giving a reporting or analytics tool write access "just in case a report needs to write something back someday" | Violates least privilege for a hypothetical, not an actual, requirement; if a genuine write need arises later, it can be granted precisely at that time instead of preemptively. |
| Treating migration/schema-management privileges the same as everyday runtime privileges | DDL changes are rare, high-impact, and usually deliberate/manual; bundling that power into the same role that handles constant live traffic multiplies the risk unnecessarily. |

## Interview Questions

1. **Q: State the principle of least privilege as it applies to database roles.**
   A: A role should be granted only the minimum set of privileges genuinely necessary for its intended function, and nothing beyond that — extra privileges granted "just in case" only increase risk without providing any corresponding benefit to that role's actual job.

2. **Q: Give a concrete example of applying least privilege to an application's own database connection.**
   A: Grant the application's connection role `SELECT`, `INSERT`, `UPDATE`, and `DELETE` on only the specific tables it actually uses, and explicitly withhold DDL privileges (`CREATE`, `ALTER`, `DROP`) and DCL privileges (`GRANT`, `CREATEROLE`), since normal application operation never requires restructuring the schema or managing other roles.

3. **Q: How does least privilege reduce the impact of a SQL injection vulnerability, given that the vulnerability itself still exists?**
   A: Least privilege doesn't prevent the injection from occurring, but it bounds what the resulting arbitrary SQL execution can actually accomplish — if the compromised role only ever held `SELECT`/`INSERT`/`UPDATE`/`DELETE` on a small set of specific tables, the attacker's exploited access inherits exactly those same limits, rather than being able to read or destroy anything in the entire database.

4. **Q: What's a reasonable trade-off least privilege introduces, and why is it usually worth accepting?**
   A: It requires more deliberate, upfront role design, and legitimate new needs require an explicit follow-up `GRANT` rather than already being covered by a broad existing credential. This is a small, fast cost (a single statement) compared to the benefit of bounding the damage from an eventual compromised credential or bug, which real systems should assume will eventually happen at some point.

## Summary

- The principle of least privilege means granting a role exactly the privileges its actual job requires, and nothing more — it is a design discipline applied on top of the `CREATE ROLE`/`GRANT`/`REVOKE` mechanisms from Topics 1 and 2, not a new SQL feature.
- A well-designed application connection role gets `SELECT`, `INSERT`, `UPDATE`, `DELETE` on only the specific tables it uses — never DDL, never DCL, never access to unrelated tables.
- A reporting role gets `SELECT` only, on only the tables it needs to read — no write access "just in case."
- "Blast radius" is the practical question this principle answers: given that some credential will eventually be compromised or misused, how much damage can that specific credential actually do? Least privilege minimizes the answer.
- This principle is the direct, applied reasoning behind the DCL guidance introduced back in Module 1 Topic 3, and it works as an independent layer of defense even when application-level protections (like Topic 3's parameterized queries) unexpectedly fail.
- This closes out Module 19 — the next topic is this module's summary, consolidating roles, grants, injection, and least privilege into one coherent security model.
