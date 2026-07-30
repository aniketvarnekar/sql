# The Relational Model and Codd's Rules

## Learning Objectives

By the end of this section you should be able to:
- Explain the historical significance of Edgar Codd's 1970 paper and the problem it solved
- Explain why a relation is formally a mathematical set of tuples, and demonstrate concretely how plain SQL tables diverge from that in practice
- Summarize, in plain English, the intent behind Codd's twelve rules — especially guaranteed access, systematic `NULL` handling, and physical/logical data independence
- Give concrete examples of where SQL (and PostgreSQL specifically) only approximates the pure relational model, rather than fully implementing it

## Prerequisites

- [Tables, Rows, and Columns](01-tables-rows-and-columns.md) — you need the relation/tuple/attribute vocabulary this topic builds directly on top of.
- [What Is a Database and a DBMS?](../01-introduction/01-what-is-a-database-and-a-dbms.md) — you need the working definition of "relational database" this topic now traces back to its actual historical and theoretical origin.
- [What Is SQL?](../01-introduction/02-what-is-sql.md) — the declarative, "what not how" framing established there is exactly what several of Codd's rules formalize.

## Motivation

It's entirely possible to use PostgreSQL productively for years without ever hearing the name "Codd" or reading a single one of his rules. But without this theoretical background, certain SQL behaviors look like arbitrary quirks or even bugs: why does a table let you insert two completely identical rows? Why does a `UNIQUE` column let you insert `NULL` twice? Why does everyone call SQL databases "relational" so confidently, when in practice they bend so many of the rules that supposedly define what "relational" even means? This topic gives you the actual historical and theoretical grounding to answer all three questions precisely, instead of shrugging them off as "just how SQL is."

## Problem Statement

Picture two very different, equally real scenarios. First: it's the early 1970s, and databases are "navigational" — to fetch a piece of data, a program has to know the exact physical chain of pointers linking one record to the next, essentially hard-coding knowledge of *how* the data happens to be stored today into every single piece of application logic that touches it. Second: it's the mid-1980s, and "relational" has become such a popular marketing term that vendors are slapping it onto products that share almost nothing structurally with the rigorous model the term was supposed to describe — a customer evaluating database products has no reliable way to tell a genuinely relational system from one merely using the word for its appeal. Both problems needed a precise, checkable answer to the same underlying question: what, exactly, does it mean for a database to be "relational"? Edgar F. Codd answered that question twice, fifteen years apart, and both answers are the subject of this topic.

## Concept

### The 1970 Paper: *A Relational Model of Data for Large Shared Data Banks*

In 1970, Edgar F. Codd, a researcher at IBM, published a paper proposing that data should be modeled as **mathematical relations** — sets of tuples over named attributes, as formalized in Topic 1 — and queried using operations grounded in set theory and predicate logic (relational algebra and relational calculus), rather than navigated via explicit, hand-followed pointers between records.

This was a genuinely radical proposal at the time. The dominant database designs of the era (hierarchical and network/"navigational" models) required application code to understand and traverse the physical structure of the data directly — follow this pointer, then that one, in this specific order — meaning any change to how data was physically organized could break every application built on top of it. Codd's insight, previewed in Module 01, was to separate **what** data you want from **how** it's physically stored and retrieved — the same logical/physical independence that makes SQL declarative today. This 1970 paper is the theoretical foundation of every relational database, including PostgreSQL, and of SQL itself, which grew directly out of an IBM research project (originally called SEQUEL) built specifically to implement Codd's ideas as a practical query language.

### Relations as Mathematical Sets — and Where SQL Diverges

Formally, a relation is defined as a subset of the Cartesian product of its attributes' domains — in plainer terms: a relation is a **set** of tuples. This single word, "set," carries a precise mathematical consequence that Topic 1 already flagged: a set, by definition, cannot contain the same element twice. Two identical tuples aren't "two rows that happen to match" — in true relational theory, they're the same element, so there is only ever one of it.

This is a good moment to be completely honest about a real gap between theory and the SQL you actually run. A plain SQL table is not automatically a "relation" in this strict sense — it behaves more like a mathematical **bag** (or multiset), which *is* allowed to contain duplicate elements. Watch this directly:

```sql
CREATE TABLE contacts (
    name   TEXT,
    phone  TEXT
);

INSERT INTO contacts (name, phone) VALUES ('Priya', '555-0101');
INSERT INTO contacts (name, phone) VALUES ('Priya', '555-0101');

SELECT * FROM contacts;
```

