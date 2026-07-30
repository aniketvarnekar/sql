# Boyce-Codd Normal Form (BCNF)

## Learning Objectives

By the end of this section you should be able to:
- State the definition of BCNF and explain precisely how it differs from 3NF
- Identify the narrow edge case where a table satisfies 3NF but not BCNF: a determinant that is not a candidate key
- Work through a table with two overlapping candidate keys and show why it violates BCNF
- Decompose a BCNF-violating table into a form that satisfies it
- Explain why BCNF is described as a "stricter version" of 3NF rather than an unrelated, separate rule

## Prerequisites

- [Second and Third Normal Form (2NF, 3NF)](03-second-and-third-normal-form.md) — BCNF is defined as a tightened version of 3NF's own definition, so you need 3NF's exact wording fresh in mind to see precisely what BCNF adds.
- [Functional Dependencies](01-functional-dependencies.md), specifically the definition of a determinant — BCNF's entire distinction from 3NF hinges on which functional dependencies are permitted to have a non-candidate-key determinant.

## Motivation

3NF eliminates transitive dependencies through non-key attributes, and in the overwhelming majority of real schemas, reaching 3NF is where practical normalization stops — it removes essentially all the update anomalies you're likely to encounter. But 3NF has one narrow blind spot, discovered by Raymond Boyce and Edgar Codd shortly after 3NF itself was defined: a specific, somewhat unusual arrangement of *overlapping candidate keys* that 3NF's own wording technically allows through, even though it still causes a real anomaly. BCNF exists to close that one remaining gap.

## Problem Statement

A different part of the same training company assigns each enrolled student a personal **advisor** for every course they take, according to two real business rules:

1. A given student, in a given course, is assigned exactly one advisor.
2. Each advisor specializes in exactly one course — an advisor who advises for `CS201` never also advises for `CS305`.

The data looks like this:

```
 student_id | course_id | advisor_id
------------+-----------+------------
 101        | CS201     | ADV-K
 101        | CS305     | ADV-O
 102        | CS201     | ADV-K
 103        | CS305     | ADV-O
 103        | CS412     | ADV-K
```

Notice something rule 2 implies: because `ADV-K` only ever advises for `CS201`, and `ADV-O` only ever advises for `CS305`... wait — look at Chen Wei's second row: `ADV-K` also appears against `CS412`. That's consistent with rule 2 only if `ADV-K` specializes in exactly one course *at a time* — for this table, assume `ADV-K` currently specializes in `CS412`, and the `CS201` rows reflect an assignment from a still-open past term. The point that matters for this topic is structural, not historical: **whenever a specific advisor ID appears anywhere in this table, it always appears next to the same course ID.** That single fact — `advisor_id → course_id` — is the functional dependency this entire topic is built around.

Suppose an administrative error assigns `ADV-K` (who specializes in `CS412`) to also advise a student in `CS201`:

```sql
INSERT INTO advising (student_id, course_id, advisor_id) VALUES (104, 'CS201', 'ADV-K');
```

Nothing about the table's structure stops this — there is no constraint anywhere that captures "an advisor's course must match every other row mentioning that advisor," even though that is exactly the real-world rule being violated. This is the anomaly BCNF exists to catch, and — as the rest of this topic shows — 3NF's own definition does not catch it.

## Concept

### Working Out the Candidate Keys

Before applying any normal form, you need every candidate key, not just an assumed one. This table has functional dependencies:

- `{student_id, course_id} → advisor_id` (rule 1: one advisor per student-course pairing)
- `advisor_id → course_id` (rule 2: an advisor specializes in exactly one course)

From these two dependencies, there are actually **two** candidate keys, not one:

- `{student_id, course_id}` — obviously a candidate key: it determines `advisor_id` directly (rule 1), and every attribute of the relation is now covered.
- `{student_id, advisor_id}` — also a candidate key, and easy to miss: since `advisor_id → course_id`, knowing `student_id` and `advisor_id` together already lets you derive `course_id` too (via the second dependency), which means `{student_id, advisor_id}` also determines every attribute in the table. It's minimal (dropping either column breaks the determination), so it qualifies as a full candidate key in its own right.

