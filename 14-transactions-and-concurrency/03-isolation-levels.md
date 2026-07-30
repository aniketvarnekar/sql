# Isolation Levels

## Learning Objectives

By the end of this section you should be able to:
- Name the SQL standard's four isolation levels in order of increasing strictness
- Explain which of those four levels PostgreSQL actually implements as distinct behaviors, and which one it treats as an alias for another
- State PostgreSQL's default isolation level
- Set the isolation level for a specific transaction using `SET TRANSACTION ISOLATION LEVEL`
- Explain, at a conceptual level, why stronger isolation costs more in concurrency and performance

## Prerequisites

- [BEGIN, COMMIT, and ROLLBACK](02-begin-commit-rollback.md) — isolation level is set per transaction, so you need to already be comfortable opening and ending an explicit transaction block.
- [ACID Properties](01-acid-properties.md), specifically the Isolation section — this topic is the detailed follow-up to the claim made there that "how isolated" a transaction is happens to be configurable.

## Motivation

Topic 1 stated that Isolation guarantees concurrent transactions can't see each other's uncommitted work — but it left one important detail open: isolation is not a single fixed guarantee, it's a *dial* with several settings, each trading off correctness guarantees against how much concurrent work the database can do at once. Understanding the isolation levels lets you deliberately choose the right amount of protection for a given piece of work, instead of blindly trusting whatever the default happens to be.

## Problem Statement

Full, perfect isolation — where every transaction behaves as if it ran completely alone, with no other transaction's activity visible or interfering at all — is easy to guarantee in one blunt way: let only one transaction run at a time, and make every other transaction wait its turn. That would be correct, but it would also throw away nearly all of the concurrency a modern multi-user database needs to be useful. On the other end, giving every transaction a completely unrestricted view of every other transaction's in-progress changes maximizes concurrency but reintroduces the dirty-read problem from Topic 1. Real databases need a middle ground — several selectable middle grounds, in fact — and the SQL standard formalizes exactly four of them.

## Concept

### The SQL Standard's Four Isolation Levels

The SQL standard defines four isolation levels, from loosest to strictest:

| Level | What the standard permits |
|---|---|
| **Read Uncommitted** | The loosest level — permits reading another transaction's uncommitted changes (dirty reads), as well as non-repeatable reads and phantom reads. |
| **Read Committed** | Never permits dirty reads, but permits non-repeatable reads and phantom reads. |
| **Repeatable Read** | Never permits dirty reads or non-repeatable reads, but the standard still permits phantom reads at this level. |
| **Serializable** | The strictest level — behaves as if every transaction ran one at a time, in some serial order; permits none of the three classic anomalies. |

(The precise definitions of dirty reads, non-repeatable reads, and phantom reads — with concrete two-transaction timelines for each — are the entire subject of Topic 4. This topic focuses on the levels themselves; read Topic 4 next, then return here once the anomalies have concrete shape.)

The pattern is: each level going down the table prevents strictly more anomalies than the one above it, at the cost of more restrictive (and more expensive) concurrency control underneath.

### What PostgreSQL Actually Implements

PostgreSQL accepts all four standard level names, but internally only implements **three distinct behaviors**:

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;   -- accepted, but behaves exactly like READ COMMITTED
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;     -- the default
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

- **Read Uncommitted** is accepted as valid syntax but PostgreSQL silently treats it identically to Read Committed. This is because PostgreSQL's storage engine (MVCC, previewed in Topic 1) is architecturally incapable of showing a transaction another transaction's uncommitted row versions — dirty reads are simply not possible in PostgreSQL at *any* isolation level, so there is nothing weaker than Read Committed for it to implement.
- **Read Committed** is PostgreSQL's default for every new transaction unless told otherwise. Each individual statement inside the transaction sees a fresh snapshot of the database taken at the moment that statement begins — meaning two `SELECT`s in the same transaction, run a few seconds apart, can see different data if another transaction committed something in between.
- **Repeatable Read** takes a single snapshot at the start of the transaction (specifically, at the first query or data-modifying statement) and uses that same snapshot for every statement in the transaction, no matter how much other transactions commit in the meantime.
- **Serializable** behaves like Repeatable Read's snapshot behavior, plus additional runtime tracking (Serializable Snapshot Isolation) that detects when the pattern of reads and writes across concurrent transactions could not have occurred in *any* possible one-at-a-time ordering — and aborts one of the conflicting transactions with an error rather than allowing a subtly incorrect outcome.

You can check which isolation level a transaction is currently using:

```sql
BEGIN;
SHOW transaction_isolation;
```

```
 transaction_isolation
------------------------
 read committed
(1 row)
```

### Setting the Isolation Level

The isolation level is set per transaction, immediately after `BEGIN`, and applies only to that one transaction:

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT balance FROM accounts WHERE owner = 'Asha';
-- ... other work, possibly minutes of it in a long-running report ...
SELECT balance FROM accounts WHERE owner = 'Asha';
-- Guaranteed to return the exact same value as the first SELECT above,
-- even if another transaction committed a change to Asha's balance in between.

