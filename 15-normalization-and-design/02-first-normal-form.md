# First Normal Form (1NF)

## Learning Objectives

By the end of this section you should be able to:
- Define what it means for a value to be **atomic** and explain why 1NF requires every column to hold only atomic values
- Identify a **repeating group** (an array-like column, or a set of numbered columns holding the same kind of thing) as a 1NF violation
- Transform a table with repeating groups or non-atomic columns into a table that satisfies 1NF
- Explain why 1NF is considered the baseline every other normal form builds on top of

## Prerequisites

- [Functional Dependencies](01-functional-dependencies.md) — this topic doesn't lean on functional dependencies as heavily as the ones that follow it, but the same "one fact, one place" reasoning applies, and the vocabulary (relation, attribute) carries over directly.

## Motivation

Before you can ask "does this table have a partial dependency?" or "does it have a transitive dependency?", you have to be able to trust that each column even holds *one, single, well-defined value* per row in the first place. If a column secretly holds a list, or if a table has "course_1," "course_2," "course_3" columns bolted on side by side, none of the later normal forms' definitions even apply cleanly — you don't yet have well-formed rows and columns to reason about. 1NF is the entry ticket to every other topic in this module.

## Problem Statement

Suppose the same training company from the previous topic instead tracks student enrollments like this, one row per student:

```
 student_id | student_name |          courses_taken
------------+---------------+----------------------------------
 101        | Asha Rao      | CS201, CS305
 102        | Ben Ochieng   | CS201
 103        | Chen Wei      | CS305, CS412
```

At a glance this looks compact — each student appears exactly once. But `courses_taken` is not one value; it's a hidden list crammed into a single text field. Try to answer some ordinary questions against this table and the cracks show immediately:

- "Which students are enrolled in `CS305`?" There's no way to write `WHERE courses_taken = 'CS305'` — Asha's row wouldn't match, even though she is enrolled in `CS305`, because her cell holds `'CS305, CS412'`, not `'CS305'` alone. You'd be forced into fragile, slow text-matching tricks like `WHERE courses_taken LIKE '%CS305%'`, which would also incorrectly match a hypothetical course code `CS3051` if one existed.
- "How many total enrollments are there?" `COUNT(*)` on this table returns 3 — the number of *students*, not the number of *enrollments* (which is actually 5). The true count is buried inside comma-separated text, invisible to any aggregate function.
- "Add a grade for Asha's `CS201` enrollment specifically." There's no column to put it in — a single `grade` column on this table couldn't know which of Asha's two courses it belongs to.

A design that has to be queried by parsing text apart is a design that has stopped using the relational model's actual strengths — filtering, counting, and joining on structured columns — and started reimplementing string parsing badly, one query at a time.

## Concept

### The Rule

> A relation is in **First Normal Form (1NF)** if every column holds a single, **atomic** value for every row — no repeating groups, no comma-separated lists, no arrays-as-a-column, and no set of numbered "sibling" columns standing in for a list.

"Atomic" here means *indivisible for the purposes of your queries* — a single value that isn't secretly a hidden collection of several smaller values glued together. This is a slightly informal but practical definition: `'CS201, CS305'` is not atomic because your application cares about each course code individually. A `full_name` column holding `'Asha Rao'` is usually treated as atomic in ordinary business schemas, even though it technically contains two "parts" (first and last name) — atomicity is about whether *your queries* need to split it, not about whether a string is theoretically splittable at all. (If you frequently need to query first and last names separately — sorting by last name, for instance — that's itself a sign `full_name` should be split into two columns; the same underlying principle, applied with judgment.)

### Two Ways 1NF Gets Violated

**1. A single column holding multiple values** (as in the Problem Statement above) — a comma-separated list, or, in PostgreSQL specifically, literal use of an array type to hold what is really a one-to-many relationship:

