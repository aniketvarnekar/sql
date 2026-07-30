# Stored Procedures

## Learning Objectives

By the end of this section you should be able to:
- Write a `CREATE PROCEDURE` statement in PL/pgSQL and invoke it with `CALL`
- State precisely why a procedure, unlike a function, is allowed to issue `COMMIT` and `ROLLBACK` inside its own body
- Explain why a procedure does not "return a value" the same way a function does, and how `INOUT` parameters partially substitute for that
- Decide whether a piece of multi-step logic belongs in a procedure rather than a function

## Prerequisites

- [User-Defined Functions](01-user-defined-functions.md) — procedures share almost all of their syntax and PL/pgSQL body structure with functions; this topic focuses on the handful of ways they genuinely differ, so read that topic first.
- [Module 14 — Transactions & Concurrency](../14-transactions-and-concurrency/00-module-overview.md) — a procedure's defining capability is committing and rolling back transactions from inside its own body; you need to already know what a transaction, a `COMMIT`, and a `ROLLBACK` guarantee before that capability means anything.

## Motivation

A function is a great fit for a single, composable calculation — but the moment you need to perform several distinct steps as an operation, some of which you may want to commit even if a later step fails or takes a long time, a function's single defining restriction becomes a wall: a function always executes inside whatever transaction its caller is already running, and cannot commit or roll back that transaction itself. A **stored procedure** exists specifically to remove that restriction — it is a named, callable, multi-step unit of work that is allowed to manage its own transaction boundaries.

## Problem Statement

Recall the classic funds-transfer scenario first introduced in Module 1's discussion of Transaction Control Language: moving money from one account to another must happen as an all-or-nothing unit, using `BEGIN`/`COMMIT`/`ROLLBACK` (fully covered in Module 14).

```sql
CREATE TABLE accounts (
    id      SERIAL PRIMARY KEY,
    owner   TEXT NOT NULL,
    balance NUMERIC NOT NULL CHECK (balance >= 0)
);
```

Every application that needs to perform this transfer has to correctly re-type the same sequence: check the sender has sufficient funds, debit the sender, credit the receiver, and either commit both changes together or roll back both if anything goes wrong. If this logic is written out fresh in every application, script, or ad hoc admin tool that ever needs to move money between accounts, a mistake in any one of those places — someone forgetting the balance check, someone committing the debit before confirming the credit succeeded — is a real risk to data correctness, and one that's easy to introduce without anyone noticing until real money is actually lost. What's needed is a single, named, transaction-aware operation: "transfer this amount from this account to that account," with the safety logic guaranteed to run the same way every single time it's invoked.

A user-defined function looks like it should solve this — until you try to give it the ability to `COMMIT` partway through and PostgreSQL refuses:

```sql
CREATE OR REPLACE FUNCTION transfer_funds_broken(p_amount NUMERIC)
RETURNS VOID
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - p_amount WHERE id = 1;
    COMMIT; -- ERROR: invalid transaction termination
END;
$$;
```

Functions execute inside the transaction of whatever called them and are not permitted to end that transaction. This is exactly the gap stored procedures fill.

## Concept

### Anatomy of `CREATE PROCEDURE`

```sql
CREATE OR REPLACE PROCEDURE procedure_name(parameter_name parameter_type, ...)
LANGUAGE plpgsql
AS $$
BEGIN
    -- procedural logic here, possibly including COMMIT / ROLLBACK
END;
$$;
```

The structure is nearly identical to `CREATE FUNCTION`, with two important differences visible right away:

- There is no `RETURNS` clause. A procedure does not hand back a value the way a function does — its purpose is to *perform an operation*, not to be evaluated as an expression.
- It is invoked with the `CALL` statement, never inside a `SELECT`:

```sql
CALL procedure_name(argument1, ...);
```

`CALL` is its own top-level SQL statement, syntactically separate from `SELECT` — this is the clearest surface-level signal that a procedure is not an expression and cannot be nested inside one, unlike a function.

### A Worked Procedure with Transaction Control

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    p_sender_id   INTEGER,
    p_receiver_id INTEGER,
    p_amount      NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_sender_balance NUMERIC;
BEGIN
    IF p_amount <= 0 THEN
        RAISE EXCEPTION 'Transfer amount must be positive, got %', p_amount;
    END IF;

    SELECT balance INTO v_sender_balance
    FROM accounts
    WHERE id = p_sender_id
    FOR UPDATE;

    IF v_sender_balance IS NULL THEN
        RAISE EXCEPTION 'Sender account % does not exist', p_sender_id;
    END IF;

    IF v_sender_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: account % has %, needs %',
            p_sender_id, v_sender_balance, p_amount;
    END IF;

    UPDATE accounts SET balance = balance - p_amount WHERE id = p_sender_id;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_receiver_id;

    COMMIT;
