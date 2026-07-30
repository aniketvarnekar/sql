# BEGIN, COMMIT, and ROLLBACK

## Learning Objectives

By the end of this section you should be able to:
- Explain PostgreSQL's default autocommit behavior and why every statement is its own implicit transaction unless told otherwise
- Open an explicit transaction with `BEGIN`, end it successfully with `COMMIT`, and undo it entirely with `ROLLBACK`
- Use `SAVEPOINT` to undo only part of a transaction while keeping the rest
- Recognize what state the database is in at each point of a multi-statement transaction, including what other connections can and cannot see

## Prerequisites

- [ACID Properties](01-acid-properties.md) — this topic is the concrete syntax behind the Atomicity and Isolation guarantees just explained; you need to know *why* transactions matter before drilling into exactly how to control one.

## Motivation

Knowing *why* transactions matter (Topic 1) is only half the picture — you also need the exact commands to open one, end it successfully, or abandon it. `BEGIN`, `COMMIT`, and `ROLLBACK` are the three keywords that give you direct control over where a transaction starts and stops, and `SAVEPOINT` gives you a finer-grained "undo just this part" tool inside a single transaction. These four keywords are used constantly in real applications — anywhere a sequence of related changes needs to succeed or fail as a unit, these are the words that make that happen.

## Problem Statement

By default, if you open `psql` and run statements one after another, each one takes effect immediately and permanently:

```sql
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';
```

The instant this statement finishes, its effect is already permanent — any other connection querying the `accounts` table would see the new balance right away, and there is no way to undo it afterward except by running another statement to manually reverse it. This is fine for a single, standalone statement. But what if you need the debit above and its matching credit to Ben to either both happen or neither happen? Running them as two separate, independently-committed statements gives you no such guarantee — you need a way to explicitly say "these statements belong together as one unit," and a way to change your mind partway through and undo everything so far.

## Concept

### Autocommit Mode — PostgreSQL's Default

By default, PostgreSQL runs in **autocommit mode**: every individual SQL statement you send is automatically wrapped in its own implicit transaction and committed immediately if it succeeds.

```sql
INSERT INTO accounts (owner, balance) VALUES ('Chen', 250);
-- This single statement is, internally, its own complete transaction:
-- an implicit BEGIN, the INSERT, and an implicit COMMIT, all before
-- psql even shows you the next prompt.
```

If that `INSERT` violates a constraint, only that one statement fails — it does not affect any statement you ran before it, because each statement, on its own, is already a fully committed (or fully failed) unit. This is convenient for quick, one-off changes, but it gives you no way to group several statements into a single all-or-nothing operation — which is exactly the gap explicit transactions fill.

### BEGIN — Starting an Explicit Transaction

`BEGIN` (PostgreSQL also accepts the SQL-standard `START TRANSACTION` as an exact synonym) turns off autocommit for everything that follows, until you explicitly end the transaction:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';
```

At this point — after both `UPDATE`s but with no `COMMIT` yet — the changes exist only *inside this transaction*. Querying from the very same connection shows the new values:

```sql
SELECT owner, balance FROM accounts WHERE owner IN ('Asha', 'Ben');
```

```
 owner | balance
-------+---------
 Asha  |     900
 Ben   |     600
(2 rows)
```

But a **second, separate connection** running the exact same query at this exact moment sees the old, still-committed values:

```
 owner | balance
-------+---------
 Asha  |    1000
 Ben   |     500
(2 rows)
```

This is Isolation (Topic 1) in action: the second connection cannot see the first connection's uncommitted work.

### COMMIT — Making the Transaction Permanent

```sql
COMMIT;
```

`COMMIT` ends the transaction successfully and makes every change inside it permanent and visible to every other connection from this point forward. After `COMMIT`, a second connection querying the same rows now sees `900` and `600` — the transaction's effects are now indistinguishable from any other committed data.

### ROLLBACK — Undoing Everything Since BEGIN

If, instead of `COMMIT`, you run:

```sql
ROLLBACK;
```

every statement executed since the matching `BEGIN` is completely undone, as if none of it had ever happened. The full sequence:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';

SELECT owner, balance FROM accounts WHERE owner IN ('Asha', 'Ben');
```

```
 owner | balance
-------+---------
 Asha  |     900
 Ben   |     600
(2 rows)
```

```sql
ROLLBACK;

SELECT owner, balance FROM accounts WHERE owner IN ('Asha', 'Ben');
```

```
 owner | balance
-------+---------
 Asha  |    1000
 Ben   |     500
(2 rows)
```

Both balances are back to their original values — `ROLLBACK` erased both `UPDATE`s entirely. This is the practical face of Atomicity: nothing you did since `BEGIN` is kept once you `ROLLBACK`, no matter how many statements ran in between.

`ROLLBACK` is also what happens automatically if a statement inside the transaction hits an unrecoverable error (like the `CHECK` constraint violation from Topic 1) — PostgreSQL aborts the entire transaction and refuses to let you run any further statements in it (other than `ROLLBACK` itself) until you explicitly roll back:

