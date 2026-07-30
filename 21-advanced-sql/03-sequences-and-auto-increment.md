# Sequences and Auto-Increment

## Learning Objectives

By the end of this section you should be able to:
- Create and use a standalone sequence with `CREATE SEQUENCE`
- Explain precisely how `SERIAL`/`BIGSERIAL` and `GENERATED ... AS IDENTITY` are implemented on top of a sequence
- Use `nextval`, `currval`, and `setval` correctly, and explain what each one does and doesn't guarantee
- Explain why gaps in sequence-generated values are normal and harmless, and why a naive `MAX(id) + 1` approach is not
- Reset a sequence deliberately (for example, after a bulk load) without breaking uniqueness

## Prerequisites

This topic depends on **Module 5 — Constraints & Keys**, specifically [Primary Keys](../05-constraints-and-keys/03-primary-keys.md), for the idea that a table needs a reliable, unique way to identify each row. It also assumes you've already used `SERIAL PRIMARY KEY` without a full explanation, in [Your First Query](../01-introduction/05-your-first-query.md) back in Module 1 — this topic is where that shorthand finally gets explained down to its actual mechanism.

## Motivation

You've written `id SERIAL PRIMARY KEY` in nearly every table definition in this course without asking exactly how PostgreSQL decides what number to assign each new row. It feels automatic, and it is — but "automatic" is doing real work under the hood, work that has to be correct even when dozens of transactions are trying to insert rows into the same table at the exact same instant. Understanding the mechanism (a **sequence**) explains not just how auto-increment works, but why its output looks the way it does — including something that alarms nearly every developer the first time they notice it: gaps.

## Problem Statement

Suppose, instead of `SERIAL`, you tried to invent auto-incrementing IDs yourself, in application code, with the seemingly obvious approach:

```sql
SELECT MAX(id) + 1 FROM orders;
-- use that value as the new row's id
```

This looks correct in isolation. It fails the moment two transactions try to insert a new order at nearly the same time:

```
 Transaction A                          Transaction B
 -------------                          -------------
 SELECT MAX(id)+1 FROM orders  → 101
                                          SELECT MAX(id)+1 FROM orders  → 101   (A hasn't committed yet)
 INSERT ... id = 101
 COMMIT
                                          INSERT ... id = 101   ← duplicate key violation, or worse,
                                                                   silently accepted if there's no
                                                                   PRIMARY KEY/UNIQUE constraint at all
```

Both transactions read the same `MAX(id)` before either committed, so both computed the same "next" value — a classic race condition. A `PRIMARY KEY` constraint (Module 5) would at least catch this as an error rather than corrupting data, but that still leaves one of the two inserts failing for a reason that has nothing to do with the actual data being inserted. What's needed is a way to hand out the "next" identifier that is **atomic** — guaranteed never to hand the same value to two concurrent callers, without either one having to wait for the other to finish its whole transaction first. That's exactly what a sequence is built to do.

## Concept

### What a Sequence Is

A **sequence** is a special, standalone database object whose entire job is to hand out a new, unique, ever-increasing (or decreasing) number every time it's asked, atomically, regardless of how many concurrent sessions are asking at once.

```sql
CREATE SEQUENCE order_id_seq START WITH 1 INCREMENT BY 1;

SELECT nextval('order_id_seq');
```

```
 nextval
---------
       1
(1 row)
```

```sql
SELECT nextval('order_id_seq');
```

```
 nextval
---------
       2
(1 row)
```

Each call to `nextval()` atomically advances the sequence and returns the new value — no two calls, no matter how many concurrent sessions are calling simultaneously, will ever receive the same number.

### `SERIAL`, `BIGSERIAL`, and `IDENTITY`: What's Really Happening

`SERIAL` is not a real, distinct column type in PostgreSQL — it's a shorthand notation that, at table-creation time, expands into three things: an `INTEGER` column, a sequence created specifically to back it, and a `DEFAULT` on the column that calls `nextval()` on that sequence. Writing:

```sql
CREATE TABLE orders (
    id     SERIAL PRIMARY KEY,
    detail TEXT
);
```

is exactly equivalent to PostgreSQL silently doing this on your behalf:

```sql
CREATE SEQUENCE orders_id_seq;

CREATE TABLE orders (
    id     INTEGER NOT NULL DEFAULT nextval('orders_id_seq') PRIMARY KEY,
    detail TEXT
);

ALTER SEQUENCE orders_id_seq OWNED BY orders.id;
```

