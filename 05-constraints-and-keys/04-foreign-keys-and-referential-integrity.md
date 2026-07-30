# Foreign Keys and Referential Integrity

## Learning Objectives

By the end of this section you should be able to:
- Explain what a foreign key is and write the `REFERENCES` syntax to declare one
- Define "referential integrity" and describe the specific problem (orphaned rows) it prevents
- Choose the correct `ON DELETE`/`ON UPDATE` action (`CASCADE`, `SET NULL`, `RESTRICT`, `NO ACTION`) for a given real-world scenario
- Predict exactly what happens to dependent rows when a referenced row is deleted or its key is updated, under each action

## Prerequisites

- [Primary Keys](03-primary-keys.md) — a foreign key's entire job is to point at another table's primary key (or another uniquely-constrained column), so you need to already understand what a primary key guarantees before "referencing" one means anything.

## Motivation

Almost no real system is modeled with a single table. Customers place orders; orders contain products; employees belong to departments. The moment you split related facts across multiple tables — which Module 2's relational model and Module 15's normalization both push you toward, for excellent reasons — you introduce a new problem: what stops an `orders` row from claiming to belong to a `customer_id` that doesn't actually exist? Foreign keys are the mechanism that keeps cross-table relationships honest.

## Problem Statement

Take the `customers` and `orders` tables built up over the previous topics, but without any link enforced between them:

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    full_name   TEXT NOT NULL
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    total_amount NUMERIC(10, 2) NOT NULL
);

INSERT INTO customers (full_name) VALUES ('Asha Rao');   -- becomes customer_id 1
INSERT INTO orders (customer_id, total_amount) VALUES (999, 149.99);
```

This insert succeeds without complaint, even though no customer with `customer_id = 999` exists anywhere in the `customers` table:

```sql
SELECT o.order_id, o.customer_id, o.total_amount, c.full_name
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id;
```

```
 order_id | customer_id | total_amount | full_name
----------+-------------+--------------+-----------
        1 |         999 |       149.99 |
(1 row)
```

`full_name` comes back blank — there's no customer 999 to join against. This order is now **orphaned**: it references a customer that doesn't exist, and nothing in the database stopped that from happening. Any report, invoice, or shipping label built from this row is broken, and the damage compounds — the same problem can happen when a real customer *is* later deleted while orders still point at them, silently turning previously-valid orders into orphans. Without a mechanism enforcing that `orders.customer_id` must always match a real row in `customers`, the two tables can drift out of sync in ways that are difficult to detect until they cause visible harm.

## Concept

### What a Foreign Key Is

A **foreign key** is a column (or set of columns) in one table that must match a value that exists in another table's primary key (or other unique-constrained column) — or be `NULL`, if the column allows it. The table containing the foreign key is often called the "child" or "referencing" table; the table it points at is the "parent" or "referenced" table.

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    full_name   TEXT NOT NULL
);

CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers (customer_id),
    total_amount NUMERIC(10, 2) NOT NULL
);
```

`customer_id INTEGER NOT NULL REFERENCES customers (customer_id)` declares that every value stored in `orders.customer_id` must already exist as a `customer_id` in the `customers` table. Now the earlier problem is caught immediately:

```sql
INSERT INTO customers (full_name) VALUES ('Asha Rao');  -- customer_id 1
INSERT INTO orders (customer_id, total_amount) VALUES (999, 149.99);
```

```
ERROR:  insert or update on table "orders" violates foreign key constraint "orders_customer_id_fkey"
DETAIL:  Key (customer_id)=(999) is not present in table "customers".
```

A valid reference succeeds normally:

```sql
INSERT INTO orders (customer_id, total_amount) VALUES (1, 149.99);
```

```
INSERT 0 1
```

The same constraint can be written as a named, table-level clause instead of inline — functionally identical, but gives you a predictable name to reference later:

```sql
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    total_amount NUMERIC(10, 2) NOT NULL,
    CONSTRAINT orders_customer_id_fkey FOREIGN KEY (customer_id) REFERENCES customers (customer_id)
);
```

A foreign key can also be added to a table that already exists, using `ALTER TABLE`:

```sql
ALTER TABLE orders
    ADD CONSTRAINT orders_customer_id_fkey
    FOREIGN KEY (customer_id) REFERENCES customers (customer_id);
```

Exactly like adding `NOT NULL` to a populated table (Topic 1), PostgreSQL checks every existing row in `orders` before accepting this constraint, and refuses if even one row already references a nonexistent customer.

### What Referential Integrity Means

**Referential integrity** is the guarantee that every foreign key value in a table always matches an existing row in the table it references — there are never any "orphaned" rows pointing at something that doesn't exist. A foreign key constraint is simply the mechanism PostgreSQL uses to enforce referential integrity automatically, on every `INSERT` and `UPDATE`, without any application code having to check it manually.

