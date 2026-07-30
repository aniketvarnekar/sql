# Date and Time Types

## Learning Objectives

By the end of this section you should be able to:
- Distinguish `DATE`, `TIME`, `TIMESTAMP`, and `INTERVAL`, and state what each represents
- Explain precisely what `TIMESTAMP WITH TIME ZONE` (`TIMESTAMPTZ`) stores and how it differs from `TIMESTAMP WITHOUT TIME ZONE`
- Explain why timezone-aware timestamps matter for any system used across more than one time zone
- Perform basic date and timestamp arithmetic using `INTERVAL`
- Choose the correct date/time type for a given real-world scenario

## Prerequisites

- [Character and String Types](02-character-and-string-types.md) — this module's previous topic; no direct dependency, but the same "match the type to what the data really is" reasoning continues here.
- [Your First Query](../01-introduction/05-your-first-query.md) — familiarity with basic `SELECT`/`WHERE`/`ORDER BY` is assumed for the query examples in this topic.

## Motivation

Almost every real table has at least one date or time column — an order's placement time, a user's signup date, an event's scheduled start. Get these types right and your application correctly handles users, servers, and events across the entire globe with no extra effort. Get them wrong — most commonly, by using a timezone-naive type where a timezone-aware one was needed — and you get a class of bug that is notoriously hard to reproduce and diagnose: timestamps that look correct in testing (because the developer, database, and test data all happen to share one time zone) and then silently disagree by hours once real users, servers, or database replicas in different time zones are involved.

## Problem Statement

Suppose you store an event's start time like this:

```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name TEXT,
    starts_at TIMESTAMP  -- no time zone information at all
);

INSERT INTO events (name, starts_at) VALUES ('Product Launch', '2026-08-15 09:00:00');
```

That value, `2026-08-15 09:00:00`, is stored exactly as written — nine in the morning, but nine in the morning *where*? If your application server runs in New York and your database session is configured for a different time zone, or if a colleague in Tokyo reads this same row, `TIMESTAMP` gives no information at all about which real-world instant in time was intended. It's simply a wall-clock reading with no anchor. This is precisely the ambiguity that `TIMESTAMP WITH TIME ZONE` exists to eliminate, and the rest of this topic explains exactly how.

## Concept

### `DATE` — A Calendar Date, No Time

`DATE` stores a calendar date with no time-of-day component at all — a year, month, and day:

```sql
CREATE TABLE employees_dates (
    id SERIAL PRIMARY KEY,
    hire_date DATE
);

INSERT INTO employees_dates (hire_date) VALUES ('2024-03-15');
INSERT INTO employees_dates (hire_date) VALUES ('2024-02-30'); -- rejected: not a real date
```

```
ERROR:  date/time field value out of range: "2024-02-30"
```

PostgreSQL validates calendar dates against the real Gregorian calendar (including leap years), rejecting nonexistent dates like February 30th outright.

### `TIME` — A Time of Day, No Date

`TIME` (`TIME WITHOUT TIME ZONE`) stores a time-of-day value with no associated calendar date — useful for things like "the store opens at 09:00 every day," where the date is irrelevant:

```sql
CREATE TABLE store_hours (
    day_of_week TEXT,
    opens_at TIME,
    closes_at TIME
);

INSERT INTO store_hours VALUES ('Monday', '09:00:00', '18:00:00');
```

PostgreSQL also offers `TIME WITH TIME ZONE`, which attaches a UTC offset to a bare time-of-day value. In practice this type is rarely useful and is explicitly discouraged by PostgreSQL's own documentation — a UTC offset without an associated date is ambiguous with respect to daylight saving transitions, and it exists mainly for SQL-standard completeness. Prefer `TIMESTAMPTZ` (below) whenever both a date and time-zone-aware behavior are genuinely needed.

### `TIMESTAMP WITHOUT TIME ZONE` — A Wall-Clock Reading

`TIMESTAMP` (equivalently `TIMESTAMP WITHOUT TIME ZONE`) stores a calendar date and time-of-day together, exactly as given, with no time zone conversion or awareness whatsoever:

```sql
CREATE TABLE events_naive (
    id SERIAL PRIMARY KEY,
    starts_at TIMESTAMP
);

INSERT INTO events_naive (starts_at) VALUES ('2026-08-15 09:00:00');

SELECT starts_at FROM events_naive;
```

```
      starts_at
---------------------
 2026-08-15 09:00:00
(1 row)
```

No matter what time zone your session, server, or client is configured for, this value always reads back exactly as `2026-08-15 09:00:00` — PostgreSQL performs no conversion on it at all. This makes `TIMESTAMP` appropriate only for values that genuinely have no inherent connection to a specific real-world instant — for example, a recurring local wall-clock rule like "this recurring meeting happens at 09:00 local time, wherever 'local' means for whoever reads the schedule."

