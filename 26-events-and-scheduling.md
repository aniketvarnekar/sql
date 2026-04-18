# Events and Scheduling

MySQL's Event Scheduler is a built-in mechanism for running SQL code automatically at scheduled times — without an external cron daemon, application process, or operating system scheduler. Events are ideal for recurring database maintenance tasks like purging old records, refreshing summary tables, or generating periodic reports.

## MySQL Event Scheduler

The Event Scheduler is a background thread in the MySQL server that wakes up periodically and executes any events whose scheduled time has arrived. It is controlled by the `event_scheduler` system variable.

### Enabling the Event Scheduler

Check whether the scheduler is running:

```sql
SHOW VARIABLES LIKE 'event_scheduler';
```

Enable it for the current server session (until restart):

```sql
SET GLOBAL event_scheduler = ON;
```

To enable it permanently, add the following to the MySQL configuration file (`my.cnf` or `my.ini`) under `[mysqld]`:

```
event_scheduler = ON
```

Without enabling the Event Scheduler, events are stored and defined but never executed.

## CREATE EVENT

The basic syntax for a one-time event:

```sql
CREATE EVENT cleanup_old_sessions
ON SCHEDULE AT '2024-12-31 23:59:00'
DO
    DELETE FROM sessions WHERE last_active < DATE_SUB(NOW(), INTERVAL 30 DAY);
```

For a recurring event:

```sql
CREATE EVENT purge_old_logs
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'
DO
    DELETE FROM application_logs WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
```

The `STARTS` clause sets when the schedule begins. Without it, the event starts immediately when created.

For events with multi-statement bodies, use `BEGIN ... END` with a delimiter change:

```sql
DELIMITER //

CREATE EVENT nightly_summary
ON SCHEDULE EVERY 1 DAY
STARTS (CURRENT_DATE + INTERVAL 1 DAY + INTERVAL 2 HOUR)
COMMENT 'Rebuilds the daily_summary table each night at 2 AM'
DO
BEGIN
    TRUNCATE TABLE daily_summary;
    INSERT INTO daily_summary (date, order_count, total_revenue)
    SELECT
        DATE(order_date),
        COUNT(*),
        SUM(total)
    FROM orders
    WHERE order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
    GROUP BY DATE(order_date);
END //

DELIMITER ;
```

### INTERVAL Units

The `EVERY` clause accepts standard interval units:

- `EVERY 1 SECOND`, `EVERY 5 MINUTE`, `EVERY 1 HOUR`
- `EVERY 1 DAY`, `EVERY 1 WEEK`, `EVERY 1 MONTH`, `EVERY 1 YEAR`
- `EVERY 30 MINUTE`, `EVERY 6 HOUR`, `EVERY 2 WEEK`

## ALTER EVENT

To modify an existing event's schedule, body, or status:

```sql
-- Change the schedule:
ALTER EVENT purge_old_logs
ON SCHEDULE EVERY 12 HOUR;

-- Disable the event without dropping it:
ALTER EVENT purge_old_logs DISABLE;

-- Re-enable it:
ALTER EVENT purge_old_logs ENABLE;

-- Rename the event:
ALTER EVENT purge_old_logs RENAME TO purge_application_logs;
```

## DROP EVENT

```sql
DROP EVENT IF EXISTS purge_old_logs;
```

## Viewing Events

```sql
-- List all events in the current database:
SHOW EVENTS;

-- Detailed view from information_schema:
SELECT
    event_name,
    event_type,
    execute_at,
    interval_value,
    interval_field,
    status,
    last_executed,
    event_comment
FROM information_schema.events
WHERE event_schema = DATABASE()
ORDER BY event_name;
```

## One-Time vs Recurring Events

A one-time event executes once at a specific timestamp and is then automatically dropped (by default):

```sql
-- Execute once and drop:
CREATE EVENT one_time_migration
ON SCHEDULE AT NOW() + INTERVAL 5 MINUTE
ON COMPLETION NOT PRESERVE
DO
    UPDATE products SET price = price * 1.05 WHERE category = 'Books';
```

`ON COMPLETION NOT PRESERVE` (the default for one-time events) drops the event after it executes. `ON COMPLETION PRESERVE` keeps the event definition in the database after it runs, with status set to `DISABLED`.

A recurring event runs repeatedly until it is explicitly dropped or disabled:

```sql
CREATE EVENT weekly_archive
ON SCHEDULE EVERY 1 WEEK
STARTS '2024-01-07 01:00:00'
ON COMPLETION PRESERVE
DO
    INSERT INTO archived_orders SELECT * FROM orders WHERE order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);
```

## Use Cases

**Automated archiving:** Move old records from active tables to archive tables periodically, keeping active tables small and fast:

```sql
CREATE EVENT archive_old_orders
ON SCHEDULE EVERY 1 DAY
STARTS CURRENT_TIMESTAMP + INTERVAL 1 HOUR
DO
BEGIN
    INSERT INTO orders_archive SELECT * FROM orders WHERE order_date < DATE_SUB(CURDATE(), INTERVAL 2 YEAR);
    DELETE FROM orders WHERE order_date < DATE_SUB(CURDATE(), INTERVAL 2 YEAR);
END;
```

**Cleanup jobs:** Delete expired sessions, tokens, temporary records, or log entries:

```sql
CREATE EVENT cleanup_expired_tokens
ON SCHEDULE EVERY 1 HOUR
DO
    DELETE FROM password_reset_tokens WHERE expires_at < NOW();
```

**Reporting:** Refresh materialized summary tables during off-peak hours:

```sql
CREATE EVENT refresh_product_stats
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    TRUNCATE TABLE product_stats;
    INSERT INTO product_stats (product_id, total_sold, revenue)
    SELECT product_id, SUM(quantity), SUM(quantity * unit_price)
    FROM order_items
    GROUP BY product_id;
END;
```

