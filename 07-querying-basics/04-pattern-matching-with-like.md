# Pattern Matching with LIKE

## Learning Objectives

By the end of this section you should be able to:
- Write `LIKE` patterns using the `%` and `_` wildcards
- Use PostgreSQL's `ILIKE` for case-insensitive pattern matching
- Escape a literal `%` or `_` character in a pattern so it's matched as text, not as a wildcard
- Explain, at a high level, that regular-expression matching also exists in PostgreSQL for cases `LIKE` can't express

## Prerequisites

- [Comparison and Logical Operators](03-comparison-and-logical-operators.md) — `LIKE` is used inside `WHERE` exactly like the comparison operators already covered, just testing a text pattern instead of an exact value.

## Motivation

Exact equality (`name = 'Desk Lamp'`) only ever finds rows you can spell out completely and precisely in advance. Real questions about text are rarely that exact: "find every product whose name contains 'Desk'," "find anything starting with 'USB'." `LIKE` is SQL's built-in way to answer questions like these — simple, portable pattern matching without needing a full regular-expression engine for the common cases.

## Problem Statement

A colleague wants to find every product related to desks in the catalog — but doesn't know (or care) about the exact, full product names. `name = 'Desk'` matches nothing (no product is named exactly "Desk"), and writing out every possible exact name in advance defeats the purpose of the search. What's needed is a way to say "contains 'Desk' somewhere," "starts with 'Desk'," or similar — a *pattern*, not an exact value.

## Concept

### The Two Wildcards: `%` and `_`

`LIKE` compares a column against a pattern containing two special characters:

| Wildcard | Meaning |
|---|---|
| `%` | Matches any sequence of characters — zero, one, or many |
| `_` | Matches exactly one character, any character |

```sql
SELECT name FROM products WHERE name LIKE 'Desk%';
```

```
   name
-----------
 Desk Lamp
(1 row)
```

`'Desk%'` means "starts with the literal text `Desk`, followed by anything (including nothing at all)." `Standing Desk` does *not* match — it doesn't *start* with `Desk`, `Desk` appears in the middle of it.

```sql
SELECT name FROM products WHERE name LIKE '%Desk%';
```

```
     name
----------------
 Standing Desk
 Desk Lamp
(2 rows)
```

Wrapping the search term in `%` on both sides — "contains `Desk` anywhere" — is the most common pattern shape for a general substring search.

### `LIKE` Is Case-Sensitive by Default

```sql
SELECT name FROM products WHERE name LIKE '%desk%';
```

```
(0 rows)
```

Zero rows — every stored name capitalizes "Desk," and plain `LIKE` treats `d` and `D` as different characters. This trips up a lot of beginners expecting search to "just work" regardless of case.

### `ILIKE` — PostgreSQL's Case-Insensitive Variant

`ILIKE` behaves exactly like `LIKE`, except it ignores letter case entirely. This is a PostgreSQL-specific convenience — not part of the SQL standard, and not available by that name in every database (Module 22 covers where other databases differ):

```sql
SELECT name FROM products WHERE name ILIKE '%desk%';
```

```
     name
----------------
 Standing Desk
 Desk Lamp
(2 rows)
```

Same pattern, same intent, now matching regardless of case. Reach for `ILIKE` deliberately, when case genuinely shouldn't matter for the search — not as a reflexive habit that papers over not knowing what case your data is actually stored in.

### Escaping Literal `%` and `_`

Both wildcard characters can themselves appear as ordinary, literal characters inside real data — and this catalog has two examples: `Blender_Pro 2000` (a literal underscore) and `100% Cotton Towel` (a literal percent sign). Without escaping, `LIKE` treats those characters as wildcards, not as the literal text they represent in the data:

```sql
SELECT name FROM products WHERE name LIKE '%_%';
```

```
          name           
-------------------------
 Wireless Mouse
 USB-C Cable
 Mechanical Keyboard
 Standing Desk
 Office Chair
 Desk Lamp
 Notebook Pack of 5
 Stapler
 Whiteboard Marker Set
 Coffee Mug
 Electric Kettle
 Blender_Pro 2000
 100% Cotton Towel
 Uncategorized Widget
(14 rows)
```

Every single row matches — because an unescaped `_` matches *any one character*, and `%_%` therefore just means "contains at least one character anywhere," which is true of every non-empty name in the table. This pattern is useless for what was actually intended (finding names with a literal underscore) because the wildcard swallowed the very character it was supposed to represent literally.

