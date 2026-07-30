# Entity-Relationship (ER) Modeling

## Learning Objectives

By the end of this section you should be able to:
- Define an entity, an attribute, and a relationship, and identify each in a real-world requirement
- Classify a relationship's cardinality as one-to-one, one-to-many, or many-to-many
- Explain why a many-to-many relationship requires a junction (associative) table to implement in a relational schema, while one-to-many does not
- Read and draw a simple ER diagram in text form for a realistic small domain
- Translate a complete ER diagram directly into a set of correct `CREATE TABLE` statements

## Prerequisites

- [Second and Third Normal Form (2NF, 3NF)](03-second-and-third-normal-form.md) and [Boyce-Codd Normal Form (BCNF)](04-boyce-codd-normal-form.md) — this topic designs, from scratch, the exact same normalized shape those topics arrived at by decomposing a bad table after the fact; you should recognize `students`, `courses`, `instructors`, and `enrollments` as the destination, not a new example.
- [Foreign Keys and Referential Integrity](../05-constraints-and-keys/04-foreign-keys-and-referential-integrity.md) — a relationship in an ER diagram is implemented, concretely, as a foreign key or a junction table built from foreign keys, so you should already be comfortable with what a foreign key guarantees.

## Motivation

Topics 1 through 4 all shared the same shape: start with a bad table, and reason your way to a good one through functional dependencies and normal forms. That's an essential skill for cleaning up existing messes, but in real work you're just as often starting from a blank page — a stakeholder describes a new system in plain English, and you need to design the tables *before* any bad table exists to fix. ER modeling is the practical technique for going directly from a plain-English requirement to a correct, already-normalized schema, without ever passing through an anomalous intermediate table at all.

## Problem Statement

Suppose you're asked to design the database for the same training company from earlier topics, but this time from a fresh requirements conversation rather than a spreadsheet to clean up:

> "We need to track students, the courses they take, and the instructors who teach those courses. Each student can enroll in many courses, and each course can have many students — we need to record the semester and final grade for each enrollment. Each course is taught by exactly one instructor, but an instructor may teach several different courses."

Nothing here is a table yet — it's a description of real-world things and how they relate. Jumping straight to `CREATE TABLE` without first identifying the *entities* (the distinct real-world things involved) and the *relationships* between them (and precisely how many of one relate to how many of another) is exactly how the tangled, redundant `enrollment_records` table from Topic 3 gets created in the first place — someone tried to write down "students, their courses, and their instructors" as one flat table because that's how the requirement was described in a sentence, not because that's the correct underlying structure.

## Concept

### Entities and Attributes

> An **entity** is a distinct real-world thing the database needs to keep information about — typically becoming one table. An **attribute** is a single piece of information describing an entity — typically becoming one column.

From the requirement above, three entities jump out immediately, simply from the nouns being tracked: **Student**, **Course**, **Instructor**. Each has its own attributes:

| Entity | Attributes |
|---|---|
| Student | student_id, name, email |
| Course | course_id, title, credits |
| Instructor | instructor_id, name, office |

Note this is precisely the same set of entities Topics 2–4 arrived at by *decomposing* a bad table — ER modeling is simply the same destination, reached by reasoning about the requirement directly instead of by fixing a flattened mistake.

### Relationships and Cardinality

> A **relationship** describes how two (or more) entities are associated with each other. Its **cardinality** describes *how many* instances of one entity can be associated with how many instances of the other.

Three cardinalities matter for ordinary schema design:

| Cardinality | Meaning | Example |
|---|---|---|
| **One-to-one (1:1)** | One instance of A relates to at most one instance of B, and vice versa. | One person has exactly one passport; one passport belongs to exactly one person. |
| **One-to-many (1:N)** | One instance of A relates to many instances of B, but each instance of B relates to only one instance of A. | One instructor teaches many courses; each course has exactly one instructor. |
| **Many-to-many (M:N)** | Many instances of A relate to many instances of B, in both directions. | Many students enroll in many courses; each course has many students, and each student takes many courses. |

Applying this to the requirement: **Instructor–Course is one-to-many** (an instructor teaches several courses; a course has exactly one instructor), and **Student–Course is many-to-many** (a student takes several courses; a course has several students).

### Why Many-to-Many Needs a Junction Table

A one-to-many relationship is straightforward to implement: put a foreign key on the "many" side pointing back to the "one" side. `courses.instructor_id REFERENCES instructors(instructor_id)` fully captures "many courses, one instructor" with a single column — no extra table required, because each course only ever needs to reference *one* instructor row.

