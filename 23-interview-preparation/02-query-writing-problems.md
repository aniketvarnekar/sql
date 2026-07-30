# Query-Writing Problems

A query-writing interview round rarely asks you to define a term — it hands you a small schema and a plain-English request and watches how you get from one to the other. The problems below are ordered roughly by difficulty. For each one, try writing your own query against the given schema before reading the worked solution.

## Problem 1 — Second-Highest Salary in Each Department

**Schema**

```sql
CREATE TABLE departments (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE employees (
    id            SERIAL PRIMARY KEY,
    name          TEXT NOT NULL,
    department_id INTEGER NOT NULL REFERENCES departments(id),
    salary        NUMERIC NOT NULL
);
```

**Problem:** For each department, find the employee(s) with the second-highest distinct salary. If a department has fewer than two distinct salary values, it shouldn't appear in the result at all.

**Solution**

```sql
WITH ranked AS (
    SELECT
        department_id,
        name,
        salary,
        DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT department_id, name, salary
FROM ranked
WHERE salary_rank = 2;
```

**Technique:** `DENSE_RANK()` is used instead of `RANK()` deliberately — if two employees tie for the highest salary in a department, `RANK()` would skip straight to rank 3 for the next distinct value, silently making "second-highest" mean something different than intended. `DENSE_RANK()` never skips numbers, so rank 2 always means "the second-highest distinct salary value." This is a window-function problem (Module 16, Window Functions), partitioned per department so the ranking restarts for each one.

## Problem 2 — Customers Who Never Placed an Order

**Schema**

```sql
CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    order_date  DATE NOT NULL
);
```

**Problem:** List every customer who has never placed a single order.

**Solution**

