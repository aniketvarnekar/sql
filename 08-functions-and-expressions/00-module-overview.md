# Module 08 — Functions and Expressions

## Module Goal

By the end of this module, you will be able to transform and compute values *inside* a query instead of just retrieving them as stored — cleaning up messy text, doing arithmetic with the correct result types, working with dates and times, branching logic with `CASE`, and converting between types while handling missing (`NULL`) data safely. Every realistic report, dashboard, or data export leans on these tools constantly: raw stored values are rarely exactly what a human or another system needs to see, and functions and expressions are how SQL bridges that gap without ever leaving the query itself.

## Topics Covered in This Module

1. **[String Functions](01-string-functions.md)** — concatenating, measuring, trimming, case-converting, extracting, replacing, splitting, and locating text within string values.
2. **[Numeric Functions](02-numeric-functions.md)** — rounding, ceiling/floor, absolute value, modulo, exponentiation, arithmetic operators, and the crucial difference between integer and numeric division.
3. **[Date and Time Functions](03-date-and-time-functions.md)** — getting the current date/time, extracting date parts, truncating to a time bucket, arithmetic with intervals, computing ages, and formatting dates for display.
4. **[CASE Expressions](04-case-expressions.md)** — conditional logic inside a query: simple vs. searched `CASE`, using it in `SELECT`, `WHERE`, and `ORDER BY`.
5. **[Casting and NULL-Handling Functions](05-casting-and-null-handling-functions.md)** — `CAST`/`::`, implicit vs. explicit conversion, and the two workhorse `NULL`-handling functions, `COALESCE` and `NULLIF`.
6. **[Module Summary](06-module-summary.md)** — Consolidated recap.

## Prerequisites

- **Module 7 — Querying Basics.** This entire module assumes you are already comfortable writing `SELECT`, filtering with `WHERE`, and sorting with `ORDER BY`. Every function and expression in this module is *used inside* those clauses — you'll be calling functions in the column list of a `SELECT`, inside a `WHERE` condition, and inside an `ORDER BY` expression throughout. If the mechanics of those three clauses aren't solid yet, the *placement* of a function call will be confusing even if the function itself is simple.
- **Module 3 — Data Types.** Functions in SQL are strongly type-specific: string functions expect text-like input, numeric functions expect numbers, date functions expect date/time values. You need to already know what `TEXT`, `INTEGER`, `NUMERIC`, `DATE`, and `TIMESTAMP` mean and how they differ before this module's function catalog will make sense. You also need Module 3's coverage of `NULL` — that a column can hold "no value at all," and that `NULL` propagates through most expressions — because Topic 5 of this module exists specifically to give you tools for handling it.

## How to Study This Module

Topics 1 through 3 (string, numeric, date/time functions) are best treated as a **reference toolkit**: read through each once so you know what exists and roughly how it behaves, but don't feel obligated to memorize every function signature on a first pass — you will come back to these pages repeatedly as you write real queries later in the course. Topics 4 and 5 (`CASE` expressions, and casting/`NULL`-handling) deserve closer, slower reading and re-reading: they are not just a catalog of functions but genuine *conceptual* tools you will reach for in nearly every query from here on — every later module in this course (aggregation, joins, subqueries, window functions, reporting queries of any kind) leans on `CASE`, `COALESCE`, and `NULLIF` constantly. Getting comfortable with them now pays off for the rest of the course.