### `TIMESTAMP WITH TIME ZONE` (`TIMESTAMPTZ`) — An Unambiguous Instant

`TIMESTAMPTZ` stores a genuine, unambiguous point in time. Internally, PostgreSQL always stores the value normalized to UTC; on input and output, it converts using the current session's `TimeZone` setting:

```sql
SET TimeZone = 'America/New_York';

CREATE TABLE events_aware (
    id SERIAL PRIMARY KEY,
    starts_at TIMESTAMPTZ
);

INSERT INTO events_aware (starts_at) VALUES ('2026-08-15 09:00:00');

SELECT starts_at FROM events_aware;
```

```
        starts_at
------------------------
 2026-08-15 09:00:00-04
(1 row)
```

The literal `'2026-08-15 09:00:00'` was interpreted as 9 AM in the session's current time zone (`America/New_York`, which is UTC-4 in August due to daylight saving), converted to UTC internally for storage, and displayed back with the offset that applies. Now change the session's time zone and re-select the *same stored row*, with no new insert:

```sql
SET TimeZone = 'Asia/Kolkata';

SELECT starts_at FROM events_aware;
```

```
        starts_at
------------------------
 2026-08-15 18:30:00+05:30
(1 row)
```

The exact same underlying instant is now displayed as 6:30 PM in India Standard Time — because 9 AM Eastern and 6:30 PM India time are, in fact, the same real-world moment. This is precisely the guarantee `TIMESTAMP` cannot give you: a `TIMESTAMPTZ` value means the same real instant no matter who queries it from where, only its *displayed representation* changes with the viewer's time zone. Note also what's lost: `TIMESTAMPTZ` does not remember the original input offset or zone name (`America/New_York`) — only the resulting UTC instant. If you need to remember "this event's official local time zone was US Eastern" as a business fact (for showing "9 AM Eastern" in a UI regardless of viewer), store the zone name in a separate column alongside the `TIMESTAMPTZ`.

### `INTERVAL` — A Span of Time

`INTERVAL` represents a duration — a span of time rather than a point in time — and supports arithmetic directly against dates and timestamps:

```sql
SELECT CURRENT_DATE AS today,
       CURRENT_DATE + INTERVAL '30 days' AS thirty_days_out,
       CURRENT_DATE - INTERVAL '1 year' AS one_year_ago;
```

```
    today   |   thirty_days_out   |  one_year_ago
------------+---------------------+----------------
 2026-07-31 | 2026-08-30 00:00:00 | 2025-07-31
(1 row)
```

Subtracting two timestamps produces an `INTERVAL` directly:

```sql
SELECT TIMESTAMP '2026-08-15 09:00:00' - TIMESTAMP '2026-08-01 14:30:00' AS gap;
```

```
        gap
--------------------
 13 days 18:30:00
(1 row)
```

`INTERVAL` values can also be constructed and combined flexibly:

```sql
SELECT INTERVAL '2 years 3 months' + INTERVAL '10 days' AS combined;
```

```
     combined
-------------------
 2 years 3 mons 10 days
(1 row)
```

### Common Date Arithmetic

```sql
SELECT
    CURRENT_DATE                                AS today,
    NOW()                                        AS right_now_with_tz,
    CURRENT_TIMESTAMP - INTERVAL '90 minutes'    AS ninety_minutes_ago,
    EXTRACT(YEAR FROM CURRENT_DATE)              AS current_year,
    DATE_TRUNC('month', CURRENT_TIMESTAMP)       AS start_of_this_month;
```

`CURRENT_DATE` and `NOW()`/`CURRENT_TIMESTAMP` give you the current date and current timestamp (as `TIMESTAMPTZ`) respectively; `EXTRACT` pulls a single field (year, month, day, hour, and so on) out of a date/time value; `DATE_TRUNC` rounds a timestamp *down* to the start of a given unit (day, month, year, etc.) — extremely useful for grouping events by day or month in later reporting queries.

### A Combined Example

```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    room_name TEXT,
    reserved_date DATE,
    start_time TIME,
    end_time TIME,
    booked_at TIMESTAMPTZ DEFAULT NOW(),
    duration INTERVAL GENERATED ALWAYS AS (end_time - start_time) STORED
);

INSERT INTO reservations (room_name, reserved_date, start_time, end_time)
VALUES ('Conference Room A', '2026-08-20', '14:00:00', '15:30:00');

SELECT room_name, reserved_date, start_time, end_time, duration, booked_at FROM reservations;
```

```
      room_name      | reserved_date | start_time | end_time | duration |          booked_at
---------------------+---------------+------------+----------+----------+-------------------------------
 Conference Room A    | 2026-08-20    | 14:00:00   | 15:30:00 | 01:30:00 | 2026-07-31 10:22:47.913201-04
(1 row)
```

