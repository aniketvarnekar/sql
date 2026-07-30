# Functional Dependencies

## Learning Objectives

By the end of this section you should be able to:
- State the formal definition of a functional dependency (`X → Y`) and explain what "determines" means in that context
- Read a sample table and identify the functional dependencies it appears to satisfy
- Distinguish a **full** functional dependency from a **partial** one when the determinant is a composite key
- Recognize a **transitive** dependency and explain why it's different from a direct one
- Explain why functional dependencies, not vague intuition, are the formal basis for every normal form covered later in this module

## Prerequisites

- Module 2 — Relational Model, for the vocabulary of a relation, a tuple (row), and an attribute (column) — a functional dependency is a statement about attributes within a single relation.
- [UNIQUE Constraints](../05-constraints-and-keys/02-unique-constraints.md) — a candidate key is itself a special, complete functional dependency (the whole key determines every other attribute), so you should already be comfortable with what "uniquely identifies a row" means before this topic formalizes it further.

## Motivation

Every normal form you'll learn in this module — 1NF, 2NF, 3NF, BCNF — is ultimately just a rule about which functional dependencies a table is and isn't allowed to contain. If you skip this topic and jump straight to "how to reach 3NF," you'll be memorizing a checklist instead of understanding a design principle, and you'll have no way to reason about a table shape you haven't seen a named rule for yet. Learn this vocabulary first, and every later topic in this module becomes a short, mechanical consequence of it rather than a new thing to memorize from scratch.

## Problem Statement

Suppose a training company hands you a spreadsheet export of everyone currently enrolled in a course, with no database design behind it at all — just rows exported directly from whatever system tracked this by hand:

```
 student_id | student_name | student_email        | course_id | course_title        | credits | instructor_id | instructor_name | instructor_office | semester | grade
------------+---------------+----------------------+-----------+----------------------+---------+----------------+------------------+--------------------+----------+-------
 101        | Asha Rao      | asha@example.com     | CS201     | Data Structures      | 4       | I10            | Dr. Kade         | Room 214           | Fall2026 | A
 101        | Asha Rao      | asha@example.com     | CS305     | Databases            | 3       | I14            | Dr. Osei         | Room 118           | Fall2026 | A-
 102        | Ben Ochieng   | ben@example.com      | CS201     | Data Structures      | 4       | I10            | Dr. Kade         | Room 214           | Fall2026 | B+
 103        | Chen Wei      | chen@example.com     | CS305     | Databases            | 3       | I14            | Dr. Osei         | Room 118           | Fall2026 | A
 103        | Chen Wei      | chen@example.com     | CS412     | Distributed Systems  | 4       | I10            | Dr. Kade         | Room 214           | Fall2026 | B
```

Stare at this for a moment and some uncomfortable facts emerge. `Asha Rao`'s name and email are typed out twice, once per course she's taking. `Data Structures`, its 4 credits, and `Dr. Kade`'s name and office are repeated on every row where anyone takes that course. If Dr. Kade moves to Room 220 next semester, how many rows need to change? Every single row mentioning `I10` — and if even one of them is missed or mistyped, the table now contains two different, contradictory answers to "what room is Dr. Kade in?" with no way for the database to know which one is right.

This isn't a vague "feels messy" complaint. Every one of these problems traces back to a specific, nameable relationship between columns — and that relationship is what a **functional dependency** formally describes.

## Concept

### The Formal Definition

> Given a relation (table) `R`, attribute (column) `Y` is **functionally dependent** on attribute `X` — written `X → Y` and read "`X` determines `Y`" — if, for any two rows in `R` that have the same value of `X`, they are guaranteed to also have the same value of `Y`.

`X` is called the **determinant**. Practically, this means: if you know the value of `X`, you can always correctly figure out the value of `Y`, no matter which row you're looking at. `X` and `Y` don't have to be single columns — either side of a functional dependency can be a set of columns taken together (a **composite** determinant), which becomes essential once a table's key is more than one column wide.

Note precisely what a functional dependency does *not* claim: it says nothing about `Y` determining `X` back. `course_id → course_title` (a course ID determines exactly one title) does not imply `course_title → course_id`, since in principle two different courses could coincidentally share the same title. A functional dependency is a one-directional guarantee.