The last line, `OWNED BY`, is why `DROP TABLE orders` also automatically drops `orders_id_seq` — PostgreSQL tracks the sequence as belonging to that specific column, so you never end up with an orphaned sequence left behind after its table is gone. `BIGSERIAL` is identical in spirit, just backed by a `BIGINT` column and sequence, for tables expected to exceed roughly two billion rows.

Since PostgreSQL 10, the SQL-standard-compliant alternative is `GENERATED ... AS IDENTITY`:

```sql
CREATE TABLE orders (
    id     INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    detail TEXT
);
```

This achieves the same underlying mechanism (a sequence, a default expression) but through standard SQL syntax rather than a PostgreSQL-specific pseudo-type, and gives you two variants with a meaningful difference:

- `GENERATED ALWAYS AS IDENTITY` — the column's value is always generated by the sequence; an explicit `INSERT` supplying its own value is rejected unless you add `OVERRIDING SYSTEM VALUE` to the statement.
- `GENERATED BY DEFAULT AS IDENTITY` — behaves like `SERIAL`: the sequence supplies a value only when you don't provide one yourself, and an explicit value in the `INSERT` is accepted without complaint.

For new schemas, `GENERATED ALWAYS AS IDENTITY` is generally the safer, more standards-aligned choice, since it makes accidentally overriding an auto-generated key an explicit, deliberate act rather than something that happens silently.

### `nextval`, `currval`, and `setval`

Three functions operate directly on a sequence:

| Function | What it does |
|---|---|
| `nextval('seq_name')` | Atomically advances the sequence and returns the new current value. This is what a `SERIAL`/`IDENTITY` column's default calls on every insert. |
| `currval('seq_name')` | Returns the value most recently obtained via `nextval()` **in the current session**. Raises an error if `nextval()` hasn't been called yet in this session. |
| `setval('seq_name', value)` | Manually sets the sequence's current value — used to realign a sequence, most commonly after a bulk load that inserted explicit id values. |

```sql
INSERT INTO orders (detail) VALUES ('First order') RETURNING id;
```

```
 id
----
  1
(1 row)
```

```sql
SELECT currval('orders_id_seq');
```

```
 currval
---------
       1
(1 row)
```

A very common real-world need for `setval()`: suppose you bulk-load historical orders with their original, explicit `id` values (perhaps migrating from another system):

```sql
INSERT INTO orders (id, detail) VALUES
    (500, 'Migrated order A'),
    (501, 'Migrated order B');
```

The sequence backing `orders.id` has no idea these explicit values were just used — it still thinks its next value is `2`. The very next ordinary insert would attempt `nextval()` → `2`, which is fine for now, but eventually the sequence will climb up to `500` on its own and collide with the manually-inserted rows. Fix this immediately after the bulk load:

```sql
SELECT setval('orders_id_seq', (SELECT MAX(id) FROM orders));
```

This tells the sequence "the highest value already in use is whatever `MAX(id)` currently is — hand out one past that next," preventing a future collision.

### Gaps Are Normal — and Expected

A sequence's guarantee is **uniqueness and increasing order**, not **contiguity**. Two ordinary, deliberate behaviors of a sequence produce gaps:

**1. A `nextval()` call is never rolled back**, even if the transaction that called it is:

```sql
BEGIN;
INSERT INTO orders (detail) VALUES ('Will be cancelled') RETURNING id;
-- id: 2
ROLLBACK;

INSERT INTO orders (detail) VALUES ('Actually kept') RETURNING id;
-- id: 3, not 2 — the sequence already handed out 2 and does not "give it back"
```

```
   detail
--------------
 First order
Actually kept
```

```
 id |    detail
----+---------------
  1 | First order
  3 | Actually kept
(2 rows)
```

`id = 2` is gone forever — no row ever has it, and no future row ever will. This is a deliberate design choice: making a sequence's `nextval()` participate in transaction rollback would mean two concurrent transactions could never safely call `nextval()` at the same time without waiting to see whether the other one commits or rolls back first, which would completely defeat the purpose of a sequence being fast and non-blocking.

