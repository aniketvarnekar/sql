# Dirty Reads, Non-Repeatable Reads, and Phantom Reads

## Learning Objectives

By the end of this section you should be able to:
- Give a precise definition of a dirty read, a non-repeatable read, and a phantom read
- Distinguish all three from each other using their exact timing and what specifically changes between the two observations
- Walk through a concrete two-transaction timeline for each anomaly
- Read a table mapping which PostgreSQL isolation level prevents which anomaly, and explain why

## Prerequisites

- [Isolation Levels](03-isolation-levels.md) — this topic gives the anomalies these levels were designed to prevent or permit their exact, concrete definitions; it's worth re-reading Topic 3's level descriptions again once you've finished this one.

## Motivation

Topic 3 named the anomalies isolation levels are designed around, but only in passing. Before the isolation level table means anything more than a list of names to memorize, you need to see, statement by statement, exactly what each anomaly looks like when two transactions run at almost the same time — and exactly which statement in the timeline is the one that "shouldn't" have been visible, or shouldn't have changed.

## Problem Statement

All three anomalies share a common shape: a transaction reads data more than once (or reads it once while another transaction is doing something), and something about what it sees is inconsistent with the idea that transactions should behave as if they ran one at a time. But the *specific* thing that goes wrong is different in each case — one is about seeing data that was never truly committed, one is about the same query returning different values for a row that already existed, and one is about the same query returning a different *set of rows* entirely. Treating them as one vague "concurrency bug" category makes it impossible to reason about which isolation level actually protects you from which specific failure.

For all three examples below, assume this starting table:

```sql
CREATE TABLE accounts (
    id      SERIAL PRIMARY KEY,
    owner   TEXT NOT NULL,
    balance NUMERIC NOT NULL CHECK (balance >= 0)
);

INSERT INTO accounts (owner, balance) VALUES ('Asha', 1000), ('Ben', 500);
```

## Concept

### Dirty Read

> A **dirty read** happens when a transaction reads data written by another transaction that has **not yet committed** — data that might still be rolled back and, if it is, was never a real, permanent fact at all.

```
Time   Transaction A                                   Transaction B
----   ---------------------------------------------   ---------------------------------------------
T1     BEGIN;
T2     UPDATE accounts SET balance = balance - 100
       WHERE owner = 'Asha';
       -- Asha's balance is now 900, but UNCOMMITTED
T3                                                      BEGIN;
T4                                                      SELECT balance FROM accounts
                                                         WHERE owner = 'Asha';
                                                         -- if this could see 900, that would be a
                                                         -- dirty read of A's uncommitted change
T5     ROLLBACK;
       -- Asha's balance reverts to 1000; it never
       -- really existed as a committed value
T6                                                      COMMIT;
       -- Transaction B has already acted on a number
       -- (900) that turned out to have never been real
```

**Why this is dangerous:** if Transaction B made a decision based on 900 — say, it approved a second withdrawal believing Asha had less money reserved than she actually did — that decision is now based on a value that was, retroactively, never true.

**Important PostgreSQL-specific fact:** this exact scenario is **not actually possible in PostgreSQL, at any isolation level**, including Read Uncommitted. PostgreSQL's MVCC engine (Topic 1, Topic 3) never exposes another transaction's uncommitted row versions to any other transaction — a `SELECT` always sees only committed data as of its snapshot. The timeline above is the textbook definition of the anomaly (and is genuinely possible in some other database products at their weakest isolation level), but PostgreSQL is architecturally immune to it unconditionally.

### Non-Repeatable Read

> A **non-repeatable read** happens when a transaction reads the same row twice, and gets two different values, because another transaction **committed** a change to that row in between the two reads.

