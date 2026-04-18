# MySQL Setup and Basics

Getting MySQL installed and knowing how to navigate the command-line interface is the practical foundation for everything else in these notes. This file covers how to connect to MySQL, the essential commands for orienting yourself in the CLI, how to run SQL files, and the case sensitivity rules that trip up many beginners.

## Installing MySQL

MySQL can be installed on Linux, macOS, and Windows. On Linux (Debian/Ubuntu), the standard approach is:

```bash
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
```

On macOS, Homebrew is the most common method:

```bash
brew install mysql
brew services start mysql
```

On Windows, download the MySQL Installer from the official MySQL website and run it. It installs the server, the CLI client, and optionally MySQL Workbench (a graphical interface).

After installation, run the secure installation script to set a root password and remove default insecure settings:

```bash
sudo mysql_secure_installation
```

## Connecting to MySQL

The MySQL command-line client is the primary tool for interacting with MySQL. To connect as the root user and be prompted for a password:

```bash
mysql -u root -p
```

The `-u` flag specifies the username and `-p` tells the client to prompt for a password. If you want to connect to a specific database immediately:

```bash
mysql -u root -p database_name
```

To connect to a remote MySQL server, add the `-h` flag for the host:

```bash
mysql -u username -p -h 192.168.1.100 database_name
```

Once connected, the prompt changes to `mysql>`, indicating that MySQL is ready to receive commands. Every statement you type must end with a semicolon (`;`) before pressing Enter, or MySQL will wait for more input. This allows multi-line statements — MySQL accumulates lines until it sees the semicolon.

## Essential CLI Commands

### Navigating Databases

To see all databases available to your user:

```sql
SHOW DATABASES;
```

To select a database to work in (all subsequent statements will run against this database):

```sql
USE database_name;
```

After running `USE`, the prompt changes to `mysql [database_name]>` in MySQL 8 to remind you which database is active.

To see all tables in the currently selected database:

```sql
SHOW TABLES;
```

To see the structure (columns, types, constraints) of a specific table:

```sql
DESCRIBE table_name;
```

`DESCRIBE` is shorthand for `SHOW COLUMNS FROM table_name`. Both produce the same output: the column name, data type, whether NULL is allowed, key information, default value, and any extras like `AUTO_INCREMENT`.

### Getting Status Information

To see the currently connected user, host, database, and MySQL version:

```sql
STATUS;
```

To see which database is currently selected:

```sql
SELECT DATABASE();
```

To see all running queries and connections:

```sql
SHOW PROCESSLIST;
```

### Getting Help

MySQL's built-in help system is useful when you forget syntax:

```sql
HELP SELECT;
HELP CREATE TABLE;
HELP 'data types';
```

## Running SQL Files

Rather than typing every statement into the CLI, you can write SQL in a `.sql` file and execute it. From within the MySQL CLI, use the `SOURCE` command:

```sql
SOURCE /path/to/file.sql;
```

The path must be an absolute path on the server. Alternatively, you can pipe the file directly when connecting:

```bash
mysql -u root -p database_name < /path/to/file.sql
```

This is useful for running migration scripts, loading seed data, or executing long setup scripts. If the file contains errors, MySQL will report them with the line number.

## Case Sensitivity Rules in MySQL

MySQL's case sensitivity behavior depends on the operating system, the file system, and configuration settings. Understanding these rules prevents hard-to-diagnose bugs.

### SQL Keywords and Function Names

SQL keywords (`SELECT`, `FROM`, `WHERE`, `CREATE TABLE`) and built-in function names (`NOW()`, `COUNT()`, `UPPER()`) are always case-insensitive in MySQL. Writing `select * from users` is identical to `SELECT * FROM users`. By convention, these notes capitalize keywords to distinguish them visually from identifiers and values, but MySQL does not require this.

### Database and Table Names

Database names and table names follow the case sensitivity of the underlying file system. On Linux (which typically uses a case-sensitive file system), `Users`, `users`, and `USERS` would be three different tables. On macOS and Windows (which default to case-insensitive file systems), they would be treated as the same table.

This creates a portability problem: code written on a case-insensitive Mac that references `Users` in some places and `users` in others will fail when deployed to a Linux server. The solution is the `lower_case_table_names` system variable:

```sql
SHOW VARIABLES LIKE 'lower_case_table_names';
```

A value of `0` means table names are stored as-is and comparisons are case-sensitive (default on Linux). A value of `1` means all table names are stored and compared in lowercase (default on Windows). A value of `2` means names are stored as given but compared in lowercase (default on macOS).

The safe practice is to always use lowercase for database and table names. This ensures consistent behavior regardless of the operating system.

### Column Names

