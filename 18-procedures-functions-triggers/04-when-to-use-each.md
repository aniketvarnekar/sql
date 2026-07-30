# When to Use Functions, Procedures, and Triggers

## Learning Objectives

By the end of this section you should be able to:
- Compare functions, procedures, triggers, and plain application-level logic against a consistent set of decision criteria
- Articulate the real trade-off between centralizing logic in the database versus keeping it in application code
- Explain why debuggability and testability concerns weigh more heavily against these tools than raw capability might suggest
- Recognize the specific signs that a function, procedure, or trigger has been reached for when it shouldn't have been

## Prerequisites

- [User-Defined Functions](01-user-defined-functions.md), [Stored Procedures](02-stored-procedures.md), and [Triggers](03-triggers.md) — this topic assumes you already know the mechanics of all three and is entirely about judgment, not new syntax.

## Motivation

By this point in the module you can write a function, a procedure, and a trigger. That's the easy part. The genuinely hard, and far more consequential, question is *when you should*. All three tools share one seductive property: they run right next to the data, invisibly, without any application code needing to know they exist. That's exactly what makes them powerful for the right problem — and exactly what makes them a liability when reached for out of habit rather than necessity. This topic is a deliberately honest look at that trade-off, because "how do I write one" and "should I write one at all" are two completely different questions, and most real-world regret with these tools comes from never asking the second one.

## Problem Statement

Imagine a team building an order-processing system faces four different small decisions in the same week:

1. A report needs the same "customer lifetime value" calculation in six different places.
2. Placing an order needs to check inventory, decrement stock, and record the order as one all-or-nothing operation.
3. Whenever an order's status changes, a `last_status_change` timestamp column elsewhere needs updating.
4. A discount calculation depends on a customer's order history, loyalty tier, and today's date, and changes fairly often as marketing runs new promotions.

All four *could* be implemented as a function, a procedure, or a trigger. Only some of them genuinely should be. Getting this judgment wrong in either direction is costly: solving problem 4 with a trigger buries a frequently-changing business rule somewhere no developer reading the application code will ever think to look; solving problem 2 *without* a procedure or an equivalent application-level transaction risks leaving inventory and order records inconsistent if something fails halfway through. This topic builds the framework for telling these apart.

## Concept

### The Four Options, Side by Side

| | **Function** | **Procedure** | **Trigger** | **Application-level logic** |
|---|---|---|---|---|
| How it's invoked | Called explicitly, inside a query, as an expression | Called explicitly, via `CALL`, as its own statement | Fires automatically on `INSERT`/`UPDATE`/`DELETE` — never called directly | Called explicitly, from whatever general-purpose code your application is written in |
| Can manage its own transaction | No — runs inside the caller's transaction | Yes — can `COMMIT`/`ROLLBACK` internally | No — runs inside the transaction of the statement that triggered it | Yes — fully in control of transaction boundaries via the statements it issues |
| Visible at the call site | Yes — a query calling it names it explicitly | Yes — a `CALL` names it explicitly | **No** — an ordinary-looking `INSERT`/`UPDATE`/`DELETE` gives no hint a trigger exists | Yes — it's just more code in the same codebase |
| Testable in isolation | Hard — needs a live database and often bespoke tooling | Hard — same, plus transactional side effects to account for | Hardest — must trigger it indirectly via a write, and verify a side effect elsewhere | Easiest — ordinary code, testable with whatever tools the application's language already provides |
| Portable across database vendors | No — procedural syntax is vendor-specific (Module 22) | No — same | No — same, plus trigger syntax itself varies more than function syntax across vendors | Yes — general-purpose application code isn't tied to a specific database vendor's procedural dialect |
| Bypassable by a direct database write | No — runs regardless of caller | No — runs regardless of caller | No — runs regardless of caller | **Yes** — anything that writes to the database by a different path skips this logic entirely |

That last row is the single most important line in the table, and it's the crux of the entire trade-off this topic is about.

### The Real Trade-off: Centralizing Logic vs. Keeping It Outside the Database