```
ERROR:  new row for relation "accounts" violates check constraint "accounts_balance_check"
```
```
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

At that point, `ROLLBACK` is the only way forward — there is no partial recovery from inside an aborted transaction.

### SAVEPOINT — Partial Rollback Within a Transaction

Sometimes you don't want to undo an *entire* transaction, only the last few statements, while keeping everything before that point. A `SAVEPOINT` is a named marker inside a transaction that you can roll back to without discarding the whole transaction:

```sql
BEGIN;

INSERT INTO accounts (owner, balance) VALUES ('Diego', 300);

SAVEPOINT before_bonus;

UPDATE accounts SET balance = balance + 1000000 WHERE owner = 'Diego';  -- clearly a mistake

ROLLBACK TO SAVEPOINT before_bonus;

-- Diego's row from the INSERT is still here, but the bad UPDATE is undone.

UPDATE accounts SET balance = balance + 50 WHERE owner = 'Diego';  -- the correct adjustment

COMMIT;
```

Tracing through this: the `INSERT` happens and is kept. `SAVEPOINT before_bonus` marks a point to return to. The oversized `UPDATE` runs (as far as this transaction is concerned, at this moment, it took effect). `ROLLBACK TO SAVEPOINT before_bonus` discards *only* that `UPDATE`, rewinding the transaction back to exactly the state it was in right after the savepoint was created — the `INSERT` is untouched. Then a corrected `UPDATE` runs, and `COMMIT` makes the final result (the `INSERT` plus the corrected `UPDATE`, but never the million-unit mistake) permanent.

A transaction can have multiple savepoints, and you can release one you no longer need with `RELEASE SAVEPOINT before_bonus;` (which forgets the savepoint but keeps everything committed so far in the transaction — it does not commit the transaction itself).

```sql
BEGIN;
INSERT INTO accounts (owner, balance) VALUES ('Elena', 100);
SAVEPOINT sp1;
INSERT INTO accounts (owner, balance) VALUES ('Farah', 200);
SAVEPOINT sp2;
UPDATE accounts SET balance = balance - 5000 WHERE owner = 'Farah';  -- mistake, would violate CHECK
ROLLBACK TO SAVEPOINT sp2;   -- undoes only the bad UPDATE; Farah's INSERT survives
RELEASE SAVEPOINT sp1;        -- no longer need this checkpoint
COMMIT;                       -- keeps both INSERTs
```

## Internal Working (Preview)

```
 BEGIN                COMMIT
   │                     │
   ▼                     ▼
 ┌─────────────────────────────────────────────┐
 │           one transaction, one xid           │
 │                                               │
 │  stmt 1 ──▶ stmt 2 ──▶ SAVEPOINT sp1 ──▶ stmt 3   │
 │                             │                │
 │                             └── ROLLBACK TO sp1 discards only stmt 3's effect
 └─────────────────────────────────────────────┘
