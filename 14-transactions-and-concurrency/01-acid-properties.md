# ACID Properties

## Learning Objectives

By the end of this section you should be able to:
- State a precise definition of Atomicity, Consistency, Isolation, and Durability
- For each property, describe a concrete scenario in which its absence causes real, tangible data corruption
- Explain, at a conceptual level, which part of PostgreSQL's engine is responsible for delivering each guarantee
- Distinguish Atomicity (all steps happen or none do) from Isolation (concurrent transactions don't see each other's in-progress work) — the two properties beginners confuse most often

## Prerequisites

- [Module 6 — Modifying Data](../06-modifying-data/00-module-overview.md) — every example in this topic is a sequence of `INSERT`/`UPDATE`/`DELETE` statements; you need to already be comfortable writing those individually before wrapping them in a transaction.
- [What Is a Database and a DBMS?](../01-introduction/01-what-is-a-database-and-a-dbms.md) — this topic first stated, at a conceptual level, that the DBMS "guarantees a change either fully happens or not at all" and "coordinates access so simultaneous changes don't collide destructively." This topic is the precise, technical unpacking of both of those sentences.
- [Module 5 — Constraints & Keys](../05-constraints-and-keys/00-module-overview.md) — `CHECK` and `NOT NULL` constraints are how the database enforces the *Consistency* property below; you need to already know what a constraint is for that section to make sense.

## Motivation

Almost every real-world data change is not a single, self-contained statement — it's a sequence of related statements that only make sense as a group. Moving money between two accounts is two `UPDATE`s. Placing an order is an `INSERT` into an orders table, an `UPDATE` decrementing inventory, and possibly another `INSERT` into a payments table. If any one statement in that sequence can be silently interrupted, or if two such sequences running at the same time can see and corrupt each other's half-finished work, your data will eventually — not maybe, *eventually* — become wrong in a way no single `SELECT` can fix after the fact. ACID is the set of four guarantees a real database makes so that you can write multi-statement sequences and trust the result, no matter what crashes, errors, or concurrent activity happens around them.

## Problem Statement

Consider the simplest possible multi-step operation: transferring 100 currency units from Asha's account to Ben's account. Written as two independent statements:

```sql
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';
```

Now imagine, with no special guarantees in place:

- **The server crashes or the connection drops** after the first `UPDATE` runs but before the second one does. Asha's balance is now permanently 100 lower, and Ben's account never received it. 100 units have vanished from existence.
- **A bug (or a missing constraint) lets Asha's balance go negative.** She had only 40 units, but the transfer proceeds anyway, and her balance becomes `-60`. The database now holds a value that should never be allowed to exist.
- **A second connection reads Asha's account balance in between the two `UPDATE`s** — after the debit has happened but before the credit — and reports a number that reflects money that is, at that exact instant, nowhere: debited from Asha, not yet credited to Ben. That reader has been shown a state of the world that should never be externally visible.
- **The transfer completes and the application reports success to the user, and then the machine loses power** a fraction of a second later, before the change was ever physically written to durable storage. On reboot, the transfer is simply gone, even though the user was told it succeeded.

Four different failure modes, four different guarantees needed. These are exactly the four letters in ACID: **A**tomicity, **C**onsistency, **I**solation, **D**urability.

## Concept

A **transaction** is a sequence of one or more SQL statements that the database treats as a single logical unit of work. ACID is the set of four properties every transaction in a real relational database management system is guaranteed to have.

### Atomicity — "All or Nothing"

> **Atomicity** guarantees that every statement inside a transaction either all take effect together, or none of them take effect at all. There is no partially-applied state.

Wrapped in an explicit transaction (full syntax in Topic 2), the transfer becomes:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';

COMMIT;
```

**What Atomicity prevents:** if the connection drops, the server crashes, or an error occurs after the first `UPDATE` but before `COMMIT`, PostgreSQL guarantees that *neither* `UPDATE` is visible to anyone, ever — including after the crashed server restarts. There is no possible outside observation of the state "debited but not credited." Either both statements' effects exist, permanently, or neither does. This is fundamentally different from just "running two statements back to back" — atomicity is an active guarantee the database enforces, not a side effect of statements running quickly.

**Without Atomicity:** the "100 units vanish" scenario above is exactly what happens. Every partial-completion bug in a system without transactions traces back to a missing atomicity guarantee.

### Consistency — "Valid State to Valid State"

> **Consistency** guarantees that a transaction can only take the database from one state that satisfies all defined rules (constraints) to another state that also satisfies all of them — it can never leave the data in a state that violates a rule you've declared, even temporarily as the final result.

Suppose the `accounts` table enforces a rule that balances can never go negative:

```sql
CREATE TABLE accounts (
    id      SERIAL PRIMARY KEY,
    owner   TEXT NOT NULL,
    balance NUMERIC NOT NULL CHECK (balance >= 0)
);