Every row above ultimately traces back to one fundamental tension:

- **Logic inside the database (functions, procedures, triggers)** is guaranteed to run no matter what touches the data — a genuine, structural guarantee, not a convention someone has to remember to follow. This is precisely why Topic 3's audit-logging trigger works: it cannot be bypassed by a stray manual `UPDATE`.
- **Logic outside the database (application code)** is easy to read, test, debug, version, and review using the same tools and workflows as the rest of a system — but it is only as reliable as every single code path that's supposed to call it. A second application, a data migration script, or someone's one-off admin fix can trivially bypass it.

Neither side of this trade-off is free. The temptation is to see the guarantee side (bypass-proof) and reach for a trigger every time, because it feels like the "more correct" choice. But that guarantee is bought at a real, recurring cost: logic that's invisible from the application code, harder to test without a live database, harder to version and code-review in the same way as everything else, and locked to whatever database vendor you're running. The right choice depends entirely on which side of that cost genuinely matters more for the specific problem at hand — not on which option feels more sophisticated.

### A Decision Framework

Work through these questions, roughly in order:

1. **Does this need to be guaranteed no matter what touches the data — including paths outside your main application?**
   If yes (compliance audit logs, data integrity that must hold even against direct database access), a trigger is a legitimate candidate. If no — if you control every single code path that could perform this action — the guarantee a trigger buys you isn't actually needed, and its costs (hidden side effects, harder testing) are paid for nothing.

2. **Can the rule already be expressed as a constraint (Module 5)?**
   If a rule is a single-row, single-table check (`salary >= 0`, `email` must be unique), a `CHECK`, `NOT NULL`, `UNIQUE`, or foreign key constraint expresses it more simply, more declaratively, and without any procedural code at all. Reaching for a trigger here — as Topic 3's `reject_negative_salary` example deliberately illustrated — solves an already-solved problem with a heavier, more opaque tool.

3. **Does this operation need to manage its own transaction — validate, act, and commit or roll back as one unit, independent of its caller?**
   If yes, that's a procedure's defining reason to exist (Topic 2). If the logic is fine inheriting whatever transaction its caller is already running, it doesn't need to be a procedure — a plain function, or even application code issuing ordinary statements, is sufficient.

4. **Is this a piece of reusable, composable calculation that multiple queries need, with no side effects?**
   If yes, a function is a strong fit — it's callable as an expression, and its cost (harder to test/version than application code) is modest for something this self-contained.

5. **Does this change often, need business stakeholders' input, or benefit from the same code review, testing, and deployment pipeline as the rest of the application?**
   If yes — as in scenario 4 from the Problem Statement, a discount rule that marketing revisits frequently — application-level logic is usually the better home, even though a function *could* technically implement it. Frequently changing business logic buried in the database is exactly the kind of thing that becomes hard to find, hard to review, and hard to safely modify under time pressure.

Applying this to the Problem Statement's four scenarios:

| Scenario | Best fit | Why |
|---|---|---|
| 1. Lifetime value calculation reused across six reports | Function | Reusable, composable, no side effects, no transaction control needed — a textbook function use case. |
| 2. Place an order: check stock, decrement it, record the order, all-or-nothing | Procedure (or equivalent application-level transaction) | Needs multi-step transactional control; whether it belongs in the database as a procedure or in application code wrapped in `BEGIN`/`COMMIT` (Module 14) depends on whether other non-application code paths also place orders — if only the application ever does, application-level transaction handling is simpler and more testable. |
| 3. Maintain `last_status_change` whenever status changes | Trigger | This is exactly the kind of rule that must hold regardless of which code path changes status — a small, narrowly-scoped `AFTER UPDATE` trigger is a legitimate, low-risk fit. |
| 4. Discount logic that changes with marketing promotions | Application-level logic | Changes frequently, benefits from ordinary code review and testing, and has no need to be bypass-proof against direct database access — burying it in a trigger or function would make it slower and riskier to iterate on. |

## Internal Working (Preview)

The decision in this topic isn't really about how the database executes each option internally — Topics 1 through 3 already covered that — it's about *where code physically lives* and what that implies operationally:

