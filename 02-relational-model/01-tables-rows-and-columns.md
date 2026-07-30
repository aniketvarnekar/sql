# Tables, Rows, and Columns

## Learning Objectives

By the end of this section you should be able to:
- Define the formal terms **relation**, **tuple**, and **attribute**, and map each onto the everyday terms table, row, and column
- State the properties a collection of data must have to qualify as a valid relation
- Explain why row order and column order carry no logical meaning in the relational model, with a concrete demonstration
- Recognize how a table's real, physical column order still shows up in practice (`SELECT *`, `\d`) even though it has no theoretical significance

## Prerequisites

- [What Is a Database and a DBMS?](../01-introduction/01-what-is-a-database-and-a-dbms.md) — you need the working definition of a relational database (data organized into tables/relations) that this topic now makes formally precise.
- [Your First Query](../01-introduction/05-your-first-query.md) — you should already be comfortable running `CREATE TABLE`, `INSERT`, and `SELECT` against the `sql_course` database, since every example below builds on that same practical loop.

## Motivation

You've already created a table, inserted rows into it, and queried it back. That was enough to get moving, but "table," "row," and "column" are informal, everyday words — useful for talking casually, but not precise enough to reason rigorously about what a database is actually guaranteeing you. The moment you start asking sharper questions — "can this table have two identical rows?", "does it matter which column I defined first?", "what does the database actually promise me about ordering?" — you need the formal vocabulary the relational model was built on. That vocabulary is what this topic gives you, and it's the foundation every later module quietly assumes.

## Problem Statement

Suppose two engineers are arguing about a table of employees. One says, "the third row is Chen's record." The other says, "there's no such thing as 'the third row' — rows don't have positions." Who's right? Separately, an application developer writes code that reads column values by position (`row[0]`, `row[1]`, `row[2]`) instead of by name, and after a teammate adds a new column in the *middle* of the table definition, half the application's data reads shift and start returning the wrong values for the wrong fields. Both disagreements come from the same missing piece: a precise understanding of what a table actually *is*, and what the database does and does not promise about the order of its rows and columns.

## Concept

### Relation, Tuple, Attribute: the Formal Vocabulary

The relational model — the mathematical theory behind every relational database, covered in full in the next topic — doesn't use the words "table," "row," and "column." It uses more precise terms, borrowed from set theory. SQL then gives you the everyday synonyms you already know.

