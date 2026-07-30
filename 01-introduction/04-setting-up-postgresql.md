# Setting Up PostgreSQL

## Learning Objectives

By the end of this section you should be able to:
- Install PostgreSQL on your operating system
- Connect to it using `psql`, the command-line client
- Understand what a "server," a "connection," and a "database" mean in practical terms
- Optionally set up a GUI client for a more visual workflow

## Prerequisites

- [Categories of SQL Commands](03-categories-of-sql-commands.md) — conceptual background; not technically required for installation, but you should know why you're doing this.

## Motivation

Every topic from here on assumes you can actually run SQL somewhere and see real results. Reading about `SELECT` without ever running it is like reading about swimming without entering water. This topic is entirely practical: get a working PostgreSQL installation and a way to talk to it.

## Problem Statement

SQL doesn't run in a vacuum — it's sent to a running database server process, which must be installed, started, and listening for connections before any query can be executed. You need: (1) the PostgreSQL server software installed and running, and (2) a client — something that lets you type SQL and see the results — connected to it.

## Concept

### The Client-Server Model

PostgreSQL (like virtually all RDBMSs) runs as a **server process** — a program running continuously in the background, listening for connections. You interact with it through a **client** — a separate program that connects to the server, sends it SQL text, and displays the results it sends back.

```
  Client (psql, or a GUI tool)  ──[connection]──▶  PostgreSQL server process ──▶ data files
        (you type SQL here)                          (parses, plans, executes)
```

This is true whether the server is running on your own laptop (as it will be for this course) or on a machine on the other side of the world — the client/server relationship is identical either way; only the network path differs.

### Installing PostgreSQL

Pick the path matching your operating system.

**macOS**
- Recommended: install via [Homebrew](https://brew.sh/) (a package manager for macOS):
  ```
  brew install postgresql@16
  brew services start postgresql@16
  ```
- Alternative: download **Postgres.app**, a self-contained macOS application that bundles the server and starts it via a menu-bar icon — no command-line package manager needed.

**Windows**
- Download the installer from the official PostgreSQL website's Windows distribution (provided by EnterpriseDB). It installs the server as a Windows service (so it starts automatically) and includes `psql` and a GUI tool called pgAdmin.

**Linux**
- Use your distribution's package manager, for example on Debian/Ubuntu-based systems:
  ```
  sudo apt update
  sudo apt install postgresql
  ```
  Most Linux package managers automatically start PostgreSQL as a background service after installation.

### Verifying the Installation

Once installed, confirm the server is running and connect to it with `psql`, PostgreSQL's official command-line client:

```
psql --version
psql -U postgres
```

- `psql -U postgres` connects as the default administrative user named `postgres`. Depending on your OS and install method, you may be prompted for a password, or you may need to run this as your system's own user account first (installers typically print exact instructions at the end of setup — follow those if this default command doesn't work).
- A successful connection drops you into an interactive prompt that looks like:
  ```
  postgres=#
  ```
  You can now type SQL directly. Try:
  ```sql
  SELECT version();
  ```
  This should print the installed PostgreSQL version — confirming everything is wired up correctly.

### Creating a Database to Practice In

A single PostgreSQL server can host many separate databases. Create one dedicated to this course:

```sql
CREATE DATABASE sql_course;
```

Then connect to it specifically:

```
\c sql_course
```

(`\c` is a `psql`-specific "meta-command" — not SQL itself — for switching the active database within the same session. Meta-commands always start with a backslash and are a `psql` convenience feature, not part of the SQL language.)

### Useful `psql` Meta-Commands

| Command | Purpose |
|---|---|
| `\l` | List all databases on the server |
| `\c dbname` | Connect to a specific database |
| `\dt` | List tables in the current database |
| `\d tablename` | Describe a table's columns and types |
| `\q` | Quit `psql` |

### Optional: A GUI Client

Typing SQL directly is the most transparent way to learn (and what this course assumes), but a graphical client can help you *visualize* tables and results, especially early on. Two common options:
- **pgAdmin** — a full-featured, PostgreSQL-specific GUI (often bundled with the Windows installer).
- **TablePlus / DBeaver** — general-purpose database GUI clients that support PostgreSQL and many other databases.

