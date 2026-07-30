# Locking Fundamentals

## Learning Objectives

By the end of this section you should be able to:
- Distinguish row-level locks from table-level locks and explain when PostgreSQL uses each
- Distinguish a shared lock from an exclusive lock and explain what each does and does not block
- Explain which statements implicitly acquire row-level locks, and exactly which rows get locked
- Use `SELECT ... FOR UPDATE` to explicitly lock rows within a transaction, and explain why you'd need to
- Describe what it means for a transaction to "block" and wait on a lock, and how that resolves

## Prerequisites

- [BEGIN, COMMIT, and ROLLBACK](02-begin-commit-rollback.md) — locks are acquired and released within the boundaries of a transaction, so you need to be comfortable with explicit transaction blocks first.
- [Dirty Reads, Non-Repeatable Reads, and Phantom Reads](04-dirty-nonrepeatable-and-phantom-reads.md) — this topic explained what isolation *guarantees*; this topic explains the actual mechanism (locks) that makes writes safe under concurrency, working alongside the MVCC snapshots already introduced.

## Motivation

Topics 1 through 4 described what the database *guarantees* under concurrency — atomicity, isolation levels, which read anomalies are or aren't possible. None of that explained how the database actually stops two transactions from corrupting the same row at the same time when they both try to *write* to it. Reads are handled by MVCC snapshots (Topic 3) — readers never block writers and writers never block readers in PostgreSQL. But when two transactions both want to *change* the same row, something has to make one of them wait. That something is a lock.

## Problem Statement

Suppose two transactions both try to update Asha's balance at almost the same moment:

```sql
-- Transaction A
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';

-- Transaction B, running concurrently
UPDATE accounts SET balance = balance - 50 WHERE owner = 'Asha';
```