`course_id` is a **prime attribute** — it's part of a candidate key (`{student_id, course_id}`) — while `advisor_id`, notably, is *also* part of a candidate key (`{student_id, advisor_id}`), so it too is prime. This detail is exactly why this example is subtle enough to need its own normal form.

### Why This Table Is Already in 3NF

Recall 3NF's definition from the previous topic: no non-key attribute may be transitively dependent on a candidate key. Check the dependency `advisor_id → course_id` against that rule carefully — `course_id` is not a non-key attribute here; it's prime (part of the candidate key `{student_id, course_id}`). **3NF's rule only restricts dependencies on non-key attributes** — a dependency whose *dependent* side is a prime attribute is explicitly permitted by 3NF's wording, no matter what its determinant looks like. So `advisor_id → course_id` does not violate 3NF, and this table genuinely does satisfy every requirement of 3NF as defined in the previous topic.

And yet the anomaly demonstrated above is real: nothing stops `ADV-K` from being paired with the wrong course on some future row, silently contradicting rule 2. 3NF, by its own precise wording, has a blind spot here — and that blind spot is exactly what BCNF was defined to close.

### The BCNF Definition

> A relation is in **Boyce-Codd Normal Form (BCNF)** if, for every non-trivial functional dependency `X → Y` in the relation, `X` is a **superkey** — that is, `X` alone is sufficient to determine every attribute in the relation (a candidate key, or a superset of one).

Compare this word-for-word against 3NF: 3NF says a dependency is fine as long as its *dependent* (`Y`) is a prime attribute, even if its *determinant* (`X`) isn't a full candidate key on its own. BCNF removes that escape hatch entirely — it doesn't ask anything about `Y`; it demands that **every** determinant, full stop, be a superkey. This is the single, precise difference between the two, and it's why BCNF is accurately described as a **stricter version of 3NF**: every relation in BCNF is automatically in 3NF too (BCNF's requirement is a strict superset of 3NF's), but the reverse doesn't hold, as this exact table demonstrates.

Apply the BCNF test to `advisor_id → course_id`: is `advisor_id` alone a superkey? No — `advisor_id` by itself does not determine `student_id` (many different students can share the same advisor for the same course across different enrollments). `advisor_id` is a determinant, but not a superkey. **This single functional dependency is what makes the table fail BCNF**, even though, as shown above, it does not make the table fail 3NF.

### Decomposing to BCNF

The fix follows the same pattern as every other normal form in this module: split off the offending dependency into its own table, keyed by its actual determinant.

```sql
-- advisor_id -> course_id lives here now, with advisor_id as the (super)key
CREATE TABLE advisors (
    advisor_id TEXT PRIMARY KEY,
    course_id  TEXT NOT NULL
);

-- student_id, advisor_id together are enough to identify an assignment;
-- course_id is no longer stored here directly -- it's looked up via advisors
CREATE TABLE student_advising (
    student_id INTEGER NOT NULL,
    advisor_id TEXT    NOT NULL REFERENCES advisors(advisor_id),
    PRIMARY KEY (student_id, advisor_id)
);
```

```
advisors:
 advisor_id | course_id
------------+-----------
 ADV-K      | CS412
 ADV-O      | CS305

student_advising:
 student_id | advisor_id
------------+------------
 101        | ADV-K
 101        | ADV-O
 102        | ADV-K
 103        | ADV-O
 103        | ADV-K
```

Now the rule "an advisor specializes in exactly one course" is enforced structurally — `advisors.advisor_id` is a primary key, so `ADV-K` can only ever appear in the `advisors` table pointing at a single `course_id`. The earlier administrative error (assigning `ADV-K` against `CS201` on some row) is no longer even expressible: `course_id` doesn't appear at all in `student_advising`, so there is no column left where a contradicting course could be written down next to `ADV-K`. To recover "which course is a student being advised on, and by whom," a join reconstructs it exactly:

```sql
SELECT sa.student_id, a.course_id, sa.advisor_id
FROM student_advising sa
JOIN advisors a ON a.advisor_id = sa.advisor_id;
```

```
 student_id | course_id | advisor_id
------------+-----------+------------
 101        | CS412     | ADV-K
 101        | CS305     | ADV-O
 102        | CS412     | ADV-K
 103        | CS305     | ADV-O
 103        | CS412     | ADV-K
```

## Internal Working (Preview)