COMMIT;
```

PostgreSQL also accepts setting the level directly in the `BEGIN` statement itself:

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

If you want every transaction on a connection (or server-wide) to default to a stricter level rather than setting it every time, you can change the default:

```sql
SET default_transaction_isolation = 'repeatable read';
```

### An Important PostgreSQL-Specific Nuance

The SQL standard's Repeatable Read level still technically permits phantom reads (a new row matching your `WHERE` condition appearing on a re-run query). PostgreSQL's actual implementation of Repeatable Read is **stricter than the standard requires** — because it works by holding a single consistent snapshot of the *entire* database for the whole transaction, it also prevents phantom reads, which the standard doesn't require it to. This is a case where the underlying MVCC mechanism happens to deliver a stronger guarantee than the standard's minimum bar for that level's name. Topic 4's anomaly-prevention table reflects this PostgreSQL-specific behavior directly, since this course teaches against real PostgreSQL behavior rather than only the abstract standard.

What PostgreSQL's Repeatable Read does *not* fully prevent, that only Serializable does, is a subtler class of problem called a **serialization anomaly** (sometimes called *write skew*) — two transactions that each individually read some data, then write based on what they read, in a way that would be impossible if they'd truly run one after another, even though neither one individually saw a dirty, non-repeatable, or phantom read. This is beyond the three classic anomalies this module focuses on, but it's worth knowing the name exists, and that Serializable is the only level PostgreSQL offers that closes it completely.

## Internal Working (Deep Dive)

```
 Read Committed:     [stmt 1: new snapshot] [stmt 2: new snapshot] [stmt 3: new snapshot]
                            │                       │                       │
                      sees latest committed   sees latest committed   sees latest committed
                      data as of THIS instant  data as of THIS instant data as of THIS instant


 Repeatable Read:    [stmt 1: snapshot taken] ────────────────────────────────▶ same snapshot
                            │                       │                       │
                      one snapshot for the entire transaction — every statement sees
                      the database exactly as it looked at the start


 Serializable:       same snapshot behavior as Repeatable Read, PLUS:
                      a runtime dependency-tracking layer (SSI) watches for read/write
                      patterns across concurrent transactions that couldn't have happened
                      in any serial (one-at-a-time) order, and aborts one transaction
                      with a serialization failure error if it detects one