```

Every explicit transaction is assigned a transaction ID (`xid`) by PostgreSQL. Every row version written inside that transaction is tagged internally with that `xid`. A `COMMIT` marks the `xid` as committed in a small internal record; from that instant on, any other transaction's MVCC snapshot (Topic 3) considers rows written by that `xid` visible. A `ROLLBACK` marks the `xid` as aborted instead — every row version tagged with it is simply treated as if it never existed, with nothing to physically "undo" row by row. A `SAVEPOINT` is implemented as a lightweight *sub-transaction* with its own internal id nested inside the main transaction; `ROLLBACK TO SAVEPOINT` aborts only that sub-transaction's id (and any sub-transactions nested after it), while the outer transaction's earlier, already-established work remains untouched.

## Real-World Analogy

Autocommit mode is like sending an email the instant you finish typing each sentence — nothing is grouped, and once sent, a sentence can't be unsent. An explicit transaction is like writing a full email in drafts: `BEGIN` opens the draft, every statement you run is a sentence added to the draft (visible to you, invisible to the eventual recipient), `COMMIT` is hitting Send (making the whole draft permanent and visible to everyone at once), and `ROLLBACK` is discarding the entire draft unsent. A `SAVEPOINT` is like a version-history checkpoint while editing that draft — you can revert back to an earlier checkpoint (undoing only the paragraphs written after it) without discarding the parts of the draft you wrote before that checkpoint and still want to keep.

## Why Explicit Transaction Control Was Designed This Way

Autocommit-by-default matches how most single statements are actually used — a lone `SELECT` or a single corrective `UPDATE` doesn't need explicit grouping, and forcing every statement to require a manual `BEGIN`/`COMMIT` pair would be needless ceremony for the overwhelmingly common case. But any time a set of statements has a logical "all or nothing" relationship (the transfer example throughout this module), the database needs an explicit way for you to declare where that unit starts and ends — hence `BEGIN` and `COMMIT` as opt-in boundaries around exactly the statements that need to be atomic together. `SAVEPOINT` exists because real multi-step business logic often has structure *within* a single transaction — try one path, and if it doesn't work out, fall back to an earlier point without abandoning everything already validated — mirroring how application code uses nested error handling rather than a single all-encompassing try/catch.

## Advantages

- **Precise control over atomicity boundaries** — you decide exactly which statements must succeed or fail together, rather than the database guessing.
- **Safe experimentation within a session** — you can run statements, inspect their effect with a `SELECT` from the same connection, and still discard everything with `ROLLBACK` if something looks wrong, before any other connection ever sees it.
- **Partial recovery via `SAVEPOINT`** — a single mistaken statement deep inside a long transaction doesn't force you to discard everything already done correctly.

## Disadvantages / Limitations

- **Open transactions hold resources** — an explicit transaction left open (neither committed nor rolled back) holds row locks (Topic 5) and prevents certain internal cleanup (autovacuum) from reclaiming old row versions, which can degrade performance for the whole database if left open too long.
- **Aborted transactions block all further statements** — once any statement inside a transaction errors, every subsequent statement in that same transaction is rejected until you `ROLLBACK` (or `ROLLBACK TO` a savepoint before the error) — you cannot simply "skip" the failed statement and continue.
- **Savepoints add a small amount of overhead** — each savepoint is a real sub-transaction PostgreSQL tracks; using dozens of them in a single transaction for fine-grained control has a real, if usually small, cost.

## Best Practices

- Keep explicit transactions as short as possible — open with `BEGIN` right before the first statement that needs to be atomic, and `COMMIT` or `ROLLBACK` as soon as the related work is done; don't leave a transaction open while waiting on user input or an external network call.
- Always pair every `BEGIN` with an explicit `COMMIT` or `ROLLBACK` in your application code — an application that crashes or times out with an open transaction can leave locks held far longer than intended (Topic 5 and Topic 6 cover the downstream consequences).
- Use `SAVEPOINT` when a transaction has a natural "try this, fall back if it fails" structure, rather than restructuring the whole transaction into smaller pieces just to get partial-undo behavior.
- After any statement fails inside a transaction, check whether you need `ROLLBACK` (undo everything) or `ROLLBACK TO SAVEPOINT` (undo back to a specific checkpoint) — don't assume you can just continue.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a statement can be undone after autocommit has already committed it | In autocommit mode, each statement is already a complete, committed transaction the instant it finishes — there is no implicit "undo the last statement"; you would need to write a new statement that manually reverses the change. |
| Continuing to run statements after one has errored inside a transaction, expecting only that one to fail | PostgreSQL marks the whole transaction as aborted after any statement error; every following statement in that same transaction is rejected until you `ROLLBACK` (or roll back to an earlier savepoint). |
| Forgetting to `COMMIT` and leaving a transaction open indefinitely | An idle open transaction still holds any row locks it acquired (Topic 5) and can block other transactions or degrade internal cleanup — "forgot to commit" is a common, real cause of mysterious production slowdowns. |
| Using `ROLLBACK TO SAVEPOINT` without first creating that `SAVEPOINT` | The name must match a savepoint actually established earlier in the same transaction with `SAVEPOINT name;` — there's no implicit checkpoint to roll back to otherwise. |

## Interview Questions

1. **Q: What does PostgreSQL do by default if you never run `BEGIN` at all?**
   A: It runs in autocommit mode — every individual statement is automatically wrapped in its own implicit transaction and committed immediately if it succeeds, with no way to group multiple statements into one atomic unit unless you explicitly `BEGIN` first.

2. **Q: What is the difference between `ROLLBACK` and `ROLLBACK TO SAVEPOINT <name>`?**
   A: `ROLLBACK` undoes every statement since the transaction's `BEGIN` and ends the transaction entirely. `ROLLBACK TO SAVEPOINT <name>` undoes only the statements run after that specific savepoint was created, but keeps the transaction open and keeps everything committed before the savepoint — you can continue running new statements afterward and still `COMMIT` the transaction later.

3. **Q: If a statement inside an explicit transaction throws an error, can you simply run the next statement and continue?**
   A: No. PostgreSQL marks the entire transaction as aborted once any statement inside it errors, and rejects every further statement (other than `ROLLBACK`, or `ROLLBACK TO` an earlier savepoint) until the transaction is explicitly ended.

4. **Q: Why is it considered bad practice to leave a `BEGIN`ed transaction open for a long time without committing?**
   A: Because any locks it has acquired (Topic 5) remain held for as long as the transaction stays open, which can block other transactions from proceeding, and it can also interfere with PostgreSQL's internal cleanup of old row versions — both of which can degrade the whole database's performance, not just the connection holding the open transaction.

## Summary

- PostgreSQL defaults to **autocommit**: every statement is its own implicit, immediately-committed transaction.
- **`BEGIN`** (or `START TRANSACTION`) opens an explicit transaction, turning off autocommit until you end it.
- **`COMMIT`** makes every change since `BEGIN` permanent and visible to all other connections.
- **`ROLLBACK`** discards every change since `BEGIN` entirely, as if none of the statements ran.
- **`SAVEPOINT name`** creates a checkpoint inside a transaction; **`ROLLBACK TO SAVEPOINT name`** undoes only the statements after that checkpoint, without discarding the whole transaction; **`RELEASE SAVEPOINT name`** forgets a savepoint you no longer need.
- Keep explicit transactions short and always end them explicitly — an open transaction holds resources (Topic 5) for as long as it stays open.