```sql
CREATE TABLE students (
    student_id   SERIAL PRIMARY KEY,
    student_name TEXT NOT NULL,
    courses_taken TEXT[]   -- an array column: 'CS201, CS305' stored as {'CS201','CS305'}
);
```

PostgreSQL's array type is a real, well-supported feature — but using it here to store a list of *related entities* (courses, each with their own attributes like title and credits) is exactly the repeating-group problem, just stored in a more structured-looking column. The array hides the same issue: you can't easily filter, join, or aggregate over the individual elements the way you can over rows in a proper table.

**2. Repeating groups spread across numbered columns** — the same problem, disguised as "more columns" instead of "one packed column":

```sql
CREATE TABLE students (
    student_id    SERIAL PRIMARY KEY,
    student_name  TEXT NOT NULL,
    course_1      TEXT,
    course_2      TEXT,
    course_3      TEXT
);
```

```
 student_id | student_name | course_1 | course_2 | course_3
------------+---------------+----------+----------+----------
 101        | Asha Rao      | CS201    | CS305    |
 102        | Ben Ochieng   | CS201    |          |
 103        | Chen Wei      | CS305    | CS412    |
```

This is arguably worse than the comma-separated column, because it also silently caps every student at exactly 3 courses — a fourth enrollment has nowhere to go without an `ALTER TABLE` adding `course_4`. Querying "which students take `CS305`" now requires checking three separate columns (`WHERE course_1 = 'CS305' OR course_2 = 'CS305' OR course_3 = 'CS305'`), and every aggregate, filter, and join has to be manually repeated per numbered column.

### The 1NF Transformation

The fix in both cases is the same: give each individual fact (one student, one course, taken together) its own row, instead of packing a variable-length collection into one row's columns.

**Before (violates 1NF):**

```
 student_id | student_name |          courses_taken
------------+---------------+----------------------------------
 101        | Asha Rao      | CS201, CS305
 102        | Ben Ochieng   | CS201
 103        | Chen Wei      | CS305, CS412
```

**After (satisfies 1NF):**

```sql
CREATE TABLE enrollments (
    student_id   INTEGER NOT NULL,
    student_name TEXT NOT NULL,
    course_id    TEXT NOT NULL,
    PRIMARY KEY (student_id, course_id)
);

INSERT INTO enrollments (student_id, student_name, course_id) VALUES
    (101, 'Asha Rao',    'CS201'),
    (101, 'Asha Rao',    'CS305'),
    (102, 'Ben Ochieng', 'CS201'),
    (103, 'Chen Wei',    'CS305'),
    (103, 'Chen Wei',    'CS412');
```

```
 student_id | student_name | course_id
------------+---------------+-----------
 101        | Asha Rao      | CS201
 101        | Asha Rao      | CS305
 102        | Ben Ochieng   | CS201
 103        | Chen Wei      | CS305
 103        | Chen Wei      | CS412
(5 rows)
```

Every column now holds exactly one atomic value per row, and every question that was awkward before becomes a single, ordinary SQL statement:

```sql
-- Which students are enrolled in CS305?
SELECT student_name FROM enrollments WHERE course_id = 'CS305';
```

```
 student_name
---------------
 Asha Rao
 Chen Wei
(2 rows)
```

```sql
-- How many total enrollments are there?
SELECT COUNT(*) FROM enrollments;
```

```
 count
-------
     5
(1 row)
```

Both queries are now exact, fast, and don't rely on any text-parsing trick. Note that this 1NF-satisfying `enrollments` table is *still* not fully normalized in every other respect — `student_name` is repeated on both of Asha's rows, which is a partial-dependency problem that Topic 3 (2NF) addresses next. 1NF only guarantees atomic values and no repeating groups; it says nothing yet about redundancy caused by a composite key. Each normal form solves one specific class of problem, and they stack.

## Internal Working (Preview)

PostgreSQL's row storage format (used internally for every ordinary table) fundamentally expects one value per column per row — it has no native way to store "a variable number of related course records" inside a single row's storage slot, other than falling back to a semi-structured escape hatch like `TEXT`, `ARRAY`, or (covered in a later module) `JSON`. Conceptually:

```
 1NF-violating design:                      1NF-satisfying design:
 ┌─────────────┬───────────────────┐        ┌─────────────┬───────────┐
 │ student_id  │ courses_taken     │        │ student_id  │ course_id │
 │             │ (packed/array)    │        │             │           │
 ├─────────────┼───────────────────┤        ├─────────────┼───────────┤
 │ 101         │ 'CS201, CS305'    │        │ 101         │ CS201     │
 │             │  (1 storage slot, │        │ 101         │ CS305     │
 │             │   2 hidden facts) │        │ 102         │ CS201     │
 └─────────────┴───────────────────┘        └─────────────┴───────────┘
                                              (each row = exactly 1 fact,
                                               1 storage slot, 1 index entry)
```

When a column holds one atomic value per row, the query planner and storage engine can filter, index, and join on that column directly and efficiently (Module 13 — Indexes). When a column instead hides a packed list, none of PostgreSQL's ordinary indexing and filtering machinery can see *inside* it without extra, more expensive tooling (for array columns specifically, PostgreSQL does support specialized `GIN` indexes for searching within an array — but this is a workaround for a design that already violates 1NF, not a reason to prefer that design in the first place).

## Real-World Analogy

Think of a filing cabinet where each folder is supposed to hold exactly one client's single contract. A 1NF violation is like someone stuffing three unrelated clients' contracts into one folder labeled with only the first client's name, because it was "convenient" at filing time. The moment anyone needs just the second client's contract, they have to pull the whole folder, dig through all three, and manually figure out which pages belong to whom — the filing system's entire benefit (labeled, individually retrievable folders) is defeated. The fix is exactly what 1NF prescribes: one folder per contract, even if that means three folders where there used to be one.

## Why 1NF Was Designed This Way

1NF formalizes the most basic promise of the relational model itself (Module 2): a relation is a set of tuples, where each tuple assigns exactly one value to each attribute. A column secretly holding a list isn't a minor style issue — it's a table that no longer satisfies the mathematical definition of a relation at all, which is why 1NF is treated as the *entry requirement* for the rest of normalization theory rather than just "the first item on a checklist." Codd's original insistence on atomic values traces directly back to wanting every column to be independently queryable through the same simple, uniform operations (filter, project, join) — the moment a column hides multiple values, those operations stop working correctly on it, and you're forced back into ad hoc, per-column string-parsing logic that the relational model was invented specifically to replace.

## Advantages

- **Every column becomes independently filterable, sortable, and indexable** — no query needs to parse text or search inside a packed value to find what it needs.
- **Aggregates (`COUNT`, `SUM`, `GROUP BY`) become correct and meaningful** — one row genuinely represents one fact, so counting rows means counting facts.
- **No artificial capacity limits** — a numbered-column design (`course_1`, `course_2`, `course_3`) silently caps how many related items a row can have; a proper one-row-per-fact design has no such ceiling.
- **Joins work as designed** — a properly atomic foreign-key-style column (like `course_id` in the `enrollments` example) can be joined against a `courses` table directly; a packed list cannot be joined against anything without first being split apart.

## Disadvantages / Limitations

- **More rows overall** — a table that used to have one row per student now has one row per (student, course) pair, which can feel like "more data" even though it's the same information, now correctly structured.
- **Requires a composite or dedicated key** — once you split repeating data into its own rows, you need something to uniquely identify each new row (in the example above, `(student_id, course_id)` together), which means thinking about composite keys (Topic 3 builds directly on this).
- **1NF alone doesn't remove all redundancy** — as seen in the worked example, satisfying 1NF can still leave a table with the exact same value (`student_name`) repeated across multiple rows; 1NF only fixes non-atomic columns, not every kind of duplication.

## Best Practices

