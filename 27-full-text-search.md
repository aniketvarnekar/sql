# Full-Text Search

MySQL's full-text search allows you to search for words and phrases within text columns efficiently, beyond what `LIKE '%pattern%'` can provide. A full-text index stores a map from individual words to the rows containing them, enabling fast natural language searches on large text datasets.

## FULLTEXT Index

A `FULLTEXT` index can be created on `CHAR`, `VARCHAR`, and `TEXT` columns. It can span multiple columns, which allows a single search to look across all of them simultaneously:

```sql
-- At table creation:
CREATE TABLE articles (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    body MEDIUMTEXT NOT NULL,
    FULLTEXT INDEX ft_articles (title, body)
);

-- On an existing table:
ALTER TABLE articles ADD FULLTEXT INDEX ft_articles (title, body);

-- Or with CREATE INDEX:
CREATE FULLTEXT INDEX ft_articles ON articles (title, body);
```

Only InnoDB (MySQL 5.6+) and MyISAM tables support `FULLTEXT` indexes. InnoDB is the correct choice for modern MySQL.

## MATCH() ... AGAINST() Syntax

Full-text searches use the `MATCH() ... AGAINST()` expression. `MATCH()` specifies the columns to search, and `AGAINST()` specifies the search string and mode:

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('MySQL database');
```

The columns in `MATCH()` must exactly match the columns in the `FULLTEXT` index. You cannot use `MATCH(title)` if the index was defined on `(title, body)` — you must list both.

`MATCH() ... AGAINST()` can also appear in the `SELECT` list to return the relevance score:

```sql
SELECT
    id,
    title,
    MATCH(title, body) AGAINST ('MySQL database') AS relevance_score
FROM articles
WHERE MATCH(title, body) AGAINST ('MySQL database')
ORDER BY relevance_score DESC;
```

The relevance score is a floating-point number. Higher scores indicate more relevant matches. Returning and sorting by the relevance score is the standard pattern for full-text search result pages.

## Natural Language Mode

Natural language mode is the default. MySQL parses the search string into words, ignores common words (stopwords), and returns rows ranked by relevance:

```sql
SELECT title, MATCH(title, body) AGAINST ('relational database design') AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('relational database design')
ORDER BY score DESC;
```

In natural language mode, MySQL automatically ignores words that appear in more than 50% of rows (extremely common words) because they have no discriminating value. Words shorter than the minimum word length (`innodb_ft_min_token_size`, default 3 characters) are also ignored.

## Boolean Mode

Boolean mode gives you fine-grained control over the search using operators:

```sql
SELECT title FROM articles
WHERE MATCH(title, body) AGAINST ('+MySQL -Oracle' IN BOOLEAN MODE);
```

Boolean mode operators:
- `+word` — the word must be present in the result (required)
- `-word` — the word must not be present (excluded)
- `word` — the word increases the relevance score but is not required
- `*` — wildcard suffix: `MySQL*` matches 'MySQL', 'MySQLi', 'MySQL8', etc.
- `"exact phrase"` — the words must appear as an exact phrase
- `>word` — increases a word's contribution to the relevance score
- `<word` — decreases a word's contribution to the relevance score
- `(word group)` — groups words for sub-expression control

```sql
-- Must contain 'MySQL', must not contain 'Oracle':
AGAINST ('+MySQL -Oracle' IN BOOLEAN MODE)

-- Must contain the exact phrase 'query optimization':
AGAINST ('"query optimization"' IN BOOLEAN MODE)

-- Must contain 'database', optionally boosted by 'relational':
AGAINST ('+database relational' IN BOOLEAN MODE)