**2. Sequences can cache multiple values per session** (`CACHE n` in `CREATE SEQUENCE`) for performance, which under concurrent, multi-session access can cause values to be consumed by different sessions in a way that isn't strictly issued in "arrival order" across sessions, though each individual session still always sees increasing values.

This ties directly back to **surrogate keys** (Module 5): a surrogate key's entire job is to uniquely and stably identify a row — it carries no business meaning, and nothing about it promises to reflect "how many rows have ever existed" or "the order things happened in real time." A gap is completely harmless *precisely because* nothing in a well-designed schema depends on ids being contiguous. The actual mistake — covered in Common Mistakes below — is treating an id column as if it *did* carry that meaning (e.g., displaying it to a customer as "you are order number 47" when the true count of successful orders is lower because of intervening rollbacks).

### Resetting a Sequence

```sql
ALTER SEQUENCE orders_id_seq RESTART WITH 1;
```

This is useful in a development or test environment when you want a clean slate after truncating a table, or immediately after a bulk restore where you've deliberately reloaded data with known, contiguous ids and want new rows to continue from the correct point. It is dangerous to do carelessly against live production data: if existing rows already occupy the ids the sequence is about to hand out again, the very next insert will collide with (or worse, if there's no uniqueness constraint, silently overwrite the intent behind) an existing row.

## Internal Working (Preview)

A sequence is implemented as its own small, independent database object — not a row inside any table, and not subject to the same transactional locking as ordinary table rows. `nextval()` takes a very short, lightweight internal lock, just long enough to atomically increment and read the sequence's counter, then immediately releases it — nowhere near the kind of lock a row update takes, and critically, **not tied to your transaction's outcome**:

```
 Session A: BEGIN; nextval('seq') → 101 ...............(still running)........... ROLLBACK
 Session B:              BEGIN; nextval('seq') → 102 ......... COMMIT
```

Both sessions get distinct values instantly, regardless of what either transaction eventually does — this is the entire reason sequences exist instead of `SELECT MAX(id) + 1`, which requires reading committed data and is only safe if no other transaction can possibly compute the same "next" value before yours commits — a guarantee `MAX(id)+1` cannot make without expensive, blocking locks that would seriously hurt concurrent insert performance.

## Real-World Analogy

Think of the "take a number" ticket dispenser at a deli counter or a government office. Every customer who pulls a ticket gets a strictly increasing number, guaranteed unique, even if fifty people are pulling tickets in the same second — the machine doesn't need to check with anyone else before handing out the next number. If a customer pulls ticket 42 and then leaves without being served, ticket 42 is simply never called — service continues at 43, 44, and onward, completely unaffected. Nobody audits the dispenser wondering "where did ticket 42 go" — the tickets exist purely to give each customer a distinct, orderly number, not to serve as an exact running count of how many customers the store has ever had.

## Why Sequences Were Designed This Way

A sequence's core promise — atomic, non-blocking, non-transactional value generation — is a direct, deliberate consequence of what a surrogate key (Module 5) is actually for: a stable, unique handle for a row, nothing more. If PostgreSQL instead guaranteed gapless, perfectly sequential ids, it would have to serialize every insert against every other concurrent insert (effectively making transactions wait on each other just to obtain an id), which would sacrifice the very thing that makes relational databases usable at scale under concurrency (Module 14). By explicitly decoupling "give me a unique number" from "commit my transaction," PostgreSQL lets id generation stay fast and lock-free, at the honest cost of occasional, harmless gaps.

## Advantages

- **Guaranteed uniqueness under concurrency, without contention** — many simultaneous transactions can call `nextval()` without ever blocking on each other or duplicating a value.
- **Extremely fast** — a sequence's internal lock is held for a tiny fraction of the time a full row-level or table-level lock would require.
- **Simple to use via `SERIAL`/`IDENTITY` shorthand** — you rarely need to interact with the underlying sequence directly at all.

## Disadvantages / Limitations