```
      Application-level logic                Database-side logic
   (general-purpose language, own       (functions, procedures, triggers)
     process, own deploy pipeline)
              │                                       │
              ▼                                       ▼
   Runs in the application's own          Runs inside the database server's
   process — scales/deploys with          own process — every connection to
   the application; testable with         the database is subject to it,
   the application's normal test          including connections your
   tooling; visible in the same           application's test suite doesn't
   code review as everything else.        control or even know about.
```

This is why the trade-off is ultimately organizational as much as technical: database-side logic runs in a place that's shared by every consumer of the database (every application, every script, every human with a `psql` prompt), for better (the guarantee) and worse (it's outside the normal code review and testing loop most teams already have for application code).

## Real-World Analogy

Think of a company's rulebook. Some rules genuinely need to be enforced by the building itself — a door that physically won't open without a badge, no matter who's trying it (the trigger/constraint guarantee: bypass-proof, but a locksmith has to be called to change it, and most employees don't even know that door has special hardware). Other rules are better as a clearly written, easily updated policy that new hires read during onboarding and that management revises whenever the business needs change (application-level logic: easy to read, discuss, and change, but only effective if people actually follow it). A company that hard-wires every single policy into physical building mechanisms — including ones that change every quarter — ends up calling a locksmith constantly and makes it needlessly hard for anyone to even find out what the current rules are, let alone change them safely.

## Why This Framework Matters

This entire module's tools were designed to fill a real gap — sometimes a rule genuinely must be enforced at the data layer, and Modules 5 and 6 already showed you that constraints and the statements they guard exist precisely because centralizing correctness there is more reliable than trusting every caller to enforce it themselves. But that same centralizing instinct, taken too far, recreates the opposite problem: instead of "every caller has to remember to enforce this," you get "every developer has to remember this hidden logic even exists." Good schema and system design isn't "always centralize in the database" or "never put any logic in the database" — it's recognizing, case by case, which side of that specific trade-off the *particular* rule in front of you actually needs, using the framework above rather than a blanket rule in either direction.

## Advantages (of Having a Framework Like This at All)

