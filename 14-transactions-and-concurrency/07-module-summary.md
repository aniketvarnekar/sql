# Module 14 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **ACID Properties** — Atomicity, Consistency, Isolation, and Durability, each demonstrated with a bank-transfer scenario showing exactly what breaks in its absence
- [x] **BEGIN, COMMIT, and ROLLBACK** — autocommit mode vs. explicit transaction blocks, undoing a transaction entirely with `ROLLBACK`, and partial rollback within a transaction using `SAVEPOINT`
- [x] **Isolation Levels** — the SQL standard's four levels, which three PostgreSQL actually implements distinctly, and how to set the level per transaction
- [x] **Dirty Reads, Non-Repeatable Reads, and Phantom Reads** — precise definitions and two-transaction timelines for all three anomalies, plus the isolation-level-vs-anomaly mapping table
- [x] **Locking Fundamentals** — row-level vs. table-level locks, shared vs. exclusive modes, implicit locking by `UPDATE`/`DELETE`, explicit locking with `SELECT ... FOR UPDATE`, and what it means for a transaction to block
- [x] **Deadlocks** — the circular-wait definition, a concrete two-transaction deadlock, PostgreSQL's automatic detection and resolution, and prevention practices

## Practical Connections

- **Any feature that changes more than one related piece of data at once** — placing an order (inserting the order, decrementing inventory, recording a payment), transferring funds, or reassigning a shared resource — relies on exactly the transaction boundaries covered in Topics 1 and 2 to guarantee that a crash or error midway never leaves the data half-changed.
- **A reporting or analytics process that runs several queries and needs them to reflect one single, consistent moment in time** — rather than a mix of old and newly-committed data from other activity happening concurrently — relies on the Repeatable Read isolation level from Topic 3, and depends on understanding the non-repeatable and phantom read anomalies from Topic 4 to know why that level is necessary in the first place.
- **Any system where many users can simultaneously modify overlapping data** — a seat reservation system, an inventory count shared across many simultaneous orders, an account balance updated by many concurrent transfers — relies on the row-level locking behavior from Topic 5 to prevent lost updates, and on the deadlock prevention practices from Topic 6 (consistent lock ordering, short transactions) to avoid the one failure mode locking alone can't resolve.
- **Any application built with real concurrency in mind** must be written to expect and gracefully retry on a `deadlock detected` error and a serialization failure under stricter isolation levels — treating these as normal, occasional conditions to handle, not proof of a bug, is a mark of production-grade transactional code.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Atomicity vs. Isolation | Atomicity is about a single transaction's own statements succeeding or failing together as one unit. Isolation is about two *different* transactions not seeing each other's uncommitted, in-progress work. |
| `ROLLBACK` vs. `ROLLBACK TO SAVEPOINT` | `ROLLBACK` undoes everything since `BEGIN` and ends the transaction entirely. `ROLLBACK TO SAVEPOINT <name>` undoes only the statements after that checkpoint, keeping the transaction open and keeping everything committed before it. |
| Non-repeatable read vs. phantom read | A non-repeatable read is an *existing row's value* changing between two reads of that row. A phantom read is the *set of rows* matching a condition changing, because rows were inserted or deleted, not because an existing row's values changed. |
| Ordinary blocking vs. a deadlock | Ordinary blocking always resolves on its own once the transaction holding the lock commits or rolls back. A deadlock is a *cycle* of waiting — each transaction waiting on a lock the other holds — which cannot resolve without one transaction being forcibly aborted. |
| Read Committed vs. Repeatable Read | Read Committed takes a fresh snapshot for every individual statement, so a transaction's own repeated reads can see different, newly-committed data. Repeatable Read takes one snapshot for the whole transaction, so every statement in it sees the exact same data throughout, immune to both non-repeatable and (in PostgreSQL specifically) phantom reads. |
| Shared lock vs. exclusive lock | A shared lock can be held by many transactions on the same row at once (for reading with a guarantee against change). An exclusive lock can be held by only one transaction at a time and blocks every other lock request, shared or exclusive, on that row. |

## What's Next

This module gave you the complete picture of how a database keeps data correct and reliable when many statements — and many connections — touch the same data at the same time: the ACID guarantees a transaction makes, the exact syntax (`BEGIN`/`COMMIT`/`ROLLBACK`/`SAVEPOINT`) to control one, the isolation levels that tune how much concurrent activity a transaction can see, the precise anomalies those levels prevent or allow, the locks that make concurrent writes safe, and the one failure mode (deadlocks) that locking alone can't resolve on its own. Every module before this one assumed a single statement running in isolation; from here forward, you can reason correctly about what happens when many are running at once. **Module 15 — Normalization & Design** shifts focus back to the shape of your data itself: functional dependencies, the normal forms (1NF through BCNF), entity-relationship modeling, and when denormalizing a well-normalized design is actually the right call — the schema-design discipline that determines how clean (or how painful) every transaction, query, and constraint you've learned so far will be to work with in practice.