These are optional. Every example in this course is shown as raw SQL you can run in `psql` directly, and that's the recommended way to follow along — a GUI can obscure exactly what's being sent to the server, which matters while you're still building your mental model.

## Internal Working (Preview)

When `psql` connects, here's roughly what happens:
1. `psql` opens a network connection to the PostgreSQL server process (by default on port `5432`).
2. The server authenticates the connection (checking username/password against its configuration).
3. Once authenticated, the server allocates a **backend process** dedicated to your session — every connected client gets its own server-side process handling its queries.
4. From then on, every SQL statement you type is sent over that connection, parsed, planned, executed, and the result sent back — exactly the pipeline previewed in Topic 2.

## Real-World Analogy

Installing PostgreSQL and connecting via `psql` is like installing a phone line and picking up the receiver: installation sets up the "phone company" (the server) so it's ready to take calls; running `psql -U postgres` is you dialing in and getting connected; the `postgres=#` prompt is the other end picking up, ready to listen to what you say next.

## Advantages of Using a Local Install for Learning

- **No cost, no network dependency** — everything runs on your own machine.
- **Safe to experiment** — you can create, break, and drop databases freely without affecting anything real.
- **Identical SQL behavior to production PostgreSQL** — what you learn here transfers directly; PostgreSQL doesn't behave differently based on where it's hosted.

## Limitations

- A local install doesn't teach you about network configuration, remote authentication, or hosting concerns real production deployments involve — those are operational topics outside the scope of this SQL-focused course.
- Some hosted/managed PostgreSQL services (e.g., cloud database providers) restrict certain administrative commands (like superuser-only operations) that a local install allows freely — a minor difference to be aware of if you later work against a managed cloud database.

## Best Practices

- Create a dedicated database for this course (as shown above) rather than experimenting inside the default `postgres` database — it keeps your practice work cleanly separated and easy to wipe and restart if needed (`DROP DATABASE sql_course;` followed by re-creating it).
- Get comfortable with `psql` before reaching for a GUI — seeing the raw prompt and typing raw SQL builds a much more accurate mental model than clicking through a visual tool, especially for a beginner.

## Common Mistakes

| Mistake | Why it's wrong |
|---|---|
| Assuming "installing PostgreSQL" and "opening psql" are the same step. | Installing sets up the server *software*; `psql` is a separate client program that must then *connect* to that already-running server. They're two distinct pieces. |
| Typing SQL into a plain terminal without first opening `psql` (or another client) and connecting to a database. | The terminal itself doesn't understand SQL — it's just your operating system's shell. You must be inside a connected client session (like the `postgres=#` or `sql_course=#` prompt) for SQL to mean anything. |
| Forgetting the semicolon at the end of a SQL statement in `psql`. | `psql` waits for a semicolon (`;`) to know a statement is complete before executing it — without it, `psql` will just show a continuation prompt (`-#`) waiting for more input. |

## Interview Questions

1. **Q: What is the relationship between a database server, a client, and a connection?**
   A: The server is the continuously running process that manages databases and executes SQL. A client (like `psql`) is a separate program that opens a connection to the server, sends it SQL, and displays returned results. The connection is the communication channel between them.

2. **Q: Why does `psql` wait after you type a SQL statement without a trailing semicolon?**
   A: `psql` treats the semicolon as the statement terminator. Without it, `psql` assumes you're still composing the statement (which may legitimately span multiple lines) and waits for more input, shown by a continuation prompt.

3. **Q: Can one PostgreSQL server host multiple independent databases?**
   A: Yes — a single running PostgreSQL server process can manage many separate databases simultaneously (e.g., `postgres`, `sql_course`, and others), each with its own tables and data, isolated from one another, switched between using a client command like `psql`'s `\c`.

## Summary

- PostgreSQL follows a **client-server model**: a server process manages the data; a client (like `psql`) connects to it and sends SQL.
- Installation differs by OS (Homebrew/Postgres.app on macOS, the official installer on Windows, your package manager on Linux) but the end result is the same: a running server plus the `psql` client.
- `psql -U postgres` connects you to the server; `CREATE DATABASE sql_course;` and `\c sql_course` give you a dedicated space to practice in.
- With a working connection, you're ready for Topic 5 — writing and understanding your very first real query.
