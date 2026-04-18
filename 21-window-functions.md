# Window Functions

Window functions compute values across a set of rows that are related to the current row — a "window" of rows — without collapsing them into a single aggregate row. Unlike `GROUP BY` which produces one row per group, window functions return one row per input row with an additional computed column. They were introduced to MySQL in version 8.0 and enable a class of analytical queries that previously required complex self-joins or subqueries.

## What Window Functions Are and How They Differ from GROUP BY

With `GROUP BY`, multiple rows are collapsed into one:

```sql
-- GROUP BY: returns one row per department
SELECT department, AVG(salary) AS avg_salary FROM employees GROUP BY department;
```

With a window function, every row is preserved and gains an additional computed value:

```sql
-- Window function: returns one row per employee, with the department average alongside
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM employees;
```

This is the fundamental difference: `GROUP BY` reduces, window functions annotate.

## OVER(), PARTITION BY, ORDER BY Inside OVER

Every window function uses an `OVER()` clause that defines the window. An empty `OVER()` means the window is the entire result set:

```sql
SELECT name, salary, AVG(salary) OVER () AS overall_avg FROM employees;
```

`PARTITION BY` divides the rows into groups (partitions), and the window function is computed independently within each partition:

```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

`ORDER BY` inside `OVER()` defines the ordering of rows within each partition. This is required by ranking functions and affects frame-based functions:

```sql
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees;
```

`PARTITION BY` and `ORDER BY` can be combined:

```sql
SELECT name, department, salary,
       SUM(salary) OVER (PARTITION BY department ORDER BY hire_date) AS running_dept_total
FROM employees;
```

## Ranking Functions: ROW_NUMBER, RANK, DENSE_RANK

`ROW_NUMBER()` assigns a unique sequential number to each row within the partition. No ties — even rows with identical sort key values get different numbers:

```sql
SELECT name, score,
       ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num
FROM test_results;
-- Row 1, 2, 3, ... regardless of ties in score
```

`RANK()` assigns ranks with gaps for ties. If two rows tie for rank 2, both get rank 2, and the next rank is 4 (not 3):

```sql
SELECT name, score,
       RANK() OVER (ORDER BY score DESC) AS rank_num
FROM test_results;
-- Scores 100, 95, 95, 90 get ranks 1, 2, 2, 4
```

`DENSE_RANK()` assigns ranks without gaps. Ties get the same rank, and the next rank increments by 1:

```sql
SELECT name, score,
       DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank_num
FROM test_results;
-- Scores 100, 95, 95, 90 get ranks 1, 2, 2, 3
```

A common use case is "top N per group" — the top 3 salespeople per region, for example:

```sql
SELECT *
FROM (
    SELECT name, region, sales,
           RANK() OVER (PARTITION BY region ORDER BY sales DESC) AS rank_in_region
    FROM salespeople
) ranked
WHERE rank_in_region <= 3;
```

## LEAD and LAG

`LAG(column, offset, default)` accesses the value from a previous row within the partition. `LEAD(column, offset, default)` accesses the value from a following row:

```sql
SELECT
    order_date,
    total,
    LAG(total) OVER (ORDER BY order_date) AS prev_day_total,
    total - LAG(total) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_sales;
```

`offset` defaults to 1 (one row back or ahead). `default` specifies the value to use when there is no previous/next row (NULL by default):

```sql
SELECT
    month,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY month) AS prev_month_revenue,
    LEAD(revenue, 1, 0) OVER (ORDER BY month) AS next_month_revenue
FROM monthly_revenue;
```

## FIRST_VALUE and LAST_VALUE

`FIRST_VALUE(column)` returns the value of the specified column from the first row in the window frame. `LAST_VALUE(column)` returns the value from the last row:

```sql
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(name) OVER (PARTITION BY department ORDER BY salary DESC) AS highest_earner
FROM employees;
```

`LAST_VALUE` requires careful frame specification because its default frame ends at the current row, making it return the current row's value in most contexts. To get the true last value in the partition, use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`:

```sql
SELECT
    name,
    salary,
    LAST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_earner
FROM employees;
```

## SUM and AVG as Window Functions

Standard aggregate functions become window functions when combined with `OVER()`. This allows running totals, moving averages, and cumulative percentages:

```sql
-- Running total of sales:
SELECT order_date, total,
       SUM(total) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- Moving 7-day average:
SELECT order_date, total,
       AVG(total) OVER (
           ORDER BY order_date
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS moving_avg_7day
FROM orders;

-- What percentage of total revenue does each order represent:
SELECT id, total,
       ROUND(total / SUM(total) OVER () * 100, 2) AS pct_of_total
FROM orders;
```

## Frame Specification: ROWS vs RANGE

The frame clause, inside `OVER()`, defines which rows are included in the window for each row being computed. It comes after `ORDER BY` inside `OVER()`:

```sql
OVER (ORDER BY column ROWS BETWEEN start AND end)
OVER (ORDER BY column RANGE BETWEEN start AND end)
```