```
 name  |   phone
-------+-----------
 Priya | 555-0101
 Priya | 555-0101
(2 rows)
```

PostgreSQL happily stores two entirely indistinguishable rows here — something a true mathematical relation, being a set, cannot contain by definition. Nothing about `contacts` as defined above tells the database these two rows should be treated as the same fact; SQL's default table behavior is bag semantics, not set semantics. To force genuinely relational, set-like behavior — no duplicates permitted — you have to opt in explicitly, typically with a `PRIMARY KEY` or `UNIQUE` constraint (Module 5):

```sql
CREATE TABLE contacts_v2 (
    name   TEXT,
    phone  TEXT,
    UNIQUE (name, phone)
);

INSERT INTO contacts_v2 (name, phone) VALUES ('Priya', '555-0101');
INSERT INTO contacts_v2 (name, phone) VALUES ('Priya', '555-0101');
```

```
ERROR:  duplicate key value violates unique constraint "contacts_v2_name_phone_key"
DETAIL:  Key (name, phone)=(Priya, 555-0101) already exists.
```

Only once a uniqueness constraint exists does `contacts_v2` actually behave like a true mathematical relation with respect to duplicate tuples. This is the central, honest nuance of this topic: **SQL calls these things "tables," and they're close cousins of relations, but a bare SQL table is not automatically a relation in the strict theoretical sense — it becomes one only once you add the constraints that force set-like uniqueness.**

### The 1985 Rules: Codd's Twelve-Point Test for "Relational"

By the mid-1980s, "relational" had become exactly the kind of loosely-used marketing term described in the Problem Statement — enough vendors were applying it to products that weren't genuinely built on Codd's model that the term risked losing all technical meaning. In response, Codd published two articles in *Computerworld* in 1985 ("Is Your DBMS Really Relational?" and "Does Your DBMS Run By the Rules?") laying out a concrete, checkable set of rules a system would have to satisfy to legitimately be called relational. There are technically thirteen rules, numbered 0 through 12 — "Rule Zero" plus twelve more — which is why they're universally referred to as "Codd's twelve rules."

| Rule | Name | Plain-English intent |
|---|---|---|
| 0 | Foundation Rule | The system must manage its data entirely through relational capabilities — you can't bolt a relational-looking front end onto a fundamentally non-relational engine underneath. |
| 1 | Information Rule | All data — including metadata about tables and columns — is represented in exactly one way: as values sitting in tables. No hidden, non-tabular representation of information is allowed. |
| 2 | Guaranteed Access Rule | Every single data value must be reachable by specifying a table name, a primary key value, and a column name — no data should be accessible *only* by some other, non-logical means (like a physical file offset). |
| 3 | Systematic Treatment of Null Values | Missing or inapplicable data must be represented by one consistent, systematic marker (`NULL`), handled uniformly across every data type, and clearly distinguished from any real value (including the number zero or an empty string). |
| 4 | Dynamic Online Catalog Based on the Relational Model | The database's own description of itself — what tables and columns exist — must be stored as ordinary relational data, queryable with the same query language used for regular data. |
| 5 | Comprehensive Data Sublanguage Rule | There must be one well-defined language supporting data definition, data manipulation, integrity constraints, and transaction control, usable both interactively and from within application code. |
| 6 | View Updating Rule | Any view that is theoretically capable of being updated (i.e., the update is unambiguous) must actually be updatable by the system, not just readable. |
| 7 | High-Level Insert, Update, and Delete | Insert, update, and delete operations must work on whole sets of rows at once, not merely one row at a time. |
| 8 | Physical Data Independence | Changing how data is physically stored or accessed (file layout, indexing strategy) must never require changing the applications or queries that use it. |
| 9 | Logical Data Independence | Changes to a table's logical structure that don't remove information an application actually uses (e.g., adding a new column) should not force that application's existing queries to change. |
| 10 | Integrity Independence | Integrity rules (like "this value must be unique" or "this value can't be negative") must be definable within the relational language itself and stored centrally, not hard-coded into every application that touches the data. |
| 11 | Distribution Independence | Applications and queries shouldn't need to know or care whether the underlying data is stored on one machine or spread across several. |
| 12 | Nonsubversion Rule | If the system offers any lower-level, record-at-a-time way of accessing data, that lower-level path must not be usable to bypass the integrity rules enforced by the higher-level relational language. |

A few of these deserve a closer look, because they connect directly to ideas already introduced in this course:

- **Rule 2 (Guaranteed Access)** is the formal justification for why every table *should* have a way to uniquely identify each row — a primary key — so that any single value in the database is always reachable by naming a table, a key value, and a column, rather than by some indirect, positional, or storage-specific means. This is exactly what Module 5 (Constraints & Keys) teaches you to declare.
- **Rule 3 (Systematic `NULL` Handling)** insists that missing data be represented one consistent way, everywhere, distinguishable from any legitimate value — including zero, an empty string, or `false`. Module 3 (Data Types) covers `NULL` and its three-valued logic in depth; this topic returns to it below, because it's also where SQL's real-world implementation visibly falls short of Codd's ideal.
- **Rules 8 and 9 (Physical and Logical Data Independence)** are the formal statement of the "what, not how" theme running through this entire course since Module 01 — the guarantee that your queries keep working correctly as storage details change (Rule 8) or as a table's structure evolves in backward-compatible ways, like adding a column (Rule 9).
- **Rule 4 (Dynamic Online Catalog)** is exactly what you already saw in Topic 2 of this module: querying `information_schema.schemata` and `pg_tables` with ordinary `SELECT` statements to inspect the database's own structure — the catalog is data, queryable the same way any other data is.

### Why No Real Database — Including PostgreSQL — Fully Satisfies All Twelve Rules

Codd's own conclusion, even in 1985, was that not a single commercial product on the market at the time satisfied all twelve rules completely — and that remains true of essentially every SQL database in use today, PostgreSQL included. This isn't a knock against PostgreSQL specifically; it's a reflection of the fact that Codd's rules describe an idealized theoretical target, while real databases make pragmatic engineering trade-offs. A few concrete, checkable examples:

**Rule 2 assumes every relation has a way to guarantee access via a key — but SQL doesn't require a primary key.** You can create and use a perfectly ordinary table with no primary key at all:

```sql
CREATE TABLE scratch_notes (
    note TEXT
);
```

This is completely legal SQL, but it means some rows may have no way to be uniquely, logically addressed at all if their `note` values happen to duplicate — a direct, permitted violation of Rule 2's guarantee, left entirely as a design choice for you to opt into (via `PRIMARY KEY`, Module 5) rather than enforced by the system.

**Rule 3 wants one systematic, consistent treatment of `NULL` — but SQL's `NULL` behaves inconsistently in a specific, well-known way around uniqueness.** Consider:

```sql
CREATE TABLE users (
    id     SERIAL PRIMARY KEY,
    email  TEXT UNIQUE
);

INSERT INTO users (email) VALUES (NULL);
INSERT INTO users (email) VALUES (NULL);

SELECT * FROM users;
```

```
 id | email
----+-------
  1 |
  2 |
(2 rows)
```

Both inserts succeed, even though `email` is declared `UNIQUE`. This is because SQL's `NULL` represents "unknown," and by the rules of SQL's three-valued logic, `NULL = NULL` never evaluates to true — it evaluates to *unknown* — so the `UNIQUE` constraint never considers two `NULL`s to be "the same value" worth rejecting. Contrast that with ordinary equality checks in a `WHERE` clause, where this exact same "unknown" behavior means `WHERE email <> NULL` never matches any row at all, including rows where `email` actually is `NULL` — you must use `IS NULL` instead. `NULL` is treated consistently in the sense that it always represents "unknown" — but that consistency itself produces behavior (multiple "duplicate" `NULL`s allowed in a `UNIQUE` column) that doesn't match how most people intuitively expect "systematic" and "duplicate-free" to behave together, and is a frequently cited example of SQL's imperfect realization of Rule 3.

**Rule 6 (View Updating) is only partially honored.** A simple view over a single table is usually updatable in PostgreSQL, but a view involving a `JOIN`, an aggregate function, or `GROUP BY` (Module 12 covers views in depth) generally is *not* automatically updatable — PostgreSQL raises an error if you try to `INSERT` or `UPDATE` through such a view directly, even in cases where a human could reasonably figure out the "obviously correct" underlying change. Rule 6 demands the system support this whenever it's theoretically unambiguous; real implementations support only a subset of those cases.

### SQL as an Approximation, Not a Perfect Implementation