INSERT INTO accounts (owner, balance) VALUES ('Asha', 40), ('Ben', 500);
```

Now attempt to transfer 100 units from Asha, who only has 40:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';  -- would make balance = -60
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';

COMMIT;
```

Running this produces:

```
ERROR:  new row for relation "accounts" violates check constraint "accounts_balance_check"
DETAIL:  Failing row contains (1, Asha, -60).
```

**What Consistency guarantees:** the instant the first `UPDATE` would produce a row violating the `CHECK` constraint, PostgreSQL rejects that statement and aborts the entire transaction — the second `UPDATE` never even needs to run, and (thanks to Atomicity, working together with Consistency here) nothing from this transaction is applied. Asha's balance remains 40, Ben's remains 500, exactly as if the transaction had never been attempted.

**Without Consistency:** the database would silently accept a `-60` balance — a value that is nonsensical for the business rule you declared, and every report, every later calculation, every downstream system now has to defensively guard against a value that should have been structurally impossible.

Consistency is somewhat different from the other three ACID properties: Atomicity, Isolation, and Durability are guarantees the *transaction manager* itself provides regardless of what your schema looks like. Consistency is a guarantee that depends on *you* — it means the database faithfully enforces whatever constraints you declared (Module 5). The database can't know your business rules unless you tell it (via `CHECK`, `NOT NULL`, foreign keys, `UNIQUE`); Consistency is the promise that once you've told it, it will never let a transaction commit a violation.

### Isolation — "Concurrent Transactions Don't Interfere"

> **Isolation** guarantees that concurrent transactions cannot see each other's in-progress, uncommitted changes, and (to a degree controlled by the isolation level — Topic 3) behave as if they had run one at a time rather than simultaneously.

Picture two things happening at almost the same instant: Transaction A is running the transfer above, and Transaction B is a completely separate connection just checking Asha's balance.

```
Time    Transaction A (the transfer)                Transaction B (checking balance)
----    -------------------------------------------  -------------------------------------------
T1      BEGIN;
T2      UPDATE accounts SET balance = balance - 100
        WHERE owner = 'Asha';
        -- Asha's balance is now 900 (uncommitted)
T3                                                    BEGIN;
T4                                                    SELECT balance FROM accounts
                                                       WHERE owner = 'Asha';
T5      UPDATE accounts SET balance = balance + 100
        WHERE owner = 'Ben';
T6      COMMIT;
T7                                                    COMMIT;
```

**What Isolation guarantees:** at T4, Transaction B's `SELECT` does **not** see the uncommitted 900 value that Transaction A produced at T2 — it sees Asha's last *committed* balance (1000, from before the transfer started), because Transaction A has not committed yet. Transaction B is fully isolated from A's in-flight, unfinished work. Once A commits at T6, a *new* read from B (or a new transaction) would see the updated value.

**Without Isolation:** Transaction B would see 900 — a value that reflects half of a transfer that, at that moment, might still fail and roll back entirely. If A's transaction later rolled back (say, the `CHECK` constraint failed on the second `UPDATE`), B would have acted on a number that never actually existed as a committed fact. This exact failure has a name — a **dirty read** — and it, along with two related anomalies, is the entire subject of Topic 4. Isolation is the property with the most nuance of the four, because *how much* isolation you get is configurable (Topic 3) — unlike Atomicity, Consistency, and Durability, which are always fully guaranteed, Isolation has selectable strength levels because full isolation has a real performance cost.

### Durability — "Once Committed, It Survives Anything"

