# Module 15 Summary

## Topic Coverage Verification

Every topic promised in the [Module Overview](00-module-overview.md) has been covered:

- [x] **Functional Dependencies** — the formal `X → Y` definition, trivial vs. non-trivial dependencies, full vs. partial dependencies, and transitive dependencies, all worked out from a realistic messy enrollment export
- [x] **First Normal Form (1NF)** — atomic values, repeating groups (both packed columns and numbered sibling columns), and a concrete before/after transformation into one-row-per-fact
- [x] **Second and Third Normal Form (2NF, 3NF)** — partial dependencies removed by 2NF (relevant only with a composite key) and transitive dependencies removed by 3NF, worked through a full three-table decomposition of the running enrollment example
- [x] **Boyce-Codd Normal Form (BCNF)** — the overlapping-candidate-key edge case 3NF's wording permits through, why BCNF is a strictly stricter version of 3NF, and a full worked decomposition of a 3NF-but-not-BCNF table
- [x] **Entity-Relationship (ER) Modeling** — entities, attributes, relationships, the three cardinalities, why many-to-many requires a junction table, a full text-form ER diagram, and its direct translation into `CREATE TABLE` statements
- [x] **Denormalization Trade-offs** — deliberately reintroducing redundancy for read performance, when it's justified versus when it isn't, and the concrete update-anomaly risk a denormalized column carries

## Practical Connections

- A production **reporting or analytics dashboard** querying millions of rows relies on exactly the judgment call this module teaches: whether the underlying schema should stay fully normalized (favoring write correctness) or accept a small, deliberately-maintained amount of denormalized redundancy (favoring read speed) for its specific, heaviest queries.
- A **schema migration** that adds a new feature to an existing application (a new "wishlist" feature added to an e-commerce database, for instance) is, in practice, an exercise in identifying new entities and relationships and correctly assessing their cardinality — getting a many-to-many relationship wrong at this stage is far more expensive to fix once real customer data already lives in a wrongly-shaped table.
- A **data import or migration from a legacy, spreadsheet-like source** (a common real-world task) almost always starts from a flattened, 1NF-violating, redundant table exactly like this module's running example — the functional-dependency reasoning and decomposition steps from Topics 1–4 are the direct, practical technique for turning that import into a correct relational schema rather than copying its flaws forward.
- An **interview schema-design question** ("design a database for a library," "design a database for a ride-sharing app") is, almost without exception, evaluated on whether the candidate correctly identifies entities and cardinalities (Topic 5) and avoids the redundancy and anomalies this module's normal forms are built to catch.

## Frequently Confused Concepts (Recap)

| Confused pair | The distinction |
|---|---|
| Partial dependency vs. transitive dependency | A partial dependency is about a *composite key*, where some non-key attribute depends on only part of the key. A transitive dependency is about a chain through another *non-key attribute*, and can occur even with a single-column key. 2NF eliminates the former; 3NF eliminates the latter. |
| 3NF vs. BCNF | 3NF permits a non-superkey determinant as long as its dependent value is a prime attribute (part of some candidate key). BCNF removes that exception, requiring every determinant to be a superkey with no exceptions — making BCNF a strictly stricter version of 3NF, not an unrelated rule. |
| Denormalization vs. never having normalized | Denormalization is a deliberate, targeted, justified, and actively-maintained exception layered on top of an already-correct normalized schema. Never normalizing is broad, unplanned redundancy with no justification and no consistency mechanism. |
| One-to-many vs. many-to-many relationships | One-to-many is implemented with a single foreign key column on the "many" side. Many-to-many cannot be represented by a single foreign key on either side and requires a separate junction table holding a foreign key to each side. |
| A functional dependency vs. a database constraint | A functional dependency is a design-time fact you reason about on paper (or a whiteboard). A constraint (`PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`) is what the database engine actually enforces at run time — normalization is the process of translating the former into the latter. |

## What's Next

This module gave you the formal theory (functional dependencies, the normal forms through BCNF) and the practical design process (ER modeling) to take any real-world requirement — whether a fresh conversation with a stakeholder or an existing, messy spreadsheet — and produce a schema that stores every fact exactly once, joined back together correctly and efficiently on demand. It also gave you the judgment to recognize the narrow, deliberate cases where breaking that rule is the right call. **Module 16 — Window Functions** builds on the correctly normalized, multi-table schemas this module produces: window functions (`OVER`, `PARTITION BY`, `ROW_NUMBER`, `RANK`, `LAG`/`LEAD`, running totals) are exactly the kind of rich analytical query that a well-normalized schema, joined together first, is built to support.
