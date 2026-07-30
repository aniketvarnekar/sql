# SQL Injection

## Learning Objectives

By the end of this section you should be able to:
- State precisely what SQL injection is, in terms of *why* it happens, not just that it's "dangerous"
- Walk through a concrete example of a vulnerable query and explain exactly how untrusted input changes that query's actual structure
- Explain what a parameterized query (prepared statement) is at the SQL/driver level, and why it eliminates this class of vulnerability structurally
- Explain why manually escaping special characters is a fragile substitute for true parameterization

## Prerequisites

- [Your First Query](../01-introduction/05-your-first-query.md) — this topic's examples build on ordinary `SELECT ... WHERE` statements with string literals; you should be completely comfortable reading those before seeing how they can be manipulated.
- [GRANT and REVOKE](02-grant-and-revoke.md) — not required to understand the vulnerability itself, but the discussion of *why the damage is bounded or unbounded* in a real breach depends on the privilege concepts from that topic, which Topic 4 develops fully.

## Motivation

SQL injection has been one of the most common and damaging categories of vulnerability in the history of software running against databases — not because it requires some exotic attack technique, but because it exploits an extremely ordinary, easy-to-write mistake: building a SQL command by gluing together fixed text and untrusted input as one string. Understanding *exactly* why that's dangerous, at the level of what the database actually parses, is what separates "I've heard SQL injection is bad" from being able to recognize a vulnerable pattern on sight and know precisely why the fix works.

## Problem Statement

Suppose an application needs to check a login by looking up a username and password in a table:

```sql
CREATE TABLE users (
    id            SERIAL PRIMARY KEY,
    username      TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL
);

INSERT INTO users (username, password_hash) VALUES
    ('alice', 'a1b2c3...'),
    ('bob',   'd4e5f6...');
```

A very common (and very dangerous) way to build the lookup query is to take whatever the user typed and glue it directly into a SQL string before sending it to the database. Written generically, independent of any particular language or driver, the *string the application hands to the database* is built like this:

```
query_text = "SELECT * FROM users WHERE username = '" + typed_username + "' AND password_hash = '" + typed_password_hash + "';"
```

When a normal user named `alice` logs in, `typed_username` is `alice`, and the resulting string handed to the database is exactly:

```sql
SELECT * FROM users WHERE username = 'alice' AND password_hash = 'a1b2c3...';
```

That runs fine and returns Alice's single row. The problem is that `typed_username` is **untrusted input** — literally whatever text arrived from outside — and nothing has stopped that text from containing SQL-meaningful characters, like a single quote. This topic works through exactly what happens when it does.

## Concept

### What SQL Injection Actually Is

**SQL injection is a vulnerability where untrusted input is concatenated directly into a SQL command string, allowing that input to change the query's actual structure — not just supply a value within it.** The key word is *structure*: the attacker isn't just providing an unexpected *value* (like a wrong password, which would simply fail the check); they are providing text that gets interpreted by the SQL parser as additional SQL syntax — new clauses, comment markers, or entirely different logic — because the application never distinguished "this part is code" from "this part is data" before handing the final string to the database.

### Walking Through the Exploit

Take the vulnerable query-building pattern from the Problem Statement, and suppose instead of a normal username, the attacker types this into the username field:

```
' OR '1'='1' --
```

The application, following exactly the same string-building logic as before, produces:

```sql
SELECT * FROM users WHERE username = '' OR '1'='1' --' AND password_hash = 'anything';
```

Look carefully at what changed. The attacker's single quote (`'`) closed the string literal early — right after `username = '`. Everything after that is no longer inside the string; it is parsed as **actual SQL**:

- `OR '1'='1'` adds a condition that is always true, for every row in the table.
- `--` starts a SQL comment, so everything after it on that line — including the entire `AND password_hash = '...'` check — is discarded by the parser and never evaluated at all.

The query the database actually executes is functionally:

```sql
SELECT * FROM users WHERE username = '' OR '1'='1';
```

Since `'1'='1'` is true for every row, this returns **every row in the `users` table** — the password check never even ran. Running it against the sample data:

```
 id | username |    password_hash
----+----------+-----------------------
  1 | alice    | a1b2c3...
  2 | bob      | d4e5f6...
(2 rows)
```

If the application's logic is "log the user in as whichever row comes back first," the attacker has just logged in as `alice` without ever knowing her password — not by guessing it, but by rewriting the query's actual logic using nothing but a text input field. This example is illustrative and deliberately simple; the same underlying mechanism — attacker-controlled text becoming attacker-controlled SQL syntax — is what every real SQL injection vulnerability, however more elaborate, ultimately comes down to.

### The Fix: Parameterized Queries (Prepared Statements)

The structural fix is to never build a SQL string by concatenating untrusted input into it *at all*. Instead, the query's text — with its structure completely fixed — is sent to the database separately from the actual data values, using **placeholders**. PostgreSQL's own SQL-level mechanism for this is `PREPARE` and `EXECUTE`:

```sql
PREPARE login_check (text, text) AS
    SELECT * FROM users WHERE username = $1 AND password_hash = $2;

EXECUTE login_check('alice', 'a1b2c3...');
```

```
 id | username | password_hash
----+----------+---------------
  1 | alice    | a1b2c3...
(1 row)
```

Now try to "inject" through the same attack string, this time passed correctly as a *parameter value* rather than concatenated into the SQL text:

```sql
EXECUTE login_check('''  OR ''1''=''1'' --', 'anything');
```

This does **not** return every row. It returns zero rows — because `$1` is bound, after the query's structure was already parsed and fixed, to the literal, inert string value `' OR '1'='1' --` (quotes included, as actual characters), and PostgreSQL looks for a row where the `username` column equals that entire literal string, character for character. There is no such username. The attacker's text was never re-interpreted as SQL syntax, because by the time it's used, the SQL structure has already been finalized — there is nothing left for the text to "break out of."

Every real database driver, in every application-side language, provides this exact mechanism under names like "parameterized queries," "prepared statements," or "bind parameters" — the SQL-level idea shown here with `PREPARE`/`EXECUTE` and `$1`/`$2` placeholders is the same idea those drivers implement, regardless of what specific syntax a given driver uses for its placeholders.

### Why Escaping Alone Is Fragile

A tempting partial fix is **escaping**: before concatenating input into a query string, transform dangerous characters (most commonly, doubling every single quote: `'` becomes `''`) so a quote in the input can't prematurely close the string literal. This can work, narrowly, but it is a fundamentally weaker defense than parameterization, for several concrete reasons:

- **It must be applied correctly, every single time, at every single injection point.** Parameterization makes forgetting structurally impossible — there's no string concatenation step to forget to protect in the first place. Escaping is a manual step that must be remembered and applied consistently across an entire codebase; missing it in even one place reopens the vulnerability there.
- **The correct escaping rule depends on context, and contexts are easy to mix up.** Escaping a value destined for a string literal (`'`) follows different rules than a value destined for an identifier (a table or column name, escaped with double quotes), a `LIKE` pattern (where `%` and `_` are also special), or a numeric context. Using the wrong rule, or assuming one universal "safe" escaping function handles every context, has historically led to bypasses.
- **It treats a structural problem as a data-cleaning problem.** Escaping tries to sanitize data *before* it gets glued into code, but the underlying design — building executable SQL text out of untrusted input at all — is still there. Parameterization removes that design entirely: the input is never part of the SQL text in the first place, so there's nothing for an escaping rule to protect against or fail to anticipate.

In short: escaping is a best-effort filter bolted onto a fundamentally risky pattern (string-concatenated SQL); parameterization removes the risky pattern itself.

## Internal Working (Preview)

The difference between the vulnerable pattern and the parameterized fix is visible in how each one flows through the database's parser:

```
 VULNERABLE (string concatenation):
 untrusted input ──► glued directly into SQL text ──► raw SQL string
      (structure is NOT fixed — attacker input IS part of the syntax)
                              │
                              ▼
                       SQL Parser parses
                    WHATEVER text resulted,
                 including any injected clauses
                              │
                              ▼
                    Executed as written — attacker's
                     logic runs as real SQL


 PARAMETERIZED (prepared statement):
 SQL text with placeholders ($1, $2) ──► Parser ──► Parse Tree
      (structure is fixed here, BEFORE any data is involved)
                              │
                              ▼
                    Planner produces a Plan with
                    typed parameter "slots"
                              │
                              ▼
        Actual values bound into those slots as pure data
                              │
                              ▼
                 Executed — bound values can only ever be
                 compared/matched as data, never parsed as syntax
```

The critical distinction is *when* parsing happens relative to the untrusted data. In the vulnerable path, parsing happens *after* the untrusted input has already become part of the text being parsed. In the parameterized path, parsing happens *before* the untrusted input is ever involved — the input arrives only as a value to slot into an already-fixed plan, so no matter what characters it contains, it cannot alter what that plan says.

## Real-World Analogy

Picture a mail-merge template letter with blanks: "Dear ___, your current balance is ___." The template's actual sentence structure is fixed and printed in advance; the blanks only ever accept a *value* to be inserted — a name, a number. If someone's name happens to be a full sentence, or contains punctuation, it still only ever fills the blank; it cannot rewrite the rest of the letter, because the letter's structure was locked in before any name was chosen.

Now imagine the opposite, error-prone process instead: someone hands you a slip of paper and asks you to retype the *entire letter* fresh each time, manually copying their "name" directly into the sentence you're writing by hand. If that slip of paper actually contains "Ignored. Actually, this letter should say your balance is $0 and you are approved for a loan," and you just transcribe it verbatim into the position where a name goes, you've let the "value" rewrite the letter's actual meaning — exactly what string-concatenated SQL lets untrusted input do to a query, and exactly what a mail-merge blank (a true parameter) structurally cannot allow.

## Why Parameterization Was Designed This Way

Prepared statements exist because a database's parser has to draw a line somewhere between "this is SQL syntax" and "this is a value the syntax refers to" — and the safest possible place to draw that line is *before* any untrusted data is involved at all. This connects directly to how SQL execution actually works internally (previewed across this course, and in Module 1's introduction of the query planner): a statement is parsed into a fixed parse tree and turned into an execution plan first, and only then are parameter values bound in as pure data — never re-parsed, never re-interpreted as grammar. Parameterization isn't a security feature bolted on top of normal query execution; it's a direct, deliberate use of the fact that parsing and data-binding are already two separate steps inside the engine, arranged so that untrusted data only ever touches the second step.

## Advantages

- **Structurally eliminates SQL injection for every parameterized value** — there is no string-concatenation step for untrusted input to corrupt, so the entire class of vulnerability simply doesn't apply to values passed as parameters.
- **Often faster on repeated execution** — a prepared statement's parse tree and plan can be produced once and reused across many `EXECUTE` calls with different parameter values, avoiding repeated parsing/planning overhead for the same query shape.
- **Makes intent explicit and self-documenting** — a query with `$1`, `$2` placeholders makes it immediately obvious, to any future reader, exactly which parts of the statement are meant to vary as data.

## Disadvantages / Limitations

- **Parameters can only stand in for values, not for identifiers.** A placeholder cannot be used where a table name, column name, or `ORDER BY` column would go — `SELECT * FROM $1` is not valid. Genuinely dynamic table/column names require a different defense (strict allow-listing against a known, fixed set of valid names, and careful quoting), since they fall outside what parameterization covers.
- **Parameterization alone doesn't limit the damage of a successful compromise through some other means** (a leaked credential, an overly broad role) — it prevents *this specific* vulnerability class, but doesn't substitute for sound privilege design, which is exactly what Topic 4 covers next.
- **A prepared statement still must be written correctly** — parameterizing the *values* in a query doesn't protect against a query whose fixed, hand-written logic is simply wrong or overly permissive; parameterization protects the boundary between code and data, not the correctness of the code itself.

## Best Practices