A many-to-many relationship cannot be implemented this way, because neither side can hold a single foreign key to the other: a student takes *multiple* courses (a single `course_id` column on `students` couldn't hold more than one), and a course has *multiple* students (a single `student_id` column on `courses` has the same problem). Attempting either would immediately reintroduce the repeating-group problem from Topic 2 (1NF).

The resolution is a **junction table** (also called an **associative table** or **link table**): a new table whose entire purpose is to represent one instance of the relationship itself, holding a foreign key to each side, with the pair of foreign keys together forming its primary key:

```sql
CREATE TABLE enrollments (
    student_id INTEGER NOT NULL REFERENCES students(student_id),
    course_id  TEXT    NOT NULL REFERENCES courses(course_id),
    semester   TEXT    NOT NULL,
    grade      TEXT,
    PRIMARY KEY (student_id, course_id)
);
```

This should look immediately familiar — it's exactly the `enrollments` table Topic 3 arrived at while decomposing the bad `enrollment_records` table. That's not a coincidence: a many-to-many relationship, correctly normalized, and a junction table, correctly designed, are the same structure viewed from two different starting points. A junction table can also hold its own attributes that describe the relationship itself rather than either side alone — `semester` and `grade` don't belong to a student, and don't belong to a course; they belong to the specific *pairing* of a student with a course, which is exactly what a row in the junction table represents.

### A Complete ER Diagram in Text Form

```
 ┌─────────────────────┐              ┌─────────────────────┐
 │      STUDENT         │              │      INSTRUCTOR      │
 ├─────────────────────┤              ├─────────────────────┤
 │ student_id (PK)      │              │ instructor_id (PK)   │
 │ name                  │              │ name                  │
 │ email                 │              │ office                │
 └──────────┬───────────┘              └──────────┬───────────┘
            │                                       │
            │  M                                 1  │
            │                                       │
            │       enrolls in / teaches            │
            │                                       │
            │  N                                    │  N
 ┌──────────▼───────────┐              ┌───────────▼──────────┐
 │     ENROLLMENT        │◄─────────────┤        COURSE         │
 ├─────────────────────┤   references  ├─────────────────────┤
 │ student_id (PK, FK)   │              │ course_id (PK)        │
 │ course_id  (PK, FK)   │──────────────► instructor_id (FK)    │
 │ semester              │              │ title                 │
 │ grade                 │              │ credits               │
 └─────────────────────┘              └─────────────────────┘

 STUDENT  ──(M:N, via ENROLLMENT junction table)──  COURSE
 INSTRUCTOR ──(1:N, direct foreign key)──  COURSE
```

Reading this diagram: the `M`/`N` labels near `STUDENT` and `COURSE` (on either side of `ENROLLMENT`) mark the many-to-many relationship, resolved by the `ENROLLMENT` junction table sitting between them, holding a foreign key to each. The `1`/`N` labels between `INSTRUCTOR` and `COURSE` mark the one-to-many relationship, resolved directly by `instructor_id` living as a plain foreign key column on `COURSE` itself — no junction table needed, because each course only ever references exactly one instructor.

### Translating the Diagram into `CREATE TABLE` Statements

Every entity becomes a table; every attribute becomes a column; every one-to-many relationship becomes a foreign key on the "many" side; every many-to-many relationship becomes a junction table.

```sql
CREATE TABLE instructors (
    instructor_id     TEXT PRIMARY KEY,
    instructor_name   TEXT NOT NULL,
    instructor_office TEXT NOT NULL
);

CREATE TABLE students (
    student_id    SERIAL PRIMARY KEY,
    student_name  TEXT NOT NULL,
    student_email TEXT NOT NULL UNIQUE
);

CREATE TABLE courses (
    course_id     TEXT PRIMARY KEY,
    course_title  TEXT NOT NULL,
    credits       INTEGER NOT NULL,
    instructor_id TEXT NOT NULL REFERENCES instructors(instructor_id)
);

CREATE TABLE enrollments (
    student_id INTEGER NOT NULL REFERENCES students(student_id),
    course_id  TEXT    NOT NULL REFERENCES courses(course_id),
    semester   TEXT    NOT NULL,
    grade      TEXT,
    PRIMARY KEY (student_id, course_id)
);
```

This is a complete, already-normalized (3NF, in fact BCNF for this particular set of dependencies) schema, produced directly from a plain-English requirement — no bad intermediate table, no decomposition after the fact. Every query the earlier topics needed a `JOIN` to answer is answerable identically here, because it's the identical schema:

```sql
SELECT s.student_name, c.course_title, e.semester, e.grade, i.instructor_name
FROM enrollments e
JOIN students s    ON s.student_id = e.student_id
JOIN courses c      ON c.course_id = e.course_id
JOIN instructors i  ON i.instructor_id = c.instructor_id
WHERE s.student_name = 'Asha Rao';
```

```
 student_name | course_title    | semester | grade | instructor_name
---------------+------------------+----------+-------+------------------
 Asha Rao      | Data Structures  | Fall2026 | A     | Dr. Kade
 Asha Rao      | Databases        | Fall2026 | A-    | Dr. Osei
(2 rows)
```

## Internal Working (Preview)

An ER diagram is a design-time artifact — a piece of documentation and reasoning that exists before, and independent of, any SQL. PostgreSQL has no internal concept of "entity" or "relationship" at all; by the time `CREATE TABLE` runs, every entity has already become a table in the system catalog (`pg_class`), and every relationship has already become either a plain column with a `FOREIGN KEY` constraint (one-to-many) or an entire separate table (many-to-many), exactly as covered for foreign keys generally in Module 5. The diagram's value is entirely in catching cardinality mistakes — a many-to-many relationship mistakenly implemented as a single foreign key column, for instance — *before* they're baked into a live schema, rather than being something the database itself consults at run time.

## Real-World Analogy

Think of an ER diagram like an architect's blueprint for a building, drawn before a single brick is laid. The blueprint identifies distinct rooms (entities: kitchen, bedroom, garage), what each room contains (attributes: a kitchen has a stove, a sink, a counter), and how rooms connect to each other (relationships: a hallway connects to many bedrooms — one-to-many; a shared driveway connects to many houses on a street, and each house might use several shared amenities — many-to-many, requiring a shared, separate structure like a clubhouse that both sides connect through, rather than one house somehow containing pieces of another). Just as no competent builder starts laying bricks before the blueprint settles these questions, no schema should reach `CREATE TABLE` before its entities, attributes, and relationship cardinalities are settled on an ER diagram first.

## Why ER Modeling Was Designed This Way

ER modeling (introduced by Peter Chen in 1976, shortly after Codd's original relational model paper) exists because normalization theory, on its own, only tells you how to *fix* a table you've already written down — it doesn't tell you what tables to write down in the first place. ER modeling fills that gap by starting from the vocabulary of the real world (things, their properties, how they relate) rather than from an existing, possibly wrong, tabular guess. Its emphasis on cardinality specifically exists because getting cardinality wrong is the single most common structural mistake in schema design — modeling a many-to-many relationship as if it were one-to-many is a mistake that normalization theory alone won't catch for you at all, since the flawed table might still, by coincidence, satisfy every normal form covered in this module while still being fundamentally unable to represent the real relationship correctly (for example, a `courses` table with a single `student_id` column can never record a second enrolled student, no matter how "normalized" that column looks in isolation).

## Advantages

- **Produces a correct schema before any bad table exists to fix** — normalization (Topics 1–4) cleans up mistakes; ER modeling prevents many of them from being made in the first place.
- **Makes cardinality mistakes visible early**, on a diagram, where they're cheap to correct — rather than late, in a live schema, where fixing a wrongly-modeled relationship means an expensive schema migration.
- **Gives non-technical stakeholders something they can actually validate** — "each course has one instructor, right?" is a question a domain expert can answer confidently by looking at a diagram, even if they can't read a `CREATE TABLE` statement.
- **Translates mechanically into `CREATE TABLE` statements** — once the diagram is right, turning it into actual DDL is close to a rote exercise, rather than a fresh design decision.

## Disadvantages / Limitations

- **A diagram can still be wrong if the underlying business rules were misunderstood** — ER modeling only formalizes whatever cardinality you were told or assumed; if "each course has exactly one instructor" turns out to be false in reality (some courses are co-taught), the diagram faithfully encodes the wrong rule until someone corrects it.
- **Notation varies across teams and tools** — the text-diagram style shown here, Chen notation, "crow's foot" notation, and UML class diagrams all express the same underlying ideas with different symbols; reading someone else's ER diagram sometimes requires first learning their particular notation's conventions.
- **Doesn't replace normalization reasoning entirely** — a rushed or incomplete ER diagram can still produce a schema with hidden functional dependency issues; treating the two techniques as complementary, rather than as alternatives, gives the most reliable result.

## Best Practices

- Identify entities from the nouns in a requirement conversation, and relationships from the verbs connecting them ("student enrolls in course," "instructor teaches course") — this simple heuristic catches the large majority of entities and relationships in an ordinary business domain.
- Always determine cardinality by asking both directions of the question explicitly: "can one student take many courses?" *and* "can one course have many students?" — getting only one direction of the question answered is how one-to-many gets mistaken for many-to-many or vice versa.
- Model a many-to-many relationship's own attributes (like `semester` and `grade`, which belong to the enrollment itself, not to the student or the course alone) as attributes of the junction table, not by awkwardly attaching them to one side or the other.
- Validate the finished diagram against a few concrete, realistic scenarios ("what happens if this instructor teaches three courses at once?") before translating it into `CREATE TABLE` statements — it's far cheaper to redraw a box on a diagram than to migrate a live table.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Modeling a many-to-many relationship as a single foreign key on one side. | A single foreign key column can only ever reference one row — it cannot represent "this course has many students" or "this student takes many courses" at once; a junction table is structurally required. |
| Only checking cardinality in one direction. | "One instructor teaches many courses" (true) doesn't automatically confirm "one course has many instructors" (false, in this domain) — both directions must be checked independently to determine whether a relationship is 1:N or M:N. |
| Attaching a many-to-many relationship's own attribute (like `grade`) to one of the entity tables instead of the junction table. | `grade` describes a specific student-course pairing, not the student alone or the course alone — it belongs on the junction table, exactly where Topic 3 also placed it when decomposing the bad table. |
| Treating the ER diagram as optional documentation to skip when "the schema is simple enough." | Cardinality mistakes are cheapest to catch on a diagram, before any table exists; skipping this step for a "simple" schema is exactly how an accidentally wrong one-to-many assumption ends up baked into a live table. |

## Interview Questions

1. **Q: What is the difference between an entity, an attribute, and a relationship in ER modeling?**
   A: An entity is a distinct real-world thing the database tracks (typically a table); an attribute is a single piece of information describing an entity (typically a column); a relationship describes how two entities are associated with each other, characterized by its cardinality (one-to-one, one-to-many, or many-to-many).

2. **Q: Why does a many-to-many relationship require a separate junction table, while a one-to-many relationship doesn't?**
   A: A one-to-many relationship can be captured with a single foreign key column on the "many" side, since each "many"-side row only ever needs to reference one "one"-side row. A many-to-many relationship has no single side that can hold just one foreign key — both sides can relate to multiple rows on the other — so a separate table is needed whose rows each represent one specific pairing, holding a foreign key to each side.

3. **Q: You're modeling a system where each employee has exactly one badge, and each badge belongs to exactly one employee. What cardinality is this, and how would you implement it?**
   A: This is a one-to-one relationship. It can be implemented by placing a foreign key on either table (whichever more naturally "belongs" to the other) with an additional `UNIQUE` constraint on that foreign key column — the `UNIQUE` constraint is what prevents the same badge from being assigned to two employees, turning an ordinary one-to-many foreign key into a strict one-to-one relationship.

4. **Q: If a junction table has attributes of its own (like `semester` and `grade` on an enrollments table), what does that indicate about those attributes?**
   A: It indicates the attributes describe the relationship itself, not either entity individually — they only make sense in the context of one specific pairing (this student, in this course), which is precisely what a row in the junction table represents.

## Summary

- **Entities** become tables, **attributes** become columns, and **relationships** connect entities, characterized by their **cardinality**: one-to-one, one-to-many, or many-to-many.
- A **one-to-many** relationship is implemented with a foreign key on the "many" side; a **many-to-many** relationship requires a **junction (associative) table** holding a foreign key to each side, since neither side alone can hold the multiplicity.
- A junction table's own attributes (like `semester` and `grade`) describe the relationship itself, not either connected entity individually.
- Translating a correct ER diagram into `CREATE TABLE` statements is close to mechanical: one table per entity, one foreign key per one-to-many relationship, one junction table per many-to-many relationship.
- ER modeling and normalization (Topics 1–4) are complementary: ER modeling designs a correct schema from a fresh requirement, while normalization repairs an already-existing, flawed one — done well, both arrive at the same destination.
- Next, Topic 6 examines when it's deliberately worth reintroducing the redundancy this entire module has been eliminating — and what risk that trade-off reintroduces.