Referential integrity is checked in **both directions** once a relationship exists:
- **On the child table (`orders`)**: you cannot insert or update a row to reference a `customer_id` that doesn't exist in `customers`.
- **On the parent table (`customers`)**: by default, you cannot delete (or change the primary key value of) a customer row that is still referenced by any `orders` row — doing so would immediately orphan those orders. This second direction is exactly what `ON DELETE` and `ON UPDATE` actions, covered next, let you control.

```sql
DELETE FROM customers WHERE customer_id = 1;
```

```
ERROR:  update or delete on table "customers" violates foreign key constraint "orders_customer_id_fkey" on table "orders"
DETAIL:  Key (customer_id)=(1) is still referenced from table "orders".
```

PostgreSQL refuses this delete by default — this default behavior is exactly `NO ACTION`, explained below alongside the other three choices.

### `ON DELETE` and `ON UPDATE` Actions

When you declare a foreign key, you can specify what should happen to referencing (child) rows when the referenced (parent) row is deleted, or when its key value is updated. PostgreSQL supports four actions, specified separately for `ON DELETE` and `ON UPDATE`:

| Action | On the parent row's deletion/update... | Typical use case |
|---|---|---|
| `NO ACTION` (default) | The operation is rejected if any child row still references the parent — checked at the end of the statement (or transaction, if deferred). | The safe default: force an explicit decision before anything is removed. |
| `RESTRICT` | The operation is rejected if any child row still references the parent — checked immediately, with no option to defer. | Same intent as `NO ACTION`, slightly stricter timing; the practical difference rarely matters outside advanced transaction patterns. |
| `CASCADE` | The child rows are automatically deleted (for `ON DELETE`) or automatically updated to the new key value (for `ON UPDATE`) along with the parent. | The child rows have no independent meaning without the parent — deleting the parent should take them with it. |
| `SET NULL` | The child rows' foreign key column is automatically set to `NULL`. | The child row still makes sense without a parent — the relationship becomes "unassigned" rather than the row itself disappearing. |

Each is best understood through a concrete scenario using the running schema.

#### `RESTRICT` / `NO ACTION` — Protect Orders From an Accidental Customer Deletion

```sql
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers (customer_id) ON DELETE RESTRICT,
    total_amount NUMERIC(10, 2) NOT NULL
);
```

A customer with existing order history should never be silently deletable — that history (for accounting, tax, and dispute-resolution reasons) needs to survive. With `ON DELETE RESTRICT` (or the default `NO ACTION`), attempting to delete customer 1 while their order exists fails outright, exactly as shown above — forcing whoever is performing the deletion to make an explicit decision (e.g., archive the customer instead of deleting them, or first decide what should happen to their orders).

#### `CASCADE` — Remove an Order's Line Items When the Order Itself Is Removed

```sql
CREATE TABLE order_items (
    order_id   INTEGER NOT NULL REFERENCES orders (order_id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL,
    quantity   INTEGER NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

An `order_items` row (Topic 3's example: "2 units of product 100 on order 1") has no independent meaning once order 1 itself is deleted — there is no legitimate scenario where you'd want a line item to survive its own order's deletion. With `ON DELETE CASCADE`:

```sql
DELETE FROM orders WHERE order_id = 1;
```

```
DELETE 1
```

This single statement automatically removes order 1 *and* every `order_items` row referencing it, with no separate `DELETE FROM order_items` needed and no risk of forgetting it. `CASCADE` is powerful, and precisely because of that, it's the action most likely to cause real damage if applied somewhere the child rows actually *do* have independent value worth preserving — see Common Mistakes below.

#### `SET NULL` — Un-assign an Employee From Orders Without Deleting the Orders

Suppose `orders` also tracks which employee processed each order, and employees sometimes leave the company:

```sql
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    full_name   TEXT NOT NULL
);

CREATE TABLE orders (
    order_id            SERIAL PRIMARY KEY,
    customer_id         INTEGER NOT NULL REFERENCES customers (customer_id) ON DELETE RESTRICT,
    handled_by_employee INTEGER REFERENCES employees (employee_id) ON DELETE SET NULL,
    total_amount         NUMERIC(10, 2) NOT NULL
);
```

Note `handled_by_employee` has no `NOT NULL` here — it must be nullable for `SET NULL` to be usable at all. When an employee leaves and their row is deleted:

```sql
DELETE FROM employees WHERE employee_id = 7;
```

```
DELETE 1
```

Every order they had handled still exists (orders are permanent business records — they must not disappear just because staffing changed), but `handled_by_employee` on those rows is automatically set to `NULL`, correctly reflecting "no longer attributable to a specific current employee" instead of pointing at a deleted, nonexistent one.

#### `CASCADE` on `ON UPDATE` — Propagating a Changed Key

`ON UPDATE CASCADE` follows the same idea, but for changes to the parent's key value rather than deletion — if a parent row's primary key value is ever changed, every referencing foreign key is automatically updated to match:

```sql
CREATE TABLE orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers (customer_id) ON UPDATE CASCADE ON DELETE RESTRICT,
    total_amount NUMERIC(10, 2) NOT NULL
);

