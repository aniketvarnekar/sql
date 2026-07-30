# Deadlocks

## Learning Objectives

By the end of this section you should be able to:
- Define a deadlock precisely, in terms of a circular wait between transactions
- Construct a concrete two-transaction example that produces a real deadlock
- Explain how PostgreSQL automatically detects a deadlock and how it resolves one
- Read PostgreSQL's actual deadlock error message and explain what each part means
- Apply concrete practices (consistent lock ordering, shorter transactions) that prevent deadlocks from occurring in the first place

## Prerequisites

- [Locking Fundamentals](05-locking-fundamentals.md) — a deadlock is a specific failure mode of the blocking behavior explained there; you need to understand ordinary blocking before understanding how blocking can go wrong.

## Motivation

Topic 5 ended by noting that a transaction that blocks on a lock will resume automatically once the lock is released — that's the ordinary, self-resolving case. But there's exactly one scenario where blocking cannot resolve itself: two transactions each waiting for a lock the *other* one holds. Neither will ever release its own lock, because each is waiting for the other to finish first. Left alone, both would wait forever. Every production database that supports row-level locking has to solve this problem, and PostgreSQL solves it automatically — but only if you understand it enough to write code that survives it happening.

## Problem Statement

Recall the transfer example: money moving between Asha's and Ben's accounts. Now imagine two transfers running concurrently, in **opposite directions**, each updating both rows but in a different order:

```sql
-- Transaction A: transfer FROM Asha TO Ben
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';  -- locks Asha's row first
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';   -- then wants Ben's row

-- Transaction B: transfer FROM Ben TO Asha, running at the same time
UPDATE accounts SET balance = balance - 50 WHERE owner = 'Ben';    -- locks Ben's row first
UPDATE accounts SET balance = balance + 50 WHERE owner = 'Asha';   -- then wants Asha's row
```

If both transactions manage to acquire their *first* lock before either attempts its *second*, each is now waiting for a row the other transaction already holds exclusively — and neither will ever release what it's holding, because each is stuck waiting to acquire its second lock before it can reach `COMMIT`.

## Concept

### What a Deadlock Is

> A **deadlock** is a situation where two (or more) transactions are each waiting for a lock held by the other, forming a cycle of waiting with no possible way for any of them to proceed.

It is fundamentally different from ordinary blocking (Topic 5): ordinary blocking always resolves on its own, because the transaction holding the lock is not waiting on anything from the blocked transaction — it will eventually commit or roll back and release the lock. A deadlock is blocking that **cannot** resolve on its own, because the transaction holding the lock the first transaction wants is itself blocked, waiting on a lock the first transaction holds.

### A Concrete Two-Transaction Deadlock

```
Time   Transaction A                                   Transaction B
----   ---------------------------------------------   ---------------------------------------------
T1     BEGIN;
T2     UPDATE accounts SET balance = balance - 100
       WHERE owner = 'Asha';
       -- A now holds an exclusive lock on Asha's row
T3                                                      BEGIN;
T4                                                      UPDATE accounts SET balance = balance - 50
                                                         WHERE owner = 'Ben';
                                                         -- B now holds an exclusive lock on Ben's row
T5     UPDATE accounts SET balance = balance + 100
       WHERE owner = 'Ben';
       -- A wants Ben's row, but B holds it -- A BLOCKS,
       -- waiting for B to release Ben's lock
T6                                                      UPDATE accounts SET balance = balance + 50
                                                         WHERE owner = 'Asha';
                                                         -- B wants Asha's row, but A holds it -- B BLOCKS,
                                                         -- waiting for A to release Asha's lock
       ---------------------------------------------------------------------------------------------
       DEADLOCK: A is waiting on B, and B is waiting on A. Neither can proceed to COMMIT, because
       neither can finish its second UPDATE, and neither will release its first lock until it does.
       ---------------------------------------------------------------------------------------------
```

At T6, the cycle is complete: A holds Asha's lock and wants Ben's; B holds Ben's lock and wants Asha's. Without intervention, both transactions would wait for each other forever.

### How PostgreSQL Detects and Resolves Deadlocks

PostgreSQL does not let this situation hang indefinitely. It runs a background **deadlock detector** that periodically checks whether the current pattern of blocked transactions forms a cycle (a "wait-for graph," where an edge means "this transaction is waiting for a lock held by that one"). The check runs whenever a transaction has been blocked for longer than the `deadlock_timeout` setting (1 second by default) — not constantly, since building and checking this graph has a real cost, and the overwhelming majority of blocked transactions resolve normally well within a second.

The instant PostgreSQL's detector finds a genuine cycle, it breaks it by choosing one of the transactions involved (typically the one whose lock request would complete the cycle) and forcibly aborting it, immediately returning an error to that transaction's connection:

```
ERROR:  deadlock detected
DETAIL:  Process 18211 waits for ShareLock on transaction 892; blocked by process 18234.
         Process 18234 waits for ShareLock on transaction 891; blocked by process 18211.
HINT:  See server log for query details.
```