> **Durability** guarantees that once a transaction has successfully `COMMIT`ted, its effects are permanent — they will survive a server crash, an operating system crash, or a power loss, with no exceptions.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE owner = 'Asha';
UPDATE accounts SET balance = balance + 100 WHERE owner = 'Ben';
COMMIT;
-- The instant COMMIT returns successfully, this change is permanent.
```

**What Durability guarantees:** if the machine running PostgreSQL loses power one microsecond after `COMMIT` returns control to the application, the transfer is still there when the database restarts. PostgreSQL does not simply update an in-memory copy of the data and hope to flush it to disk "eventually" — a `COMMIT` does not report success back to you until the change has been recorded to durable storage in a way that survives a crash (the mechanism is the Write-Ahead Log, covered in Internal Working below).

**Without Durability:** an application could tell a user "your transfer succeeded," the user walks away, and a crash thirty seconds later silently erases it — with the user having no idea, since they were already told it worked. This is the single most damaging kind of bug to trust in a system, because it violates a promise that was already actively communicated as fulfilled.

## Internal Working (Deep Dive)

Each ACID letter maps to a real, distinct piece of engineering inside PostgreSQL:

```
        Atomicity & Durability                    Isolation                      Consistency
               │                                       │                              │
               ▼                                       ▼                              ▼
     Write-Ahead Log (WAL)                MVCC (Multiversion Concurrency        Constraint checks
                                            Control) — Topic 3 & 4 go deep      (CHECK, NOT NULL,
     Every change is first written                                              UNIQUE, foreign keys —
     to an append-only log on disk        Each transaction sees a              Module 5) evaluated
     *before* the table itself is         "snapshot" of the database,          before any row change
     modified. COMMIT does not            so concurrent writers don't          is allowed to be
     return until the WAL record is       corrupt each other's view.           finalized.
     safely on disk. If the server
     crashes, PostgreSQL replays the
     WAL on restart to redo any
     committed-but-not-yet-applied
     changes and undo any changes
     from transactions that never
     committed.
