# Mock Interview Walkthrough

Reading model answers in isolation doesn't prepare you for what an interview actually feels like: a live conversation, a blank editor, and an interviewer who is watching your process as much as your final answer. What follows is a single, realistic mock interview transcript — warm-up conceptual questions, a full query-writing problem worked through out loud from a blank start, and a short wrap-up — followed by concrete guidance on presenting a SQL solution well.

## The Transcript

**Interviewer:** Thanks for joining. Let's start with something quick — can you tell me the difference between `WHERE` and `HAVING`?

**Candidate:** Sure. `WHERE` filters individual rows before any grouping happens, and `HAVING` filters groups after `GROUP BY` has collapsed rows together — so `HAVING` is the one you use when your filter condition involves an aggregate, like `COUNT(*) > 1`. If I tried to put an aggregate condition in `WHERE`, the aggregate doesn't exist yet at that point in query evaluation, so it'd either error or just not do what I want.

**Interviewer:** Good. And what about `LEFT JOIN` versus `INNER JOIN` — when would you reach for one over the other?

**Candidate:** `INNER JOIN` only keeps rows that match on both sides, so if I only care about rows that definitely have a related row in the other table, that's the right choice. `LEFT JOIN` keeps every row from the left table no matter what, filling in `NULL`s for the right side when there's no match — so I'd use that whenever I specifically need the "no match" case to show up, like finding customers with no orders, where the whole point is to see the customers that *don't* join to anything.

**Interviewer:** Perfect segue, actually — let's do exactly that as a live problem. I'll give you a schema.

```sql
CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    order_date  DATE NOT NULL,
    amount      NUMERIC NOT NULL
);
```

**Interviewer:** Write a query that returns, for each customer, their three most recent orders.

**Candidate:** Okay, let me make sure I understand the requirement first — for every customer, I want up to three rows, the ones with the most recent `order_date`, and if a customer has fewer than three orders total, I just want all of them. Is that right?

**Interviewer:** That's right.

**Candidate:** And is there any tie-breaking I should worry about if two orders land on the exact same date for the same customer?

**Interviewer:** Good question — let's say just pick any three in that case, no specific tie-break required.

**Candidate:** Got it. So my first instinct here is that this is a "top-N per group" problem, and the standard tool for that is a window function — specifically `ROW_NUMBER()`, partitioned per customer. Let me think through why a plain `GROUP BY` wouldn't work here first, actually, just to make sure I'm reaching for the right tool: `GROUP BY customer_id` would let me get one row per customer, but I need three rows per customer, not one aggregated row — so `GROUP BY` alone can't produce this shape of result at all. That confirms a window function is the right call, since it lets me rank rows within each customer without collapsing them.

So I'll start with a CTE that numbers each customer's orders from most recent to least recent.

```sql
WITH ranked_orders AS (
    SELECT
        id,
        customer_id,
        order_date,
        amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
```

**Candidate:** `PARTITION BY customer_id` restarts the numbering at 1 for every customer, and `ORDER BY order_date DESC` means rank 1 is that customer's most recent order. Then I just filter on the outer query.

```sql
SELECT id, customer_id, order_date, amount
FROM ranked_orders
WHERE rn <= 3;
```

**Candidate:** Let me trace through this mentally with a small example before I call it done — say customer 1 has five orders dated Jan through May. Partitioned and ordered descending, May gets rn 1, April gets rn 2, March gets rn 3, February gets rn 4, January gets rn 5. Filtering `rn <= 3` keeps May, April, March — exactly the three most recent. And if customer 2 only has one order total, that order gets rn 1, which passes the filter too, so the "fewer than three, just return them all" case is automatically handled — I don't need a separate branch for it.

**Interviewer:** Nice. One follow-up: this table has 50 million orders in production. Would this scale, and is there anything you'd want to check?

**Candidate:** The first thing I'd want to look at is `EXPLAIN ANALYZE` on this query against a realistic data size, because a window function like this needs to sort each partition by `order_date`, and if `customer_id` isn't indexed — or better, if there's a composite index on `(customer_id, order_date)` — the planner may end up doing an expensive sort across the whole table before it can compute the window function efficiently. A composite index in exactly that column order would let it read each customer's orders already in date order, which should turn that sort into something much cheaper, potentially just a scan of already-ordered index entries per customer.

**Interviewer:** Great. Last one — not a coding question. Tell me about a time you had to debug a slow query in production.

**Candidate:** At a previous project, a reporting query that used to run in under a second started taking twenty-plus seconds after a few months of data growth. I ran `EXPLAIN ANALYZE` on it instead of guessing, and the plan showed the estimated row count for one of the joins was wildly off from the actual row count — the table's statistics were stale because autovacuum hadn't run an `ANALYZE` recently after a big bulk load. I ran `ANALYZE` manually on the affected tables, the planner picked a completely different, much better join order immediately, and the query dropped back to under a second. The bigger fix afterward was tuning the autovacuum settings so statistics wouldn't go stale like that again after future bulk loads.

**Interviewer:** That's a solid answer — thanks, that's everything from my side today.

## Presentation Guidance

**Talk before you type.** State your understanding of the problem, and any assumptions you're making, before writing a single line of SQL. In the transcript above, the candidate confirmed the tie-break behavior *before* writing code — that single question would have been much more awkward to ask after presenting a finished, silent query.

**Narrate your reasoning, not just your syntax.** The candidate explained *why* a window function was the right tool — by first explaining why `GROUP BY` alone couldn't do the job — instead of just typing `ROW_NUMBER() OVER (...)` and hoping the interviewer inferred the reasoning. Interviewers are evaluating your thought process far more than they're evaluating whether you remember exact syntax; a candidate who narrates "here's why" consistently scores higher than one who silently produces a correct-looking query.

**Trace through your own query with a concrete example.** Before declaring a solution finished, the candidate walked through a small hypothetical example by hand. This catches real bugs before the interviewer has to point them out, and it visibly demonstrates a habit — verifying your own work — that matters just as much in a real job as it does in an interview.

**What interviewers are actually evaluating:**
- Whether you clarify ambiguous requirements instead of guessing and hoping.
- Whether you can explain *why* you chose one technique (a window function, a join, a subquery) over another plausible one.
- Whether you can reason about scale and performance beyond just getting a correct answer on a small example.
- Whether you communicate clearly enough that a teammate could follow your thinking in a real code review, not just whether the final query is correct.

**Common self-sabotage patterns to avoid:**
- **Silent typing.** Writing an entire query without saying a word turns a conversation into a test, and denies the interviewer the chance to redirect you early if you've misunderstood something — which is far worse for you than getting caught later.
- **Skipping clarifying questions.** Diving straight into code on an ambiguous prompt (what happens on a tie, what counts as "recent," should cancelled orders count) risks solving the wrong problem confidently.
- **Going quiet when stuck.** Freezing in silence reads as being lost; talking through a partial idea, even a wrong one — "my first instinct is X, but that might double-count in this case, let me reconsider" — reads as a strong engineer thinking in real time.
- **Never testing your own output.** Presenting a query as finished without mentally (or actually) running it against a small example misses bugs you could have caught yourself, and it's a habit interviewers specifically probe for.
- **Over-apologizing.** Repeatedly saying "sorry, this is probably wrong" undermines a correct answer just as much as an actual mistake would; state your reasoning confidently, and correct course calmly if you do find an issue.