Frame boundaries:
- `UNBOUNDED PRECEDING` — all rows before the current row (from the start of the partition)
- `n PRECEDING` — n rows before the current row
- `CURRENT ROW` — the current row
- `n FOLLOWING` — n rows after the current row
- `UNBOUNDED FOLLOWING` — all rows after the current row (to the end of the partition)

`ROWS` mode counts physical rows. `RANGE` mode includes all rows with the same `ORDER BY` value as the current row. The default when `ORDER BY` is specified is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

```sql
-- Cumulative sum (all rows from start up to current):
SUM(total) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- 3-row moving average (current row plus one before and one after):
AVG(total) OVER (ORDER BY order_date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)
```

## Practice Problems

### Problem 1
Write a query that returns each employee's name, department, salary, their rank within their department by salary (highest first), and the department's highest salary for comparison.

**Schema used:**
```sql
CREATE TABLE employees (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    department VARCHAR(100) NOT NULL,
    salary DECIMAL(10, 2) NOT NULL
);

INSERT INTO employees (name, department, salary) VALUES
    ('Alice', 'Engineering', 95000),
    ('Bob', 'Engineering', 85000),
    ('Carol', 'Engineering', 105000),
    ('Dave', 'Marketing', 65000),
    ('Eve', 'Marketing', 72000),
    ('Frank', 'Sales', 55000),
    ('Grace', 'Sales', 55000);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    name,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank,
    MAX(salary) OVER (PARTITION BY department) AS dept_max_salary,
    ROUND(salary / MAX(salary) OVER (PARTITION BY department) * 100, 1) AS pct_of_dept_max
FROM employees
ORDER BY department, salary_rank;
```

**Explanation:**
`RANK() OVER (PARTITION BY department ORDER BY salary DESC)` ranks employees within each department by salary in descending order. Frank and Grace in Sales both earn $55,000, so they share rank 1 (and there is no rank 2). `MAX(salary) OVER (PARTITION BY department)` computes the maximum salary within each department and shows it alongside every row — this would be impossible with a plain `GROUP BY` without losing individual employee rows. The percentage calculation shows how each salary compares to the department maximum.

### Problem 2
Write a query that shows each month's sales total alongside the previous month's total and the difference between them (month-over-month change).

**Schema used:**
```sql
CREATE TABLE monthly_sales (
    month_date DATE NOT NULL PRIMARY KEY,
    total DECIMAL(12, 2) NOT NULL
);

INSERT INTO monthly_sales VALUES
    ('2023-01-01', 45000),
    ('2023-02-01', 52000),
    ('2023-03-01', 48000),
    ('2023-04-01', 61000),
    ('2023-05-01', 57000);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    DATE_FORMAT(month_date, '%Y-%m') AS month,
    total AS current_month,
    LAG(total) OVER (ORDER BY month_date) AS previous_month,
    total - LAG(total) OVER (ORDER BY month_date) AS month_over_month_change,
    ROUND(
        (total - LAG(total) OVER (ORDER BY month_date))
        / LAG(total) OVER (ORDER BY month_date) * 100,
        1
    ) AS pct_change
FROM monthly_sales
ORDER BY month_date;
```

**Explanation:**
`LAG(total) OVER (ORDER BY month_date)` returns the `total` from the previous row when rows are sorted by month. For January (the first row), `LAG` returns NULL because there is no preceding row. The difference and percentage change for January will also be NULL, which is correct — there is nothing to compare January against. `DATE_FORMAT(month_date, '%Y-%m')` formats the date as 'YYYY-MM' for readability.

### Problem 3
Write a query that computes a 3-month moving average of sales totals, where the average includes the current month and the two preceding months.

**Schema used:**
```sql
CREATE TABLE monthly_sales (
    month_date DATE NOT NULL PRIMARY KEY,
    total DECIMAL(12, 2) NOT NULL
);

INSERT INTO monthly_sales VALUES
    ('2023-01-01', 45000),
    ('2023-02-01', 52000),
    ('2023-03-01', 48000),
    ('2023-04-01', 61000),
    ('2023-05-01', 57000),
    ('2023-06-01', 70000);
```

**Your query:**
```sql
-- Write the SELECT here.
```

**Solution:**
```sql
SELECT
    DATE_FORMAT(month_date, '%Y-%m') AS month,
    total,
    ROUND(
        AVG(total) OVER (
            ORDER BY month_date
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    ) AS moving_avg_3month
FROM monthly_sales
ORDER BY month_date;
```

**Explanation:**
`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` defines a frame that includes the current row and the two rows immediately before it (sorted by `month_date`). For January, the frame has only 1 row (no preceding rows), so the average is just January's value. For February, the frame has 2 rows, giving a 2-month average. From March onward, the frame has 3 rows, giving a true 3-month rolling average. `ROWS` mode (rather than `RANGE`) is correct here because we want exactly 2 preceding rows, not rows within a date range.