The mechanical fix is identical in kind to every earlier normal form's decomposition — extract the dependency into its own table so the engine's ordinary `PRIMARY KEY` uniqueness check enforces it — but it's worth being precise about *why* 3NF's own decomposition process didn't already catch this on its own:

```
 3NF's check, per non-key attribute Y:
   "does Y depend on something less than the full candidate key,
    or on some other non-key attribute?"
        │
        ▼
   advisor_id -> course_id: course_id is NOT a non-key attribute here
   (it's part of the candidate key {student_id, course_id})
        │
        ▼
   3NF's check simply never fires on this dependency -- it only
   ever inspects dependencies whose right-hand side is non-key

 BCNF's check, per ANY non-trivial dependency X -> Y (regardless of Y):
   "is X, by itself, a superkey?"
        │
        ▼
   advisor_id -> course_id: is advisor_id alone a superkey? No.
        │
        ▼
   BCNF's check fires here -- and forces the same kind of
   decomposition 3NF would have required, had Y been non-key
```

The gap exists because 3NF was defined first, with non-key attributes as its focus, and this specific overlapping-candidate-key situation was noticed only afterward as a case 3NF's wording happened not to cover.

## Real-World Analogy

Picture a small clinic where each appointment slot is booked by a (patient, doctor) pair, and separately, each doctor is scheduled in exactly one specialty room for the whole day (Dr. Kade is always in Room 3; Dr. Osei is always in Room 5). A booking sheet that lists (patient, doctor, room) together looks perfectly reasonable at a glance — every row seems to make sense on its own. But nothing on that sheet stops a rushed receptionist from writing Dr. Kade next to Room 5 on one particular line, quietly contradicting every other line that correctly ties Dr. Kade to Room 3. The real, structural rule — "each doctor has exactly one room, always" — is a fact about *doctors*, not about *appointments*, and it belongs on its own separate roster (doctor → room), not repeated, and therefore corruptible, on every single booking line. That separate roster is exactly BCNF's `advisors` table.

## Why BCNF Was Designed This Way

BCNF exists because functional dependency theory is about eliminating *every* determinant that isn't a superkey, full stop — not just the ones whose dependent side happens to be a non-key attribute. 3NF was a hugely useful, practical approximation of that full goal, but it was defined slightly asymmetrically (only policing dependencies that land on non-key attributes), which happens to let through the rare situation of overlapping candidate keys demonstrated in this topic. BCNF is what you get when you apply the same "every non-trivial dependency must have a superkey determinant" reasoning without carving out 3NF's exception for prime attributes — which is precisely why it's correctly described as a tightened, purer version of the same underlying principle, discovered as a direct refinement once the overlapping-key edge case was noticed, rather than as an unrelated fifth rule invented independently.

## Advantages

- **Closes the one remaining gap 3NF leaves open** — any redundancy or update anomaly caused by a non-superkey determinant, even one that lands on a prime attribute, is eliminated.
- **Conceptually simpler to state than 3NF, once understood** — "every determinant must be a superkey" is a single, uniform rule, with none of 3NF's extra carve-out about prime vs. non-prime dependents.
- **A useful diagnostic even when you don't decompose to it** — recognizing a BCNF violation (a determinant that isn't a superkey) tells you exactly where a hidden business rule (like "an advisor specializes in one course") is not yet structurally enforced.

## Disadvantages / Limitations

- **The situation BCNF addresses is genuinely rare in ordinary business schemas** — it requires two or more overlapping candidate keys with a specific cross-dependency between them, which is uncommon outside of a handful of classic textbook-style scenarios (like this one).
- **Decomposing to BCNF can occasionally lose the ability to enforce every original functional dependency using only primary/foreign keys** (a property called "dependency preservation"), whereas a 3NF decomposition is always guaranteed to preserve every original dependency. In practice this is a rare, advanced concern; 3NF decompositions almost always suffice, and reaching for BCNF is usually a deliberate, occasional step rather than a routine one.
- **It's easy to over-apply** — chasing BCNF on ordinary tables that don't have overlapping candidate keys accomplishes nothing beyond what 3NF already achieved, since a table with only one candidate key is already automatically at BCNF the moment it satisfies 3NF.

## Best Practices