UPDATE customers SET customer_id = 5001 WHERE customer_id = 1;
```

Every `orders` row with `customer_id = 1` is automatically updated to `customer_id = 5001` as part of the same operation. In practice, `ON UPDATE CASCADE` matters far less than `ON DELETE` actions when the parent's primary key is a surrogate `SERIAL`/`GENERATED ALWAYS AS IDENTITY` value (Topic 6) that's never intentionally changed after creation — it becomes far more relevant when the primary key is a natural key that could plausibly be edited, such as a product's SKU.

### Choosing the Right Action

| Question to ask | Points toward |
|---|---|
| Does the child row have any meaning at all without its parent? | No → `CASCADE` (delete the child along with the parent). |
| Should the child row survive, just with the relationship cleared? | Yes → `SET NULL` (requires the foreign key column to be nullable). |
| Should deleting the parent be actively prevented while children exist? | Yes → `RESTRICT` / `NO ACTION` (force an explicit decision before any deletion). |

## Internal Working (Preview)

```
DELETE FROM customers WHERE customer_id = 1;
        │
        ▼
For every table with a FOREIGN KEY referencing customers(customer_id):
        │
        ├─ ON DELETE RESTRICT / NO ACTION → any matching child row? → yes → ABORT the DELETE entirely
        │
        ├─ ON DELETE CASCADE  → delete every matching child row first, then the parent row
        │
        └─ ON DELETE SET NULL → set the foreign key column to NULL on every matching child row, then delete the parent row
