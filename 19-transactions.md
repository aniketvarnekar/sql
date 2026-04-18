# Transactions

A transaction is a sequence of SQL statements that are treated as a single unit of work. Either all statements in the transaction succeed and their effects are permanently saved, or the entire group of changes is discarded. Transactions are the mechanism that keeps databases consistent in the face of errors, concurrent access, and system failures.

## What a Transaction Is

Without transactions, a partial failure leaves the database in an inconsistent state. The classic example is a bank transfer: deducting $500 from account A and adding $500 to account B must both succeed or both fail. If the deduction succeeds but the server crashes before the addition, the $500 is simply gone. A transaction prevents this by grouping both operations into an all-or-nothing unit.

## ACID Properties

Transactions guarantee four fundamental properties, known as ACID:

**Atomicity** means the transaction is all-or-nothing. If any statement in the transaction fails, all preceding statements in the transaction are automatically undone. The database is left exactly as if the transaction never started.

**Consistency** means a transaction transforms the database from one valid state to another valid state. Every constraint, trigger, and rule must be satisfied after the transaction completes. A transaction that would violate a foreign key constraint, for example, is not allowed to commit.

**Isolation** means transactions are invisible to each other while they are in progress. A transaction reading data sees a snapshot of the database that is not affected by concurrent transactions that haven't committed yet. The exact isolation guarantee depends on the isolation level setting.

**Durability** means once a transaction commits, its changes are permanent — even if the server immediately crashes. MySQL achieves durability through the InnoDB redo log, which records all changes and can replay them after a crash.

## START TRANSACTION, COMMIT, ROLLBACK

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 500.00 WHERE id = 1;
UPDATE accounts SET balance = balance + 500.00 WHERE id = 2;

COMMIT;
```

`START TRANSACTION` begins the transaction. All subsequent statements are part of this transaction until `COMMIT` or `ROLLBACK`. `COMMIT` saves all changes permanently. `ROLLBACK` discards all changes made since `START TRANSACTION` and restores the database to its state before the transaction began:

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 500.00 WHERE id = 1;

-- Something went wrong:
ROLLBACK;
-- The deduction is undone; account 1 still has its original balance.
```

`BEGIN` is a synonym for `START TRANSACTION` in MySQL.

## SAVEPOINT and ROLLBACK TO SAVEPOINT

Savepoints allow partial rollbacks within a transaction. You can roll back to a named point without discarding the entire transaction:

```sql
START TRANSACTION;

INSERT INTO orders (customer_id, total) VALUES (1, 250.00);
SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 5, 2);
-- If this item insert fails for some reason:
ROLLBACK TO SAVEPOINT after_order;
-- Only the item insert is undone; the order row remains.

-- Now try inserting a different item:
INSERT INTO order_items (order_id, product_id, quantity) VALUES (LAST_INSERT_ID(), 7, 1);

COMMIT;
```

`RELEASE SAVEPOINT savepoint_name` discards a savepoint without rolling back to it.

## Autocommit Mode

MySQL runs in autocommit mode by default: every single SQL statement is automatically wrapped in its own transaction and committed immediately. This is why changes from a plain `INSERT` or `UPDATE` take effect immediately without an explicit `COMMIT`.

```sql
-- Check autocommit status:
SHOW VARIABLES LIKE 'autocommit';

-- Disable autocommit for the current session:
SET autocommit = 0;

-- Now statements accumulate until you COMMIT or ROLLBACK:
INSERT INTO products (name, price) VALUES ('Test', 9.99);
-- Not yet visible to other connections

COMMIT;
-- Now visible

-- Re-enable autocommit:
SET autocommit = 1;
```

`START TRANSACTION` implicitly disables autocommit for the duration of that transaction, regardless of the session's autocommit setting.