```

- **Atomicity** is implemented by tracking every change a transaction makes against its transaction ID; if the transaction never commits (crash, error, explicit `ROLLBACK`), PostgreSQL treats every row version it wrote as invisible/dead, as if it never happened — nothing needs to be "undone" row by row, because uncommitted work was never made visible to begin with.
- **Durability** is implemented by the WAL: PostgreSQL always writes a durable log record of a change before acknowledging the commit, so even an immediate power loss can be recovered from by replaying the log on restart.
- **Isolation** is implemented by MVCC: rather than transactions locking each other out to read, PostgreSQL keeps multiple versions of each row and gives each transaction a consistent "snapshot" of which versions it's allowed to see, based on which transactions had committed at the moment its snapshot was taken. Topics 3 and 4 explain exactly how much of another transaction's changes a given isolation level lets you see.
- **Consistency** is implemented by the constraint-checking machinery from Module 5, run against every row a transaction tries to insert or modify, before that transaction is permitted to commit.

## Real-World Analogy

Think of a bank wire transfer processed through a clearinghouse rather than by physically moving cash:

- **Atomicity** is the clearinghouse's all-or-nothing settlement rule: the debit from one bank and the credit to the other are recorded as a single settlement instruction. If the process is interrupted, the whole instruction is voided — you never end up with a debit recorded at one bank and no matching credit at the other.
- **Consistency** is the clearinghouse refusing to process an instruction that would leave an account below its allowed minimum — the rule is checked as part of the settlement itself, not corrected after the fact.
- **Isolation** is the fact that no other bank can see "in-progress" settlement instructions — they only ever see completed, settled balances, never an account caught mid-transfer.
- **Durability** is the clearinghouse's permanent settlement ledger: once an instruction is confirmed settled, it is written to a permanent record that survives even if the clearinghouse's building loses power that night.

## Why ACID Was Designed This Way

The relational model (Module 1's framing of "the DBMS coordinates access so simultaneous changes don't collide destructively") only means something concrete if it's backed by an explicit, checkable contract — and ACID is that contract. Early database systems that lacked these guarantees pushed all of this responsibility onto application code: every application had to manually implement crash recovery, manually re-check business rules on every write, and manually coordinate with every other program that might be touching the same data at the same time — and inevitably, some of them got it wrong. By making ACID a guarantee of the DBMS itself rather than a discipline application developers had to reinvent for every project, the relational model moved "is my data going to end up correct" out of application logic and into infrastructure that has been tested for decades across an enormous range of failure conditions. This is the same theme as SQL's declarative nature (Module 1): you state *what* transactional behavior you want (`BEGIN` ... `COMMIT`), and the database is responsible for *how* it's actually guaranteed underneath.

## Advantages

- **Correctness under failure** — crashes, power loss, and application bugs mid-operation cannot leave your data in a half-changed state; this is Atomicity and Durability working together.
- **Enforced business rules** — Consistency means constraints declared once (Module 5) are actually impossible to violate via any transaction, not just "checked by well-behaved application code."
- **Safe concurrency by default** — Isolation means you can write a multi-statement operation without personally reasoning about every other transaction that might run at the same instant; the database handles the interference for you.
- **A trustworthy commit acknowledgment** — when `COMMIT` returns successfully, you can rely on that fact permanently, which lets applications report success to users with confidence.

## Disadvantages / Limitations

- **Performance cost** — durability (flushing to the WAL before acknowledging commit) and strong isolation both cost time and I/O compared to a system that made no such guarantees; extremely high-throughput systems sometimes deliberately relax specific guarantees (e.g., asynchronous replication, weaker isolation levels) in exchange for speed, accepting the trade-off consciously.
- **Isolation strength is a spectrum, not a single guarantee** — unlike the other three properties, "how isolated" a transaction is depends on a configurable isolation level (Topic 3); the strongest level (Serializable) has the highest correctness guarantee but the highest concurrency cost, and picking the wrong level for your workload is a common real-world mistake.
- **Consistency only enforces what you declare** — the database cannot protect you from a business rule you never wrote down as a constraint; ACID guarantees the rules you state are unbreakable, not that you thought of every rule you needed.

## Best Practices

- Always wrap multi-statement operations that must succeed or fail together in an explicit transaction (Topic 2) — never rely on "the two statements ran right after each other and nothing went wrong" as a substitute for atomicity.
- Push business rules into real constraints (Module 5) wherever possible, rather than only checking them in application code first — constraints are what make the Consistency guarantee actually protect you, and they protect every writer, not just the one you remembered to add a check to.
- Don't treat ACID as something you need to manually implement — it's a property of correctly using transactions (`BEGIN`/`COMMIT`/`ROLLBACK`), not a separate feature you opt into.
- When evaluating whether a workload can tolerate a weaker isolation level or relaxed durability setting for performance, make that decision deliberately and document it — don't discover the trade-off accidentally in production.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Confusing Atomicity with Isolation | Atomicity is about a *single* transaction's own steps succeeding or failing together. Isolation is about *two different* transactions not seeing each other's uncommitted work. They solve different problems and both can fail independently of one another. |
| Believing ACID is optional "extra safety" you can skip for simple operations | Even a "simple" two-statement update (like the transfer example) is exactly the kind of operation that silently corrupts data without transactional guarantees — ACID is not overhead for edge cases, it's the baseline correctness contract for any multi-step change. |
| Assuming Consistency means "the data is always correct" in a general sense | Consistency only guarantees that *declared* constraints are never violated by a committed transaction. It cannot catch a business rule you never wrote down as a `CHECK`, `NOT NULL`, or foreign key. |
| Treating all isolation levels as giving the same protection | Only Atomicity, Consistency, and Durability are unconditionally guaranteed at full strength. Isolation strength is selectable, and the default level (Read Committed, Topic 3) permits some anomalies that a stricter level would prevent. |

## Interview Questions

1. **Q: What does each letter in ACID stand for, and give a one-sentence guarantee for each?**
   A: Atomicity — all statements in a transaction succeed together or none take effect at all. Consistency — a transaction can only move the database between states that satisfy all declared constraints. Isolation — concurrent transactions cannot see each other's uncommitted, in-progress changes. Durability — once a transaction commits, its effects survive any subsequent crash or power loss.

2. **Q: In a bank transfer implemented as two `UPDATE` statements, which ACID property specifically prevents the case where the debit succeeds but the credit is lost due to a crash in between?**
   A: Atomicity. It guarantees that either both `UPDATE`s take effect (on `COMMIT`) or neither does (on crash or `ROLLBACK`) — there is no state where only the debit is permanently recorded.

3. **Q: Why is Isolation described as having "levels" while the other three ACID properties are not?**
   A: Atomicity, Consistency, and Durability are binary guarantees always delivered at full strength by the transaction manager. Isolation involves a genuine trade-off between correctness (preventing more concurrency anomalies) and performance/concurrency throughput, so the SQL standard defines multiple isolation levels of increasing strictness, and the database (and the developer, per transaction) chooses which level to use — covered fully in Topic 3.

4. **Q: How does PostgreSQL physically guarantee Durability once a `COMMIT` succeeds?**
   A: Through the Write-Ahead Log (WAL) — before a `COMMIT` is acknowledged as successful, its change record has already been written to the durable WAL on disk. If the server crashes immediately afterward, PostgreSQL replays the WAL on restart to reconstruct any committed changes that hadn't yet been fully applied to the actual data files.

## Summary

- **Atomicity** guarantees a transaction's statements all take effect together or not at all — no partially-applied changes survive a crash or error.
- **Consistency** guarantees a transaction can never commit a state that violates a declared constraint (Module 5), moving the database only between valid states.
- **Isolation** guarantees concurrent transactions cannot see each other's uncommitted, in-progress changes — the exact strength of this guarantee is configurable via isolation levels (Topic 3).
- **Durability** guarantees that once `COMMIT` succeeds, the change survives any subsequent crash, thanks to the Write-Ahead Log.
- Atomicity and Durability are delivered by the WAL; Isolation is delivered by MVCC snapshots; Consistency is delivered by constraint enforcement — four distinct guarantees, four distinct mechanisms, all activated the moment you use a transaction.
- Topic 2 gives you the actual syntax (`BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`) to control transactions explicitly.