```

PostgreSQL enforces this using the same B-tree index mechanism that backs `UNIQUE` and `PRIMARY KEY` constraints (Topics 2–3) — checking whether a value exists in the referenced table's primary key index is an efficient index lookup, not a full table scan, which is why foreign key checks remain fast even as both tables grow large.

## Real-World Analogy

Think of a company's org chart and its office directory. An employee's directory entry (`orders`, the child) lists a department (`customers`, the parent) they belong to — a `RESTRICT` policy is like HR refusing to dissolve a department while employees are still assigned to it ("reassign your people first"). A `CASCADE` policy is like shredding an individual visitor's day-pass log entries the moment the *visit itself* is deleted from the system — the log entries have no purpose independent of the visit. A `SET NULL` policy is like what happens to a company car's "assigned driver" field when that driver leaves the company — the car isn't deleted, it just becomes "unassigned" until someone new claims it.

## Why Foreign Keys Were Designed This Way

Splitting data across multiple related tables (rather than one giant flattened table) is central to the relational model and to normalization (Module 15) — it avoids repeating a customer's full details on every single one of their orders. But splitting data introduces exactly the risk shown in the Problem Statement: nothing inherently keeps the split tables consistent with each other. Foreign keys exist to make the *relationship itself* a first-class, enforced fact, not an assumption baked into application code that every developer must remember to respect. The four `ON DELETE`/`ON UPDATE` actions exist because "what should happen to dependents when a parent disappears" is not one universal answer — it is a genuine business decision that differs by relationship (an order's line items should die with the order; an order itself should never die just because the handling employee left) — so the database gives you an explicit, declarative way to state that decision once, at the schema level, rather than re-implementing it inconsistently in every place a delete might occur.

## Advantages

- **Prevents orphaned rows automatically** — no application code path can ever leave a child row pointing at a nonexistent parent, regardless of how many different places in a system perform deletes or inserts.
- **Makes relationships explicit and discoverable** — reading a table's `CREATE TABLE` (or `\d tablename` in `psql`) immediately shows every table it depends on and every table that depends on it.
- **Centralizes cascade/cleanup logic** — `ON DELETE CASCADE` guarantees dependent cleanup happens every time, everywhere, rather than relying on every deletion code path remembering to also clean up related rows.
- **Enables the query planner and other tooling to reason about relationships** — joins (Module 10), for instance, are far easier to write and reason about correctly when the relationship between two tables is a known, enforced fact rather than an informal convention.

## Disadvantages / Limitations

- **`CASCADE` can delete far more data than intended if misapplied** — a chain of cascading foreign keys can turn a single `DELETE` into a much larger, harder-to-predict deletion; it must be used only where child rows genuinely have no independent value.
- **Foreign keys add write overhead** — every `INSERT`/`UPDATE` on a child table, and every `DELETE`/`UPDATE` on a parent table, must check the relevant index, which is fast but not free.
- **Bulk data loading order matters** — you generally must insert parent rows before child rows (and delete child rows before parent rows, unless `CASCADE` handles it), which complicates certain bulk-loading or data-migration scripts that would otherwise insert in any order.
- **Cross-database references aren't possible** — a foreign key can only reference a table within the same database; enforcing consistency across separate databases requires application-level or other mechanisms entirely outside this constraint.

## Best Practices

- Default to `RESTRICT`/`NO ACTION` for any relationship where deleting the parent should be a deliberate, examined decision (customers with billing history, for instance) — it's always safer to be forced into an explicit choice than to silently lose or orphan data.
- Reserve `CASCADE` for true "belongs entirely to" relationships — a genuinely dependent, meaningless-without-its-parent child, like `order_items` belonging to `orders`.
- Use `SET NULL` specifically when the relationship is optional and the child record's own existence must outlive the parent — remember the foreign key column must be nullable (not `NOT NULL`) for this to even be legal.
- Always specify `ON DELETE` explicitly rather than relying silently on the default (`NO ACTION`) — writing it out makes the intended behavior visible directly in the schema, rather than requiring a reader to know PostgreSQL's default.
- Index foreign key columns on the child table (PostgreSQL does not do this automatically, unlike the primary key side) — Module 13 covers why this matters for both lookup and cascade-delete performance.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `ON DELETE CASCADE` "just in case," without checking whether child rows have independent value | This can silently delete far more data than intended — e.g., cascading from `customers` all the way through `orders` could delete a customer's entire purchase history the moment their account is removed, which is rarely the actual intent. |
| Assuming a foreign key automatically creates an index on the child table's referencing column | Unlike a primary key, PostgreSQL does not automatically index a plain foreign key column — you typically need to create that index explicitly for good join and cascade performance (Module 13). |
| Forgetting that the referenced column must itself be `UNIQUE` or a `PRIMARY KEY` | A foreign key cannot reference an arbitrary, non-unique column — PostgreSQL requires the referenced column(s) to already be constrained unique, since otherwise "the matching row" would be ambiguous. |
| Trying to delete a parent row and being surprised by a foreign key error, without realizing `RESTRICT`/`NO ACTION` is the default | Every foreign key defaults to blocking the parent's deletion while children exist unless `CASCADE` or `SET NULL` was explicitly specified — this default is a safety feature, not a bug. |

## Interview Questions

1. **Q: What is referential integrity, and what specific problem does it prevent?**
   A: Referential integrity is the guarantee that every foreign key value in a table matches an existing row in the table it references. It prevents "orphaned" rows — child rows that point at a parent that no longer exists (or never existed), which would otherwise silently corrupt reports, joins, and any logic relying on that relationship being valid.

2. **Q: A `departments` table is referenced by an `employees` table via a foreign key. An HR administrator tries to delete a department that still has employees assigned to it. What happens under `ON DELETE RESTRICT` versus `ON DELETE CASCADE`, and which is more appropriate here?**
   A: Under `RESTRICT`, the deletion fails immediately with a foreign key violation, forcing the administrator to first reassign or remove the employees. Under `CASCADE`, every employee row referencing that department would be deleted along with it. `RESTRICT` is almost always more appropriate here — deleting a department should not silently delete the employee records themselves; employees should be reassigned, not destroyed, when a department is dissolved.

3. **Q: When would you use `ON DELETE SET NULL` instead of `ON DELETE CASCADE`?**
   A: When the child row has meaning and should persist independently of the parent, but the specific relationship becomes invalid once the parent is gone — for example, an order's `handled_by_employee` column should become `NULL` if that employee leaves the company, rather than deleting the entire order, since the order itself remains a valid, necessary business record.

4. **Q: Can a foreign key reference any column in the parent table, or only certain ones?**
   A: Only a column (or set of columns) that is already constrained `UNIQUE` or is the table's `PRIMARY KEY`. This is required because the database must be able to guarantee that any given foreign key value matches at most one row in the parent table — referencing a non-unique column would make "the" matching parent row ambiguous.

## Summary

- A **foreign key** is a column that must match an existing value in another table's primary key (or unique column), or be `NULL` if allowed — declared with `REFERENCES`.
- **Referential integrity** is the guarantee, enforced automatically by foreign keys, that no row ever references a nonexistent parent row (no orphaned rows).
- `ON DELETE`/`ON UPDATE` actions decide what happens to child rows when a parent row is deleted or its key changes: `RESTRICT`/`NO ACTION` (block the operation), `CASCADE` (propagate the delete/update to children), `SET NULL` (clear the relationship but keep the child row).
- Choosing the right action is a business decision, not a syntax choice — ask whether the child row has independent meaning without its parent.
- Foreign keys are checked via the same fast index mechanism as `UNIQUE`/`PRIMARY KEY`, but unlike primary keys, PostgreSQL does not automatically index the referencing (child-side) column — that's a manual best practice worth remembering (Module 13).