Putting all of this together: SQL and the databases that implement it (PostgreSQL very much included) are best understood as **engineering approximations** of Codd's relational model, not perfect, literal implementations of it. SQL borrows the vocabulary (tables, rows, columns as the practical names for relations, tuples, attributes), the core theoretical grounding (set-based operations, a declarative query language), and the spirit of several rules (a queryable system catalog, a comprehensive data sublanguage) — but it deliberately or historically diverges from strict compliance in specific, well-understood ways: tables permit duplicate rows unless you add a key; `NULL` interacts with uniqueness in a way that surprises newcomers; not every view is updatable; not every table has a guaranteed-unique key. None of this makes SQL databases "not really relational" in any practically meaningful sense — the term stuck industry-wide as the accepted, if imperfect, label for this entire family of systems — but understanding exactly *where* the approximation shows its seams is what lets you predict and reason about SQL's behavior in the exact cases (duplicate rows, `NULL` and `UNIQUE`, non-updatable views) that otherwise look like arbitrary inconsistencies.

## Internal Working (Preview)

Rule 4 (the dynamic, queryable catalog) is worth seeing directly, since it's the one rule you can inspect on your own system with total confidence PostgreSQL upholds it faithfully:

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;
```

```
 table_schema |    table_name
--------------+-------------------
 billing      | invoices
 public       | books
 public       | color_swatches
 public       | contacts
 public       | contacts_v2
 public       | scratch_notes
 public       | users
 sales        | invoices