### Reading Functional Dependencies Out of the Sample Data

Looking back at the messy export above, several functional dependencies are apparent — some because the data enforces them, and some because we know the underlying real-world rule even if only a few rows are shown:

| Functional dependency | Why it holds |
|---|---|
| `student_id → student_name, student_email` | Every row with `student_id = 101` shows the exact same name and email — a student ID identifies one specific student. |
| `course_id → course_title, credits, instructor_id` | Every row with `course_id = CS201` shows the same title, credit count, and instructor — a course ID identifies one specific course offering. |
| `instructor_id → instructor_name, instructor_office` | Every row mentioning `I10` shows the same instructor name and office — an instructor ID identifies one specific person. |
| `{student_id, course_id} → semester, grade` | A given student, in a given course, has exactly one recorded semester and one recorded grade — this pair, taken together, determines the outcome of that specific enrollment. |

Notice `{student_id, course_id} → semester, grade` needs *both* columns together as its determinant — knowing only `student_id` doesn't tell you the grade (a student can be in several courses, each with a different grade), and knowing only `course_id` doesn't tell you the grade either (many students share a course, each with a different grade). Only the *pair* pins down a single grade.

### Trivial vs. Non-Trivial Dependencies

A functional dependency `X → Y` is called **trivial** if `Y` is already part of `X` — for example, `{student_id, course_id} → student_id` is trivially true of any relation, since you can't have two rows agree on `{student_id, course_id}` and disagree on `student_id` alone; that would be a contradiction in terms, not a real discovery about the data. Trivial dependencies are always true and carry no design information. Every functional dependency worth writing down and reasoning about — including every one in the table above — is **non-trivial**: `Y` genuinely is *not* part of `X`, and the dependency reflects a real fact about how the data behaves.

### Full vs. Partial Functional Dependencies

This distinction only becomes meaningful once a determinant is a **composite** (multi-column) key, and it is the exact idea Topic 3 of this module (2NF) is built on:

> A functional dependency `X → Y` is a **full** functional dependency if removing *any* single column from `X` breaks the dependency (`Y` is no longer determined). It is a **partial** functional dependency if `Y` is already determined by some proper subset of `X`.

Take the candidate key `{student_id, course_id}` from the sample data above:

- `{student_id, course_id} → grade` is **full** — neither `student_id` alone nor `course_id` alone determines the grade; you genuinely need both.
- `{student_id, course_id} → student_name` is **partial** — you don't need `course_id` at all here; `student_id` alone already determines `student_name`. The `course_id` in the determinant is dead weight.
- `{student_id, course_id} → course_title` is likewise **partial** — `course_id` alone already determines `course_title`; `student_id` contributes nothing to that fact.

Partial dependencies are exactly what makes `student_name` and `course_title` redundant across multiple rows in the messy export above — they are direct evidence, in FD form, of the very duplication that opened this topic.

### Transitive Functional Dependencies

A **transitive** dependency is a chain: `X → Y` and `Y → Z`, where `Z` is not part of `X`, together implying `X → Z` indirectly, through `Y`, rather than `X` determining `Z` for its own direct reason.

In the sample export: `course_id → instructor_id` (a course has one instructor) and `instructor_id → instructor_office` (an instructor has one office) together mean `course_id → instructor_office` — but only *transitively*, through the instructor. `course_id` doesn't determine an office because of anything intrinsic to courses; it determines it only because it happens to determine an instructor, who in turn has an office. This is precisely the shape of dependency Topic 3 (3NF) rules out.

## Internal Working (Preview)

It is important to be precise about what a real database engine does and does not know about functional dependencies:

```
 Design time (you, reasoning about the data)          Run time (PostgreSQL, executing statements)
 ───────────────────────────────────────────          ────────────────────────────────────────────
 "student_id determines student_name"                  No built-in concept of "X determines Y"
 "course_id determines instructor_id"                   exists anywhere in the engine.
 ...reasoned about on paper / in your head                    │
        │                                                      ▼
        ▼                                              The engine only enforces what you
 Translated into actual table structure:                translate into real constraints:
 separate tables, one per determinant,                  PRIMARY KEY, UNIQUE, FOREIGN KEY.
 each with its own PRIMARY KEY
```