- **Prevents both failure modes.** Without a deliberate framework, teams tend to drift toward one extreme or the other — either avoiding these tools entirely (missing out on genuine bypass-proof guarantees where they matter) or overusing them (burying frequently-changing logic where it's hard to find and review).
- **Makes trade-offs explicit and discussable.** "Should this be a trigger?" becomes a concrete question with concrete criteria, rather than a matter of individual taste or habit during a design discussion or code review.
- **Scales with team size.** The bigger a team and its surrounding tooling (deployment pipelines, code review norms, testing infrastructure) get, the more expensive it becomes to have logic living somewhere those systems don't reach — this framework surfaces that cost before it's paid.

## Disadvantages / Limitations

- **No framework replaces judgment entirely.** Real schemas have genuinely ambiguous cases (scenario 2 in the Problem Statement is a good example) where reasonable, well-informed engineers can land on different answers — the framework narrows the decision, it doesn't always fully resolve it.
- **Organizational context outweighs the technical framework sometimes.** A team with strong database-migration tooling, database-level code review, and database-specific testing infrastructure can reasonably tolerate more database-side logic than a team without any of that — the "right" answer depends partly on factors outside this topic entirely.

## Best Practices

- Default to the *simplest* tool that satisfies the actual requirement: a constraint over a trigger, a function over a procedure, application code over any of the three, unless a concrete requirement (bypass-proof enforcement, internal transaction control, composability inside a query) forces a heavier tool.
- Treat "does this need to be bypass-proof against non-application access to the database" as the single highest-leverage question — it's the one criterion application-level logic structurally cannot satisfy, no matter how well-written it is.
- Keep an explicit, discoverable inventory of every function, procedure, and trigger in a schema (in schema documentation, migration scripts, or a dedicated catalog) — the whole risk of these tools is that they're invisible from application code, so compensate deliberately by making them maximally visible everywhere else.
- Revisit the decision when circumstances change — a rule that started as a rarely-touched compliance requirement (good trigger fit) can evolve into something the business wants to iterate on constantly (better as application logic); don't treat the original choice as permanent just because it was correct once.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Reaching for a trigger as the default way to enforce any rule | Most single-row, single-table rules are better and more simply expressed as a constraint (Module 5); a trigger should be reserved for logic a constraint genuinely cannot express. |
| Putting frequently-changing business logic (promotions, pricing rules) inside a database function or trigger | This logic usually benefits most from the same code review, testing, and deployment practices as the rest of an application — burying it in the database makes it slower and riskier to safely iterate on. |
| Assuming a stored procedure is always "more powerful" so it's always the better choice over a function | A procedure's only genuine advantage over a function is internal transaction control; if that's not needed, a procedure only adds the restriction of not being usable inside a query as an expression. |
| Never using triggers or procedures at all, on principle | Some guarantees (audit trails that must hold even against direct database access, multi-step operations needing internal transaction control) are genuinely and structurally better served by these tools — avoiding them entirely on principle can leave real gaps in data integrity. |

## Interview Questions

1. **Q: What is the fundamental trade-off between putting business logic inside the database versus in application code?**
   A: Database-side logic (functions, procedures, triggers) is guaranteed to run no matter what touches the data, since it's structurally impossible to bypass — but it's harder to test, debug, and review with normal application tooling, and it's tied to a specific database vendor's procedural syntax. Application-side logic is easy to test, review, and iterate on using standard tooling, but is only effective if every code path that could act on the data actually goes through it — it can be silently bypassed by a different code path or direct database access.

2. **Q: Why are triggers specifically called out as easy to overuse, more so than functions or procedures?**
   A: Because triggers are invisible at the call site by design — an ordinary-looking `INSERT`/`UPDATE`/`DELETE` gives no indication a trigger will fire, unlike a function or procedure call, which at least names what it's invoking. This makes triggers the option most likely to produce hidden side effects that a developer debugging unrelated code has no reason to suspect exist.

3. **Q: Give a concrete example of a rule that should be a trigger, and one that shouldn't, and explain the difference.**
   A: A compliance audit trail that must record every change to a sensitive column, even changes made outside the main application (e.g., a manual data fix), is a good trigger fit — the guarantee that it can't be bypassed is the entire point. A discount percentage tied to a marketing promotion that changes weekly is a poor trigger fit — it needs frequent, reviewed, testable changes, which a trigger buried in the schema makes harder, not easier, with no real bypass-proofing benefit since only the application ever needs to apply it.

4. **Q: A colleague argues "we should implement this validation as a trigger so it's enforced everywhere." What follow-up question should you ask before agreeing?**
   A: Whether the rule can already be expressed as a constraint (`CHECK`, `NOT NULL`, foreign key) — if it's a single-row, single-table rule, a constraint achieves the same "enforced everywhere, unconditionally" guarantee more simply and declaratively, without introducing procedural code or the debugging/testing costs that come with a trigger.

## Summary

- Functions, procedures, triggers, and application-level logic each occupy a distinct point on the same trade-off: how bypass-proof the logic is versus how easy it is to test, debug, version, and review with ordinary tooling.
- Database-side logic (all three tools from this module) is uniquely bypass-proof — it runs regardless of what touches the data — but that guarantee is paid for with reduced testability, reduced visibility to developers, and vendor-specific syntax.
- Constraints (Module 5) should be preferred over triggers for any rule they can already express; procedures should be preferred over functions only when internal transaction control is genuinely needed; application-level logic should be preferred whenever bypass-proofing isn't actually required.
- The most common real-world mistake in either direction is not asking the "does this genuinely need to be bypass-proof" question at all — either avoiding these tools out of caution when a real guarantee is needed, or reaching for them out of habit when application code would have been simpler, more testable, and easier to change safely.
- These tools are genuinely powerful for the narrow set of problems they were built for, and genuinely easy to overuse outside that set — the judgment in this topic matters at least as much as the syntax from Topics 1 through 3.