(8 rows)
```

This query is asking the database a question about its *own structure* using the exact same `SELECT` syntax you'd use to ask a question about any ordinary business data — because, per Rule 4, that structural information genuinely *is* ordinary relational data underneath, stored in tables like `pg_class` and `pg_namespace` and exposed through the standardized `information_schema` views. There is no separate, special-purpose "metadata language" you need to learn — the same relational query language reaches all the way down into the database describing itself.

## Real-World Analogy

Think of Codd's twelve rules like the strict legal definition of "Champagne." Only sparkling wine produced in the Champagne region of France, under a specific set of methods, is legally permitted to be called Champagne — anything else, however similar, must be labeled "sparkling wine." Codd's rules were his attempt to draw an equally strict line around the word "relational": a genuine test, not just a vibe, for whether a database earns the label. In practice, though, "relational database" never stayed as strictly policed a term as "Champagne" — the word spread across the entire industry as a loose, widely understood label for "a database built on tables, keys, and SQL," even though, by Codd's own strict twelve-point test, virtually no commercial product — including the one you're using in this course — checks every single box.

## Why This Model Was Designed This Way

The relational model exists to solve exactly the two problems in the Problem Statement: replace physical, pointer-based data access with a logical, declarative one (the 1970 paper), and give the industry a rigorous, checkable definition of what "relational" actually requires, in response to the term being diluted by marketing (the 1985 rules). Rules 8 and 9 (physical and logical data independence) in particular are the direct theoretical ancestor of everything Module 01 told you about SQL being declarative — you describe *what* you want, and the database is free to change *how* it stores and retrieves data underneath you, precisely because Codd insisted, in rigorous rule form, that this independence was a non-negotiable property of a genuinely relational system.

## Advantages

- **A rigorous, vendor-neutral theoretical foundation** — because the relational model is grounded in set theory and predicate logic rather than any one vendor's implementation choices, its core ideas (declarative queries, keys, logical independence) transfer across every relational database you'll ever use.
- **A concrete, checkable definition of "relational"** — Codd's twelve rules give you a genuine yardstick, rather than marketing language, for evaluating how closely any given system actually adheres to the model.
- **Explains, rather than just describes, SQL's behavior** — understanding the theory turns SQL quirks (duplicate rows, `NULL` and `UNIQUE`) from mysterious inconsistencies into predictable, explainable consequences of exactly where SQL approximates rather than perfectly implements the model.

## Disadvantages / Limitations

- **The rules were written before modern distributed and non-relational systems existed** — Rule 11 (distribution independence), for instance, reads very differently in a world of horizontally-sharded and globally-distributed databases than it did in 1985, and the rules don't attempt to address the trade-offs those modern systems intentionally make.
- **Strict compliance can be seen as an academic ideal rather than a practical requirement** — no widely used database fully satisfies all twelve rules, and that hasn't stopped the relational model from being extraordinarily successful in practice; treating the rules as a hard pass/fail checklist for real-world system choices would be the wrong takeaway.
- **The permissive, bag-like default behavior of SQL tables (allowing duplicates) is sometimes a genuine feature, not just a gap** — an event log table that intentionally records "this exact event happened again" benefits from not being forced into strict, key-enforced uniqueness by default; the theoretical "flaw" is, in some designs, exactly the flexibility you want.

## Best Practices

- Always add an explicit `PRIMARY KEY` or `UNIQUE` constraint when you actually need a table to behave like a true relation with respect to uniqueness — don't assume SQL gives you that automatically, because by default it doesn't (Module 5).
- Be deliberate about `NULL` in any column with a `UNIQUE` constraint — know that PostgreSQL allows multiple `NULL`s in a `UNIQUE` column by default, and if that's not the behavior you want, you need additional logic (such as a partial unique index, covered alongside indexing in Module 13) rather than assuming the constraint alone prevents it.
- Use `information_schema` and PostgreSQL's system catalog views to introspect your own database's structure programmatically (for tooling, documentation, or validation scripts) rather than hardcoding assumptions about what tables or schemas exist — this is Rule 4 in direct, practical action.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "My table is relational, so PostgreSQL automatically won't let me insert duplicate rows." | Plain SQL tables permit duplicate rows by default (bag semantics); true set-like uniqueness only exists once you add a `PRIMARY KEY` or `UNIQUE` constraint yourself. |
| "A `UNIQUE` column should reject a second `NULL`, since it already has one `NULL` in it." | SQL's three-valued logic means `NULL = NULL` is never considered true (it's "unknown"), so a `UNIQUE` constraint does not treat two `NULL`s as duplicates of each other — multiple `NULL`s are permitted by default. |
| "Any database that has tables and supports `JOIN` is fully Codd-compliant." | Having tables and joins is necessary but nowhere near sufficient — most relational databases, PostgreSQL included, only partially satisfy Codd's twelve rules (e.g., primary keys are optional, and not every view is automatically updatable). |
| "Codd's twelve rules and Codd's 1970 relational model paper are the same publication." | The 1970 paper introduced the relational model itself; the twelve rules were published fifteen years later, in 1985, specifically as a practical test to distinguish genuinely relational products from those merely marketed as such. |

## Interview Questions

1. **Q: What is the historical relationship between Codd's 1970 paper and his 1985 twelve rules?**
   A: The 1970 paper, "A Relational Model of Data for Large Shared Data Banks," introduced the relational model itself — data as mathematical relations, queried declaratively instead of navigated via physical pointers. The twelve rules, published in 1985 across two *Computerworld* articles, were a later, separate contribution: a concrete, checkable test for whether a database product genuinely implements that model, written in response to vendors loosely marketing non-relational products as "relational."

2. **Q: Why is a plain SQL table not technically a "relation" in the strict mathematical sense until a key constraint is added?**
   A: A relation is formally defined as a *set* of tuples, and a set cannot contain duplicate elements by definition. A plain SQL table permits fully duplicate rows unless a `PRIMARY KEY` or `UNIQUE` constraint is added, meaning it behaves like a bag (multiset) rather than a true set until that constraint forces genuine uniqueness.

3. **Q: Name two of Codd's twelve rules and explain their intent in plain English.**
   A: Rule 2 (Guaranteed Access) requires that every value in the database be reachable by specifying a table name, a key value, and a column name — never only through some indirect, physical means. Rule 8 (Physical Data Independence) requires that changes to how data is physically stored or accessed never force changes to the applications or queries built on top of it — the same "what, not how" separation that makes SQL declarative.

4. **Q: Give a concrete example of where SQL's `NULL` handling deviates from Codd's vision of "systematic" treatment.**
   A: A column declared `UNIQUE` will accept multiple `NULL` values, because SQL's three-valued logic never treats `NULL = NULL` as true (it evaluates to unknown), so the uniqueness check never flags two `NULL`s as duplicates of each other — a behavior that surprises many newcomers expecting `UNIQUE` to mean "no two rows can look the same," `NULL`s included.

## Summary

- Edgar Codd's **1970 paper** introduced the relational model itself: data as mathematical relations, accessed declaratively rather than navigated via physical pointers — the theoretical root of every relational database, including PostgreSQL.
- Codd's **1985 twelve rules** (Rule 0 through Rule 12) were a later, separate contribution — a concrete, checkable test for genuinely relational systems, published in response to the term "relational" being diluted by marketing.
- A true relation is a mathematical *set* of tuples, meaning duplicates are impossible by definition — but plain SQL tables permit duplicate rows by default (bag semantics) unless you explicitly add a `PRIMARY KEY` or `UNIQUE` constraint (Module 5).
- No mainstream database, PostgreSQL included, satisfies all twelve rules completely — optional primary keys, `NULL`'s interaction with `UNIQUE`, and partially-updatable views are all concrete, demonstrable examples of where SQL only approximates Codd's ideal.
- Despite this gap, "relational database" remains the accepted industry term for this entire family of systems — understanding exactly where the approximation shows its seams is what turns SQL's apparent quirks into predictable, explainable behavior.