-- Words starting with 'index':
AGAINST ('index*' IN BOOLEAN MODE)
```

In boolean mode, MySQL does not sort by relevance by default (though a relevance score is still computed if you request it). You must add `ORDER BY MATCH(...) AGAINST (...)` explicitly for ranked results.

## Query Expansion Mode

Query expansion first performs a natural language search, identifies the top-ranked result rows, expands the search query with significant words from those rows, then performs a second search. This helps find related documents even when the user's search terms don't exactly appear:

```sql
SELECT title
FROM articles
WHERE MATCH(title, body) AGAINST ('database' WITH QUERY EXPANSION)
ORDER BY MATCH(title, body) AGAINST ('database' WITH QUERY EXPANSION) DESC;
```

Query expansion is useful for short or ambiguous queries but can produce surprising results because the expansion is automatic and not always intuitive.

## Minimum Word Length Setting

Words shorter than `innodb_ft_min_token_size` (default: 3) are not indexed. This means searching for a 2-character word like 'AI' or 'db' will return no results with a full-text search. Check and adjust this setting if needed:

```sql
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';
```

To change it, update `my.cnf`:

```
[mysqld]
innodb_ft_min_token_size = 2
```

After changing this setting, the server must be restarted and all FULLTEXT indexes rebuilt:

```sql
REPAIR TABLE articles QUICK;
-- or
OPTIMIZE TABLE articles;
```

## Limitations of MySQL Full-Text Search

MySQL's built-in full-text search is suitable for moderate-scale text search requirements but has several important limitations:

- Results are not as sophisticated as dedicated search engines (Elasticsearch, Apache Solr). There is no support for fuzzy matching, stemming across languages, synonym expansion, or fine-grained relevance tuning.
- The 50% threshold in natural language mode can cause common words to be silently excluded from results, confusing users.
- Full-text indexes on InnoDB tables consume significant disk space and slow down write operations.
- No support for Chinese, Japanese, or Korean (CJK) character-based tokenization by default (a special `ngram` parser is available for these).
- Cross-database or cross-table full-text searches are not possible in a single `MATCH() AGAINST()` expression.

For applications where search quality and flexibility are critical, using MySQL for structured data storage alongside Elasticsearch for text search is a common and effective architecture.

## Practice Problems

### Problem 1
Create a `FULLTEXT` index on a `blog_posts` table and write a query that finds posts matching the phrase "query optimization" using boolean mode, sorted by relevance score.

**Schema used:**
```sql
CREATE TABLE blog_posts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    body TEXT NOT NULL,
    published_at DATE NOT NULL
);

INSERT INTO blog_posts (title, body, published_at) VALUES
    ('Introduction to MySQL', 'MySQL is a relational database management system. Learning SQL query basics.', '2024-01-10'),
    ('Query Optimization Techniques', 'Query optimization is essential for database performance. Slow queries hurt applications. Query optimization with indexes.', '2024-02-15'),
    ('Database Indexing Guide', 'Indexes speed up queries by reducing full table scans. Proper query optimization requires good indexes.', '2024-03-01');
```

**Your query:**
```sql
-- Create the FULLTEXT index and write the search query.
```

**Solution:**
```sql
ALTER TABLE blog_posts ADD FULLTEXT INDEX ft_posts (title, body);

SELECT
    id,
    title,
    published_at,
    ROUND(MATCH(title, body) AGAINST ('"query optimization"' IN BOOLEAN MODE), 4) AS relevance
FROM blog_posts
WHERE MATCH(title, body) AGAINST ('"query optimization"' IN BOOLEAN MODE)
ORDER BY relevance DESC;
```

**Explanation:**
The `ALTER TABLE` adds a `FULLTEXT` index after the table is created. The search uses boolean mode with `"query optimization"` in double quotes, which requires the two words to appear as an exact phrase in that order. The post titled 'Query Optimization Techniques' matches with the highest relevance because both the title and body contain the phrase multiple times. The third post ('Database Indexing Guide') also contains the phrase 'query optimization' in the body, so it also appears in results. The `ROUND(..., 4)` formats the relevance score to 4 decimal places for readability.

### Problem 2
Write a boolean mode search that finds posts that must contain the word "database", must not contain the word "Oracle", and optionally boost results that also contain "performance".

**Schema used:**
```sql
-- Use the blog_posts table from Problem 1 with the FULLTEXT index already created.
CREATE TABLE blog_posts (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(300) NOT NULL,
    body TEXT NOT NULL
);