| Formal term (relational theory) | SQL / everyday term | Plain-English meaning |
|---|---|---|
| **Relation** | Table | A named collection of data, structured as a set of tuples that all share the same attributes. |
| **Tuple** | Row (or "record") | One single entry in the relation — one complete, ordered assignment of a value to every attribute. |
| **Attribute** | Column (or "field") | One named property that every tuple in the relation has a value for. |
| **Domain** | Data type | The set of legal values an attribute is allowed to hold (e.g., "any integer between -2147483648 and 2147483647" for PostgreSQL's `INTEGER`). |
| **Degree** | Number of columns | How many attributes a relation has. |
| **Cardinality** | Number of rows | How many tuples a relation currently contains. |

You'll see "table," "row," and "column" used constantly throughout this course, exactly as you already have been — they're not wrong, they're the practical, day-to-day names for relation, tuple, and attribute. But when precision matters (interviews, formal design discussions, understanding *why* a rule works the way it does), the formal terms are worth reaching for, and this course will use both interchangeably from here on, favoring whichever is clearest in context.

### Building a Relation From Scratch

Consider a small table tracking books in a library:

```sql
CREATE TABLE books (
    isbn        TEXT NOT NULL,
    title       TEXT NOT NULL,
    author      TEXT NOT NULL,
    pages       INTEGER
);

INSERT INTO books (isbn, title, author, pages) VALUES
    ('978-0132350884', 'Clean Code',            'Robert C. Martin', 464),
    ('978-1491954249', 'Designing Data-Intensive Applications', 'Martin Kleppmann', 616),
    ('978-0201633610', 'Design Patterns',        'Erich Gamma',     395);
```

In relational terms:
- `books` is the **relation**.
- Each of the three rows just inserted is a **tuple**.
- `isbn`, `title`, `author`, and `pages` are the **attributes**.
- The relation's **degree** is 4 (four attributes). Its current **cardinality** is 3 (three tuples).
- The **domain** of `pages` is "any value `INTEGER` can hold"; the domain of `title` is "any value `TEXT` can hold." Module 3 (Data Types) covers domains — i.e., data types — in full depth.

### What Makes a Collection of Data a Valid Relation

Not just any pile of data qualifies as a relation. To be a valid relation, a collection of data must satisfy several structural properties:

1. **Every attribute has a name, and that name is unique within the relation.** You cannot have two columns both named `title` in the same table — each attribute name must uniquely identify one property.
2. **Every attribute draws its values from a single, defined domain.** The `pages` column only ever holds integers; it is never sometimes an integer and sometimes a sentence of text. This is what a column's data type enforces (Module 3).
3. **Every tuple has exactly one value for every attribute — no more, no fewer.** A relation cannot have one row with five populated fields and another row with only three; every tuple conforms to the same fixed set of attributes, even if some values are absent (represented by `NULL`, covered in Module 3).
4. **Every attribute value is atomic — a single, indivisible value, not a list or a nested structure.** A `pages` column holding `395` is atomic. A hypothetical `authors` column holding the single text blob `"Erich Gamma, Richard Helm, Ralph Johnson"` looks convenient but is *not* atomic in the relational sense — it's really three facts crammed into one value, which makes it impossible to reliably query "find every book written by Ralph Johnson" without fragile text-parsing. Designing relations so every column holds one true, indivisible fact is a foundational discipline covered in depth in Module 4 (Database & Table Design) and formalized rigorously in Module 15 (Normalization & Design).
5. **No two tuples in the relation are identical.** Conceptually, a relation is a mathematical *set* of tuples, and a set — by definition — cannot contain the same element twice. If two rows have the exact same value in every single attribute, they are not two different facts; they are the same fact, listed redundantly. This is where a genuinely important, honest nuance shows up: **plain SQL tables do not automatically enforce this.** PostgreSQL will happily let you insert two byte-for-byte identical rows into a table with no constraints. Guaranteeing true, meaningful uniqueness — usually by designating one or more attributes as a **key** — is the job of `PRIMARY KEY` and `UNIQUE` constraints, covered in full in Module 5 (Constraints & Keys). For now, hold onto this idea: a *true* relation shouldn't have duplicate tuples, but SQL leaves enforcing that up to you. The next topic in this module returns to this exact gap and explains why it exists.

### Row Order Doesn't Matter

Because a relation is formally a *set* of tuples, and sets have no inherent order, **the relational model makes no promise about what order rows come back in**, unless you explicitly ask for one. Watch this in action:

```sql
SELECT * FROM books;
```

```
      isbn       |                 title                  |      author       | pages
-----------------+-----------------------------------------+-------------------+-------
 978-0132350884  | Clean Code                              | Robert C. Martin  |   464
 978-1491954249  | Designing Data-Intensive Applications   | Martin Kleppmann  |   616
 978-0201633610  | Design Patterns                         | Erich Gamma       |   395
(3 rows)
```

This happens to come back in insertion order today. But nothing in the SQL standard, and nothing PostgreSQL promises, guarantees that. If you add an index on `title`, or the table grows large enough that PostgreSQL chooses a different physical storage strategy, or a future `VACUUM` operation reorganizes the table's storage, the *exact same query* could return these three rows in a completely different order — while still being 100% correct, because the relational model never promised an order in the first place. The only way to guarantee an order is to ask for it explicitly, with `ORDER BY`:

```sql
SELECT * FROM books ORDER BY title;
```

```
      isbn       |                 title                  |      author       | pages
-----------------+-----------------------------------------+-------------------+-------
 978-0132350884  | Clean Code                              | Robert C. Martin  |   464
 978-0201633610  | Design Patterns                         | Erich Gamma       |   395
 978-1491954249  | Designing Data-Intensive Applications   | Martin Kleppmann  |   616
(3 rows)
```

This is the same underlying set of tuples as before — nothing about the *data* changed — only the presentation order did, and only because it was explicitly requested. This is a direct application of the declarative principle from Module 01's [What Is SQL?](../01-introduction/02-what-is-sql.md): you describe the shape of the result you want (including, if you care, its order); you never rely on incidental storage order to mean anything.

### Column Order Doesn't Matter Either — Mathematically

The same logic applies to attributes. In the strict mathematical relational model, a relation's attributes are also an unordered set — there is no formal concept of the "first" or "second" column. `{isbn, title, author, pages}` is exactly the same relation schema as `{title, pages, author, isbn}`; the *names* identify each attribute, not their position.

In practice, SQL is more forgiving of this theory than it is with rows: `CREATE TABLE` does fix a left-to-right column order, and `SELECT *` returns columns in that defined order, so column order is not entirely invisible day to day. But that order is a storage/definition detail, not a semantic guarantee you should ever *rely on* in code. Prove it to yourself:

```sql
SELECT title, author FROM books WHERE pages > 500;
```

```
                 title                  |      author
-----------------------------------------+-------------------
 Designing Data-Intensive Applications   | Martin Kleppmann
(1 row)
```

Here, `title` appears before `author` — the reverse of their order in the `CREATE TABLE` statement. The result's column order is entirely determined by what you list in `SELECT`, not by the table's underlying definition order. This is exactly the same point made in Module 01's [Your First Query](../01-introduction/05-your-first-query.md): "the output order is whatever you list in `SELECT`, not the table's underlying storage order" — this topic is simply giving you the theoretical reason *why* that's true, rather than just the observation that it is.

### A Full Worked Example

Putting it together — proving both that duplicate insertion order doesn't matter for correctness, and that a relation is really about the *set of facts it holds*, not a sequence:

```sql
CREATE TABLE color_swatches (
    name  TEXT,
    hex   TEXT
);

INSERT INTO color_swatches (name, hex) VALUES ('Crimson', '#DC143C');
INSERT INTO color_swatches (name, hex) VALUES ('Amber',   '#FFBF00');

-- Query it one way:
SELECT hex, name FROM color_swatches ORDER BY name;
```

```
   hex   |  name
---------+---------
 #FFBF00 | Amber
 #DC143C | Crimson
(2 rows)
```

Nothing about `color_swatches` "the relation" changed between the two `INSERT` statements and this `SELECT` — it holds exactly the same two facts throughout. All that changed is which columns were asked for, in which order, and whether a sort was requested. That separation — the relation as a fixed set of facts versus the query as a free choice of how to view them — is the single most important idea in this topic.

## Internal Working (Preview)

It's worth being honest about what PostgreSQL actually does under the hood, since it explains why row order can *appear* stable in small examples like the ones above, even though it is never guaranteed:

```
 CREATE TABLE books (...)
        │
        ▼
 PostgreSQL records the table's structure in its system catalog
 (pg_attribute stores each column's name, type, and its defined
  left-to-right position — this is where "column order" lives)
        │
        ▼
 INSERT statements append each new tuple into the table's
 physical storage (called a "heap" in PostgreSQL) — by default,
 simply after whatever was already there
        │
        ▼
 A plain SELECT (no ORDER BY) reads the heap using whatever
 access path the query planner chooses — often a straightforward
 top-to-bottom scan for a small table, which is why insertion
 order often "looks" preserved — but this is an implementation
 detail, not a guarantee, and changes as soon as an index,
 a different query plan, or storage maintenance (VACUUM) is involved
```

Column order *is* tracked by the system catalog (`pg_attribute`) and does have a real, defined position for a given table — that's why `SELECT *` and tools like `\d` show columns in a consistent order. What has no such guarantee is *row* order, and even column order should be treated as a display convenience, not something application logic should depend on positionally (always refer to columns by name).

## Real-World Analogy

Think of a relation like a box of index cards, one card per fact, all cards using the exact same printed template (same labeled fields in the same positions on the card). If you dump the box out on a table and pick the cards up again, you get the same *collection* of facts back — nothing was gained or lost — even though the physical order the cards happen to be sitting in changed completely. You'd never say "the fact about Amber is the second fact" as if that were a meaningful, permanent property of the data — you'd say "the card with `name = 'Amber'`," identifying it by its content, not its position in the stack. A relational table works the same way: identify rows by what they contain (their key values, covered in Module 5), never by an assumed position.

## Why This Design Was Chosen

If row and column order mattered logically, every query result would silently depend on incidental facts about how data happened to be stored or inserted — the opposite of the declarative, "what not how" principle established in Module 01. By defining a relation as a *set* of tuples over a *set* of named attributes, Codd's model deliberately strips away any notion of position as meaningful, which frees the database engine to store, reorganize, compress, index, and reorder data however is most efficient, without ever changing the logical meaning of a single query. This is the same logical/physical independence previewed in Module 01's [What Is a Database and a DBMS?](../01-introduction/01-what-is-a-database-and-a-dbms.md) and is explored formally, as one of Codd's original design goals, in the next topic.

## Advantages

- **Freedom for the engine to optimize** — because no query may rely on physical order, PostgreSQL is free to store, compress, reorganize, and access data however is fastest, without breaking any application's logic.
- **Queries stay stable as data grows** — a query that doesn't `ORDER BY` never silently "worked because of order" that later breaks when the table doubles in size or gets reindexed; the contract was always "no guaranteed order" from day one.
- **A clean, precise mental model** — thinking of a table as a set of facts, not a sequence, makes it far easier to reason correctly about what operations like `JOIN` (Module 10) and aggregation (Module 9) actually do, since both are fundamentally set operations.

## Disadvantages / Limitations

- **Counterintuitive for beginners used to spreadsheets or arrays** — most people's first mental model for "a grid of data" is a spreadsheet, where row 5 really is "row 5" and stays that way; unlearning that positional instinct takes deliberate practice.
- **Requires extra syntax for anything order-sensitive** — you must remember to add `ORDER BY` explicitly any time order matters to your application (e.g., "most recent first"); relying on default behavior is a subtle, easy-to-miss bug.
- **The gap between theory and practice around duplicate rows** — since a plain SQL table doesn't automatically enforce uniqueness the way a strict mathematical relation does, this power to store duplicates is occasionally exactly what you want (e.g., a log of every time an event occurred, where the "same" event repeating is meaningful) — the trade-off is that it removes a guarantee you might have assumed was automatic.

## Best Practices

- Never write application code, exercises, or queries that assume a specific row will always be "the first row" or "the last row" of an unordered result — always add an explicit `ORDER BY` if position matters at all.
- Never reference columns positionally in application code (`row[0]`, `row[1]`) when a name-based reference is available — column position is a definition detail, not a semantic guarantee, and inserting a new column later can silently shift positional references.
- When you need genuine, enforced uniqueness of rows (the true-relation property), don't assume it exists by default — explicitly declare a `PRIMARY KEY` or `UNIQUE` constraint (Module 5).

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `SELECT * FROM table_name;` (no `ORDER BY`) will always return rows in the same order it did last time. | Row order without `ORDER BY` is never guaranteed by the relational model or by PostgreSQL specifically — it can change as the table grows, gets indexed, or is reorganized by maintenance operations, even if the query text never changes. |
| Believing a plain table automatically refuses duplicate, byte-for-byte identical rows because "that's what a database does." | True relational theory says a relation shouldn't contain duplicate tuples, but plain SQL tables do not enforce this automatically — duplicate rows are entirely possible unless you add a `PRIMARY KEY` or `UNIQUE` constraint yourself (Module 5). |
| Referring to a column by its position in application code instead of by name. | A table's column order is a definition detail tracked in the system catalog, not a stable contract application logic should depend on — adding a new column later can silently break positional references. |

## Interview Questions

1. **Q: What are the formal relational-model terms for "table," "row," and "column," and what does each mean precisely?**
   A: "Table" is formally a **relation** — a named set of tuples sharing the same attributes. "Row" is formally a **tuple** — one complete entry, with one value per attribute. "Column" is formally an **attribute** — one named property every tuple has a value for, drawn from a defined **domain** (data type).

2. **Q: Does SQL guarantee the order rows come back in if you don't use `ORDER BY`?**
   A: No. A relation is conceptually a set, which has no inherent order. Without an explicit `ORDER BY`, the order PostgreSQL returns rows in is an implementation detail that can change based on storage, indexing, or query plan — it should never be relied upon.

3. **Q: Can a plain PostgreSQL table contain two completely identical rows?**
   A: Yes, unless a constraint prevents it. This is a genuine gap between pure relational theory (where a relation, being a set, cannot contain duplicate tuples by definition) and practical SQL, which permits duplicate rows by default. Enforcing true uniqueness requires an explicit `PRIMARY KEY` or `UNIQUE` constraint.

4. **Q: Why does the relational model treat row and column order as meaningless?**
   A: Because doing so lets the database engine freely choose how to physically store, index, compress, and reorganize data for performance, without ever changing what any query logically returns — this is the same logical/physical independence that underlies SQL's declarative design.

## Summary

- A **relation** (table) is a named set of **tuples** (rows), each holding one value for every **attribute** (column), with each attribute drawing values from a defined **domain** (data type).
- A valid relation requires uniquely named attributes, one fixed set of attributes per tuple, atomic (indivisible) attribute values, and — in pure theory — no duplicate tuples.
- Plain SQL does **not** automatically enforce that last property: duplicate rows are possible unless you add a key constraint (Module 5) — an important, honest gap between theory and practice, explored further in Topic 3.
- Row order is never guaranteed without an explicit `ORDER BY`; column order is fixed by the table's definition for display purposes (`SELECT *`) but should never be relied on positionally in application logic — always refer to columns by name.
- This "set of facts, not a sequence" mental model is exactly what makes SQL's declarative style (Module 01) coherent, and it underlies how joins and aggregation (Modules 9 and 10) are formally defined.