```
Time   Transaction A (Read Committed)                  Transaction B
----   ---------------------------------------------   ---------------------------------------------
T1     BEGIN;
T2     SELECT balance FROM accounts
       WHERE owner = 'Asha';
       -- returns 1000
T3                                                      BEGIN;
T4                                                      UPDATE accounts SET balance = 700
                                                         WHERE owner = 'Asha';
T5                                                      COMMIT;
T6     SELECT balance FROM accounts
       WHERE owner = 'Asha';
       -- returns 700 -- a DIFFERENT value than T2's read,
       -- within the SAME transaction, for the SAME row
T7     COMMIT;
```

Unlike a dirty read, everything Transaction A saw was genuinely committed data at the moment it read it — the problem is that Transaction A's *own* two reads, of the exact same row, disagree with each other, because the underlying data legitimately changed and was committed by someone else in between.

**This happens under Read Committed** (PostgreSQL's default), because Read Committed takes a fresh snapshot for every individual statement (Topic 3) — so a `SELECT` at T6 correctly sees whatever was most recently committed, even if that's different from what an earlier `SELECT` in the same transaction saw at T2.

**This does not happen under Repeatable Read or Serializable** — both take a single snapshot for the entire transaction, so Transaction A's `SELECT` at T6 would still return 1000, exactly matching T2, regardless of what Transaction B committed in between.

### Phantom Read

> A **phantom read** happens when a transaction re-runs a query with a `WHERE` condition matching a *set* of rows, and gets a different set of rows the second time, because another transaction **inserted or deleted** a row matching that condition in between.

```
Time   Transaction A (Read Committed)                  Transaction B
----   ---------------------------------------------   ---------------------------------------------
T1     BEGIN;
T2     SELECT owner, balance FROM accounts
       WHERE balance > 400;
       -- returns Asha (1000) and Ben (500) -- 2 rows
T3                                                      BEGIN;
T4                                                      INSERT INTO accounts (owner, balance)
                                                         VALUES ('Chen', 900);
T5                                                      COMMIT;
T6     SELECT owner, balance FROM accounts
       WHERE balance > 400;
       -- returns Asha, Ben, AND Chen -- 3 rows
       -- Chen is a "phantom" row that appeared between
       -- two runs of the exact same query in one transaction
T7     COMMIT;
```

The difference from a non-repeatable read is precise: a non-repeatable read is about an *existing row's column value* changing between two reads of that same row; a phantom read is about the *set of rows matching a condition* changing between two reads of that condition, because rows were added or removed, not because an existing row's values changed.

**This happens under Read Committed**, for the same reason as the non-repeatable read example — each statement gets a fresh snapshot, so a newly committed row becomes visible to the next statement in the same transaction.

**In the SQL standard, Repeatable Read still technically permits this.** But as Topic 3 noted, **PostgreSQL's actual Repeatable Read implementation prevents this too** — because it takes one snapshot of the *entire database* at the start of the transaction, Chen's row (inserted and committed after that snapshot was taken) would not appear in Transaction A's second `SELECT` at T6 either. Rerunning the exact same scenario with `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` at the start of Transaction A yields 2 rows both times, not 3.

### Isolation Level vs. Anomaly — What PostgreSQL Actually Prevents

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted (behaves as Read Committed in PostgreSQL) | Prevented | Possible | Possible |
| Read Committed (PostgreSQL's default) | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Prevented (stricter than the SQL standard requires at this level) |
| Serializable | Prevented | Prevented | Prevented |

A useful way to remember the shape of this table: PostgreSQL never allows dirty reads, at any level — that row of the table is "Prevented" all the way across. Read Committed is the only level that allows the other two anomalies; both are eliminated the moment you move up to Repeatable Read, thanks specifically to PostgreSQL's single whole-transaction snapshot. Serializable prevents everything Repeatable Read does, plus the subtler serialization anomalies (write skew) mentioned in Topic 3, which are beyond the three anomalies in this table.

## Internal Working (Deep Dive)

```
 Read Committed:   stmt1 ──▶ [snapshot@T1] ... stmt2 ──▶ [snapshot@T2] ... stmt3 ──▶ [snapshot@T3]
                    each statement gets its own, independently-refreshed view of "what's committed
                    right now" — anything another transaction committed between statements becomes
                    visible to the NEXT statement in this transaction


 Repeatable Read:  stmt1 ──▶ [snapshot@T1] ─────────────────────────────────────▶ same snapshot for
                    /Serializable                                                  every later statement
                    one snapshot, taken once, used for the whole transaction — commits from other
                    transactions after that point are invisible to this transaction, for both
                    existing rows (no non-repeatable reads) and new rows (no phantom reads)
```

Both non-repeatable reads and phantom reads have the exact same root cause under the hood: whether a transaction's snapshot is refreshed per-statement (Read Committed) or fixed once for the whole transaction (Repeatable Read/Serializable). PostgreSQL doesn't need a separate mechanism for "protect against changed values in existing rows" versus "protect against new/removed rows matching a condition" — a single consistent snapshot mechanically prevents both at once, which is exactly why PostgreSQL's Repeatable Read ends up stricter than the SQL standard's minimum definition of that level.

## Real-World Analogy

Picture auditing a shared spreadsheet of account balances by glancing at it repeatedly during a work session:

- A **dirty read** would be seeing a number someone typed into a cell but hasn't saved yet — and that number gets deleted (undone) a moment later. You reacted to something that was never actually finalized. (PostgreSQL's equivalent of "glancing at unsaved edits" simply cannot happen — it only ever shows you saved, finalized values.)
- A **non-repeatable read** is glancing at one specific cell, writing down its value, glancing away, then glancing back at the *same* cell later and finding a different, but now properly saved, number — because someone else genuinely edited and saved it while you were looking elsewhere.
- A **phantom read** is counting how many rows in the spreadsheet meet some criterion (say, "balance over 400"), looking away, then counting again and getting a different total — not because an existing row's value changed, but because someone added or deleted an entire row that matches your criterion while you weren't looking.

## Why These Anomalies Are Categorized This Way

The SQL standard defines these three specific anomalies (rather than one vague "concurrency bug" category) because each one has a *different* underlying cause that requires a *different* mechanism to prevent, and a database vendor needs to be precise about exactly what guarantee each isolation level provides. Dirty reads are prevented simply by never exposing uncommitted data (something every serious relational database does by default, PostgreSQL included, at every level). Non-repeatable and phantom reads require a stronger mechanism — a stable snapshot spanning the whole transaction — because they're both fundamentally about *time*: whether the database re-checks "what's true right now" on every statement, or commits to one fixed view of "what was true when I started." Separating these named anomalies from each other is what makes an isolation level's guarantee something you can reason about precisely, rather than a vague promise of "more safety."

## Advantages

- **Precise vocabulary** — naming each anomaly separately lets you reason exactly about which guarantee a given isolation level does or doesn't provide, rather than a fuzzy sense of "more isolation is safer."
- **Maps directly onto isolation level selection** — knowing your workload's actual risk (does it re-read the same row? does it re-run a range query it needs to stay stable?) tells you precisely which level you need, per Topic 3.
- **PostgreSQL's stronger-than-standard Repeatable Read is a genuine, free benefit** — you get phantom-read protection at that level without needing to reach for the more expensive Serializable level.

## Disadvantages / Limitations

- **Recognizing which anomaly applies requires care** — non-repeatable reads and phantom reads are easy to conflate; the deciding question is always "did an existing row's value change, or did the set of matching rows change?"
- **None of these three anomalies cover everything that can go wrong under concurrency** — the serialization anomaly (write skew) mentioned in Topic 3 is a real fourth category that only Serializable eliminates, and it can occur even when none of these three named anomalies do.

## Best Practices

- Before choosing an isolation level for a piece of work, explicitly ask: does this transaction re-read the same row and require it to stay stable (non-repeatable read risk)? Does it re-run a range query and require the row set to stay stable (phantom read risk)? The answers point directly at Repeatable Read if either risk applies to your transaction's correctness.
- Don't assume dirty reads are something you need to defend against in PostgreSQL specifically — verify, when working with a different database product (Module 22), whether its default level actually permits them, since not every database is as strict as PostgreSQL on this point.
- When writing a report-generating transaction that runs several related queries, default to Repeatable Read so the whole report reflects one single, self-consistent moment in time.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Calling any concurrency bug a "dirty read" generically | Dirty read has a precise meaning — reading another transaction's *uncommitted* data — and is impossible in PostgreSQL at any isolation level. A vague "the data looked wrong" bug is far more likely a non-repeatable read or phantom read, both of which are possible under PostgreSQL's default Read Committed level. |
| Confusing a non-repeatable read with a phantom read | A non-repeatable read is about an *existing row's* value changing between two reads. A phantom read is about the *set of rows* returned by a range condition changing, due to rows being inserted or deleted, not existing values changing. |
| Assuming Repeatable Read in PostgreSQL still allows phantom reads because "that's what the textbook says" | Many textbooks describe the SQL standard's minimum guarantee, which does permit phantom reads at Repeatable Read. PostgreSQL's actual implementation is stricter and prevents them too — always verify against PostgreSQL's specific documented behavior, not the abstract standard, when working in PostgreSQL. |
| Believing the three named anomalies are the only things Serializable protects against | Serializable also prevents serialization anomalies (write skew) — subtler cross-transaction read/write patterns that none of these three named anomalies describe, and that Repeatable Read does not fully prevent. |

## Interview Questions

1. **Q: Define a dirty read, a non-repeatable read, and a phantom read, and give the one distinguishing detail between the latter two.**
   A: A dirty read is reading another transaction's uncommitted data. A non-repeatable read is re-reading the same row and getting a different value because another transaction committed a change to it in between. A phantom read is re-running the same range query and getting a different *set* of rows because another transaction inserted or deleted a matching row in between. The distinguishing detail: non-repeatable reads are about a value changing on an existing row; phantom reads are about the row set itself changing.

2. **Q: Can a dirty read ever occur in PostgreSQL?**
   A: No, at any isolation level, including one explicitly set to Read Uncommitted — PostgreSQL's MVCC engine never exposes another transaction's uncommitted row versions; Read Uncommitted is simply treated as Read Committed.

3. **Q: Does setting Repeatable Read in PostgreSQL protect against phantom reads?**
   A: Yes — even though the SQL standard's minimum definition of Repeatable Read still technically permits phantom reads, PostgreSQL's actual implementation takes one snapshot of the whole database for the entire transaction, which mechanically prevents phantom reads as well, making PostgreSQL's Repeatable Read stricter than the standard requires.

4. **Q: A report-generating transaction runs the same aggregate query twice, five minutes apart, and gets two different totals even though no error occurred. Which isolation level would you check first, and why?**
   A: Check whether the transaction is running under Read Committed (PostgreSQL's default) — under that level, each statement takes a fresh snapshot, so committed changes from other transactions become visible between the two queries, causing exactly this kind of non-repeatable (or phantom, if rows were added/removed) read. Switching the transaction to Repeatable Read would fix it by fixing the snapshot for the whole transaction.

## Summary

- A **dirty read** is reading another transaction's *uncommitted* data — impossible in PostgreSQL at any isolation level.
- A **non-repeatable read** is re-reading the same row and getting a different, but genuinely committed, value — possible under Read Committed, prevented from Repeatable Read upward.
- A **phantom read** is re-running a range query and getting a different set of matching rows due to inserts/deletes elsewhere — possible under Read Committed, but prevented under PostgreSQL's Repeatable Read (stricter than the SQL standard's minimum for that level) and Serializable.
- The root mechanism behind preventing both non-repeatable and phantom reads is the same: whether a transaction's snapshot refreshes per statement (Read Committed) or is fixed once for the whole transaction (Repeatable Read/Serializable).
- Serializable additionally prevents serialization anomalies (write skew), a subtler fourth category beyond these three named anomalies.
- Topic 5 shifts from *what the database guarantees* to *how it physically enforces it* — through locks.
