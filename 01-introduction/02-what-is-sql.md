# What Is SQL?

## Learning Objectives

By the end of this section you should be able to:
- Give a precise definition of SQL and explain what "declarative" means
- Explain the relationship between the SQL standard and specific database products
- Correctly pronounce and expand the acronym, and know its origin

## Prerequisites

- [What Is a Database and a DBMS?](01-what-is-a-database-and-a-dbms.md) — you need the concept of an RDBMS before "a language for talking to an RDBMS" means anything.

## Motivation

You're about to spend an entire course learning a language. Before learning its grammar, you should know *what kind* of language it is — because that shapes how you should think about every statement you write from here on. SQL is unlike most general-purpose programming languages you may have encountered, and that difference is the single most important thing to internalize before writing your first query.

## Problem Statement

Suppose you want "all employees in the Sales department, sorted by salary, highest first." In a general-purpose programming language, you'd typically have to write out the *steps*: open the data source, loop through every record, check if its department field equals "Sales," collect the matches into a list, sort that list by salary, then return it.

Now imagine the underlying data is one million rows. Should you loop through every row every time? Should you build some kind of lookup structure first? Should the sort happen before or after filtering? A general-purpose imperative approach forces *you*, the person asking the question, to also be the one who decides *how* to answer it efficiently — and that decision changes depending on data size, hardware, and how the data happens to be stored today.

## Concept

### Definition

> **SQL (Structured Query Language)** is a **declarative**, special-purpose language used to define, manipulate, and query data held in a relational database.

"SQL" is typically pronounced either as the letters "S-Q-L" or as the word "sequel" — both are common and accepted; the "sequel" pronunciation is a holdover from SQL's predecessor, a language called SEQUEL, developed at IBM in the early 1970s based directly on Codd's relational model.

### Declarative vs. Imperative

This is the single most important distinction to understand about SQL.

| | Imperative | Declarative |
|---|---|---|
| **You specify** | The exact steps to compute the result | *What* result you want |
| **The system figures out** | Nothing — it just does what you said, in order | *How* to compute it efficiently |
| **Example instruction** | "Open the file, loop through each line, if department equals Sales add it to a list, then sort that list by salary descending" | "Give me all employees where department is Sales, ordered by salary descending" |
| **SQL analogy** | — | `SELECT * FROM employees WHERE department = 'Sales' ORDER BY salary DESC;` |

In SQL, you never tell the database *how* to fetch your data — no loops, no "check this row, then that row." You describe the **shape of the result you want**, and the DBMS's **query planner/optimizer** decides the actual retrieval strategy (which we'll open up properly in Module 20 — Performance Tuning). This is the same "what, not how" separation introduced in Topic 1 — it is the defining trait of the entire relational model, and SQL is simply the language built to express it.

### The SQL Standard vs. Real Databases

SQL is standardized by ANSI (American National Standards Institute) and ISO (International Organization for Standardization) — the first standard was published in 1986, and it has been revised many times since (SQL:1992, SQL:1999, SQL:2003, SQL:2016, SQL:2023, and others).

However, no real database implements the standard 100% exactly, and every vendor adds its own extensions on top. This is precisely analogous to how different countries drive on the same basic principle (a road, lanes, traffic lights) but differ in details (which side of the road, specific sign shapes). Because of this:

- The core you'll learn in this course (`SELECT`, `WHERE`, `JOIN`, `GROUP BY`, and so on) is standard SQL and works essentially identically across PostgreSQL, MySQL, SQL Server, Oracle, and SQLite.
- Some features (certain functions, certain syntax for pagination, procedural extensions) differ by vendor. This course uses **PostgreSQL** as its reference database (explained in the root README), and Module 22 is dedicated entirely to mapping out where other databases diverge.

### What SQL Is Used For

SQL statements fall into several purposes — previewed here, covered in full in Topic 3 of this module:

- **Defining** the structure of data (creating tables, defining columns) — Module 4.
- **Manipulating** data (inserting, updating, deleting rows) — Module 6.
- **Querying** data (asking questions and retrieving results) — Modules 7 through 17.
- **Controlling access** to data (granting/revoking permissions) — Module 19.
- **Controlling transactions** (making a group of changes succeed or fail together) — Module 14.

## Internal Working (Preview)

When you type a SQL statement and run it, roughly this happens inside the DBMS:

```
 Your SQL text
      │
      ▼
   Parser            (checks syntax is valid SQL)
      │
      ▼
 Query Planner /      (decides the most efficient execution strategy —
 Optimizer             e.g., "use this index" vs. "scan the whole table")
      │
      ▼
 Execution Engine     (actually reads/writes the data, following the plan)
      │
      ▼
   Result set
```

You write *what* you want (the SQL text). Steps 2 and 3 — deciding *how* — are entirely the DBMS's job, and are invisible to you unless you deliberately inspect them (Module 13 and Module 20 teach you how, using `EXPLAIN`).