## Internal Working (Preview)

Internally, `DATE` is stored as a 4-byte integer counting days relative to a fixed epoch (January 1, 2000); `TIMESTAMP` and `TIMESTAMPTZ` are both stored as an 8-byte integer counting microseconds relative to the same epoch. The *only* structural difference between `TIMESTAMP` and `TIMESTAMPTZ` at the storage layer is that `TIMESTAMPTZ` values are always normalized to UTC before being stored, and converted back to the session's time zone only when displayed or compared against a zone-aware literal:

```
 INSERT '2026-08-15 09:00:00' with TimeZone = America/New_York
      │
      ▼
 parsed as 09:00 Eastern (UTC-4 in August)
      │
      ▼
 converted to UTC for storage:  2026-08-15 13:00:00 UTC   ◀── stored value (TIMESTAMPTZ)
      │
      ▼
 SELECT with TimeZone = Asia/Kolkata
      │
      ▼
 converted from UTC to session zone for display: 2026-08-15 18:30:00+05:30
```

`TIMESTAMP WITHOUT TIME ZONE` skips both conversion steps entirely — the stored microsecond count directly reflects whatever was typed, with no reference to UTC or any zone at all, which is exactly why it cannot represent "this same instant, viewed from a different location."

## Real-World Analogy

`TIMESTAMP WITHOUT TIME ZONE` is like a clock hanging on a wall with no label saying which city or time zone it's set to — it faithfully reads "9:00," but that reading is meaningless to anyone who doesn't already know where the clock physically is. `TIMESTAMPTZ` is like recording an event using a universal reference — for example, "this flight departs at Unix epoch second 1786867200" — which any airport anywhere in the world can unambiguously convert into its own correct local departure-board time, because the underlying instant is fixed and the *display* adapts to the viewer, not the other way around.

## Why Date/Time Types Were Designed This Way

The design deliberately separates two genuinely different concepts that are easy to conflate: a **wall-clock reading** (a date and time with no inherent connection to any specific real-world instant — `TIMESTAMP`) and a **point in time** (a globally unambiguous instant, expressible relative to a universal reference like UTC — `TIMESTAMPTZ`). Most real-world event data — "when did this order get placed," "when did this user last log in," "when is this flight scheduled to depart" — actually describes points in time that matter across time zones, even if a single-time-zone application never notices the difference during development. By storing `TIMESTAMPTZ` values internally as UTC and converting only at the input/output boundary, PostgreSQL lets every part of the system reason in UTC internally (where arithmetic and comparison are unambiguous) while still displaying values in whatever time zone is convenient for a given viewer — consistent with the relational model's broader principle, from Module 02, that the database stores canonical, unambiguous data and lets each query shape the presentation, rather than baking a specific display format into storage itself.

## Advantages

- **`TIMESTAMPTZ` eliminates an entire class of "what time zone did they mean" ambiguity**, because the stored value is always a single, unambiguous, comparable instant regardless of who inserted it or who reads it.
- **`DATE`/`TIME` model exactly what they represent with no wasted information** — a birth date or a recurring store-opening time genuinely has no time-zone component, and using the narrower type documents that clearly.
- **`INTERVAL` arithmetic is expressive and calendar-aware** — adding `INTERVAL '1 month'` to January 31st correctly lands on the last valid day of February, rather than naively adding a fixed number of days.
- **PostgreSQL validates real calendar rules automatically** (leap years, days-in-month, valid hour ranges), catching malformed date/time input immediately rather than storing nonsense silently.

## Disadvantages / Limitations

- **`TIMESTAMPTZ` discards the original input offset/zone name** — if you need to remember "this was originally entered as 9 AM Eastern" as a business fact (not just the resulting instant), you must store that zone separately in its own column.
- **`TIMESTAMP WITHOUT TIME ZONE` is easy to reach for by habit and get wrong** — it looks and behaves almost identically to `TIMESTAMPTZ` in a single-time-zone development environment, hiding the bug until multiple time zones are actually involved in production.
- **`TIME WITH TIME ZONE` is a genuinely awkward type with little practical use** — a bare offset without an associated date can't correctly account for daylight saving transitions, which is why PostgreSQL's own documentation discourages it.
- **Interval arithmetic across daylight saving boundaries can be subtle** — adding "1 day" as an interval to a `TIMESTAMPTZ` near a DST transition advances the calendar day correctly but the elapsed wall-clock time may be 23 or 25 hours, which can surprise code that assumes a day is always exactly 24 hours.

## Best Practices

