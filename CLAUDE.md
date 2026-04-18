# CLAUDE.md — MySQL Notes Project

## Project Structure

All note files live directly inside `mysql-notes/` with no subfolders. The directory contains:

- `README.md` — project overview and table of contents
- `CLAUDE.md` — this file; instructions for future Claude Code sessions
- `01-introduction-to-databases.md` through `30-json-in-mysql.md` — topic files, numbered sequentially

There are no nested directories. Every `.md` file except `README.md` and `CLAUDE.md` is a numbered topic file.

## Adding a New Topic File

When adding a new topic file:

1. Place the file directly in the root of `mysql-notes/` — no subfolders.
2. Name it with the next sequential number, zero-padded to two digits, followed by a hyphen and a lowercase-kebab-case topic name, with the `.md` extension. For example: `31-new-topic.md`.
3. Update `README.md` by adding a new entry to the Table of Contents section, following the same link format as the existing entries: `- [NN - Topic Name](nn-topic-name.md)`.
4. Write the file content following all formatting conventions described below.

## Updating the README Table of Contents

The Table of Contents in `README.md` is a flat list under the heading `## Table of Contents`. Each line has the format:

```
- [NN - Topic Title](nn-topic-filename.md)
```

When adding a new file, append the new entry at the end of the list. Do not reorder existing entries.

## MySQL Version

All notes use MySQL 8.x syntax. When writing new content, do not use syntax or features exclusive to MySQL 5.x or earlier unless explicitly comparing the two. When MySQL 8.x introduced or changed a feature, note it explicitly.

## Formatting Conventions

Every topic file must follow these conventions without exception:

1. The file begins with a top-level heading (`#`) that matches the topic name exactly.
2. Immediately after the heading, include a single paragraph summarizing what the topic covers and why it matters. No subheadings before this paragraph.
3. Use `##` for major sections and `###` for sub-sections within a topic.
4. All SQL code must be in fenced code blocks tagged with `sql`.
5. All shell or terminal commands must be in fenced code blocks tagged with `bash`.
6. Conceptual explanations are written in plain prose — complete sentences and paragraphs. Do not use bullet points for conceptual content.
7. Bullet points are acceptable only for lists of options, parameters, syntax variants, or quick-reference items.
8. No emojis anywhere in any file.
9. Every file ends with a `## Practice Problems` section containing at least three problems.

## Practice Problems Format

Each problem in the Practice Problems section must follow this exact structure:

```
### Problem N
[Problem statement in plain English]

**Schema used:**
```sql
-- setup SQL
```

**Your query:**
```sql
-- space for reader
```

**Solution:**
```sql
-- full working solution
```

**Explanation:**
[One paragraph explaining the solution and what to watch out for]
```

Do not deviate from this structure. Every problem needs all five parts: statement, schema, query space, solution, and explanation.