- Before worrying about BCNF, explicitly check whether a table actually has more than one candidate key — if there's only one, reaching 3NF already means you've reached BCNF, and no further work is needed.
- When a table does have overlapping candidate keys, test every functional dependency's determinant against the full list of candidate keys, not just the one you first assumed was "the" key — that's exactly how `{student_id, advisor_id}` was found as a second candidate key in this topic's example.
- Treat 3NF as the practical default target for most schema design, and reach for BCNF specifically when you spot a determinant that isn't a superkey — don't treat "always decompose all the way to BCNF" as a blanket rule independent of whether the situation actually calls for it.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming a table with a composite candidate key can only have that one candidate key. | As this topic's worked example shows, a second, overlapping candidate key can exist precisely because of another functional dependency in the table — missing it means missing the BCNF violation entirely. |
| Believing 3NF and BCNF are unrelated rules that happen to both be about normalization. | BCNF is a strictly stricter version of 3NF's exact same principle — every BCNF-satisfying relation is automatically in 3NF, but not every 3NF relation satisfies BCNF, as this topic's example demonstrates directly. |
| Thinking any functional dependency whose determinant "sounds important" must be fine. | The only test that matters is whether the determinant is a superkey — `advisor_id` sounding like a meaningful identifier doesn't matter if it doesn't, by itself, determine every attribute in the relation. |
| Chasing BCNF decomposition on a table that only has a single candidate key. | Such a table is automatically in BCNF the moment it's in 3NF — there's no overlapping-key edge case for BCNF to catch, so additional decomposition accomplishes nothing. |

## Interview Questions

1. **Q: What is the precise difference between 3NF and BCNF?**
   A: 3NF forbids a non-key attribute from being transitively dependent on the key, but explicitly only restricts dependencies whose *dependent* side is a non-key (non-prime) attribute. BCNF drops that restriction and requires every non-trivial functional dependency's *determinant*, regardless of what its dependent side is, to be a superkey. BCNF is therefore strictly stricter than 3NF.

2. **Q: Can a table be in 3NF but not in BCNF? Describe a scenario.**
   A: Yes. This happens when a table has two or more overlapping candidate keys and a functional dependency whose determinant is not itself a candidate key but whose dependent value happens to be part of another candidate key (a prime attribute). For example, a table `(student_id, course_id, advisor_id)` where `{student_id, course_id}` and `{student_id, advisor_id}` are both candidate keys, and `advisor_id → course_id` holds: `course_id` is prime, so 3NF permits the dependency, but `advisor_id` isn't a superkey, so BCNF forbids it.

3. **Q: Why is BCNF described as a "stricter" version of 3NF rather than a completely separate normal form?**
   A: Because every relation that satisfies BCNF automatically satisfies 3NF as well — BCNF's requirement (every determinant is a superkey) implies 3NF's weaker requirement (every non-key attribute either depends on the whole key directly, or is itself prime). The reverse isn't true, which is exactly what makes BCNF a genuine, strict tightening of the same underlying rule rather than an unrelated criterion.

4. **Q: If a table has only one candidate key, is it possible for it to be in 3NF but not BCNF?**
   A: No. With only one candidate key, every attribute not in that key is automatically non-prime, so 3NF's restriction on dependencies landing on non-key attributes already covers every possible dependency in the table — there's no prime-attribute escape hatch left for BCNF to additionally close. The 3NF/BCNF gap only ever appears when a table has multiple, overlapping candidate keys.

## Summary

- **BCNF** requires every non-trivial functional dependency's determinant to be a superkey — a strictly tighter requirement than 3NF's.
- 3NF permits a dependency to have a non-superkey determinant as long as its dependent value is a prime attribute (part of some candidate key); BCNF removes that exception entirely.
- The gap between 3NF and BCNF only appears when a table has two or more overlapping candidate keys, which is a comparatively rare real-world situation.
- The classic example: `{student_id, course_id}` and `{student_id, advisor_id}` as overlapping candidate keys, with `advisor_id → course_id` violating BCNF while still satisfying 3NF.
- The decomposition technique is identical to every other normal form in this module: extract the offending dependency into its own table, keyed by its true determinant, and reconnect with a foreign key.
- With the normal forms now complete, the next topic shifts from *fixing* a bad table to *designing* a good one from scratch, using entity-relationship modeling.