- **Values are not gapless** — rollbacks and session caching mean the sequence of ids in a table is increasing but not necessarily contiguous; anything that assumes "the ids are a perfect count of rows ever inserted" will be wrong.
- **A sequence's current value is not part of transactional rollback** — a rolled-back insert's consumed value is gone permanently, which is a deliberate trade-off (see Why Designed This Way), but surprises people expecting fully transactional behavior everywhere.
- **Exposing sequence-derived ids externally can leak information** — a customer-facing URL like `/orders/10482` reveals an approximate sense of how many orders exist and how fast that number is growing; when this matters, a non-sequential surrogate value (like a UUID, a trade-off discussed further in Module 5's natural-vs-surrogate key material) is sometimes preferred instead.

## Best Practices

- Prefer `GENERATED ALWAYS AS IDENTITY` over `SERIAL` for new PostgreSQL schemas — it's the modern, SQL-standard-compliant form and makes accidental manual overrides an explicit, deliberate act.
- Never treat an id column's value as meaningful business data (a row count, a "how many customers do we have" figure) — use `COUNT(*)` for that; ids exist only to identify, not to count or communicate order/volume.
- Always call `setval()` immediately after any bulk load that inserts explicit id values into a `SERIAL`/`IDENTITY` column, to prevent a future collision.
- Don't try to "fix" gaps in production id sequences — they are not data loss, not a bug, and not worth the risk of manually renumbering live foreign-keyed rows.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Computing the next id in application code as `MAX(id) + 1` | Not safe under concurrency — two simultaneous transactions can read the same `MAX(id)` before either commits, producing a duplicate or a race condition; this is exactly the problem sequences exist to solve. |
| Being alarmed that a table's ids "have gaps" | Gaps are a normal, harmless side effect of rolled-back transactions and sequence caching — not data loss or corruption. A surrogate key's job is uniqueness, not contiguity. |
| Bulk-loading rows with explicit id values and forgetting to call `setval()` afterward | The sequence still thinks its next value should continue from before the bulk load, and will eventually try to hand out an id that's already in use, causing a unique-constraint violation on a later insert. |
| Assuming `currval()` returns "the last id anyone inserted" | `currval()` only reflects a value obtained via `nextval()` in the *current session* — it has nothing to do with what other sessions have done, and calling it before this session has ever called `nextval()` raises an error. |

## Interview Questions

1. **Q: Why not just compute the next id as `SELECT MAX(id) + 1 FROM table`?**
   A: It's not safe under concurrency — two transactions running at the same time can both read the same `MAX(id)` before either commits, and both compute the same "next" value, producing a duplicate-key error or worse. A sequence's `nextval()` is atomic and lock-free, guaranteeing distinct values across any number of concurrent callers without that race.

2. **Q: How is `SERIAL` actually implemented under the hood in PostgreSQL?**
   A: `SERIAL` is shorthand, not a distinct type. Declaring a column `SERIAL` causes PostgreSQL to create a dedicated sequence, make the column an `INTEGER` with a `DEFAULT` of `nextval()` on that sequence, and mark the sequence as "owned by" that column so it's automatically dropped along with the table.

3. **Q: Why can a table's auto-generated ids have gaps, and why is that acceptable?**
   A: A `nextval()` call is never rolled back even if the transaction using it is, and sequences may cache multiple values per session for performance — both of which can leave some values never used by any row. This is acceptable because a surrogate key's only job is to uniquely and stably identify a row; nothing in a correctly designed schema should depend on ids being gapless.

4. **Q: What's the difference between `GENERATED ALWAYS AS IDENTITY` and `GENERATED BY DEFAULT AS IDENTITY`?**
   A: With `ALWAYS`, an explicit value supplied in an `INSERT` is rejected unless the statement includes `OVERRIDING SYSTEM VALUE`. With `BY DEFAULT`, an explicit value is accepted silently, and the sequence only fills in a value when none is provided — behaving like `SERIAL`.

## Summary

- A **sequence** is a standalone database object that atomically hands out unique, increasing values, safely even under heavy concurrency.
- `SERIAL`/`BIGSERIAL` are PostgreSQL shorthand that create and wire up a sequence automatically; `GENERATED ... AS IDENTITY` is the modern, SQL-standard-compliant equivalent.
- `nextval()` advances and returns the next value; `currval()` returns the current session's last obtained value; `setval()` manually realigns a sequence, most commonly after a bulk load with explicit ids.
- Gaps in sequence-generated values are normal and expected — caused by rolled-back transactions and session caching — and are harmless precisely because a surrogate key's job is uniqueness, not contiguity.
- Reset a sequence with `ALTER SEQUENCE ... RESTART WITH n`, carefully, to avoid future id collisions with existing rows.
- Next, Topic 4 looks at temporary tables — a completely different tool for staging data that exists only for the life of a session or transaction.
