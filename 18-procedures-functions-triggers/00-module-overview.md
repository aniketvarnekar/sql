# Module 18 — Procedures, Functions & Triggers

## Module Goal

By the end of this module, you will be able to move small pieces of logic *into* the database itself instead of always writing them in whatever application code talks to it. You will know how to package a reusable calculation as a **user-defined function** that can be called like any built-in expression inside a query, how to write a **stored procedure** that can perform multi-step work and control its own transaction boundaries with `COMMIT`/`ROLLBACK`, and how to attach a **trigger** so the database reacts automatically — with no application code involved at all — whenever a row is inserted, updated, or deleted. Just as importantly, you will come away with a clear-eyed, honest framework for *when* each of these tools is the right call and when it is a trap: all three are powerful precisely because they run invisibly, close to the data, and that same property makes them easy to overuse into a tangle of hidden logic nobody can find, test, or debug.

## Topics Covered in This Module

1. **[User-Defined Functions](01-user-defined-functions.md)** — `CREATE FUNCTION` with PL/pgSQL, parameters and return types, `RETURNS TABLE` for set-returning functions, and when a function is the right tool.
2. **[Stored Procedures](02-stored-procedures.md)** — `CREATE PROCEDURE`, the `CALL` statement, and the defining difference from a function: a procedure can manage its own transactions.
3. **[Triggers](03-triggers.md)** — `CREATE TRIGGER`, `BEFORE`/`AFTER`, row-level vs. statement-level firing, the `NEW`/`OLD` record variables, and a worked audit-logging example.
4. **[When to Use Functions, Procedures, and Triggers](04-when-to-use-each.md)** — a decision framework comparing all three against each other and against plain application-level logic, and an honest look at the trade-offs of putting logic inside the database.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap.

## Prerequisites

This module assumes you have completed Modules 1 through 17 in order. Three earlier modules matter more than the rest for the specific material here:

- **Module 6 (Modifying Data)** — every trigger in this module fires in direct response to an `INSERT`, `UPDATE`, or `DELETE` statement (see the [Module 6 overview](../06-modifying-data/00-module-overview.md) and its [INSERT](../06-modifying-data/01-insert.md) and [UPDATE](../06-modifying-data/02-update.md) topics). You cannot reason about *when* a trigger runs, or what `NEW` and `OLD` contain, without already being comfortable with what those three statements do to a row.
- **Module 5 (Constraints & Keys)** — constraints (see the [Module 5 overview](../05-constraints-and-keys/00-module-overview.md), [NOT NULL and DEFAULT](../05-constraints-and-keys/01-not-null-and-default.md), and [Primary Keys](../05-constraints-and-keys/03-primary-keys.md)) are the simpler, declarative, database-level enforcement mechanism that a trigger is sometimes reached for instead of. You need to already understand what a `CHECK` constraint or foreign key *can* express before you can judge whether a trigger is genuinely necessary or is solving a problem a constraint already solved more simply.
- **Module 14 (Transactions & Concurrency)** — see its [module overview](../14-transactions-and-concurrency/00-module-overview.md). Stored procedures are the one piece of this module that can issue `COMMIT` and `ROLLBACK` from inside a body of procedural code, so you need to already understand what a transaction is and what those two statements guarantee before that capability means anything.

Beyond those three, this module also assumes the general query-writing fluency built across Modules 7–17 — you will see `SELECT`, joins, and aggregate-style reasoning used freely inside function and trigger bodies without re-explanation.

## How to Study This Module

Read Topics 1 through 3 in order — they build a single mental ladder. Topic 1 (functions) is the simplest and most self-contained case: a named, reusable piece of logic you call like any other expression. Topic 2 (procedures) looks almost identical on the surface but exists specifically to lift one restriction functions have — the inability to control transactions — so read it with Topic 1 fresh in mind and pay close attention to *why* that one restriction forces two separate database objects to exist instead of one. Topic 3 (triggers) is the most conceptually different of the three: instead of something you call, it is something the database calls *for* you, invisibly, whenever a matching statement runs — this is also the topic with the largest gap between "easy to write" and "easy to reason about once several of them exist," so don't rush the Internal Working and Disadvantages sections. Topic 4 is not a syntax topic at all — it is a judgment-call topic, and arguably the most important one in the module: come back to it every time you're tempted to reach for a function, procedure, or trigger in a real schema, long after you've memorized the syntax of all three.