PostgreSQL has no notion of "functional dependency" as a first-class internal concept the way it does `PRIMARY KEY` or `CHECK`. A functional dependency is a fact about your data's *meaning* that you, the designer, must recognize — the database only ever enforces the narrower, mechanical rules you explicitly declare (a `PRIMARY KEY` guarantees uniqueness of the key; a `FOREIGN KEY` guarantees a referenced row exists). This is exactly why normalization matters operationally, not just academically: the entire technique of normalization (Topics 2–4) is a systematic way of turning a functional dependency you've identified on paper into a real, engine-enforced guarantee — by making the determinant of that dependency into an actual primary key of its own table, so the database itself refuses to let the dependent value contradict itself across rows.

## Real-World Analogy

Think of a country's postal code system. A ZIP/postal code **determines** a city and a state (`zip_code → city, state`) — if you know the ZIP code, you can always correctly state the city and state it belongs to, because that mapping is fixed by the postal authority. But a city does *not* determine a ZIP code (`city → zip_code` does not hold) — a single city can have several ZIP codes. This is exactly the one-directional nature of `X → Y`: knowing the determinant always gets you the dependent value, but knowing the dependent value doesn't reliably get you back the determinant. It's also why postal systems maintain a single master table of ZIP codes with their city/state — precisely so "what city is ZIP 94103 in" always has one authoritative answer, rather than being re-typed, and potentially contradicted, on every letter, form, and database that happens to mention it.

## Why Functional Dependencies Were Designed This Way

Functional dependencies are a direct formalization of the relational model's core promise (Module 2, and [What Is a Database and a DBMS?](../01-introduction/01-what-is-a-database-and-a-dbms.md)): that a database centrally enforces the *truth* of data, rather than leaving every application free to store its own possibly-contradictory copy of a fact. Before this theory existed (developed by Edgar Codd and refined by others in the early-to-mid 1970s), "good table design" was a matter of experience and intuition — two designers could disagree about whether a table was well-shaped with no rigorous way to settle the argument. Functional dependency theory gives a precise, checkable test: for any two columns, does one actually determine the other in reality? Once that's answered honestly, whether a table needs to be split apart (and exactly how) stops being a matter of taste and becomes a mechanical consequence of the answer — which is exactly the mechanical process Topics 2 through 4 walk through.

## Advantages

- **Precision over intuition** — instead of "this table feels redundant," you can point to the exact functional dependency responsible and reason about it explicitly.
- **A rigorous test for a decomposition's correctness** — every normal form later in this module is defined purely in terms of which functional dependencies are and aren't allowed to survive, giving you an objective standard rather than a style preference.
- **Surfaces update anomalies before they happen** — once you notice `course_id → instructor_office` is only transitively true, you can predict, before writing a single row of data, exactly what will go wrong if an instructor moves offices.
- **Transfers to any table, in any domain** — the vocabulary (determinant, full/partial, transitive) applies identically whether you're modeling students and courses, orders and products, or patients and prescriptions.

## Disadvantages / Limitations

- **It's a design-time reasoning tool, not something PostgreSQL enforces on its own** — identifying `X → Y` doesn't make the database respect it; you still have to translate that insight into an actual `PRIMARY KEY`/`UNIQUE`/`FOREIGN KEY` structure (Topics 2–5) for the engine to guarantee it.
- **Real-world dependencies can be "almost always true" rather than strictly true**, which is easy to mis-model — e.g., assuming `email → student_name` because emails are usually per-person, when in reality a shared family email account could break that assumption; always verify a suspected dependency against the actual business rule, not just a small sample of data that happens to look consistent.
- **Enumerating every functional dependency in a wide, complex table by hand is genuinely tedious** — in practice, you focus on the dependencies that matter for key design and skip trivial ones, rather than exhaustively listing every possible determinant/dependent pair.

## Best Practices

