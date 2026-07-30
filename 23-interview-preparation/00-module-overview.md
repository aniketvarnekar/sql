# Module 23 — Interview Preparation

## Module Goal

By the end of this module, you will be able to walk into a SQL interview and perform at the level your first 22 modules actually support — not just know the material, but retrieve it fast, explain it out loud in interview-length answers, write correct queries under time pressure, reason through a schema-design prompt from a blank page, and narrate your thinking the way an interviewer is actually listening for. This module doesn't teach any new SQL feature; it is entirely retrieval practice, applied practice, and presentation practice over everything Modules 01 through 22 already gave you.

## Topics Covered in This Module

1. **[Core Conceptual Questions — Ranked & Synthesized](01-core-conceptual-questions-ranked.md)** — the highest-signal conceptual questions from across the whole course, reorganized by theme instead of module order, each with an interview-length model answer.
2. **[Query-Writing Problems](02-query-writing-problems.md)** — realistic query-writing prompts of increasing difficulty, each with a small schema, a plain-English problem statement, and a complete worked PostgreSQL solution.
3. **[Schema Design Questions](03-schema-design-questions.md)** — realistic "design a schema for X" prompts, each worked from entity identification through a full set of `CREATE TABLE` statements with proper keys and constraints.
4. **[Mock Interview Walkthrough](04-mock-interview-walkthrough.md)** — a full narrated mock interview transcript, showing what strong out-loud reasoning through a query problem actually sounds like, plus guidance on presenting a solution.
5. **[Module Summary](05-module-summary.md)** — Consolidated recap, and a note on how to use the rest of this course going forward.

## Prerequisites

This module assumes you have completed **Modules 01 through 22, in full**. It does not introduce new syntax or new concepts — every question, query, and schema in this module draws directly on material already covered:

- **Module 05 (Constraints & Keys)** and **Module 15 (Normalization & Design)** are the two modules leaned on hardest in the schema-design topic — you need to already be comfortable identifying primary keys, foreign keys, and normal-form violations before you can defend a schema design under interview questioning.
- **Module 10 (Joins & Set Operations)**, **Module 11 (Subqueries)**, **Module 16 (Window Functions)**, and **Module 17 (CTEs & Recursion)** are the four modules the query-writing topic draws on most heavily — the problems in that topic are deliberately chosen so that several distinct techniques from those four modules are each the "right" tool for at least one problem.
- **Module 13 (Indexes)**, **Module 14 (Transactions & Concurrency)**, **Module 19 (Security & Access Control)**, and **Module 20 (Performance Tuning)** supply the conceptual questions in this module that go beyond query syntax into how a database actually behaves under load, under concurrent access, and under a hostile input.

## How to Study This Module

Do not read this module passively. Topic 1 is designed for active recall — cover the model answer, answer out loud from memory, then check yourself; reading straight through defeats the entire purpose. Topic 2 and Topic 3 should be attempted with a pen or a blank editor before you look at the worked solution — write your own query or your own schema first, then compare. Topic 4 is best read once straight through to absorb the pattern of narrated reasoning, and then used as a model the next time you rehearse a mock interview of your own, out loud, ideally with another person or a recording. Treat this module as the last rehearsal before the real thing, not as new material to learn.