- **Always use your database driver's parameterized query / prepared statement mechanism for any SQL built with untrusted input — never build SQL text by string concatenation of that input.** This is the single most important takeaway of this topic.
- **Treat every source of external input as untrusted** — form fields, URL parameters, API request bodies, even values that "should" already be safe (like an ID from a previous, trusted-looking step) — and parameterize consistently, rather than selectively deciding some inputs "don't need it."
- **For genuinely dynamic identifiers (table/column names) that can't be parameterized, validate against a fixed, known allow-list** rather than accepting arbitrary text — never fall back to escaping raw user text into an identifier position.
- **Apply the principle of least privilege (Topic 4) as defense in depth** — even correctly parameterized applications should run under a database role with the minimum privileges needed, so that any other unforeseen vulnerability still has a bounded blast radius.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Concatenating user input directly into a SQL string, even "just this once" for a quick internal tool | The vulnerability doesn't care about the tool's intended audience — any code path that builds SQL text from untrusted input this way is exploitable the same way shown in this topic's worked example. |
| Relying only on manually escaping quotes before concatenation | Escaping must be applied correctly and consistently at every injection point and in the correct context (string literal vs. identifier vs. pattern); a single missed or misapplied case reopens the vulnerability, whereas parameterization removes the risky pattern structurally. |
| Assuming client-side (e.g., browser-side) input validation is sufficient protection | Client-side checks can be trivially bypassed by anyone sending requests directly to the server; the database-facing code must independently treat all incoming input as untrusted, regardless of what earlier validation claims to have done. |
| Parameterizing values correctly but still concatenating a dynamic table or column name from user input | Parameter placeholders cannot represent identifiers at all; a dynamic identifier built from untrusted text is just as exploitable as a dynamic value, and needs allow-list validation instead. |

## Interview Questions

1. **Q: Define SQL injection precisely — not just "it's when hackers attack your database."**
   A: SQL injection is a vulnerability where untrusted input is concatenated directly into a SQL command's text, allowing that input to alter the query's actual structure (adding clauses, closing string literals early, commenting out parts of the statement) rather than merely supplying an unexpected value within an already-fixed structure.

2. **Q: Walk through, at a mechanical level, how the input `' OR '1'='1' --` exploits a vulnerable, concatenated login query.**
   A: The leading single quote closes the string literal the application intended to hold just the username, so everything after it is parsed as real SQL rather than as data. `OR '1'='1'` adds a condition that's always true, matching every row regardless of the intended `WHERE` filter. The trailing `--` starts a SQL comment, discarding the rest of the original statement (including the password check) entirely.

3. **Q: Why does a parameterized query prevent this, at the level of what the database actually does?**
   A: Because the SQL text (with placeholders) is parsed into a fixed parse tree and plan *before* any parameter values are involved. Values are bound into that already-fixed plan purely as data afterward, so no matter what characters a value contains, it can never be reinterpreted as SQL syntax — there's no parsing step left for it to influence.

4. **Q: Why is manually escaping special characters considered a weaker defense than parameterized queries, even though it can technically block the same basic attack?**
   A: Escaping must be applied correctly, consistently, and in the right context (string literal vs. identifier vs. pattern) at every single point where untrusted input reaches SQL text — a single missed or wrong-context escape reopens the vulnerability. Parameterization removes the underlying risky pattern (building SQL text out of untrusted input) entirely, so there is no escaping step to ever get wrong or forget.

## Summary

- SQL injection happens when untrusted input is concatenated directly into a SQL command's text, letting that input change the query's actual structure rather than just its data.
- The worked example showed how `' OR '1'='1' --` closes a string literal early, adds an always-true condition, and comments out the rest of the intended query — turning a login check into "return every row."
- Parameterized queries / prepared statements (PostgreSQL's `PREPARE`/`EXECUTE` with `$1`, `$2` placeholders) fix this structurally by parsing the SQL's structure fully before any untrusted value is bound in as pure data.
- Escaping special characters can block the same specific attack but is fragile — it depends on remembering to apply the right rule, in the right context, at every injection point, whereas parameterization removes the risky concatenation pattern entirely.
- Parameterization protects values; it does not cover dynamic identifiers (table/column names), which need separate allow-list validation.
- Next, Topic 4 shows how bounding privileges with the principle of least privilege provides a second, independent layer of defense — so that even an unforeseen vulnerability has a limited blast radius.