To match `_` or `%` as literal characters, precede them with an **escape character**, declared with the `ESCAPE` clause:

```sql
SELECT name FROM products WHERE name LIKE '%!_%' ESCAPE '!';
```

```
       name
-------------------
 Blender_Pro 2000
(1 row)
```

```sql
SELECT name FROM products WHERE name LIKE '%!%%' ESCAPE '!';
```

```
        name
---------------------
 100% Cotton Towel
(1 row)
```

In `'%!_%'`, the `!` immediately before `_` tells PostgreSQL "the next character is literal, not a wildcard" — so this pattern means "contains a literal underscore anywhere," matching exactly the one product that has one. The same logic applies to the literal `%` in the second example. PostgreSQL actually treats a plain backslash (`\`) as the default escape character for `LIKE` even without an explicit `ESCAPE` clause, but relying on that default is a bad habit for two reasons: it depends on server configuration you may not control, and a backslash inside a string literal has its own escaping subtleties depending on how PostgreSQL is configured to parse strings. Declaring `ESCAPE` explicitly, with a character that has no special meaning to string literals at all (`!`, as used here, or `\` if you prefer and are comfortable with it), sidesteps that ambiguity entirely and is also more portable to other databases that don't default to backslash (Module 22).

### Regular Expressions Exist Too — Briefly

`LIKE`'s two wildcards are deliberately simple and cover the common cases, but they can't express everything — there's no way to say "one of several alternative substrings" or "a digit followed by a letter" with `LIKE` alone. PostgreSQL supports full POSIX regular-expression matching through the `~` operator (`~*` for case-insensitive, `!~` and `!~*` for negation):

```sql
SELECT name FROM products WHERE name ~ '^(Desk|Office) ';
```

```
     name
---------------
 Desk Lamp
 Office Chair
(2 rows)
```

This course doesn't dive deep into regular-expression syntax here — the point is simply to know `~` exists for the cases where `LIKE`'s two wildcards genuinely aren't expressive enough, so you're not tempted to contort a `LIKE` pattern (or chain several `OR`ed `LIKE`s together) into doing a regex's job.

## Internal Working (Preview)

Conceptually, `LIKE` walks the pattern and the target string together, character by character, matching literal characters exactly and letting `%`/`_` consume characters flexibly:

```
 pattern:  D  e  s  k  %
 string:   D  e  s  k  L  a  m  p
           ✓  ✓  ✓  ✓  └─ % absorbs "Lamp" (and would absorb nothing, too)
```

This matters for performance, not just correctness: a pattern with a fixed, literal prefix (`'Desk%'`) lets the database narrow its search efficiently, because a supporting index sorted on `name` can jump straight to entries starting with "Desk" without checking every row (Module 13 covers index-assisted searches). A pattern with a **leading wildcard** (`'%Desk%'`), on the other hand, could match "Desk" starting at any position, so there's no fixed prefix to jump to — this generally forces the engine to check every row's full text, one at a time, regardless of how many rows the table has. The wildcard's *position* in the pattern, not just its presence, has real performance consequences at scale.

## Real-World Analogy

Think of a library card catalog sorted alphabetically by title. A search for "titles starting with Desk" is like flipping straight to the "De" section of the catalog and reading forward a little — fast, because the catalog's physical order already groups matching entries together. A search for "titles containing Desk anywhere" is like being handed the entire catalog and told to read every single card front to back, because a match could be hiding in the middle or end of any title, with no shortcut the catalog's alphabetical ordering can offer you.

## Why LIKE Was Designed This Way

The SQL standard defines `LIKE` with exactly these two simple wildcards to provide portable, lightweight pattern matching without requiring every conforming database to implement a full regular-expression engine — a deliberate scope decision, not an oversight. `ILIKE` exists in PostgreSQL specifically because case-insensitive text search is such a routine, everyday need that spelling it out with a function every time (wrapping both sides in `LOWER(...)`) would be needless friction for something this common; PostgreSQL chose to bake the convenience directly into the operator instead.

## Advantages

- **Simple, readable syntax** for the overwhelming majority of real text-search needs — prefix, suffix, and substring matches cover most everyday questions.
- **`LIKE` (with `%`/`_`) is part of the SQL standard**, so the core syntax is portable across virtually every relational database, unlike more vendor-specific text search features.
- **`ILIKE` removes a common annoyance** (case mismatches) without requiring a separate function call on every comparison, in PostgreSQL specifically.

## Disadvantages / Limitations

- **Limited expressiveness** — no alternation ("A or B"), character classes, or anchoring beyond the string's start/end; genuinely complex patterns need `~` (regex) instead.
- **Leading-wildcard patterns (`'%text%'`) generally can't use a standard index efficiently**, forcing a full scan of every row's text on large tables — a real performance concern at scale (Module 13, Module 20).
- **Case sensitivity by default is easy to forget**, leading to silently empty results until `ILIKE` (or an explicit `LOWER()`) is used deliberately.
- **`ILIKE` is PostgreSQL-specific** — code relying on it isn't portable to a database that doesn't support it (Module 22 covers cross-database differences).

## Best Practices

- Reach for `ILIKE` (or an explicit `LOWER(column) LIKE LOWER(pattern)`) deliberately when a search genuinely should ignore case — don't use it as an unexamined default without considering whether case actually matters for the data.
- Declare `ESCAPE` explicitly whenever a pattern needs to match a literal `%` or `_`, using a character with no special meaning in string literals (like `!`), rather than relying on the database's default escape behavior.
- Be aware that a leading `%` wildcard defeats standard index usage — for search-heavy features on large tables, this is a real reason to look into PostgreSQL's specialized full-text search capabilities (outside this module's scope) instead of chaining `LIKE '%...%'` conditions.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming `LIKE` is case-insensitive by default | PostgreSQL's `LIKE` is case-sensitive; use `ILIKE` (PostgreSQL-specific) or wrap both sides in `LOWER()` for a portable case-insensitive comparison. |
| Forgetting to escape a literal `%` or `_` that appears in the actual data being searched for | An unescaped wildcard character in a pattern is interpreted as a wildcard, not literal text, often matching far more (or differently) than intended — as `'%_%'` matching every row demonstrated. |
| Believing `ILIKE` is standard SQL | `ILIKE` is a PostgreSQL extension; other databases require a different approach (commonly wrapping both sides in a case-folding function) for the same effect (Module 22). |
| Reaching for a complicated chain of `OR`ed `LIKE` conditions to express "any of these alternatives" | This is exactly the situation the `~` regex operator (or, for simple fixed lists, `IN`, covered next) handles far more directly. |

## Interview Questions

1. **Q: What do `%` and `_` mean in a `LIKE` pattern?**
   A: `%` matches any sequence of characters, including none at all; `_` matches exactly one character, whatever it is.

2. **Q: What's the difference between `LIKE` and `ILIKE` in PostgreSQL?**
   A: They behave identically except for case sensitivity — `LIKE` is case-sensitive, `ILIKE` ignores letter case entirely. `ILIKE` is a PostgreSQL-specific extension, not part of the SQL standard.

3. **Q: Why can a leading-wildcard pattern like `'%foo'` be slower than a trailing-wildcard pattern like `'foo%'` on a large table?**
   A: A trailing wildcard leaves a fixed literal prefix, which a standard index sorted on that column can use to jump directly to matching entries. A leading wildcard means a match could start anywhere in the string, giving the database no fixed prefix to search from — it generally has to check every row's full text instead.

4. **Q: How do you match a literal percent sign using `LIKE`?**
   A: Precede it with an escape character declared via the `ESCAPE` clause, for example `LIKE '%!%%' ESCAPE '!'` to match any string containing a literal `%` — without escaping, `%` is always interpreted as the "any sequence of characters" wildcard.

## Summary

- `LIKE` matches text against a pattern using two wildcards: `%` (any sequence, including none) and `_` (exactly one character).
- `LIKE` is case-sensitive by default; PostgreSQL's `ILIKE` performs the same matching while ignoring case, at the cost of portability to other databases.
- A literal `%` or `_` in the data being matched must be escaped (via an explicit `ESCAPE` clause) so it's treated as text rather than a wildcard — forgetting this can silently match far more rows than intended.
- A pattern's wildcard *position* matters for performance: a trailing wildcard can use a standard index; a leading wildcard generally forces a full scan of every row's text.
- PostgreSQL's `~` operator provides full regular-expression matching for patterns `LIKE` can't express — not covered in depth here, but worth knowing it exists.
- The next topic, [BETWEEN, IN, and IS NULL](05-between-in-and-is-null.md), covers range checks, list membership, and the correct way to test for missing values.
