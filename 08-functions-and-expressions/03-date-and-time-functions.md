# Date and Time Functions

## Learning Objectives

By the end of this section you should be able to:
- Retrieve the current date/time using `NOW()`, `CURRENT_DATE`, and `CURRENT_TIMESTAMP`, and know when to use which
- Pull individual components (year, month, day of week, etc.) out of a date/time value with `EXTRACT`
- Round a date/time value down to a time bucket (day, month, hour) with `DATE_TRUNC`
- Perform date arithmetic using `INTERVAL`, and compute a human-readable elapsed duration with `AGE()`
- Format a date/time value for display using `TO_CHAR`

## Prerequisites

- [Numeric Functions](02-numeric-functions.md) — not a strict dependency, but date arithmetic reuses the same "expression has a predictable result type" mindset established there.
- Module 3 — Data Types, specifically `DATE`, `TIMESTAMP`, and `TIMESTAMPTZ` — this topic assumes you already know these types exist and roughly what each stores; here we focus on *computing with* them, not defining them.

## Motivation

Almost every business question involves time: "how long ago was this order placed," "how many orders came in this month," "what day of the week do most signups happen." None of that is answerable from a raw stored timestamp alone — you need to extract parts of it, compare it against the current moment, bucket it into a coarser unit, or format it for a human to read. Date and time functions are how SQL turns "a stored instant" into "an answerable question about time."

## Problem Statement

Consider an `orders` table:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_name TEXT,
    placed_at TIMESTAMP,
    ship_date DATE
);

INSERT INTO orders (customer_name, placed_at, ship_date) VALUES
    ('Asha Verma', '2026-03-14 09:32:00', '2026-03-16'),
    ('Ben Okafor', '2026-05-01 17:05:22', '2026-05-03'),
    ('Chen Wei',   '2026-07-20 23:59:59', NULL);
```

You're asked: which day of the week does each order fall on, how many days elapsed between order and shipment, which month should each order be grouped into for a monthly report, and — assuming today is **2026-07-31** — how long ago was each order placed, in a form a human can read at a glance (not a raw number of seconds)? Every one of these requires a dedicated date/time function; raw comparison operators (`<`, `>`, `-`) alone can't answer "which day of the week" or "how long ago, in years/months/days" on their own.

## Concept

### `NOW()`, `CURRENT_DATE`, `CURRENT_TIMESTAMP`

```sql
SELECT NOW();
```

```
              now
-------------------------------
 2026-07-31 14:22:07.512395+00
(1 row)
```

`NOW()` returns the current date **and** time, including timezone offset, as of the start of the current transaction (it does not change if called multiple times within the same transaction — a detail Module 14 revisits). `CURRENT_TIMESTAMP` is the SQL-standard spelling of the same thing and is interchangeable with `NOW()` in PostgreSQL.

```sql
SELECT CURRENT_DATE;
```

```
 current_date
---------------
 2026-07-31
(1 row)
```

`CURRENT_DATE` returns only the date portion, with no time component at all — useful whenever time-of-day is irrelevant, such as "which day did this happen on."

### `EXTRACT()` — Pulling Out a Component

`EXTRACT(field FROM source)` pulls a single named component out of a date/time value.

```sql
SELECT customer_name,
       EXTRACT(YEAR FROM placed_at)  AS yr,
       EXTRACT(MONTH FROM placed_at) AS mo,
       EXTRACT(DAY FROM placed_at)   AS dy,
       EXTRACT(DOW FROM placed_at)   AS day_of_week
FROM orders
WHERE customer_name = 'Asha Verma';
```

```
 customer_name | yr |  mo | dy | day_of_week
----------------+----+-----+----+--------------
 Asha Verma    | 2026 |   3 | 14 |            6