Column names are case-insensitive in MySQL regardless of the operating system. `SELECT Email FROM users` and `SELECT email FROM users` are equivalent.

### String Comparisons

String comparisons (in `WHERE` clauses and `LIKE` patterns) depend on the collation of the column. The default collation for MySQL 8 is `utf8mb4_0900_ai_ci`, where `ci` stands for case-insensitive. This means:

```sql
SELECT * FROM users WHERE email = 'USER@EXAMPLE.COM';
-- will match rows where email is 'user@example.com'
```

If you need case-sensitive string matching, use a case-sensitive collation or the `BINARY` keyword:

```sql
SELECT * FROM users WHERE BINARY email = 'user@example.com';
```

## The MySQL Configuration File

MySQL reads its configuration from `my.cnf` (Linux/macOS) or `my.ini` (Windows). The location varies by installation but common paths include `/etc/mysql/my.cnf` and `/etc/my.cnf`. To see all active variables:

```sql
SHOW VARIABLES;
SHOW VARIABLES LIKE 'max_connections';
```

Variables can also be changed at runtime (for the current session or globally) without restarting the server:

```sql
SET SESSION max_allowed_packet = 67108864;
SET GLOBAL slow_query_log = 1;
```

## Practice Problems

### Problem 1
You have just installed MySQL and connected as root. Write the sequence of CLI commands to create a new database called `bookstore`, select it, and confirm that it is currently active.

**Schema used:**
```sql
-- No pre-existing schema needed.
```

**Your query:**
```sql
-- Write the sequence of commands here.
```

**Solution:**
```sql
CREATE DATABASE bookstore;
USE bookstore;
SELECT DATABASE();
```

**Explanation:**
`CREATE DATABASE` creates the new database container. `USE` switches the active context so that all subsequent statements operate within `bookstore`. `SELECT DATABASE()` returns the name of the currently active database, which confirms that the switch was successful. You could also verify by running `STATUS;` and looking at the "Current database" line in the output.

### Problem 2
You have a file at `/home/ubuntu/seed.sql` that creates and populates a `products` table. Write two different ways to execute this file against the `bookstore` database.

**Schema used:**
```sql
-- No pre-existing schema needed.
```

**Your query:**
```bash
-- Write both methods here.
```

**Solution:**
```bash
# Method 1: pipe the file when connecting from the shell
mysql -u root -p bookstore < /home/ubuntu/seed.sql

# Method 2: use SOURCE from within the MySQL CLI after connecting
# First connect:
mysql -u root -p

# Then inside the CLI:
# USE bookstore;
# SOURCE /home/ubuntu/seed.sql;
```

**Explanation:**
Both methods execute the same SQL. The shell pipe method is useful in scripts and automation because it does not require an interactive session. The `SOURCE` method is useful when you are already inside the MySQL CLI and want to run a file without leaving the session. Note that when using `SOURCE`, the path must be accessible from the server's file system, not the client's — this matters if your MySQL server is on a remote machine.

### Problem 3
A developer on a macOS machine created a table named `Orders` and a table named `orders` in the same database, expecting them to be two different tables. The code worked on their Mac but broke on the Linux production server. Explain why, and describe the best practice to prevent this from happening.

**Schema used:**
```sql
-- Illustrative example only:
CREATE DATABASE IF NOT EXISTS shop;
USE shop;
CREATE TABLE orders (id INT PRIMARY KEY);
-- On Linux, attempting this would cause an error because "orders" already exists.
-- On macOS with lower_case_table_names=2, both names resolve to the same table.
```

**Your query:**
```sql
SHOW VARIABLES LIKE 'lower_case_table_names';
```

**Solution:**
```sql
-- On macOS (lower_case_table_names = 2):
-- "Orders" and "orders" are treated as the same table.
-- The second CREATE TABLE would fail with "table already exists"
-- or silently overwrite, depending on IF NOT EXISTS usage.

-- On Linux (lower_case_table_names = 0):
-- "Orders" and "orders" are treated as distinct tables
-- because the file system is case-sensitive.

-- Best practice: always use lowercase snake_case for all table names.
-- Set lower_case_table_names = 1 in my.cnf on all environments
-- (this must be done before initializing the data directory on MySQL 8):
-- [mysqld]
-- lower_case_table_names = 1
```

**Explanation:**
The root cause is that macOS uses a case-insensitive file system by default, making `Orders` and `orders` point to the same underlying file. Linux uses a case-sensitive file system, so they are different files. Because MySQL stores each table as one or more files on disk, the behavior of table name resolution follows the file system. The safest resolution is to enforce a consistent naming convention (always lowercase) and, if possible, configure `lower_case_table_names = 1` consistently across all environments. Note that in MySQL 8.x, this variable must be set before the data directory is initialized — changing it afterward is not supported.