Reading this error: PostgreSQL identifies the exact two backend processes involved, and states plainly that each is waiting on a lock held by the other — the cycle, made explicit. The transaction that receives this error is automatically rolled back by PostgreSQL — every one of its changes so far is undone, exactly as if it had run `ROLLBACK` itself. The **other** transaction involved in the cycle is immediately unblocked once the aborted transaction's locks are released, and proceeds normally to completion, as if the deadlock had never involved it at all:

```
Time   Transaction A                                   Transaction B
----   ---------------------------------------------   ---------------------------------------------
T7     -- deadlock detected; PostgreSQL chooses to
       -- abort Transaction A
       ERROR: deadlock detected
       -- A's changes are automatically rolled back;
       -- Asha's balance lock is released
T8                                                      -- B's blocked UPDATE now proceeds immediately
                                                         -- (Asha's row is free); B continues normally
T9                                                      COMMIT;
                                                         -- B's transfer completes successfully
```

The application connected to Transaction A receives the `deadlock detected` error and is expected to handle it — typically by retrying the entire transaction from the beginning. This is the critical, easy-to-miss detail: a deadlock is not a bug the database silently "fixes" for you; it's an error condition your application must be prepared to catch and react to, usually by simply trying the whole transaction again.

## Internal Working (Deep Dive)

```
                    Wait-for graph, built by the deadlock detector

        Transaction A ────waits for────▶ Transaction B
              ▲                                │
              │                                │
              └────────waits for───────────────┘

        A cycle exists ⇒ deadlock. The detector picks one edge to break
        by aborting the transaction at the tail of that edge.
```

- Every time a transaction blocks waiting for a lock, PostgreSQL records which transaction it's waiting on.
- The deadlock detector, triggered after a transaction has been blocked longer than `deadlock_timeout` (default 1 second), builds this wait-for graph among currently blocked transactions and searches it for a cycle.
- If no cycle is found, nothing happens — the transaction simply continues waiting normally, exactly as ordinary blocking (Topic 5) describes.
- If a cycle is found, PostgreSQL selects one of the transactions in the cycle to abort (generally the one that most directly completes the cycle) and immediately raises the `deadlock detected` error on that connection, releasing its locks and allowing the rest of the cycle to proceed.
- This check has a real, if small, performance cost, which is exactly why it only runs after the `deadlock_timeout` threshold rather than continuously — ordinary blocking resolves in well under a second the vast majority of the time, so most blocked transactions never trigger this check at all.

## Real-World Analogy