END;
$$;
```

New pieces of syntax, beyond what Topic 1 already introduced:

| Piece | Meaning |
|---|---|
| `RAISE EXCEPTION 'message %', value;` | Aborts execution immediately with an error, formatting `value` into the message at the `%` placeholder — the PL/pgSQL equivalent of a guard clause that stops bad input in its tracks. |
| `SELECT ... INTO v_sender_balance ... FOR UPDATE` | Reads a value into a local variable while also locking the selected row (`FOR UPDATE`, part of Module 14's locking coverage) so no other transaction can change the sender's balance between this check and the debit below it. |
| `COMMIT;` | Ends the current transaction, making the debit and credit permanent. This is the statement a function is categorically forbidden from issuing — and the entire reason this logic is written as a procedure. |

Calling it:

```sql
CALL transfer_funds(1, 2, 15000);
```

```
CALL
```

```sql
SELECT id, owner, balance FROM accounts ORDER BY id;
```

```
 id |  owner  | balance
----+---------+---------
  1 | Asha    |   80000
  2 | Chen    |   35000
(2 rows)
```

Calling it again with an amount larger than the sender's balance demonstrates the guard clause:

```sql
CALL transfer_funds(1, 2, 999999);
```

```
ERROR:  Insufficient funds: account 1 has 80000, needs 999999
```

Because the exception was raised *before* the `COMMIT`, PostgreSQL automatically rolls back any changes the procedure made earlier in the same call — nothing partial is left behind.

### `INOUT` Parameters — How a Procedure "Returns" Data

Since a procedure has no `RETURNS` clause, the closest equivalent to "sending a value back to the caller" is an `INOUT` parameter — a parameter that carries a value in, and carries a (possibly different) value back out after the call completes:

```sql
CREATE OR REPLACE PROCEDURE transfer_funds_reporting(
    p_sender_id       INTEGER,
    p_receiver_id     INTEGER,
    p_amount          NUMERIC,
    INOUT p_new_sender_balance NUMERIC DEFAULT NULL
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_sender_id;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_receiver_id;

    SELECT balance INTO p_new_sender_balance
    FROM accounts WHERE id = p_sender_id;

    COMMIT;
END;
$$;
```

```sql
CALL transfer_funds_reporting(1, 2, 5000, NULL);
```

```
 p_new_sender_balance
-----------------------
                 75000
(1 row)
```

This is a meaningfully different mechanism from a function's `RETURN` — it's a side channel on a named parameter, not a value the procedure call itself evaluates to, and it only exists because procedures otherwise have no way to hand information back to the caller.

## Internal Working (Preview)

A key restriction to understand: `COMMIT`/`ROLLBACK` are only permitted inside a procedure when that procedure is called at the "top level" — directly via `CALL` from a client, with no explicit surrounding transaction already open (PostgreSQL's default `autocommit` mode, covered in Module 14, treats each standalone statement as its own transaction unless you explicitly `BEGIN` one). If you wrap the `CALL` in your own explicit `BEGIN ... COMMIT` block, or call the procedure from inside a function, the procedure's internal `COMMIT`/`ROLLBACK` calls are rejected, because PostgreSQL does not support nested transactions — there can only be one active top-level transaction being managed at a time, and it must be unambiguous who is managing it.

```
 Client sends: CALL transfer_funds(1, 2, 15000);
                       │
                       ▼
   No explicit BEGIN already open (autocommit mode)
                       │
                       ▼
     Procedure body runs as part of a NEW, procedure-
     owned transaction — it is free to COMMIT/ROLLBACK
     partway through its own body.
```

Each statement inside the procedure body (the `SELECT ... FOR UPDATE`, each `UPDATE`) is planned and executed by the same query planner and transaction/locking machinery covered throughout Module 14 — a procedure body is not a special execution mode, it is ordinary SQL statements, sequenced by PL/pgSQL, with the added privilege of being allowed to close out the transaction they're running in.

## Real-World Analogy

A stored procedure is like a bank teller authorized to perform an entire multi-step transaction at their own discretion — check the sender's balance, debit one account, credit another, and finalize the transaction in the bank's ledger — all as one visit to the counter, with the authority to abort the whole thing and hand your card back if a rule is violated partway through. A function, by contrast, is more like a calculator sitting on the teller's counter: useful for a quick computation the teller plugs into their own larger process, but with no authority to finalize anything in the ledger itself.

## Why Stored Procedures Were Designed This Way

The restriction that functions cannot commit or roll back isn't an arbitrary limitation — it protects a core guarantee established in Module 14: a single SQL statement (and by extension, an expression evaluated as part of one) must behave as one atomic, self-contained unit of work from the perspective of the transaction that's running it. If a function embedded inside a `SELECT`'s column list could silently commit the enclosing transaction, the meaning of "run this one query" would become unpredictable — you could no longer reason about a statement's transactional boundary just by reading the statement. Procedures are deliberately *not* embeddable inside a query expression — they are invoked standalone, with their own dedicated statement (`CALL`) — precisely so that granting them transaction control doesn't compromise the predictability of ordinary queries and functions everywhere else in the language.

## Advantages

- **Genuine transaction control.** A procedure can validate, act, and commit — or detect a problem and roll back — as a single named, reusable operation, exactly the capability the funds-transfer scenario needs.
- **Encapsulates multi-step operations safely.** Complex admin or batch operations (nightly cleanup, payroll runs, data migrations) can be written once as a procedure and re-invoked identically every time, rather than re-implemented as an ad hoc script.
- **Can commit incrementally on large batches.** A procedure processing many rows can commit periodically partway through (releasing locks and making progress durable) instead of holding one enormous transaction open for the entire batch — something no function is permitted to do at all.

## Disadvantages / Limitations

- **Cannot be used as an expression.** A procedure cannot appear inside a `SELECT` list, `WHERE` clause, or anywhere else an expression is expected — it is only invoked via its own standalone `CALL` statement, which limits how it composes with the rest of a query.
- **Shares every downside a function has.** Debuggability, testability, and vendor portability concerns (fully covered in Topic 1's Disadvantages) apply equally here — PL/pgSQL procedures are just as PostgreSQL-specific as PL/pgSQL functions.
- **Internal transaction control adds real complexity.** A procedure that commits partway through means a caller can no longer assume "the whole `CALL` either fully happened or fully didn't" the way they safely could with an ordinary statement — the procedure's author has to document and reason about exactly which parts are and aren't atomic with each other.

## Best Practices

- Only reach for a procedure once you actually need transaction control inside the logic itself — if a function's ordinary transactional inheritance from its caller is sufficient, prefer the function; it's simpler and composes into queries.
- Validate inputs and raise clear, specific exceptions (as `transfer_funds` does) *before* any data-changing statement runs, so a rejected call leaves nothing partially applied.
- Be explicit in documentation about exactly where a procedure commits internally — a caller relying on being able to roll back the *entire* call needs to know if the procedure has already committed part of its work.
- Keep a single procedure focused on one coherent operation (like one funds transfer) rather than folding many unrelated administrative tasks into one giant procedure that's hard to reason about or reuse in pieces.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Trying to call a procedure with `SELECT procedure_name(...)` | Procedures are invoked with `CALL`, never `SELECT` — `SELECT` is for evaluating expressions (including function calls), and a procedure is not an expression. |
| Expecting `COMMIT` inside a procedure to work when the `CALL` itself is wrapped in an explicit `BEGIN ... COMMIT` block | PostgreSQL does not support nested transaction control; a procedure's internal `COMMIT`/`ROLLBACK` is only legal when the procedure is the top-level unit managing the transaction, not when it's nested inside a caller-managed one. |
| Writing logic as a procedure purely out of habit, when it never actually needs to control a transaction | Adds unnecessary restrictions (can't be used inside a query as an expression) for no real benefit — if it doesn't need `COMMIT`/`ROLLBACK` internally, it should almost always be a function instead. |
| Assuming a procedure "returns" a value the way a function does | A procedure has no `RETURNS` clause and cannot be assigned like `x := procedure_name(...)`; the only way data comes back to the caller is through `INOUT` parameters. |

## Interview Questions

1. **Q: What is the single defining difference between a stored procedure and a user-defined function in PostgreSQL?**
   A: A procedure can issue `COMMIT` and `ROLLBACK` from inside its own body, managing its own transaction boundaries; a function always executes inside the transaction of whatever statement called it and cannot commit or roll back that transaction. This is also why procedures are invoked with a standalone `CALL` statement rather than embedded inside a query as an expression.

2. **Q: Why can't a procedure's internal `COMMIT` be used if the `CALL` is issued inside an explicit `BEGIN` block the caller already opened?**
   A: PostgreSQL does not support nested transactions — only one entity can be responsible for the current top-level transaction's boundary at a time. If the caller already opened a transaction explicitly, the procedure is nested inside it and is not permitted to commit or roll back that outer transaction itself.

3. **Q: How does a procedure communicate a computed value back to its caller, given that it has no `RETURNS` clause?**
   A: Through `INOUT` (or `OUT`) parameters — a parameter that is assigned a value inside the procedure body and is readable by the caller after the `CALL` completes, functioning as a side channel rather than a return value evaluated by the call expression itself.

4. **Q: You need a piece of logic that only reads data and computes a derived value, with no need to control a transaction. Should it be a function or a procedure?**
   A: A function — it should be usable directly inside a query as an expression, has no need for `COMMIT`/`ROLLBACK`, and a procedure would only add the restriction of needing a standalone `CALL` with no real benefit in return.

## Summary

- A stored procedure is created with `CREATE PROCEDURE`, has no `RETURNS` clause, and is invoked with `CALL`, never inside a query as an expression.
- Its defining capability, and the entire reason it exists as a separate object from a function, is that it can issue `COMMIT` and `ROLLBACK` from inside its own body — something a function is categorically forbidden from doing, because a function always runs inside the transaction of whatever called it.
- Internal transaction control is only legal when the procedure is called at the top level, with no explicit surrounding transaction already open — PostgreSQL does not support nested transaction management.
- `INOUT` parameters are how a procedure hands data back to its caller, since there is no `RETURN` value to evaluate the way a function has.
- Reach for a procedure specifically when a multi-step operation needs to manage its own transaction boundaries (validate, act, and commit or roll back as one named unit); otherwise, prefer a function.