## Real-World Analogy

Think of SQL like ordering food from a restaurant menu, versus imperative code like cooking the meal yourself.

- When you order "a medium-rare steak with fries," you describe the **outcome** you want. You don't tell the kitchen which pan to use, what temperature to preheat the oven to, or in what order to season it — that's the kitchen's job, and different kitchens (different DBMSs) might use slightly different techniques to arrive at the same dish.
- If you had to cook it yourself (imperative), you'd need to know every step, and if the kitchen's equipment changed (different hardware, different data volume), you'd have to rewrite your entire process.
- SQL is you ordering from the menu. The DBMS's query planner is the kitchen deciding how to actually produce what you asked for.

## Why SQL Was Designed This Way

SQL's declarative design traces directly back to Codd's relational model (Topic 1): if data access is described logically rather than physically, the same query keeps working correctly even as the underlying storage, indexing strategy, hardware, or data volume changes dramatically over the years. This was a deliberate bet in the 1970s that paid off enormously — a `SELECT` statement written for a 1980s mainframe database with a few thousand rows is, syntactically, nearly identical to one written today against a table with a billion rows, even though the actual retrieval mechanics underneath are entirely different.

## Advantages of This Design

- **Readable and close to natural language** — `SELECT name FROM employees WHERE department = 'Sales'` is almost self-explanatory to a non-programmer.
- **Portable core knowledge** — the fundamentals transfer across virtually every relational database product.
- **Optimizer improvements benefit you for free** — as the DBMS's query planner gets smarter over software versions, your unchanged SQL queries can get faster without you rewriting anything.
- **Separates intent from mechanism** — you can reason about *what data you need* without simultaneously reasoning about *how to fetch it efficiently*.

## Disadvantages of This Design

- **Less direct control** — if the query planner makes a poor decision, you can't simply "tell it what to do" the way you'd control an imperative loop; you have to learn how to *influence* it (indexes, query rewriting, hints in some databases) — this is a large part of Module 20.
- **Vendor extensions fragment portability** — the moment you use a vendor-specific feature, your SQL is no longer portable to another database without changes (Module 22).
- **Debugging performance requires new skills** — because you don't write the retrieval steps yourself, diagnosing *why* a query is slow requires learning to read the DBMS's own execution plan (Module 13, Module 20), which is a different skill than debugging a loop.

## Best Practices

- When writing a query, always ask yourself "what result do I want?" rather than "how would I compute this step by step?" — falling back into imperative thinking is the most common way beginners write unnecessarily complicated SQL.
- Learn the standard core deeply before learning any one vendor's extensions — it's the 90% of SQL knowledge that transfers everywhere.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| "SQL is a programming language like Python or Java." | SQL is a special-purpose declarative language for relational data, not a general-purpose imperative/object-oriented language — it lacks general control flow as a core feature (though procedural extensions exist, covered in Module 18). |
| "If I write standard SQL, it'll run identically everywhere with zero changes." | Mostly true for the core (this course's focus) but not guaranteed for every feature — see Module 22 for the specific gaps. |
| "SQL tells the database exactly how to fetch data." | SQL describes *what* you want; the query planner/optimizer inside the DBMS decides *how* — this is the entire point of it being declarative. |

## Interview Questions

1. **Q: What does "SQL is declarative" mean, and why does it matter?**
   A: It means you describe *what* data you want, not the step-by-step procedure to retrieve it. The DBMS's query optimizer decides the actual execution strategy. This matters because your queries stay valid and can get faster automatically as the optimizer improves or as data grows, without you rewriting retrieval logic.

2. **Q: Is SQL standardized? Does that mean every database behaves identically?**
   A: Yes, SQL is standardized by ANSI/ISO, and the core (SELECT, WHERE, JOIN, GROUP BY, etc.) behaves near-identically across major relational databases. However, every vendor adds proprietary extensions and has small behavioral differences, so 100% portability isn't guaranteed — only the standardized core is reliably portable.

3. **Q: What is the practical difference between SQL and a language like Python, from a database-interaction standpoint?**
   A: Python (imperative) requires you to write the logic to loop through, filter, and sort data yourself. SQL (declarative) requires you to only specify the conditions and shape of the desired result; the DBMS performs the retrieval and computation internally.

## Summary

- **SQL** is a declarative, special-purpose language for defining, manipulating, and querying relational data.
- Declarative means you specify *what* you want; the DBMS's query planner decides *how* to get it — a direct consequence of the relational model's logical/physical separation.
- SQL is standardized (ANSI/ISO), but real databases add their own extensions — this course teaches the portable standard core using PostgreSQL as its concrete reference.
- Next, Topic 3 breaks down the different *categories* of SQL statements (DDL, DML, DQL, DCL, TCL) you'll be learning throughout this course.