Picture two cars meeting head-on on a narrow single-lane bridge, each having already driven onto the bridge from opposite ends. Neither can go forward (the other car is in the way) and neither wants to back up first (each is waiting for the other to concede). Left alone, both cars simply sit there indefinitely — this is the deadlock. A traffic officer (PostgreSQL's deadlock detector) who notices the standoff after a short while and orders one specific car to reverse off the bridge (aborting one transaction) immediately clears the way for the other car to proceed normally. The car that was ordered to reverse has to go back to where it started and, if it still needs to cross, try again from the beginning (the application retrying the aborted transaction).

## Why Deadlock Handling Was Designed This Way

A database that allowed row-level locking (essential for concurrency, Topic 5) but had no deadlock detection would be forced to choose between two bad options: let genuinely deadlocked transactions hang forever (unacceptable for any real system), or refuse to allow the kind of fine-grained, flexible locking that makes high concurrency possible in the first place (by, say, only ever letting one transaction hold any lock at a time, which defeats the point of row-level locking). PostgreSQL's approach — allow the flexible locking, but actively watch for the one specific failure mode (a cycle) it can produce, and resolve it automatically by sacrificing one transaction — gets the concurrency benefits of row-level locking without the risk of an unresolvable hang. Requiring the application to retry the aborted transaction, rather than PostgreSQL silently re-running it, is consistent with the rest of transaction handling in this module: the database guarantees correctness and makes the failure explicit and detectable (via a specific, distinctly named error), but it is the application's responsibility to decide what "try again" means for its own business logic.

## Advantages

- **No indefinite hangs** — a genuine deadlock is always detected and broken automatically, typically within about a second (`deadlock_timeout`), rather than leaving two transactions frozen forever.
- **The guarantee is automatic and requires no manual intervention** — you don't need to build your own deadlock-watching logic; PostgreSQL's lock manager does this for every transaction, all the time.
- **The error is specific and actionable** — `deadlock detected` is a distinct, identifiable error (SQLSTATE `40P01`) that application code can specifically catch and respond to with a retry, rather than a generic failure.

## Disadvantages / Limitations

- **One transaction's committed-so-far work is always discarded** — deadlock resolution isn't a graceful negotiation; the chosen transaction is fully rolled back and must restart entirely, discarding whatever progress it had made.
- **Detection isn't instantaneous** — because the check only runs after `deadlock_timeout` (1 second by default) of blocking, a deadlocked transaction pair sits blocked for up to that long before either connection sees the error, which briefly looks identical to ordinary, resolvable blocking (Topic 5) from the outside.
- **Requires the application to implement retry logic** — a deadlock is not silently fixed on your behalf; an application that doesn't catch and retry on this specific error will simply surface a failed operation to whatever called it.

## Best Practices

- **Establish a consistent lock acquisition order across your whole application**, and follow it everywhere — for example, always updating accounts in ascending `id` order regardless of which direction money is conceptually moving. If both transactions in the earlier example had updated the lower `id` first, one would simply block waiting for the other (ordinary blocking, Topic 5), and no cycle — no deadlock — could ever form.
- **Keep transactions as short as possible**, for the same reason Topic 2 and Topic 5 recommend it — the less time a transaction spends holding a lock before releasing it, the smaller the window in which another transaction's conflicting lock request could create a cycle.
- **Acquire all the locks a transaction will need as early as possible, in a single, predictable pattern**, rather than acquiring them piecemeal interleaved with other logic — predictable locking patterns are far less likely to interleave into a cycle with another transaction.
- **Always write retry logic around code paths that run inside a transaction and might deadlock**, catching the specific `deadlock detected` condition (SQLSTATE `40P01`) and re-attempting the whole transaction from its `BEGIN`.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming PostgreSQL "fixes" a deadlock and both transactions eventually succeed | PostgreSQL only breaks the cycle by aborting one transaction entirely; that transaction's work is discarded and must be explicitly retried by the application — the other transaction proceeds, but the aborted one does not silently recover. |
| Updating related rows in an inconsistent order across different parts of an application (one code path updates Asha then Ben, another updates Ben then Asha) | This inconsistency is exactly what creates the circular wait that causes a deadlock; a single, consistent ordering everywhere eliminates the cycle entirely, turning what would be a deadlock into ordinary, resolvable blocking. |
| Treating a `deadlock detected` error as an application bug to eliminate entirely | Deadlocks can still occur even with reasonable code, simply due to timing, especially as concurrency increases — the correct response is not "make sure this can never happen" (often impossible to guarantee completely) but "always catch this specific error and retry the transaction." |
| Confusing a deadlock with ordinary lock waiting | Ordinary blocking (Topic 5) always resolves on its own once the holding transaction ends; a deadlock is specifically a *cycle* of waiting that cannot resolve without one of the transactions being forcibly aborted. |

## Interview Questions

1. **Q: What is a deadlock, precisely?**
   A: A situation where two or more transactions are each waiting for a lock held by another transaction in the same group, forming a cycle of waiting — meaning none of them can ever proceed unless the cycle is broken from outside, since each is waiting on a lock the other refuses to release until it finishes.

2. **Q: How does PostgreSQL detect and resolve a deadlock?**
   A: A background deadlock detector runs after a transaction has been blocked longer than `deadlock_timeout` (1 second by default), builds a wait-for graph of which blocked transactions are waiting on which others, and checks for a cycle. If one is found, PostgreSQL aborts one of the transactions in the cycle, returning a `deadlock detected` error and rolling back that transaction's changes, which releases its locks and allows the remaining transaction(s) to proceed.

3. **Q: Give a concrete way to prevent the classic "two transfers in opposite directions" deadlock.**
   A: Have every transaction that updates multiple rows in the same table acquire its locks in a single, consistent order regardless of the business direction of the operation — for example, always locking the row with the lower primary key value first. If both transactions follow the same order, one will simply block waiting for the other (ordinary, resolvable blocking) instead of each waiting on the other in a cycle.

4. **Q: Is a `deadlock detected` error something an application should treat as a fatal bug?**
   A: No — it's an expected, occasionally-occurring condition even in well-written concurrent code, identified by a specific SQLSTATE (`40P01`). The correct handling is for the application to catch this specific error and retry the entire transaction from the beginning, not to treat its occurrence as proof of a logic error.

## Summary

- A **deadlock** is a cycle of transactions each waiting for a lock held by another, which cannot resolve on its own — unlike ordinary blocking (Topic 5), which always resolves once the holding transaction ends.
- The classic cause is two transactions updating the same set of rows in **opposite orders**, each acquiring one lock and then blocking on the other's.
- PostgreSQL's background deadlock detector checks for cycles after a transaction has been blocked past `deadlock_timeout` (1 second by default), and automatically aborts one transaction in the cycle with a `deadlock detected` error, releasing its locks so the rest of the cycle can proceed.
- The aborted transaction's work is fully rolled back; the application is responsible for catching this specific error and retrying the whole transaction.
- The most effective prevention is **consistent lock acquisition ordering** across the entire application, combined with **keeping transactions short** — both directly shrink the window in which a cycle can form.
- This closes out the module's mechanics; Topic 7 recaps everything from ACID through deadlocks as one connected picture.
