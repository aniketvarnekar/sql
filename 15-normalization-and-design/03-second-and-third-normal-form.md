# Second and Third Normal Form (2NF, 3NF)

## Learning Objectives

By the end of this section you should be able to:
- Explain why 2NF only becomes a meaningful concern when a table's primary key is composite
- Identify a partial dependency in a table and decompose the table to eliminate it, reaching 2NF
- Identify a transitive dependency in a table and decompose the table to eliminate it, reaching 3NF
- State the informal one-line definitions of 2NF and 3NF ("every non-key column depends on the whole key, and nothing but the key") and explain what each half of that sentence rules out

## Prerequisites

- [Functional Dependencies](01-functional-dependencies.md) — 2NF and 3NF are defined entirely in terms of the full/partial and transitive dependency vocabulary introduced there; this topic is where that vocabulary is finally put to direct use.
- [First Normal Form (1NF)](02-first-normal-form.md) — 2NF and 3NF are only defined for relations that already satisfy 1NF; you cannot meaningfully apply either to a table with non-atomic columns or repeating groups.

## Motivation

1NF fixed the "hidden lists" problem, but the `enrollments` table it produced still repeats a student's name on every course they take, and (as you'll see below) a course's full details on every student who takes it. That redundancy isn't cosmetic — it's a live risk of the database contradicting itself the moment someone updates one copy of a repeated fact and misses another. 2NF and 3NF are the two rules that systematically remove this remaining redundancy, and together they're the normal forms most real-world schema design actually lives by day to day.

## Problem Statement

Picking up the running example from Topic 1, here is the full, un-decomposed `enrollment_records` table — 1NF-compliant (every column holds one atomic value), but nothing more:

```
 student_id | student_name | student_email     | course_id | course_title        | credits | instructor_id | instructor_name | instructor_office | semester | grade
------------+---------------+--------------------+-----------+----------------------+---------+----------------+------------------+--------------------+----------+-------
 101        | Asha Rao      | asha@example.com   | CS201     | Data Structures      | 4       | I10            | Dr. Kade         | Room 214           | Fall2026 | A
 101        | Asha Rao      | asha@example.com   | CS305     | Databases            | 3       | I14            | Dr. Osei         | Room 118           | Fall2026 | A-
 102        | Ben Ochieng   | ben@example.com    | CS201     | Data Structures      | 4       | I10            | Dr. Kade         | Room 214           | Fall2026 | B+
 103        | Chen Wei      | chen@example.com   | CS305     | Databases            | 3       | I14            | Dr. Osei         | Room 118           | Fall2026 | A
```

Its candidate key is `{student_id, course_id}` — a given student enrolls in a given course at most once, and only that pair of columns together is guaranteed unique across all rows. Everything else in the table is a non-key attribute that must relate to this key somehow. But look closely at *how* each one relates to it:

- `student_name` and `student_email` don't actually need `course_id` at all — they're fully determined by `student_id` alone. `course_id` is dead weight in the key as far as these two columns are concerned.
- `course_title` and `credits` don't need `student_id` at all — they're fully determined by `course_id` alone.
- `instructor_name` and `instructor_office` don't depend on the key at all, not even partially — they depend on `instructor_id`, which itself is just another non-key column.

Concretely, this creates three distinct anomalies:

- **Update anomaly**: if Dr. Kade's office changes, every row mentioning `I10` must be updated — miss one, and the table now disagrees with itself about Dr. Kade's office.
- **Insertion anomaly**: you cannot record that a new course, `CS499`, exists and is taught by Dr. Osei, until at least one student enrolls in it — there's no row to hold `course_title`/`credits`/`instructor_id` independent of a specific enrollment.
- **Deletion anomaly**: if Ben Ochieng (the only student in `CS201` this term) withdraws and his row is deleted, every fact about `CS201` — its title, its credits, its instructor — disappears from the database along with him, even though the course itself still exists.

## Concept

### Second Normal Form (2NF)

> A relation is in **Second Normal Form (2NF)** if it is in 1NF, and **every non-key attribute is fully functionally dependent on the whole candidate key** — no non-key attribute depends on only *part* of a composite key.

The crucial phrase is "composite key." **2NF is only a meaningful concern when the candidate key consists of more than one column.** If a table's key is a single column, every non-key attribute is automatically either fully dependent on it or not dependent on it at all — there's no "part of the key" to be partially dependent on, so a single-column-key table always trivially satisfies 2NF. This is precisely why 2NF violations only ever show up in tables like `enrollment_records`, whose key genuinely is a pair (or more) of columns.

**Decomposing to reach 2NF:** split off every attribute that depends on only part of the composite key into its own table, keyed by that part alone.