If both transactions were allowed to read the current balance, compute a new value, and write it back completely independently, one of two things could go wrong: either the final result reflects only *one* of the two changes (a "lost update," where whichever write happens last simply overwrites the other's effect as if it never happened), or the database ends up in an inconsistent internal state entirely. The database needs a mechanism to guarantee that when two transactions want to change the same row, one of them genuinely waits for the other to finish before its own change is calculated and applied. That mechanism is locking.

## Concept

### Lock Granularity: Row-Level vs. Table-Level

A **lock** is a mechanism the database uses to control concurrent access to a piece of data by making one transaction wait until another releases its claim on that data. PostgreSQL supports locks at different **granularities** — how much of the data a single lock covers:

- **Row-level locks** apply to a single row. PostgreSQL uses these for ordinary data-modifying statements (`UPDATE`, `DELETE`, `SELECT ... FOR UPDATE`) so that two transactions modifying two *different* rows of the same table never block each other at all — only transactions touching the *same* row contend.
- **Table-level locks** apply to an entire table. PostgreSQL takes these automatically for structural changes (`ALTER TABLE`, `DROP TABLE`, `TRUNCATE` — Module 6) because those operations affect every row and the table's definition itself, not any one row in particular.

```sql
-- Row-level lock only on the row where owner = 'Asha'
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';

-- Table-level lock on the entire accounts table
ALTER TABLE accounts ADD COLUMN opened_on DATE;
```

This is a deliberate design choice: ordinary data changes should be as concurrent as possible (row-level, so unrelated rows never contend), while structural changes need the strongest guarantee (table-level, since every row and every query against the table is affected by a structural change).

### Lock Modes: Shared vs. Exclusive

Within a given granularity, a lock is also acquired in a specific **mode** that determines what it does and doesn't allow other transactions to do at the same time:

- A **shared lock** ("I'm reading this, but I'm not changing it") can be held by multiple transactions on the same row simultaneously — many readers holding a shared lock never block each other.
- An **exclusive lock** ("I'm about to change this") can only be held by one transaction at a time on a given row — while one transaction holds an exclusive lock on a row, no other transaction can acquire *any* kind of lock (shared or exclusive) on that same row until the first one finishes.

| | Another transaction wants a shared lock | Another transaction wants an exclusive lock |
|---|---|---|
| **I hold a shared lock** | Allowed — both can hold it at once | Blocked — must wait |
| **I hold an exclusive lock** | Blocked — must wait | Blocked — must wait |

### UPDATE and DELETE Implicitly Acquire Row Locks

You never have to explicitly ask for a lock to run an ordinary `UPDATE` or `DELETE` — PostgreSQL acquires an exclusive row-level lock automatically on exactly the rows the statement touches, and holds it until the transaction ends (`COMMIT` or `ROLLBACK`):

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';
-- An exclusive lock is now held on Asha's row, automatically,
-- for the rest of this transaction.
```

Crucially, this lock only covers the row(s) matched by the `WHERE` clause — Ben's row is completely untouched and unlocked, so a concurrent transaction updating Ben's balance proceeds without any contention at all:

```
Time   Transaction A                                Transaction B
----   -------------------------------------------  -------------------------------------------
T1     BEGIN;
T2     UPDATE accounts SET balance = balance - 100
       WHERE owner = 'Asha';
       -- exclusive lock on Asha's row
T3                                                   BEGIN;
T4                                                   UPDATE accounts SET balance = balance + 50
                                                      WHERE owner = 'Ben';
                                                      -- exclusive lock on Ben's row -- DIFFERENT
                                                      -- row, no contention, proceeds immediately
T5                                                   COMMIT;
T6     COMMIT;
```

An important detail specific to PostgreSQL's MVCC design: a plain `SELECT` **does not** acquire any row lock at all, and is never blocked by an `UPDATE`'s exclusive lock, and never blocks an `UPDATE` in return — a reader simply sees a consistent snapshot (Topic 3) of the data as of its own transaction's visibility rules, regardless of any exclusive locks other transactions are holding for writes. This is exactly what "readers don't block writers, writers don't block readers" means in PostgreSQL.

### SELECT ... FOR UPDATE — Explicit Row Locking

Sometimes an application needs to lock a row *before* deciding how to change it — for example, reading a row to check a condition, and only then deciding what `UPDATE` to run, while making sure no other transaction can sneak in and change that row in between the read and the eventual write. `SELECT ... FOR UPDATE` explicitly acquires an exclusive row lock at read time, before any `UPDATE` is even issued:

```sql
BEGIN;

SELECT balance FROM accounts WHERE owner = 'Asha' FOR UPDATE;
-- Exclusive lock acquired on Asha's row right now, even though
-- this is only a SELECT — no other transaction can acquire any
-- lock on this row until this transaction ends.

-- ... application logic decides, based on the balance just read,
-- exactly how much to deduct ...

UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';

COMMIT;
```

Without `FOR UPDATE`, a second transaction could read the same balance in between the `SELECT` and the `UPDATE`, make its own decision based on now-stale data, and the two transactions' decisions could conflict once both eventually commit. `FOR UPDATE` closes that gap by locking the row the instant it's read, not just at the moment it's written.

A weaker variant, `SELECT ... FOR SHARE`, acquires a shared lock instead — useful when you need to prevent a row from being *changed* by anyone else while you're relying on its value, but you don't mind other transactions also reading it (with their own shared lock) at the same time.

### Blocking — What It Means for a Transaction to Wait

When a transaction requests a lock that conflicts with a lock another transaction already holds, it does not fail immediately — it **blocks**: it simply pauses and waits until the conflicting lock is released (i.e., until the other transaction commits or rolls back), then proceeds as if nothing unusual happened.

```
Time   Transaction A                                Transaction B
----   -------------------------------------------  -------------------------------------------
T1     BEGIN;
T2     UPDATE accounts SET balance = balance - 100
       WHERE owner = 'Asha';
       -- exclusive lock acquired on Asha's row
T3                                                   BEGIN;
T4                                                   UPDATE accounts SET balance = balance - 50
                                                      WHERE owner = 'Asha';
                                                      -- SAME row -- Transaction B now BLOCKS here,
                                                      -- waiting for A's exclusive lock to be released
T5     COMMIT;
       -- A's exclusive lock is released
T6                                                   -- Transaction B's UPDATE now proceeds, using
                                                      -- the balance AFTER A's committed change
T7                                                   COMMIT;
```

Transaction B's `UPDATE` statement does not return control back to the application until T5 — from the application's point of view, that statement simply takes however long Transaction A's transaction took to finish. This is normal, expected behavior, not an error — it is exactly the mechanism that prevents the "lost update" problem from the Problem Statement above. (Topic 6 covers the one case where blocking goes wrong: when two transactions end up waiting on each other in a cycle, with neither able to proceed — a deadlock.)

## Internal Working (Deep Dive)

```
                     Lock Manager
                          │
        ┌─────────────────┴─────────────────┐
        │                                     │
  Lock table: which transaction        Wait queue: which transactions
  holds which lock, on which           are blocked, waiting for a
  row/table, in which mode             conflicting lock to be released
        │                                     │
        └──────────────┬──────────────────────┘
                        │
             When a lock is released (COMMIT/ROLLBACK),
             the next waiting transaction in the queue
             is granted the lock and resumes execution
```

Internally, PostgreSQL maintains a lock table tracking exactly which transaction holds which lock, in which mode, on which specific row or table. When a transaction requests a lock that conflicts with an existing one, it's placed on a wait queue for that specific lock rather than failing outright. The moment the holding transaction ends (releasing all its locks at once, whether by `COMMIT` or `ROLLBACK`), the lock manager grants the lock to the next waiting transaction in line, which then resumes exactly where it left off. Row-level locks are lightweight and designed to be acquired and released constantly, in huge numbers, without meaningfully slowing down unrelated transactions touching different rows — this is precisely why PostgreSQL prefers row-level granularity for ordinary data changes.

## Real-World Analogy

Think of a shared office with individual desks (rows) rather than one single shared room (the whole table):

- A **shared lock** is like several people being allowed to read a memo pinned to a desk at the same time — nobody's reading blocks anyone else's reading.
- An **exclusive lock** is like someone physically removing the memo to rewrite it — while they're doing that, nobody else can even read the old memo *or* start their own rewrite until it's put back.
- A **row-level lock** only reserves one specific desk — someone working at a different desk across the room is completely unaffected.
- A **table-level lock** is like closing the entire office floor for renovation — everyone, at every desk, has to wait, because the whole floor's layout is changing.
- **Blocking** is simply queuing at that one desk, waiting your turn, rather than being turned away — you get to proceed the instant the person ahead of you steps away.

## Why Locking Was Designed This Way

Locking exists because MVCC's snapshot mechanism (which solves concurrent *reading* beautifully — Topic 3) does not, by itself, solve concurrent *writing*: two transactions computing a new value from the same starting row and both trying to save their result need an actual mutual-exclusion mechanism, not just a consistent view of history. PostgreSQL chooses row-level granularity as the default for data changes specifically because the whole point of a relational database serving many concurrent users is that unrelated rows should never contend with each other — locking the entire table for every `UPDATE` would defeat the purpose of storing many independent records in one table to begin with. Table-level locks are reserved for genuinely table-wide operations (structural changes) where there is no meaningful way to "partially" apply the change to just some rows. This mirrors the same reasoning as isolation levels (Topic 3): match the strength (and cost) of the mechanism to the actual scope of what's changing.

## Advantages

- **Row-level granularity maximizes concurrency** — transactions touching different rows of the same table never contend with each other at all, which is essential for a busy, many-user table like `accounts`.
- **Readers never block on writer locks in PostgreSQL** — because reads use MVCC snapshots rather than shared locks for ordinary `SELECT`s, a long-running `UPDATE` never stalls unrelated `SELECT` queries.
- **Blocking (rather than failing immediately) is usually the right default** — a transaction that would conflict simply waits its turn and then proceeds correctly, without the application needing to catch an error and retry for the ordinary case.
- **`SELECT ... FOR UPDATE` closes the read-then-write gap precisely** — it lets you safely read a value, reason about it, and write based on that reasoning, without another transaction changing the row out from under you in between.

## Disadvantages / Limitations

- **A transaction holding a lock for a long time blocks every other transaction that needs that same row** — a slow or idle transaction (Topic 2's warning about leaving transactions open) can stall other work indefinitely, even though no error ever occurs.
- **Row locks are held until the whole transaction ends, not until the individual statement finishes** — a transaction that acquires a lock early and then does unrelated, slow work afterward keeps that lock the entire time, which is easy to overlook.
- **Overuse of `SELECT ... FOR UPDATE`** — locking rows "just in case" when no actual write-after-read decision is being made needlessly increases contention and the risk of blocking chains.
- **Circular waiting between transactions is possible** — when two transactions each hold a lock the other is waiting for, blocking alone cannot resolve it; this specific failure mode, a deadlock, is the entire subject of Topic 6.

## Best Practices

- Keep the time between acquiring a lock and releasing it (i.e., between the locking statement and the transaction's `COMMIT`/`ROLLBACK`) as short as possible — don't do slow, unrelated work (network calls, waiting on user input) while holding locks.
- Use `SELECT ... FOR UPDATE` specifically when you read a row's value in order to decide how to change it, and need to guarantee no other transaction changes that value in between your read and your write.
- Don't reach for table-level locking manually (`LOCK TABLE`) unless you have a specific, well-understood structural reason — let PostgreSQL's automatic row-level locking handle ordinary data changes.
- When diagnosing an application that "hangs" on a specific statement, check first whether it's simply blocked waiting on a row lock held by another, possibly idle, transaction — this is a very common real-world cause of a query that "never returns."

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a plain `SELECT` blocks or is blocked by a concurrent `UPDATE` | In PostgreSQL, ordinary `SELECT` statements don't acquire row locks at all and rely on MVCC snapshots instead — they never block writers and are never blocked by writers' exclusive locks. |
| Believing `UPDATE`/`DELETE` lock the whole table | They acquire row-level locks only on the specific rows the statement actually matches (via its `WHERE` clause) — unrelated rows in the same table are completely unaffected. |
| Reading a value with a plain `SELECT`, then issuing an `UPDATE` based on it, assuming nothing else could have changed it in between | Without `SELECT ... FOR UPDATE`, another transaction can commit a change to that exact row in the gap between your read and your write, since a plain `SELECT` acquires no lock to prevent that. |
| Treating "my query is taking a long time" as a performance problem to optimize with indexes | It may simply be blocked, waiting on a lock held by another transaction — check for blocking before assuming the query itself is inefficient. |

## Interview Questions

1. **Q: What is the difference between a row-level lock and a table-level lock, and when does PostgreSQL use each?**
   A: A row-level lock covers a single row, allowing unrelated rows in the same table to be modified concurrently without contention; PostgreSQL uses these automatically for ordinary `UPDATE`/`DELETE` statements. A table-level lock covers an entire table and is used automatically for structural changes (like `ALTER TABLE`) that affect the whole table's definition, not any single row.

2. **Q: Does a plain `SELECT` statement in PostgreSQL ever get blocked by a concurrent `UPDATE`'s lock?**
   A: No. PostgreSQL's MVCC design means ordinary reads use a consistent snapshot rather than acquiring a row lock, so readers never block on writers' exclusive locks, and writers never wait on readers.

3. **Q: Why would you use `SELECT ... FOR UPDATE` instead of a plain `SELECT` followed by an `UPDATE`?**
   A: `SELECT ... FOR UPDATE` acquires an exclusive lock on the selected rows at the moment of the read, guaranteeing no other transaction can change those rows before your later `UPDATE` runs. A plain `SELECT` acquires no lock, leaving a gap in which another transaction could commit a conflicting change between your read and your write.

4. **Q: What does it mean for a transaction to "block," and is that itself an error condition?**
   A: Blocking means a transaction requested a lock that conflicts with a lock another transaction currently holds, so it pauses and waits in a queue until that lock is released. It is not an error — it's expected, normal behavior that resolves automatically once the holding transaction commits or rolls back, unless the waiting forms a cycle between transactions, which is a deadlock (Topic 6).

## Summary

- Locks come in two **granularities** — row-level (used for ordinary `UPDATE`/`DELETE`, allowing unrelated rows to proceed concurrently) and table-level (used for structural changes affecting the whole table).
- Locks come in two relevant **modes** — shared (many readers can hold at once) and exclusive (only one transaction, blocking every other lock request on that row).
- `UPDATE` and `DELETE` implicitly acquire exclusive row-level locks on exactly the rows they match, held until the transaction ends.
- Plain `SELECT` statements acquire no row lock at all in PostgreSQL — readers never block writers, and writers never block readers, thanks to MVCC.
- `SELECT ... FOR UPDATE` explicitly locks rows at read time, closing the gap between reading a value and later writing based on it.
- A transaction that requests a conflicting lock **blocks** — it waits in a queue and resumes automatically once the lock is released; this is normal, except when it forms a cycle between two transactions, which is a deadlock, covered next in Topic 6.