- Whenever you find yourself wanting to name columns `_1`, `_2`, `_3` for "more of the same kind of thing," stop — that is almost always a repeating group that belongs in its own table with its own rows instead.
- Be suspicious of any column whose values contain internal separators (commas, pipes, semicolons) that your application code parses back apart after querying — if you're splitting a value in your application after every `SELECT`, the database should very likely be storing it split already.
- Reserve PostgreSQL's `ARRAY` type for genuinely atomic, fixed-purpose collections your queries don't need to filter or join on individually (e.g., a small set of tags displayed together as-is) — not for related entities that have their own attributes and deserve their own table.
- Apply the "does my application ever need to query, sort, or aggregate on just part of this value?" test to every column — if yes, that value isn't atomic for your purposes and should be split.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using a comma-separated `TEXT` column to store a one-to-many relationship (e.g., a student's list of courses). | It defeats indexing, filtering, and joining on the individual items, and requires fragile pattern-matching (`LIKE '%CS305%'`) instead of an exact, indexable comparison. |
| Adding numbered columns (`phone_1`, `phone_2`, `phone_3`) instead of a separate related table. | It silently caps how many items a row can have, and forces every query to repeat itself across each numbered column instead of filtering one column once. |
| Assuming any string with punctuation inside it violates 1NF. | Atomicity is about whether your *queries* need to split the value, not whether a string is theoretically divisible. A `full_name` column holding `'Asha Rao'` is ordinarily fine in 1NF if the application never needs to query the first and last name separately. |
| Believing PostgreSQL's `ARRAY` type "solves" the repeating-group problem because it's a proper data type rather than a hacky string. | An array column still hides a one-to-many relationship inside a single row's single column — it's a more structured-looking repeating group, but it's still a repeating group if the elements are really related entities your queries need to work with individually. |

## Interview Questions

1. **Q: What does it mean for a value to be "atomic," in the context of 1NF?**
   A: A value is atomic if it isn't secretly a hidden collection of several smaller values that your queries need to work with individually. It's a practical definition tied to how the data is actually used, not a claim that the value is mathematically indivisible — a full name string is usually atomic in practice, while a comma-separated list of course codes is not, because queries need to filter, count, and join on individual course codes.

2. **Q: What is a "repeating group," and give two different ways it commonly shows up in a poorly designed table.**
   A: A repeating group is a set of logically repeated values crammed into a single row instead of given their own rows. It commonly shows up either as a single column holding a packed list (comma-separated text, or an array of related entities), or as a set of numbered sibling columns (`course_1`, `course_2`, `course_3`) standing in for a variable-length list.

3. **Q: Why is 1NF described as the "entry requirement" for the rest of normalization theory, rather than just one normal form among several?**
   A: Because every later normal form's definition assumes you already have a well-formed relation — one row per fact, one atomic value per column — to reason about. Functional dependency analysis (2NF, 3NF, BCNF) doesn't cleanly apply to a column that's secretly hiding multiple values; you have to reach 1NF first before those later, stricter rules even make sense to evaluate.

4. **Q: Does satisfying 1NF eliminate all redundancy in a table?**
   A: No. 1NF only guarantees atomic values and the absence of repeating groups — it says nothing about redundancy caused by a composite key, where a non-key value (like a student's name) can still be repeated across multiple rows that share part of the key. That specific problem is addressed by 2NF, covered next.

## Summary

- **1NF** requires every column to hold a single atomic value per row, with no repeating groups and no arrays-as-columns standing in for a one-to-many relationship.
- Violations show up either as a packed/list-like single column, or as numbered sibling columns (`course_1`, `course_2`, ...) standing in for a variable-length list.
- The fix is always the same shape: give each individual fact its own row in its own (possibly new) table, identified by an appropriate key.
- 1NF makes every column independently filterable, sortable, joinable, and aggregatable — none of which reliably works against a packed value.
- 1NF is necessary but not sufficient — a table can satisfy 1NF and still have significant redundancy, which is exactly what 2NF and 3NF (covered next) go on to address.