```sql
-- Attributes depending only on student_id
CREATE TABLE students (
    student_id    INTEGER PRIMARY KEY,
    student_name  TEXT NOT NULL,
    student_email TEXT NOT NULL
);

-- Attributes depending only on course_id (instructor_id kept for now — addressed by 3NF next)
CREATE TABLE courses (
    course_id     TEXT PRIMARY KEY,
    course_title  TEXT NOT NULL,
    credits       INTEGER NOT NULL,
    instructor_id TEXT NOT NULL
);

-- Attributes needing the FULL composite key
CREATE TABLE enrollments (
    student_id INTEGER NOT NULL REFERENCES students(student_id),
    course_id  TEXT    NOT NULL REFERENCES courses(course_id),
    semester   TEXT    NOT NULL,
    grade      TEXT,
    PRIMARY KEY (student_id, course_id)
);
```

Populated:

```
students:
 student_id | student_name | student_email
------------+---------------+--------------------
 101        | Asha Rao      | asha@example.com
 102        | Ben Ochieng   | ben@example.com
 103        | Chen Wei      | chen@example.com

courses:
 course_id | course_title    | credits | instructor_id
-----------+------------------+---------+----------------
 CS201     | Data Structures  | 4       | I10
 CS305     | Databases        | 3       | I14

enrollments:
 student_id | course_id | semester | grade
------------+-----------+----------+-------
 101        | CS201     | Fall2026 | A
 101        | CS305     | Fall2026 | A-
 102        | CS201     | Fall2026 | B+
 103        | CS305     | Fall2026 | A
```

Every row in `students` now holds one student's details exactly once, no matter how many courses they take. Every row in `courses` holds one course's title and credits exactly once, no matter how many students take it. The insertion anomaly is already fixed — `CS499` can now be inserted into `courses` the moment it's created, with zero enrolled students, because `courses` no longer depends on `enrollments` existing at all.

### Third Normal Form (3NF)

> A relation is in **Third Normal Form (3NF)** if it is in 2NF, and **no non-key attribute is transitively dependent on the key** — every non-key attribute depends on the key directly, not by way of some other non-key attribute.

Look again at the `courses` table produced above:

```
 course_id | course_title    | credits | instructor_id | instructor_name | instructor_office
-----------+------------------+---------+----------------+------------------+--------------------
 CS201     | Data Structures  | 4       | I10            | Dr. Kade         | Room 214
 CS305     | Databases        | 3       | I14            | Dr. Osei         | Room 118
```