(1 row)
```

`DOW` (day of week) returns `0` for Sunday through `6` for Saturday — March 14, 2026 falls on a Saturday, hence `6`. This is exactly the kind of question ("do most orders come in on weekends?") that isn't answerable without `EXTRACT`.

### `DATE_TRUNC()` — Rounding Down to a Time Bucket

`DATE_TRUNC(unit, value)` rounds a timestamp *down* to the start of the given unit — the opposite of extracting a single field, this keeps the value as a full timestamp, just zeroed out below the requested precision.

```sql
SELECT placed_at,
       DATE_TRUNC('month', placed_at) AS month_bucket,
       DATE_TRUNC('day', placed_at)   AS day_bucket,
       DATE_TRUNC('hour', placed_at)  AS hour_bucket
FROM orders
WHERE customer_name = 'Asha Verma';
```

```
      placed_at       |    month_bucket     |     day_bucket      |     hour_bucket
------------------------+----------------------+----------------------+----------------------
 2026-03-14 09:32:00    | 2026-03-01 00:00:00 | 2026-03-14 00:00:00 | 2026-03-14 09:00:00
(1 row)
```

This is the standard tool for "group orders by month" style reporting: `GROUP BY DATE_TRUNC('month', placed_at)` (aggregation is covered fully in Module 9) buckets every order in March into a single group, regardless of the exact day or time it was placed, without losing the fact that the bucket itself is still a real date.

### Date Arithmetic with `INTERVAL`

PostgreSQL lets you add or subtract an explicit `INTERVAL` — a duration value like "7 days" or "2 hours" — directly to/from a date or timestamp.

```sql
SELECT placed_at, placed_at + INTERVAL '7 days' AS one_week_later
FROM orders
WHERE customer_name = 'Asha Verma';
```

```
      placed_at       |    one_week_later
------------------------+------------------------
 2026-03-14 09:32:00    | 2026-03-21 09:32:00
(1 row)
```

Subtracting two dates directly gives a plain integer number of days:

```sql
SELECT customer_name, ship_date - placed_at::DATE AS days_to_ship
FROM orders;
```

```
 customer_name | days_to_ship
----------------+---------------
 Asha Verma    |             2
 Ben Okafor    |             2
 Chen Wei      |
(3 rows)
```

Chen Wei's row is blank because `ship_date` is `NULL` — arithmetic against a `NULL` produces `NULL` (the same propagation rule seen with strings and numbers), correctly signaling "not yet shipped" rather than a false zero.

### `AGE()` — Human-Readable Elapsed Time

`AGE(timestamp)` computes the difference between `CURRENT_DATE` and the given timestamp, returned as a single, calendar-aware `INTERVAL` broken into years, months, days, and time — rather than one raw unit like total days or total seconds. Assuming these examples run on **2026-07-31**:

```sql
SELECT customer_name, placed_at, AGE(placed_at) AS time_since_order
FROM orders
WHERE customer_name = 'Asha Verma';
```

```
 customer_name |      placed_at       |    time_since_order
----------------+------------------------+--------------------------
 Asha Verma    | 2026-03-14 09:32:00    | 4 mons 16 days 14:28:00
(1 row)
```

`AGE()` also accepts two explicit timestamps, `AGE(later, earlier)`, if you want the elapsed time between two specific values rather than between "now" and one value — for instance, `AGE(ship_date, placed_at)` to express shipping turnaround the same calendar-aware way.

### `TO_CHAR()` — Formatting for Display

`TO_CHAR(value, format)` renders a date/time (or number) as a specifically-formatted text string, using a format string built from pattern codes (`YYYY` for a 4-digit year, `MM` for a zero-padded month, `Mon` for an abbreviated month name, `Day` for a full day name, `HH24`/`MI` for 24-hour time, and many others).

```sql
SELECT TO_CHAR(placed_at, 'YYYY-MM-DD HH24:MI') AS iso_style,
       TO_CHAR(placed_at, 'FMDay, DD Mon YYYY')  AS long_style
FROM orders
WHERE customer_name = 'Asha Verma';
```

```
     iso_style      |        long_style
----------------------+---------------------------
 2026-03-14 09:32    | Saturday, 14 Mar 2026