- For every column that isn't part of a chosen key, explicitly ask: "if I know the key's value, do I *actually* know this column's value for a real-world reason, or does it just happen to match in my sample data?" — the second case is a false dependency waiting to break in production.
- Write down suspected functional dependencies as `X → Y` notation before designing tables, not after — it turns "does this table need to be split?" from a gut feeling into a checklist you can verify against Topics 2–4.
- When a determinant is a composite key, explicitly test each column of every non-key attribute against each *individual* column of the key, not just the whole key at once — this is exactly how partial dependencies (Topic 3) get missed.
- Validate a suspected functional dependency against real, messy production data (or a realistic sample), not a small, clean hand-written example — real data is where "almost always true" dependencies reveal themselves as false.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `X → Y` implies `Y → X`. | Functional dependency is one-directional. `course_id → course_title` doesn't mean `course_title → course_id`; two different courses could share a title. |
| Treating a functional dependency that merely happens to hold in a small sample as a guaranteed real-world rule. | A handful of rows agreeing by coincidence isn't the same as a business rule guaranteeing agreement forever — always confirm against the actual meaning of the data, not just what a few rows show. |
| Confusing a partial dependency with a transitive dependency. | A partial dependency is about a *composite key* where part of the key is unnecessary for determining a value. A transitive dependency is about a chain through a *non-key* attribute, regardless of whether the key is composite at all. They are caught by different normal forms (2NF vs. 3NF, Topic 3). |
| Believing PostgreSQL automatically "knows" and enforces functional dependencies you've identified. | The engine enforces only what you explicitly declare as constraints (`PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, `CHECK`) — a functional dependency you've merely noticed on paper has zero effect on the database until you structure tables and keys to reflect it. |

## Interview Questions

1. **Q: What is a functional dependency? Give an example.**
   A: A functional dependency `X → Y` means that for any two rows sharing the same value of `X`, they are guaranteed to also share the same value of `Y` — knowing `X` always tells you `Y`. Example: `student_id → student_email`, since a given student ID always corresponds to exactly one email address.

2. **Q: What's the difference between a full and a partial functional dependency?**
   A: Both describe dependencies where the determinant is a composite (multi-column) key. A full functional dependency requires every column of the composite key to determine the dependent value — removing any one column breaks the dependency. A partial functional dependency means some proper subset of the composite key (potentially just one column of it) already determines the dependent value on its own, making the rest of the key unnecessary for that particular fact.

3. **Q: How does a transitive functional dependency differ from a direct one?**
   A: A direct dependency `X → Z` means `X` determines `Z` for its own reason. A transitive dependency exists when `X → Y` and `Y → Z` hold, and `Z` ends up determined by `X` only because it passes through `Y` — `X` has no direct relationship to `Z` at all. For example, `course_id → instructor_id` and `instructor_id → instructor_office` together make `course_id` transitively determine `instructor_office`, even though a course has no direct relationship to an office.

4. **Q: Why does functional dependency theory matter if the database doesn't enforce it automatically?**
   A: Because it's the precise, checkable reasoning tool that tells you *how* to structure tables and keys so that the database's actual enforced constraints (primary keys, unique constraints, foreign keys) end up guaranteeing the real-world facts you care about. Without identifying the dependency first, you have no principled way to decide which table a piece of data belongs in.

## Summary

- A **functional dependency** `X → Y` means knowing the value of `X` always tells you the value of `Y`, for every row in a relation; `X` is the determinant, and either side can be a set of columns.
- Functional dependencies are one-directional — `X → Y` does not imply `Y → X`.
- A dependency involving a composite determinant is **full** if the entire determinant is needed, and **partial** if some subset of it already suffices.
- A **transitive** dependency is a chain (`X → Y → Z`) where the final dependent value is only indirectly tied to the original determinant.
- PostgreSQL does not track or enforce functional dependencies on its own — they are a design-time reasoning tool that you translate into real constraints (primary keys, unique constraints, foreign keys) through the normalization process covered in the rest of this module.
- Every normal form covered next (1NF, 2NF, 3NF, BCNF) is defined entirely in terms of which functional dependencies a table is and isn't allowed to contain.