(Shown here with `instructor_name` and `instructor_office` still attached, as they would be if you'd stopped after only removing the partial dependencies above.) This table is in 2NF — its key, `course_id`, is a single column, so there's no "part of the key" for anything to be partially dependent on. But it still has a problem: `instructor_name` and `instructor_office` don't depend on `course_id` for their own sake — they depend on `instructor_id`, and `instructor_id` happens to depend on `course_id`. This is a transitive dependency: `course_id → instructor_id → instructor_name, instructor_office`.

The update anomaly this causes is identical to the one in the Problem Statement: if Dr. Kade moves offices, every course row mentioning `I10` needs updating, and a missed row leaves the database self-contradictory.

**Decomposing to reach 3NF:** split off the transitively dependent attributes into their own table, keyed by the attribute they actually depend on directly.

```sql
CREATE TABLE instructors (
    instructor_id     TEXT PRIMARY KEY,
    instructor_name   TEXT NOT NULL,
    instructor_office TEXT NOT NULL
);

CREATE TABLE courses (
    course_id     TEXT PRIMARY KEY,
    course_title  TEXT NOT NULL,
    credits       INTEGER NOT NULL,
    instructor_id TEXT NOT NULL REFERENCES instructors(instructor_id)
);
```

```
instructors:
 instructor_id | instructor_name | instructor_office
---------------+------------------+--------------------
 I10           | Dr. Kade         | Room 214
 I14           | Dr. Osei         | Room 118

courses:
 course_id | course_title    | credits | instructor_id
-----------+------------------+---------+----------------
 CS201     | Data Structures  | 4       | I10
 CS305     | Databases        | 3       | I14
```

`courses.instructor_id` is now a foreign key pointing at `instructors` — the exact same relationship the transitive dependency described, now expressed as an explicit, engine-enforced link (Module 5's foreign keys) instead of an implicit fact repeated in text. Now updating Dr. Kade's office is one `UPDATE` against one row in `instructors`, and every course taught by `I10` reflects the change automatically the moment it's joined back in — because there is no second copy of the office to forget.

```sql
UPDATE instructors SET instructor_office = 'Room 220' WHERE instructor_id = 'I10';

SELECT c.course_id, c.course_title, i.instructor_name, i.instructor_office
FROM courses c
JOIN instructors i ON i.instructor_id = c.instructor_id;
```

```
 course_id | course_title    | instructor_name | instructor_office
-----------+------------------+------------------+--------------------
 CS201     | Data Structures  | Dr. Kade         | Room 220
 CS305     | Databases        | Dr. Osei         | Room 118
(2 rows)
```

### The One-Line Summary of Both

A common shorthand for 1NF through 3NF together, worth memorizing precisely:

> Every non-key column must depend on **the key**, **the whole key**, and **nothing but the key** — so help you Codd.

- "the key" — rules out non-atomic values that don't relate cleanly to any key at all (1NF).
- "the whole key" — rules out partial dependencies on part of a composite key (2NF).
- "nothing but the key" — rules out transitive dependencies through some other non-key column (3NF).

### The Full Decomposition, Side by Side

| Table stage | Key | Problem present |
|---|---|---|
| Original `enrollment_records` | `{student_id, course_id}` | Partial dependencies (`student_name`, `course_title`, ...) and a transitive dependency (`instructor_name` via `instructor_id`) |
| After 2NF split | `students(student_id)`, `courses(course_id, instructor_id, ...)`, `enrollments(student_id, course_id)` | Transitive dependency remains inside `courses` |
| After 3NF split | `students`, `courses(course_id, instructor_id)`, `instructors(instructor_id)`, `enrollments` | None — every non-key column depends on its own table's whole key, directly |

## Internal Working (Preview)

Both 2NF and 3NF violations manifest, mechanically, as the same underlying storage problem: the same byte sequence (a name, a title, an office) is physically written to disk in multiple different row locations, rather than once.

```
 Before decomposition (one wide table):
 disk page 1: row(101, 'Asha Rao', ..., 'CS201', 'Data Structures', ..., 'Dr. Kade', 'Room 214', ...)
 disk page 1: row(101, 'Asha Rao', ..., 'CS305', 'Databases',       ..., 'Dr. Osei', 'Room 118', ...)
 disk page 2: row(102, 'Ben Ochieng', ..., 'CS201', 'Data Structures', ..., 'Dr. Kade', 'Room 214', ...)
                              ▲ 'Data Structures' and 'Dr. Kade'/'Room 214' physically duplicated

 After decomposition (three narrow tables):
 students:     one physical copy of each student's details
 courses:      one physical copy of each course's details
 instructors:  one physical copy of each instructor's details
 enrollments:  narrow rows that only reference the others by key
```

An `UPDATE` against a normalized `instructors` table only ever has to locate and rewrite a single row's worth of storage; the same real-world change against the un-decomposed wide table requires the engine to locate and rewrite *every* row that happens to mention that instructor — more work per update, and, critically, a correctness risk if any qualifying row is missed by whatever `WHERE` clause performs the update (Module 6's coverage of `UPDATE` and the danger of an incomplete or missing `WHERE` clause applies directly here).

## Real-World Analogy

Imagine a company that, instead of keeping one HR record per employee, instead writes each employee's full details (name, department, manager, manager's phone number) onto every single expense report they ever file. Filing ten expense reports means writing that employee's name ten times, their department ten times, and their manager's phone number ten times, across ten separate physical documents. The moment that employee changes departments, someone has to hunt down and correct every expense report they've ever filed — and if even one from three years ago is missed, the company's paperwork now permanently disagrees with itself about which department that person belonged to on that date. A properly organized HR system instead keeps exactly one master employee record, and each expense report simply references the employee by ID — exactly the 2NF/3NF fix of pulling attribute-owning facts out of a table that only needs to reference them.

## Why 2NF and 3NF Were Designed This Way

Both rules exist to eliminate a table storing the same fact in more places than it has independent reasons to be true. A functional dependency (Topic 1) that isn't full, or isn't direct, is really a signal that the attribute in question *belongs to a different entity* than the one the table is nominally about — `instructor_office` isn't really a fact about a course, it's a fact about an instructor that a course row merely happens to mention by way of its instructor. 2NF and 3NF formalize "move each attribute to the table it actually belongs to," which is the same underlying discipline the relational model has pushed since Module 2: split data into narrow, single-purpose tables connected by keys, and let joins (Module 10) reconstruct the full picture whenever it's actually needed — rather than pre-flattening everything into one wide table and paying for that flattening in redundancy and anomaly risk on every write.

## Advantages

- **Update anomalies are eliminated** — each fact is stored in exactly one place, so there is no "other copy" that can be missed and left contradicting the one you just changed.
- **Insertion anomalies are eliminated** — a new course or instructor can be recorded independently of any enrollment existing yet, since `courses` and `instructors` no longer depend on `enrollments` for their own existence.
- **Deletion anomalies are eliminated** — removing the last enrollment for a course no longer erases the course's own details, since they live in a separate table entirely.
- **Storage is smaller in aggregate** — a name or title stored once, referenced by a small integer/text key many times, is typically far more compact than the same value duplicated across every referencing row.