(1 row)
```

The `FM` prefix ("fill mode") in `'FMDay, ...'` suppresses a detail that trips people up: without it, `'Day'` pads the day name with trailing spaces out to a fixed width of nine characters (so `'Saturday'` — eight letters — would render as `'Saturday '` with a trailing space, while `'Day'` alone for Sunday would render as `'Sunday   '` with three trailing spaces) so that every formatted day name lines up to the same width in a fixed-width report. `FM` removes that padding when you don't want it.

## Internal Working (Preview)

Internally, PostgreSQL stores `TIMESTAMP` values as a single integer count of microseconds relative to a fixed epoch (2000-01-01, internally), and `DATE` as a count of days relative to the same epoch. `EXTRACT`, `DATE_TRUNC`, and `AGE()` all work by converting that internal integer representation into calendar fields (year, month, day, hour, minute, second) using the server's date/time calculation library, then either returning one field (`EXTRACT`), zeroing out fields below a threshold (`DATE_TRUNC`), or computing a field-by-field calendar difference with borrowing between units (`AGE()` — precisely how "borrowing" a day to make a negative time component positive was demonstrated in the worked example above). `TO_CHAR` runs the reverse process: calendar fields are converted back into a formatted string according to the pattern codes you supply.

```
 Stored value (microseconds/days since epoch)
        │
        ▼
 Decomposed into calendar fields (year, month, day, hour, minute, second, weekday...)
        │
        ├─▶ EXTRACT      → returns one field
        ├─▶ DATE_TRUNC    → zeroes out fields below the requested unit
        ├─▶ AGE()         → field-by-field calendar subtraction, with borrowing
        └─▶ TO_CHAR       → fields re-assembled into a formatted string
