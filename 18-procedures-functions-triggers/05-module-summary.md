# Module 18 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **User-Defined Functions** — `CREATE FUNCTION` in PL/pgSQL (and plain `SQL`), scalar and `RETURNS TABLE` set-returning forms, calling a function as an ordinary expression, and why volatility (`IMMUTABLE`/`STABLE`/`VOLATILE`) affects both correctness and planner optimization.
- [x] **Stored Procedures** — `CREATE PROCEDURE` and the `CALL` statement, the defining ability to `COMMIT`/`ROLLBACK` from inside the procedure body (which a function is forbidden from doing), and `INOUT` parameters as the substitute for a return value.
- [x] **Triggers** — `CREATE TRIGGER`, `BEFORE`/`AFTER` timing, `FOR EACH ROW`/`FOR EACH STATEMENT` granularity, the `NEW`/`OLD` record variables, and a worked audit-log trigger recording every change to a table automatically and unconditionally.
- [x] **When to Use Functions, Procedures, and Triggers** — a decision framework weighing all three against plain application-level logic, centered on the real trade-off between bypass-proof guarantees and testability/debuggability/portability.

## Practical Connections

- A payments platform recording an immutable, tamper-evident log of every balance change — regardless of which internal service, script, or support-tooling console touched the account — relies on exactly the kind of table-level trigger built in this module, because application-level logging alone can never guarantee it covers every code path that can reach the database.
- A nightly batch job that reconciles inventory across thousands of warehouse locations, committing progress in manageable chunks so a mid-run failure doesn't force the entire multi-hour job to restart from zero, is a realistic large-scale case for a stored procedure's internal transaction control.
- A reporting layer serving dozens of dashboards that all need the same derived metric (a customer risk score, a tenure band, a churn flag) computed identically everywhere is a realistic large-scale case for a reusable, composable user-defined function — one definition, called consistently, instead of the same formula subtly re-implemented dashboard by dashboard.
- A team's decision to keep frequently-changing pricing and promotion logic in application code rather than database triggers or functions — even though the database *could* technically enforce it — reflects the real organizational trade-off this module's final topic centers on: iteration speed and code-review normalcy versus bypass-proof enforcement.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Function vs. procedure | A function is called inside a query as an expression and always runs inside its caller's transaction; a procedure is called with a standalone `CALL` statement and can manage its own transaction with `COMMIT`/`ROLLBACK`. |
| `RETURNS TABLE` function vs. an ordinary scalar function | A scalar function returns one value per call, used as an expression; a `RETURNS TABLE` function returns an entire result set and is queried with `SELECT ... FROM function_name(...)`, treating it as a row source. |
| `NEW` vs. `OLD` in a trigger | `NEW` is the row's values after the change (available for `INSERT`/`UPDATE`); `OLD` is the row's values before the change (available for `UPDATE`/`DELETE`). Neither exists for the event where it wouldn't make sense. |
| `BEFORE` vs. `AFTER` trigger timing | A `BEFORE` trigger runs before the write and can modify or cancel it by what it returns; an `AFTER` trigger runs once the write is already final and can only react to it, not change it. |
| `FOR EACH ROW` vs. `FOR EACH STATEMENT` | Row-level fires once per affected row, with access to `NEW`/`OLD`; statement-level fires exactly once per statement regardless of row count, with no access to `NEW`/`OLD`. |
| A trigger vs. a `CHECK` constraint | Both can reject bad data unconditionally, but a `CHECK` constraint is limited to a single row judging its own column values; a trigger can reference other tables and prior row state, at the cost of being far less declarative and harder to reason about. |

## What's Next

This module gave you the tools to move logic into the database itself — as reusable expressions (functions), transaction-aware multi-step operations (procedures), and automatic reactions to data changes (triggers) — along with, just as importantly, a framework for judging when doing so is actually the right call. Module 19 (Security & Access Control) builds directly on top of this: once logic and data both live inside the database, controlling exactly *who* is allowed to call a function, execute a procedure, or even trigger a table write at all becomes essential, and `GRANT`/`REVOKE` — first previewed all the way back in Module 1's discussion of DCL — is where that control is actually implemented.