ALTER TABLE blog_posts ADD FULLTEXT INDEX ft_posts (title, body);

INSERT INTO blog_posts (title, body) VALUES
    ('MySQL Performance', 'Improving database performance with proper indexing.'),
    ('Oracle vs MySQL', 'Comparing Oracle database with MySQL database systems.'),
    ('PostgreSQL Basics', 'PostgreSQL is an open-source database with advanced features.');
```

**Your query:**
```sql
-- Write the boolean mode query.
```

**Solution:**
```sql
SELECT
    id,
    title,
    MATCH(title, body) AGAINST ('+database -Oracle >performance' IN BOOLEAN MODE) AS score
FROM blog_posts
WHERE MATCH(title, body) AGAINST ('+database -Oracle >performance' IN BOOLEAN MODE)
ORDER BY score DESC;
```

**Explanation:**
`+database` requires the word "database" — posts without it are excluded. `-Oracle` excludes any post containing "Oracle" — the 'Oracle vs MySQL' row is filtered out even though it contains "database". `>performance` is optional but boosts the relevance score for posts that also contain "performance", so the MySQL Performance post ranks higher than the PostgreSQL post. The PostgreSQL post appears because it contains "database" and not "Oracle", satisfying the required conditions even though it doesn't contain "performance".

### Problem 3
Demonstrate the minimum word length limitation and show how to work around it when searching for short terms.

**Schema used:**
```sql
CREATE TABLE tech_docs (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    content TEXT NOT NULL
);

ALTER TABLE tech_docs ADD FULLTEXT INDEX ft_docs (content);

INSERT INTO tech_docs (content) VALUES
    ('AI and ML are transforming how we use databases'),
    ('SQL is the standard language for relational databases'),
    ('API design is crucial for modern applications');
```

**Your query:**
```sql
-- Demonstrate the short-word limitation and the LIKE workaround.
```

**Solution:**
```sql
-- Check minimum token size:
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';
-- Returns: 3 (default)

-- This returns NO results because 'AI' (2 chars) is below the minimum:
SELECT id FROM tech_docs
WHERE MATCH(content) AGAINST ('AI' IN BOOLEAN MODE);
-- Empty result set

-- This also returns NO results because 'ML' is 2 chars:
SELECT id FROM tech_docs
WHERE MATCH(content) AGAINST ('ML' IN BOOLEAN MODE);

-- Workaround 1: Use LIKE for short terms (slower but correct):
SELECT id, content FROM tech_docs WHERE content LIKE '%AI%';

-- Workaround 2: Combine FULLTEXT for the main filter with LIKE for the short word:
SELECT id, content FROM tech_docs
WHERE MATCH(content) AGAINST ('databases' IN BOOLEAN MODE)
  AND content LIKE '%AI%';

-- Workaround 3: Rebuild the index with a lower min token size
-- (requires my.cnf change + server restart + OPTIMIZE TABLE)
-- [mysqld]
-- innodb_ft_min_token_size = 2
-- Then: OPTIMIZE TABLE tech_docs;
```

**Explanation:**
The default `innodb_ft_min_token_size = 3` means 1-2 character words are never indexed and never matched by full-text search. For technical content where short abbreviations like "AI", "ML", "DB", or "ID" are significant search terms, this is a real limitation. The immediate workarounds are to use `LIKE '%AI%'` (which scans the full column but works correctly) or to combine a FULLTEXT search for longer terms with `LIKE` for short ones. The permanent fix is reducing `innodb_ft_min_token_size` in the server configuration, rebuilding the index, which requires a planned server restart.