Note that DDL statements (`CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `TRUNCATE TABLE`) cause an implicit commit in MySQL — they cannot be rolled back. If you issue a DDL statement inside a transaction, the transaction is committed up to that point before the DDL executes.

## Transaction Isolation Levels

The SQL standard defines four isolation levels that control how transactions see each other's uncommitted changes. MySQL's InnoDB uses `REPEATABLE READ` as the default.

### READ UNCOMMITTED

The lowest isolation level. A transaction can read changes made by other transactions that haven't committed yet (called "dirty reads"). This is rarely appropriate because you might read data that is subsequently rolled back — data that never truly existed.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

### READ COMMITTED

A transaction only sees changes that have been committed at the moment each individual read happens. This prevents dirty reads but allows "non-repeatable reads": if you read the same row twice within a transaction, a committed change by another transaction between the two reads could cause them to return different values.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### REPEATABLE READ (MySQL Default)

A transaction sees a consistent snapshot of the database as of the moment the transaction began. If another transaction commits changes to rows you've already read, your transaction still sees the original values. This prevents dirty reads and non-repeatable reads. MySQL's implementation (using MVCC — Multi-Version Concurrency Control) also prevents most phantom reads. This is the InnoDB default and the right choice for most applications.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### SERIALIZABLE

The strongest isolation level. Transactions are executed as if they ran one at a time, in sequence. MySQL adds shared locks to all reads, meaning other transactions cannot modify any rows you've read until your transaction completes. This completely prevents dirty reads, non-repeatable reads, and phantom reads, but significantly reduces concurrency. Use this only when the application logic truly requires it.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

To change the isolation level for future transactions system-wide:

```sql
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

To check the current level:

```sql
SHOW VARIABLES LIKE 'transaction_isolation';
```

## Practice Problems

### Problem 1
Write a transaction that transfers funds between two bank accounts. If either update would result in a negative balance, roll back the entire transaction. Use a check after the deduction to verify sufficient funds.

**Schema used:**
```sql
CREATE TABLE accounts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    owner VARCHAR(200) NOT NULL,
    balance DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    CONSTRAINT chk_balance CHECK (balance >= 0)
);

INSERT INTO accounts (owner, balance) VALUES ('Alice', 1000.00), ('Bob', 200.00);
```

**Your query:**
```sql
-- Write the transaction to transfer $300 from Alice (id=1) to Bob (id=2).
```

**Solution:**
```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 300.00 WHERE id = 1;

-- Verify the balance is still non-negative after the deduction:
SET @new_balance = (SELECT balance FROM accounts WHERE id = 1);

IF @new_balance < 0 THEN
    ROLLBACK;
    SELECT 'Transfer failed: insufficient funds' AS result;
ELSE
    UPDATE accounts SET balance = balance + 300.00 WHERE id = 2;
    COMMIT;
    SELECT 'Transfer successful' AS result;
END IF;
```

**Explanation:**
The `CHECK (balance >= 0)` constraint (MySQL 8.0.16+) on the accounts table provides a database-level safety net — if the application code fails to catch a negative balance, the constraint prevents the commit. The explicit check after the deduction and `ROLLBACK` handles the case gracefully at the application level. Note that in application code, this logic would typically live in a stored procedure or in the application's transaction handling, not in raw SQL session variables. The `FOR UPDATE` locking shown in the stored procedures example is also important in concurrent systems to prevent race conditions.

### Problem 2
Demonstrate the difference between autocommit mode and explicit transactions by showing how changes are committed with and without `START TRANSACTION`.

**Schema used:**
```sql
CREATE TABLE test_autocommit (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    value VARCHAR(100) NOT NULL
);
```

**Your query:**
```sql
-- Show autocommit behavior vs explicit transaction behavior.
```

**Solution:**
```sql
-- === Autocommit mode (default) ===
-- Each statement auto-commits immediately:
INSERT INTO test_autocommit (value) VALUES ('Row 1 - auto committed');
-- Immediately visible to other connections. Cannot be rolled back.

-- === Explicit transaction ===
START TRANSACTION;
INSERT INTO test_autocommit (value) VALUES ('Row 2 - inside transaction');
-- NOT yet committed; other connections do not see this row.

-- At this point, if the server crashes, Row 2 is lost.
-- Another connection running SELECT would see only Row 1.

-- Option A: commit and make it permanent:
COMMIT;
-- Row 2 is now permanent.

-- Option B: discard the change:
-- ROLLBACK;
-- Row 2 is gone.

-- === Checking autocommit status ===
SHOW VARIABLES LIKE 'autocommit';
```

**Explanation:**
With autocommit enabled (the default), each statement is its own transaction — it commits immediately and its effects are visible to other connections right away. With `START TRANSACTION`, changes accumulate in an uncommitted state until `COMMIT` or `ROLLBACK`. During this period, the changes are invisible to other connections (under the `REPEATABLE READ` isolation level, other connections see the database as it was before your transaction started). This isolation is what allows multiple transactions to run concurrently without seeing each other's partial work.

### Problem 3
Use savepoints to demonstrate partial rollback: insert three records, create a savepoint after the first, then roll back to the savepoint after the second, and commit the third separately.

**Schema used:**
```sql
CREATE TABLE log_entries (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(200) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**Your query:**
```sql
-- Demonstrate partial rollback with savepoints.
```

**Solution:**
```sql
START TRANSACTION;

-- Insert first record:
INSERT INTO log_entries (message) VALUES ('Entry 1 - will be committed');

SAVEPOINT after_first;

-- Insert second record:
INSERT INTO log_entries (message) VALUES ('Entry 2 - will be rolled back');

-- Decide to undo the second insert:
ROLLBACK TO SAVEPOINT after_first;
-- Entry 2 is undone; Entry 1 is still in the transaction.

-- Insert a replacement third record:
INSERT INTO log_entries (message) VALUES ('Entry 3 - replacement for Entry 2');

COMMIT;

-- Verify final state:
SELECT * FROM log_entries;
-- Result: Entry 1 and Entry 3 are committed. Entry 2 does not exist.
```

**Explanation:**
`ROLLBACK TO SAVEPOINT after_first` only undoes changes made after the savepoint was created — it does not discard the entire transaction. Entry 1 is still in the pending transaction after the partial rollback. The savepoint itself is released when you roll back to it (you would need to set a new savepoint if you wanted another checkpoint). After the `COMMIT`, only Entry 1 and Entry 3 are in the table. Savepoints are useful in complex stored procedures where you want to attempt a risky operation but recover gracefully without abandoning all prior work in the transaction.