```sql
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

**Technique:** This is the anti-join pattern via a correlated subquery (Module 11, Subqueries): `NOT EXISTS` re-checks, per customer row, whether any matching order exists. It's preferred over `NOT IN (SELECT customer_id FROM orders)` because `NOT IN` returns zero rows for every customer the moment the subquery's list contains even one `NULL` `customer_id` — a trap `NOT EXISTS` doesn't have.

## Problem 3 — Find and Remove Duplicate Rows, Keeping the Earliest

**Schema**

```sql
CREATE TABLE customers (
    id         SERIAL PRIMARY KEY,
    email      TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

**Problem:** Some rows were accidentally inserted twice with the same email address. First, find every duplicated email. Then, write a single statement that removes the duplicate rows, keeping only the earliest-created row for each email.

**Solution**

```sql
-- Step 1: find the duplicated emails
SELECT email, COUNT(*) AS occurrences
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;

-- Step 2: delete every duplicate except the earliest row per email
DELETE FROM customers c
USING (
    SELECT
        id,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at ASC) AS rn
    FROM customers
) ranked
WHERE c.id = ranked.id
  AND ranked.rn > 1;
```

**Technique:** `GROUP BY` with `HAVING COUNT(*) > 1` (Module 09, Aggregation) identifies which values repeat. The delete step uses `ROW_NUMBER()` (Module 16, Window Functions) to number every row within each email group in creation order, then `DELETE ... USING` (Module 06, Modifying Data) removes every row whose rank is greater than 1 — a single set-based statement instead of a row-by-row loop.

## Problem 4 — Employees Earning More Than Their Manager

**Schema**

```sql
CREATE TABLE employees (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    salary     NUMERIC NOT NULL,
    manager_id INTEGER REFERENCES employees(id)
);
```

**Problem:** List every employee who earns more than their direct manager.

**Solution**

```sql
SELECT
    e.name   AS employee,
    e.salary AS employee_salary,
    m.name   AS manager,
    m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

**Technique:** A self-join (Module 10, Self-Joins): the same `employees` table is joined to itself, aliased once as `e` ("the employee") and once as `m` ("their manager"), connected through the self-referencing `manager_id` foreign key. Aliasing is mandatory here — without it, the two roles the table plays would be ambiguous.

## Problem 5 — The 3 Most Recent Orders per Customer

**Schema**

```sql
CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    id            SERIAL PRIMARY KEY,
    customer_id   INTEGER NOT NULL REFERENCES customers(id),
    order_date    DATE NOT NULL,
    total_amount  NUMERIC NOT NULL
);
```

**Problem:** For every customer, return only their 3 most recent orders.

**Solution**

```sql
WITH ranked_orders AS (
    SELECT
        id,
        customer_id,
        order_date,
        total_amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
SELECT id, customer_id, order_date, total_amount
FROM ranked_orders
WHERE rn <= 3;
```

**Technique:** The classic "top-N per group" pattern (Module 16, Window Functions). `ROW_NUMBER()` numbers each customer's orders from most recent to least recent, restarting at 1 for every customer because of `PARTITION BY customer_id`; filtering `rn <= 3` then keeps exactly the top 3 rows per group.

## Problem 6 — Running Total and 7-Day Moving Average of Daily Sales

**Schema**

```sql
CREATE TABLE daily_sales (
    sale_date DATE PRIMARY KEY,
    revenue   NUMERIC NOT NULL
);
```

**Problem:** For each day, show that day's revenue, a running total of all revenue up to and including that day, and a trailing 7-day moving average of revenue.

**Solution**

```sql
SELECT
    sale_date,
    revenue,
    SUM(revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    AVG(revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_sales
ORDER BY sale_date;
```

**Technique:** Explicit window frame clauses (Module 16, Running Totals and Moving Averages). `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` includes every row from the start of the ordered set through the current one, producing a cumulative sum. `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` includes exactly the current row plus the six before it — a trailing 7-row window — for the moving average.

## Problem 7 — Products That Have Never Been Ordered

**Schema**

```sql
CREATE TABLE products (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE order_items (
    id         SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL REFERENCES products(id),
    order_id   INTEGER NOT NULL,
    quantity   INTEGER NOT NULL CHECK (quantity > 0)
);
```

**Problem:** List every product that has never appeared in any order.

**Solution**

```sql
SELECT p.id, p.name
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
WHERE oi.id IS NULL;
```

**Technique:** The same anti-join goal as Problem 2, expressed with the other equally valid form: a `LEFT JOIN` (Module 10, Joins & Set Operations) keeps every product row regardless of a match, filling `order_items` columns with `NULL` where there's no match, and `WHERE oi.id IS NULL` keeps only the rows where that non-match happened. Either this or the `NOT EXISTS` form from Problem 2 is a correct answer in an interview — knowing both, and being able to say why they're equivalent, is the actual signal.

## Problem 8 — Employees Earning Above Their Department's Average

**Schema**

```sql
CREATE TABLE employees (
    id            SERIAL PRIMARY KEY,
    name          TEXT NOT NULL,
    department_id INTEGER NOT NULL,
    salary        NUMERIC NOT NULL
);
```

**Problem:** List every employee whose salary is above the average salary of their own department.

**Solution**

```sql
SELECT e.id, e.name, e.department_id, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

**Technique:** A correlated subquery (Module 11, Subqueries): the inner `AVG` is recomputed against a different set of rows for every outer employee, because the inner query's `WHERE` clause references the outer row's `e.department_id`. This is a good problem to also solve with a window function (`AVG(salary) OVER (PARTITION BY department_id)` compared in an outer query) and mention the alternative out loud — it shows you know the same result can be reached from Module 11 or Module 16's toolbox.

## Problem 9 — All Direct and Indirect Reports of a Manager

**Schema**

```sql
CREATE TABLE employees (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    manager_id INTEGER REFERENCES employees(id)
);
```

**Problem:** Given a manager's name, list every employee who reports to them, directly or indirectly, at any depth in the hierarchy.

**Solution**

```sql
WITH RECURSIVE subordinates AS (
    -- Anchor member: employees who report directly to the given manager
    SELECT id, name, manager_id
    FROM employees
    WHERE manager_id = (SELECT id FROM employees WHERE name = 'Priya')

    UNION ALL

    -- Recursive member: employees who report to anyone already found
    SELECT e.id, e.name, e.manager_id
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT id, name
FROM subordinates;
```

**Technique:** A recursive CTE (Module 17, CTEs & Recursion). The anchor member finds the manager's direct reports; the recursive member repeatedly joins the `employees` table back against everyone the CTE has found so far, adding the next level down each pass, until a pass finds no new matches and the recursion stops naturally.

## Problem 10 — Monthly Revenue Pivoted into Columns per Category

**Schema**

```sql
CREATE TABLE sales (
    id        SERIAL PRIMARY KEY,
    category  TEXT NOT NULL,
    sale_date DATE NOT NULL,
    revenue   NUMERIC NOT NULL
);
```

**Problem:** Produce one row per category, with a separate column for that category's total revenue in each of January, February, and March.

**Solution**

```sql
SELECT
    category,
    SUM(revenue) FILTER (WHERE EXTRACT(MONTH FROM sale_date) = 1) AS jan_revenue,
    SUM(revenue) FILTER (WHERE EXTRACT(MONTH FROM sale_date) = 2) AS feb_revenue,
    SUM(revenue) FILTER (WHERE EXTRACT(MONTH FROM sale_date) = 3) AS mar_revenue
FROM sales
GROUP BY category
ORDER BY category;
```

**Technique:** Conditional aggregation via PostgreSQL's `FILTER` clause (Module 21, Pivoting Data with Conditional Aggregation), layered on top of ordinary `GROUP BY` (Module 09, Aggregation). Each `SUM(...) FILTER (WHERE ...)` computes a separate conditional sum within the same `GROUP BY category` pass, turning what would otherwise be separate rows (one per month) into separate columns of a single row per category.

## Problem 11 — Longest Consecutive-Day Login Streak per User

**Schema**

```sql
CREATE TABLE logins (
    id         SERIAL PRIMARY KEY,
    user_id    INTEGER NOT NULL,
    login_date DATE NOT NULL
);
```

**Problem:** For each user, find every run of consecutive calendar days on which they logged in, and report the length, start date, and end date of each such streak.

**Solution**

```sql
WITH distinct_logins AS (
    SELECT DISTINCT user_id, login_date
    FROM logins
),
numbered AS (
    SELECT
        user_id,
        login_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn
    FROM distinct_logins
),
grouped AS (
    SELECT
        user_id,
        login_date,
        (login_date - (rn || ' days')::INTERVAL)::DATE AS streak_group
    FROM numbered
)
SELECT
    user_id,
    COUNT(*)        AS streak_length,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end
FROM grouped
GROUP BY user_id, streak_group
ORDER BY user_id, streak_length DESC;
```

**Technique:** This is the "gaps and islands" pattern, the hardest problem in this set precisely because the trick isn't obvious the first time you see it. `ROW_NUMBER()` (Module 16, Window Functions) assigns a sequential number per user ordered by date; for any unbroken run of consecutive calendar days, both the date and the row number advance by exactly one per row, so subtracting the row number (as an interval) from the date produces the *same* resulting value — `streak_group` — for every row inside one streak, and a different value as soon as a day is skipped. `GROUP BY user_id, streak_group` (Module 09, Aggregation) then collapses each streak into a single summary row. Staging the computation through CTEs (Module 17, CTEs & Recursion) keeps each step legible instead of nesting three transformations inline.