- Default to `TIMESTAMPTZ` for any column recording "when something happened or will happen" in a system with users, servers, or replicas that could ever span more than one time zone — which, in practice, is almost every production system.
- Reserve `TIMESTAMP WITHOUT TIME ZONE` for values that are genuinely timezone-agnostic by nature — a recurring local wall-clock rule, or a value you deliberately want to treat as "whatever time zone the viewer is in."
- Avoid `TIME WITH TIME ZONE` entirely; use `TIMESTAMPTZ` whenever both a date and time-zone correctness matter.
- Set your database server's and application's session time zone explicitly and consistently (commonly UTC) rather than relying on whatever the operating system defaults to, to avoid environment-dependent surprises.
- Use `INTERVAL` for date/time arithmetic instead of hand-computing offsets in seconds or days — it correctly accounts for variable month lengths, leap years, and (for `TIMESTAMPTZ`) daylight saving transitions.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `TIMESTAMP` (no time zone) for an "event happened at" column | It stores a bare wall-clock reading with no anchor to a real instant — the same stored value means different real-world moments depending on which time zone it's later interpreted in, which is rarely the intent. |
| Assuming `TIMESTAMPTZ` remembers the original time zone that was used on insert | It only stores the resulting UTC instant; the original offset/zone name is not preserved and must be stored separately if it matters as a business fact. |
| Treating `TIME WITH TIME ZONE` as a safe way to store "time of day in a specific zone" | A bare offset with no date cannot correctly account for daylight saving transitions; PostgreSQL's documentation itself discourages this type. |
| Computing "days between two dates" by naively dividing a timestamp difference by 86400 seconds | `INTERVAL` and `AGE()`/subtraction already express this correctly and account for calendar irregularities (varying month lengths, DST) that a raw seconds-based calculation would get wrong. |
| Assuming a table's timestamps are already comparable across rows without checking whether the column is `TIMESTAMP` or `TIMESTAMPTZ` | Two `TIMESTAMP` (non-tz) values from different sources may represent completely different real instants even if the numbers look close, since neither carries any time zone context. |

## Interview Questions

1. **Q: What is the difference between `TIMESTAMP` and `TIMESTAMPTZ` in PostgreSQL?**
   A: `TIMESTAMP` (`WITHOUT TIME ZONE`) stores a date and time exactly as given, with no time zone conversion or awareness — the same value reads back identically regardless of session time zone. `TIMESTAMPTZ` (`WITH TIME ZONE`) represents an unambiguous point in time: PostgreSQL converts the input to UTC for storage using the session's time zone, and converts it back to the session's time zone whenever it's displayed or compared, so the same stored instant can be viewed correctly from any time zone.

2. **Q: Why is `TIMESTAMPTZ` generally recommended over `TIMESTAMP` for recording when events occur?**
   A: Because most real event data — order placement, login times, scheduled events — describes a genuine point in time that matters consistently regardless of who or what server records or reads it. `TIMESTAMPTZ` guarantees that instant is unambiguous and correctly comparable across time zones; `TIMESTAMP` only looks correct until more than one time zone is actually involved, at which point identical-looking values can silently represent different real moments.

3. **Q: If you insert `'2026-08-15 09:00:00'` into a `TIMESTAMPTZ` column while your session's time zone is set to `America/New_York`, then query it back after switching the session time zone to `Asia/Kolkata`, what do you see, and why?**
   A: You see the same underlying instant displayed as the equivalent India Standard Time — roughly `2026-08-15 18:30:00+05:30` — because `TIMESTAMPTZ` stores a single UTC-normalized instant and only converts to the session's time zone for display; the stored value itself never changes, only its textual representation adapts to the viewer's configured time zone.

4. **Q: What does `INTERVAL` represent, and how is it different from a `TIMESTAMP`?**
   A: `INTERVAL` represents a *duration* — a span of time, like "3 days" or "2 hours 30 minutes" — rather than a specific point in time. It's used for date/time arithmetic (adding/subtracting a span from a date or timestamp) and is itself the natural result of subtracting one timestamp from another; unlike `TIMESTAMP`, it has no fixed calendar anchor of its own.

## Summary

- `DATE` stores a calendar date only; `TIME` stores a time-of-day only; both have no inherent time-zone concept.
- `TIMESTAMP` (`WITHOUT TIME ZONE`) stores a bare wall-clock reading with no conversion or awareness of time zone; `TIMESTAMPTZ` (`WITH TIME ZONE`) stores a genuinely unambiguous instant, normalized to UTC internally and converted for display based on the session's time zone.
- `TIMESTAMPTZ` does not remember the original input offset/zone name — only the resulting instant — so store the zone separately if that context matters as a business fact.
- `INTERVAL` represents a duration and supports calendar-aware arithmetic against dates and timestamps, correctly handling varying month lengths and (for `TIMESTAMPTZ`) daylight saving transitions.
- Default to `TIMESTAMPTZ` for anything recording "when something happened or will happen" in any system that could ever span more than one time zone — which is the practical default for almost all production systems.