## Disadvantages / Limitations

- **More tables to manage and more joins to write** — a query that used to read from one wide table now needs `JOIN`s across three or four narrower ones to reconstruct the same report (Module 10).
- **A small amount of per-query join overhead** — reconstructing the full picture at read time costs something, even if PostgreSQL's query planner (Module 20) is generally very good at making that cost small, especially with appropriate indexes (Module 13) on the foreign key columns.
- **The "right" decomposition requires correctly identifying dependencies first** — 2NF/3NF only help if you've correctly worked out which attributes really depend on which key in Topic 1's sense; a wrong assumption about a dependency leads to a wrong decomposition.

## Best Practices

- Whenever a table's key is composite, explicitly test every non-key column against each individual column of the key, not just the key as a whole — this is exactly how a partial dependency is caught before it becomes a live redundancy problem.
- Ask, for every non-key column, "does this depend on the key directly, or only because it happens to travel along with some other non-key column?" — a "yes, but only because of..." answer is a transitive dependency in disguise.
- Name the new table after the real-world entity the extracted attributes describe (`instructors`, not `course_extra_details`) — it keeps the schema self-documenting and makes the correct home for future attributes obvious.
- Don't stop decomposing at 2NF if a transitive dependency is still visible — 2NF and 3NF are cumulative requirements, not alternatives to pick between.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Applying 2NF reasoning to a table with a single-column primary key. | 2NF violations require a composite key — with a single-column key, there's no "part of the key" to be partially dependent on, so such a table is automatically in 2NF; look for a transitive dependency (3NF) instead. |
| Believing a table is "done" once partial dependencies are removed. | That only reaches 2NF. A transitive dependency (a non-key column depending on another non-key column) can still remain and needs its own decomposition to reach 3NF. |
| Forgetting to add a foreign key back after splitting a transitively dependent attribute into its own table. | Without `REFERENCES`, nothing stops `courses.instructor_id` from pointing at an instructor that doesn't exist — the redundancy problem is fixed, but referential integrity (Module 5) must still be explicitly declared. |
| Assuming every composite-key table automatically has a 2NF violation. | Only true if some non-key attribute actually depends on part of the key alone. A composite-key table where every non-key attribute genuinely needs the full key (like the final `enrollments` table's `semester` and `grade`) is already in 2NF as-is. |

## Interview Questions

1. **Q: Why is 2NF only relevant to tables with a composite primary key?**
   A: 2NF is about whether a non-key attribute depends on only part of the key rather than the whole key. With a single-column key, there is no "part" smaller than the whole key to depend on, so every non-key attribute is trivially either fully dependent on it or unrelated — a single-column-key table can never have a 2NF violation.

2. **Q: A table `order_items(order_id, product_id, product_name, quantity)` has the composite key `{order_id, product_id}`. Does it violate 2NF, and if so, how would you fix it?**
   A: Yes — `product_name` depends only on `product_id`, not on `order_id`, making it a partial dependency. The fix is to move `product_name` into its own `products(product_id, product_name)` table, leaving `order_items(order_id, product_id, quantity)` with only attributes that genuinely need the full composite key.

3. **Q: Give an example of a transitive dependency and explain how you'd remove it.**
   A: In a table `employees(employee_id, department_id, department_name)`, `department_name` doesn't depend on `employee_id` directly — it depends on `department_id`, which depends on `employee_id`. This is transitive. The fix is to create a separate `departments(department_id, department_name)` table and replace `employees.department_name` with a foreign key `employees.department_id REFERENCES departments(department_id)`.

4. **Q: What real-world problems does reaching 3NF actually prevent, concretely?**
   A: It prevents update anomalies (a repeated fact being changed in some rows but not others, leaving the database self-contradictory), insertion anomalies (being unable to record a fact about one entity, like a course, until an unrelated fact, like an enrollment, also exists), and deletion anomalies (deleting one entity's only remaining reference accidentally erasing facts about a completely different entity it was linked to).

## Summary

- **2NF** requires every non-key attribute to be fully — not just partially — dependent on a composite key; it's only a meaningful concern when the key spans more than one column.
- **3NF** requires every non-key attribute to depend on the key directly, with no transitive dependency through another non-key attribute.
- The informal rule for both, together with 1NF: every non-key column depends on the key, the whole key, and nothing but the key.
- Decomposition to reach both normal forms always takes the same shape: move an offending attribute (and the attribute it actually depends on) into its own table, keyed appropriately, and connect it back with a foreign key.
- Reaching 2NF and 3NF eliminates update, insertion, and deletion anomalies by ensuring every fact is stored in exactly one place.
- The next topic, BCNF, addresses one narrow, easily-missed situation that a table can satisfy 3NF and still fall short of — a stricter version of the exact same underlying idea.