## Considerations and Limitations

Events run within the MySQL server process and consume server resources. Long-running events can impact database performance, particularly if they run during peak hours. Schedule maintenance events during off-peak hours using the `STARTS` clause.

Events run with the privileges of the user who created them. Ensure the creating user has the necessary permissions to perform the event's operations.

If an event takes longer to execute than its schedule interval, MySQL does not queue multiple concurrent executions — the event scheduler will skip the missed run and wait for the next scheduled time.

Events are not replicated by default in MySQL replication setups. On a primary/replica pair, events typically run only on the primary. Check `log_bin_trust_function_creators` and event-specific replication settings when working in replicated environments.

## Practice Problems

### Problem 1
Create a recurring event that runs every day at 3 AM and deletes all sessions from a `user_sessions` table where `last_active` is more than 24 hours ago.

**Schema used:**
```sql
CREATE TABLE user_sessions (
    id VARCHAR(64) PRIMARY KEY,
    user_id INT UNSIGNED NOT NULL,
    last_active TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Ensure the event scheduler is on:
SET GLOBAL event_scheduler = ON;
```

**Your query:**
```sql
-- Write the CREATE EVENT statement.
```

**Solution:**
```sql
CREATE EVENT cleanup_expired_sessions
ON SCHEDULE EVERY 1 DAY
STARTS (CURRENT_DATE + INTERVAL 1 DAY + INTERVAL 3 HOUR)
ON COMPLETION PRESERVE
COMMENT 'Removes inactive sessions older than 24 hours'
DO
    DELETE FROM user_sessions
    WHERE last_active < DATE_SUB(NOW(), INTERVAL 24 HOUR);
```

**Explanation:**
`CURRENT_DATE + INTERVAL 1 DAY + INTERVAL 3 HOUR` calculates tomorrow at 3 AM, so the event first runs the next calendar day at 3 AM and then repeats every 24 hours. `ON COMPLETION PRESERVE` keeps the event definition active so it continues to repeat indefinitely. `DELETE FROM user_sessions WHERE last_active < DATE_SUB(NOW(), INTERVAL 24 HOUR)` removes any session not active in the last 24 hours. For large session tables, adding `LIMIT 10000` per run and letting multiple runs clean up in chunks prevents a single long-running delete from holding locks.

### Problem 2
Create a one-time event that runs 10 minutes from now and applies a 5% price increase to all products in the 'Seasonal' category. The event should be dropped after it runs.

**Schema used:**
```sql
CREATE TABLE products (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(100) NOT NULL,
    price DECIMAL(8, 2) NOT NULL
);

INSERT INTO products (name, category, price) VALUES
    ('Summer Hat', 'Seasonal', 19.99),
    ('Winter Gloves', 'Seasonal', 14.99),
    ('Laptop', 'Electronics', 999.99);
```

**Your query:**
```sql
-- Write the one-time CREATE EVENT.
```

**Solution:**
```sql
CREATE EVENT seasonal_price_increase
ON SCHEDULE AT NOW() + INTERVAL 10 MINUTE
ON COMPLETION NOT PRESERVE
COMMENT 'One-time 5% price increase for Seasonal category'
DO
    UPDATE products
    SET price = ROUND(price * 1.05, 2)
    WHERE category = 'Seasonal';
```

**Explanation:**
`AT NOW() + INTERVAL 10 MINUTE` schedules the event to fire exactly 10 minutes from when the `CREATE EVENT` statement runs. `ON COMPLETION NOT PRESERVE` (which is actually the default for one-time events) automatically drops the event definition from the database after it executes. The `UPDATE` applies a 5% increase with `ROUND(..., 2)` to keep prices at two decimal places. One-time events are useful for scheduled schema changes, data migrations, or timed price adjustments that would otherwise require an external scheduler.

### Problem 3
Create a recurring event that runs every hour and populates a `hourly_stats` summary table with the order count and total revenue for the past hour.

**Schema used:**
```sql
CREATE TABLE orders (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    total DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE hourly_stats (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    period_start DATETIME NOT NULL,
    order_count INT NOT NULL DEFAULT 0,
    total_revenue DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    computed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**Your query:**
```sql
-- Write the CREATE EVENT.
```

**Solution:**
```sql
DELIMITER //

CREATE EVENT compute_hourly_stats
ON SCHEDULE EVERY 1 HOUR
STARTS (DATE_FORMAT(NOW(), '%Y-%m-%d %H:00:00') + INTERVAL 1 HOUR)
ON COMPLETION PRESERVE
COMMENT 'Computes order stats for the previous hour'
DO
BEGIN
    DECLARE v_period_start DATETIME;
    SET v_period_start = DATE_FORMAT(NOW() - INTERVAL 1 HOUR, '%Y-%m-%d %H:00:00');

    INSERT INTO hourly_stats (period_start, order_count, total_revenue)
    SELECT
        v_period_start,
        COUNT(*),
        COALESCE(SUM(total), 0)
    FROM orders
    WHERE created_at >= v_period_start
      AND created_at < v_period_start + INTERVAL 1 HOUR;
END //

DELIMITER ;
```

**Explanation:**
`DATE_FORMAT(NOW(), '%Y-%m-%d %H:00:00') + INTERVAL 1 HOUR` calculates the start of the next full hour so the event aligns to the top of each hour. Inside the event body, `v_period_start` is set to the start of the previous hour (the period just completed). The `INSERT ... SELECT` counts orders and sums revenue for exactly that one-hour window. `COALESCE(SUM(total), 0)` handles the case where no orders were placed in the hour. This pattern creates an append-only history of hourly statistics that can be queried quickly without hitting the raw `orders` table.