```

## Real-World Analogy

Think of a stored timestamp like a single, precise reading on a scientific instrument — a specific instant, down to the microsecond. `EXTRACT` is like a technician asking "just tell me what year this reading was taken" — pulling one fact out of the full precision. `DATE_TRUNC` is like rounding that precise reading to the nearest hour marked on a wall clock, for a report that doesn't need microsecond precision. `AGE()` is like a person calculating their exact age for a form — not "how many total days old," but "X years, Y months, and Z days," the same calendar-aware breakdown humans naturally think in. `TO_CHAR` is the instrument's display panel, choosing how to print that reading for whoever's looking at it — military time vs. 12-hour clock, abbreviated vs. full month name.

## Why These Functions Were Designed This Way

Dates and times are unusually irregular for arithmetic: months have different lengths, some years have an extra day, and "how many months between two dates" isn't answerable by simple subtraction the way "how many days" is. SQL's date/time functions exist specifically to hide that irregularity behind a calendar-aware API: `INTERVAL` arithmetic and `AGE()` correctly account for months of different lengths and leap years internally, so you never have to hand-write "days in this month" logic yourself. This mirrors SQL's broader declarative philosophy (Module 1) — you describe *what* time calculation you want ("the age of this order," "the start of this order's month") and the engine's date/time library handles the calendar mechanics correctly, every time, regardless of which specific month or year is involved.

## Advantages

- **Calendar-correctness built in.** Leap years, variable month lengths, and daylight-saving edge cases (for timezone-aware types) are handled by the engine, not by hand-written application logic prone to off-by-one bugs.
- **A single toolkit covers extraction, bucketing, arithmetic, and formatting.** `EXTRACT`, `DATE_TRUNC`, `INTERVAL` arithmetic, `AGE()`, and `TO_CHAR` between them answer nearly every common "when" question without needing anything more specialized.
- **Consistent formatting logic centralized in the query**, rather than every downstream consumer of the data re-implementing its own date formatting.

## Disadvantages / Limitations

- **Timezone handling adds real complexity.** `TIMESTAMP` (without time zone) stores a "wall clock" value with no timezone context, while `TIMESTAMPTZ` normalizes everything to UTC internally and converts on display — mixing the two carelessly is a common source of subtly wrong reports, especially across daylight-saving transitions.
- **`DATE_TRUNC` and `EXTRACT` field names are not always identical across database vendors** — this is exactly the kind of thing Module 22 (SQL Across Databases) maps out in detail; the concepts transfer, the exact function names sometimes don't.
- **`TO_CHAR` format codes must be memorized or looked up** — there's no getting around learning the pattern codes (`YYYY`, `MM`, `Mon`, `HH24`, and so on) since they aren't self-explanatory the first time you see them.

## Best Practices

- Use `TIMESTAMPTZ` (not plain `TIMESTAMP`) for anything that will ever be viewed by users in different timezones — Module 3 covers this distinction, but it's worth restating here since it directly affects how `NOW()`/`EXTRACT`/`AGE()` behave.
- Use `DATE_TRUNC` (not manual `EXTRACT`-based reconstruction) whenever you need to group rows into a time bucket for reporting — it's both simpler and less error-prone.
- Prefer `INTERVAL` arithmetic (`date + INTERVAL '1 month'`) over manually adding a fixed number of days when the intent is calendar-based ("one month later"), since a fixed day count doesn't correctly account for variable month lengths.
- Always test `TO_CHAR` format strings against a couple of edge-case dates (start/end of month, a single-digit day) before trusting the output in a report.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Using `NOW() - column` to compute "how long ago" and expecting readable output | This returns a raw `INTERVAL` in a single unit form that can be awkward to read (e.g., a huge number of days) — `AGE()` gives the same information broken into years/months/days, which is what most reports actually want. |
| Assuming subtracting two `TIMESTAMP` values (not `DATE`) gives a whole number of days | Subtracting two timestamps gives an `INTERVAL` that can include hours/minutes/seconds, not necessarily a clean whole-day count — cast to `::DATE` first if you specifically want whole days. |
| Forgetting the `FM` prefix and being confused by unexpected trailing spaces in `TO_CHAR` output | Without `FM`, `TO_CHAR` pads certain fields (like `Day`) to a fixed width with trailing spaces so output columns align — this is intentional default behavior, not a bug, but it surprises people expecting a plain trimmed string. |
| Adding a fixed number of days to approximate "a month later" | Months vary from 28 to 31 days; `date + INTERVAL '30 days'` is not reliably "one month later" — use `date + INTERVAL '1 month'` instead, which is calendar-aware. |

## Interview Questions

1. **Q: What's the difference between `EXTRACT` and `DATE_TRUNC`?**
   A: `EXTRACT(field FROM value)` returns a single numeric component (e.g., just the year, or just the day of week) and discards everything else. `DATE_TRUNC(unit, value)` returns a full date/time value, rounded down so everything below the requested unit is zeroed out — useful for bucketing rows into a coarser time granularity while keeping a real, comparable date/time value.

2. **Q: How does `AGE()` differ from simply subtracting one timestamp from another?**
   A: Plain subtraction between two timestamps yields a raw `INTERVAL`, typically expressed in a way that may not be calendar-broken-down (e.g., a large day count). `AGE()` explicitly computes a calendar-aware breakdown into years, months, days, and time — the same way a person would naturally express "how old" or "how long ago" something is.

3. **Q: Why might `date_column + 30` behave differently from what you'd expect for "add one month"?**
   A: Because months don't all have the same number of days — adding a flat 30 days is not equivalent to advancing exactly one calendar month for every starting date. `date_column + INTERVAL '1 month'` correctly advances by a full calendar month regardless of that month's actual length.

## Summary

- `NOW()`/`CURRENT_TIMESTAMP` return the current date and time; `CURRENT_DATE` returns only the date, with no time component.
- `EXTRACT` pulls out one calendar field; `DATE_TRUNC` rounds an entire value down to the start of a given unit, keeping it a full date/time value.
- `INTERVAL` arithmetic lets you add/subtract calendar-aware durations directly; subtracting two dates gives a plain day count, while subtracting two timestamps gives a full interval.
- `AGE()` produces a human-readable, calendar-aware breakdown of elapsed time (years/months/days/time), rather than one raw unit.
- `TO_CHAR` formats a date/time value into a display string using pattern codes — remember the `FM` prefix if you don't want fixed-width padding.
- `NULL` propagates through date arithmetic exactly as it does with strings and numbers — a missing date correctly produces a `NULL` result, not a misleading zero.