```

Each level's stronger guarantee is bought by doing more work: Read Committed re-derives a snapshot on every single statement (cheap per-statement, but inconsistent across the transaction); Repeatable Read holds one snapshot for the whole transaction (slightly more bookkeeping, since PostgreSQL must keep old row versions around longer for any transaction that might still need to see them); Serializable adds active monitoring of read/write dependencies between concurrently running transactions, which is the most expensive of the three and can cause transactions to be aborted purely because of *how* they overlapped with other transactions, not because either one did anything individually wrong.

## Real-World Analogy

Imagine three people trying to read and possibly annotate a shared physical ledger book:

- **Read Committed** is like glancing at the ledger fresh every single time you look at it — each glance shows you whatever the most recently finalized entry is, even if that's different from what you saw a minute ago, because someone else might have added a finalized entry in between your glances.
- **Repeatable Read** is like photocopying the entire ledger the moment you start your work session, and working only from that photocopy for your whole session — no matter what anyone else finalizes in the real ledger while you're working, your photocopy (and everything you read from it) stays exactly as it was when you made it.
- **Serializable** is like that same photocopy approach, but with an auditor also watching everyone's session in real time, ready to flag "the way your work and someone else's work overlapped could never have produced this outcome if you'd each taken the ledger and finished one at a time" — and forcing one of you to redo your work if that happens.

## Why Isolation Levels Were Designed This Way

The SQL standard deliberately leaves isolation level as a choice rather than mandating one fixed behavior, because different work genuinely needs different trade-offs: a web request updating one row benefits from Read Committed's cheap, per-statement freshness and rarely needs anything stronger; a financial report that must present a single, internally consistent view of the whole database while it runs benefits from Repeatable Read's stable snapshot; and code that enforces an invariant across multiple rows that could be violated by a subtle concurrent interleaving (not just a single anomaly) needs Serializable's active guarantee. Rather than force every transaction to pay Serializable's cost, or force every transaction to accept Read Committed's looser guarantees, the standard — and PostgreSQL's implementation of it — puts the choice explicitly in the hands of whoever writes the transaction, matching SQL's general philosophy (Module 1) of letting you declare the level of guarantee you need rather than hard-coding one-size-fits-all behavior into the language.

## Advantages

- **Matches the guarantee to the workload** — a short, single-row update doesn't need to pay for Serializable's dependency tracking; a long financial report benefits enormously from a stable Repeatable Read snapshot.
- **Explicit and per-transaction** — the isolation level is a conscious choice made in the transaction itself, not a hidden global setting that silently changes behavior across your whole application.
- **PostgreSQL's Repeatable Read is unusually strong for its name** — you get phantom-read protection at that level for free, beyond what the standard technically requires.

## Disadvantages / Limitations

- **Stronger levels reduce concurrency** — Repeatable Read and especially Serializable can cause more transactions to wait on each other, or to be aborted outright with a serialization failure, compared to Read Committed.
- **Serializable requires retry logic** — an application using Serializable isolation must be prepared to catch a serialization failure error and simply retry the whole transaction; it is not a "set and forget" setting.
- **The standard's definitions and PostgreSQL's actual behavior diverge slightly** — as noted above, PostgreSQL's Repeatable Read is stricter than the standard's minimum requirement for that name, which is easy to get wrong if you learned the isolation levels from a database-agnostic textbook rather than PostgreSQL's own documentation.

## Best Practices

- Leave the default (Read Committed) in place for the overwhelming majority of ordinary application transactions — it is sufficient for most single-row or simple multi-row operations.
- Reach for Repeatable Read specifically when a transaction needs to read the same data multiple times and requires those reads to be mutually consistent, such as generating a report from several queries that must reflect one single moment in time.
- Reach for Serializable only for transactions enforcing an invariant across concurrent transactions that a weaker level cannot guarantee (a classic example: ensuring a resource is never double-booked by two transactions that each independently check availability and then reserve it) — and make sure the application layer retries on a serialization failure.
- Always test what your application does when a transaction using Repeatable Read or Serializable receives a serialization failure error — untested retry logic is a common source of silent data-loss bugs.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Believing `READ UNCOMMITTED` lets you see dirty (uncommitted) data in PostgreSQL | PostgreSQL accepts the syntax but always behaves like Read Committed — dirty reads are architecturally impossible in PostgreSQL at any isolation level, unlike in some other database products. |
| Assuming Repeatable Read in PostgreSQL still allows phantom reads, because that's what the SQL standard technically permits at that level | PostgreSQL's actual implementation of Repeatable Read is stricter than the standard requires and does prevent phantom reads, thanks to how its single-snapshot MVCC mechanism works — always verify behavior against PostgreSQL's documentation, not just the abstract standard. |
| Reaching for Serializable everywhere "to be safe" | Serializable has the highest overhead and requires the application to handle serialization failures with retry logic; using it by default for transactions that don't need it needlessly reduces concurrency and adds failure modes that must be handled. |
| Thinking isolation level is a global, permanent setting | It's set per transaction (right after `BEGIN`) unless you change `default_transaction_isolation`; different transactions on the same connection can use different levels. |

## Interview Questions

1. **Q: Name the SQL standard's four isolation levels in order from loosest to strictest.**
   A: Read Uncommitted, Read Committed, Repeatable Read, Serializable.

2. **Q: Which isolation level does PostgreSQL use by default, and what happens if you request Read Uncommitted?**
   A: PostgreSQL defaults to Read Committed. Requesting Read Uncommitted is accepted as valid syntax but behaves identically to Read Committed, since PostgreSQL's MVCC storage engine never exposes another transaction's uncommitted data at any isolation level.

3. **Q: Does PostgreSQL's Repeatable Read isolation level allow phantom reads?**
   A: No — even though the SQL standard technically permits phantom reads at the Repeatable Read level, PostgreSQL's implementation is stricter than the standard requires: because it takes one consistent snapshot of the whole database for the entire transaction, it also prevents phantom reads at that level.

4. **Q: Why would an application choose Serializable isolation over Repeatable Read, given that it's more expensive?**
   A: Repeatable Read still allows a subtler class of problem called a serialization anomaly (write skew), where two transactions each read data and write based on it in a way that couldn't happen if they'd truly run one after another, even though neither saw a dirty, non-repeatable, or phantom read individually. Serializable adds runtime dependency tracking that detects and prevents exactly this case, at the cost of needing the application to retry transactions that get aborted with a serialization failure.

## Summary

- The SQL standard defines four isolation levels — Read Uncommitted, Read Committed, Repeatable Read, Serializable — each preventing strictly more anomalies than the one before it.
- PostgreSQL implements only three distinct behaviors: Read Uncommitted is silently treated as Read Committed, since dirty reads are impossible in PostgreSQL at any level.
- **Read Committed** (the default) takes a fresh snapshot per statement; **Repeatable Read** takes one snapshot for the whole transaction; **Serializable** adds active dependency tracking on top of Repeatable Read's snapshot behavior.
- PostgreSQL's Repeatable Read is stricter than the SQL standard requires — it also prevents phantom reads, which the standard technically still allows at that level.
- Set the level with `SET TRANSACTION ISOLATION LEVEL ...` right after `BEGIN`, or change the connection-wide default with `default_transaction_isolation`.
- Higher isolation levels cost more in concurrency and, for Serializable, require the application to retry transactions that are aborted due to a detected serialization conflict.
