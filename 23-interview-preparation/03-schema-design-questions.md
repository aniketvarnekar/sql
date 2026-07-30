# Schema Design Questions

A schema-design interview prompt is deliberately open-ended — the interviewer wants to watch your process, not just receive a finished answer. For each prompt below, the worked solution follows the same process every time: identify the entities, work out the relationships and their cardinality, and only then write `CREATE TABLE` statements — leaning on **Module 15 (Normalization & Design)** to decide how data should be split across tables, and **Module 05 (Constraints & Keys)** to decide exactly which keys and constraints enforce the rules those tables are supposed to guarantee.

## 1. Design a Schema for a Simple E-Commerce System

**Requirements:** customers can browse products organized into categories, place orders containing multiple products, and each order should record the price at the time of purchase.

**Entities:** `customers`, `categories`, `products`, `orders`, `order_items`.

**ER-style breakdown:**
- A `category` has many `products`; a `product` belongs to exactly one `category` (one-to-many).
- A `customer` places many `orders`; an `order` belongs to exactly one `customer` (one-to-many).
- An `order` and a `product` are many-to-many — one order can contain many products, and one product can appear on many orders — implemented through the `order_items` junction table (Module 15's standard resolution for a many-to-many relationship).

```sql
CREATE TABLE categories (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE customers (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
);

CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    category_id INTEGER NOT NULL REFERENCES categories(id),
    price       NUMERIC NOT NULL CHECK (price >= 0)
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    order_date  TIMESTAMP NOT NULL DEFAULT now(),
    status      TEXT NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled'))
);

CREATE TABLE order_items (
    order_id   INTEGER NOT NULL REFERENCES orders(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity   INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC NOT NULL CHECK (unit_price >= 0),
    PRIMARY KEY (order_id, product_id)
);
```

**Design reasoning:** `unit_price` is deliberately copied onto `order_items` rather than looked up from `products.price` at read time — a product's price will change over time, and a historical order must keep showing what the customer actually paid (a controlled, intentional denormalization, per Module 15's discussion of denormalization trade-offs). `order_items` uses a composite primary key `(order_id, product_id)` rather than a surrogate key (Module 05, Natural vs. Surrogate Keys) because the pair is naturally unique — a given product shouldn't appear twice as separate line items on the same order — and the composite key enforces that directly.

## 2. Design a Schema for a Library Book-Lending System

**Requirements:** the library catalogs books (which may have multiple authors), members can borrow books, and the system must track when a book is due back and whether it's been returned.

**Entities:** `authors`, `books`, `book_authors`, `members`, `loans`.

**ER-style breakdown:**
- A `book` can have multiple `authors`, and an `author` can write multiple `books` — many-to-many, resolved with the `book_authors` junction table.
- A `member` can have many `loans`; a `loan` belongs to exactly one `member` (one-to-many).
- A `book` can be loaned out many times over its lifetime, but the schema must prevent the same physical book being recorded as loaned out to two people at once — enforced at the application/query level by only ever loaning books with no open (`returned_date IS NULL`) loan.

```sql
CREATE TABLE authors (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE books (
    id     SERIAL PRIMARY KEY,
    title  TEXT NOT NULL,
    isbn   TEXT NOT NULL UNIQUE
);

CREATE TABLE book_authors (
    book_id   INTEGER NOT NULL REFERENCES books(id),
    author_id INTEGER NOT NULL REFERENCES authors(id),
    PRIMARY KEY (book_id, author_id)
);

CREATE TABLE members (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
);

CREATE TABLE loans (
    id            SERIAL PRIMARY KEY,
    book_id       INTEGER NOT NULL REFERENCES books(id),
    member_id     INTEGER NOT NULL REFERENCES members(id),
    loan_date     DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date      DATE NOT NULL,
    returned_date DATE,
    CHECK (due_date > loan_date),
    CHECK (returned_date IS NULL OR returned_date >= loan_date)
);
```

**Design reasoning:** `isbn` gets its own `UNIQUE` constraint (Module 05, UNIQUE Constraints) separate from the surrogate `id` primary key — it's a real-world natural identifier worth enforcing uniqueness on even though it isn't the table's primary key. The two `CHECK` constraints on `loans` (Module 05, CHECK Constraints) encode business rules a data type alone can't express: a due date must be after the loan date, and a return date (once known) can't logically precede the loan date. Splitting `book_authors` out as its own table rather than a repeating `author_name` column on `books` is a direct application of First Normal Form (Module 15) — a single column must hold a single, atomic value, not a list of co-authors.

## 3. Design a Schema for a Ride-Booking System

**Requirements:** riders request rides, drivers (each with one vehicle) fulfill them, and every completed ride has exactly one payment.

**Entities:** `riders`, `drivers`, `vehicles`, `rides`, `payments`.

**ER-style breakdown:**
- A `driver` owns exactly one `vehicle` in this simplified model (one-to-one).
- A `rider` requests many `rides`; a `ride` belongs to exactly one `rider` (one-to-many).
- A `driver` fulfills many `rides` over time; a `ride` is fulfilled by at most one `driver` (one-to-many, nullable until a driver accepts).
- A `ride` has exactly one `payment` (one-to-one, once the ride completes).

```sql
CREATE TABLE riders (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    phone TEXT NOT NULL UNIQUE
);

CREATE TABLE drivers (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    phone TEXT NOT NULL UNIQUE
);

CREATE TABLE vehicles (
    id         SERIAL PRIMARY KEY,
    driver_id  INTEGER NOT NULL UNIQUE REFERENCES drivers(id),
    plate      TEXT NOT NULL UNIQUE,
    model      TEXT NOT NULL
);

CREATE TABLE rides (
    id           SERIAL PRIMARY KEY,
    rider_id     INTEGER NOT NULL REFERENCES riders(id),
    driver_id    INTEGER REFERENCES drivers(id),
    status       TEXT NOT NULL DEFAULT 'requested'
                 CHECK (status IN ('requested', 'accepted', 'in_progress', 'completed', 'cancelled')),
    requested_at TIMESTAMP NOT NULL DEFAULT now(),
    completed_at TIMESTAMP,
    fare         NUMERIC CHECK (fare >= 0)
);

CREATE TABLE payments (
    id      SERIAL PRIMARY KEY,
    ride_id INTEGER NOT NULL UNIQUE REFERENCES rides(id),
    amount  NUMERIC NOT NULL CHECK (amount >= 0),
    method  TEXT NOT NULL CHECK (method IN ('card', 'cash', 'wallet')),
    paid_at TIMESTAMP NOT NULL DEFAULT now()
);
```

**Design reasoning:** the one-to-one relationships (`drivers` to `vehicles`, `rides` to `payments`) are each enforced with a `UNIQUE` constraint on the foreign key column itself (Module 05, UNIQUE Constraints) — an ordinary foreign key alone only enforces many-to-one, so `UNIQUE` is what narrows it down to exactly one. `driver_id` on `rides` is nullable, deliberately, because a ride can exist in the `requested` state before any driver has accepted it — a foreign key column is never required to be `NOT NULL` unless the relationship really is mandatory from the moment the row is created.

## 4. Design a Schema for a Blogging Platform with Tags

**Requirements:** users write posts, other users can comment on posts, and each post can carry multiple tags (and each tag can apply to multiple posts).

**Entities:** `users`, `posts`, `comments`, `tags`, `post_tags`.

**ER-style breakdown:**
- A `user` writes many `posts`; a `post` has exactly one author (one-to-many).
- A `post` has many `comments`; a `comment` belongs to exactly one `post` and one commenting `user` (two separate one-to-many relationships into `comments`).
- A `post` and a `tag` are many-to-many, resolved with the `post_tags` junction table.

```sql
CREATE TABLE users (
    id       SERIAL PRIMARY KEY,
    username TEXT NOT NULL UNIQUE,
    email    TEXT NOT NULL UNIQUE
);

CREATE TABLE posts (
    id         SERIAL PRIMARY KEY,
    author_id  INTEGER NOT NULL REFERENCES users(id),
    title      TEXT NOT NULL,
    body       TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE comments (
    id         SERIAL PRIMARY KEY,
    post_id    INTEGER NOT NULL REFERENCES posts(id),
    author_id  INTEGER NOT NULL REFERENCES users(id),
    body       TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE tags (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE post_tags (
    post_id INTEGER NOT NULL REFERENCES posts(id),
    tag_id  INTEGER NOT NULL REFERENCES tags(id),
    PRIMARY KEY (post_id, tag_id)
);
```

**Design reasoning:** `tags.name` carries its own `UNIQUE` constraint (Module 05) so the same tag text is never duplicated as two different rows — without it, "sql" and a second, accidental "sql" row would silently split what should be one tag's posts across two tag IDs. `post_tags`' composite primary key both identifies each row and, just as importantly, prevents the same tag from being attached to the same post twice — a direct enforcement of a business rule through a key choice rather than application-level checking (Module 05's general argument for pushing invariants into the schema itself).

## 5. Design a Schema for a Simple Banking Ledger

**Requirements:** customers hold one or more accounts, and every deposit, withdrawal, or transfer must be recorded as an immutable transaction against an account, with the running balance always derivable from the transaction history.

**Entities:** `customers`, `accounts`, `transactions`.

**ER-style breakdown:**
- A `customer` holds many `accounts`; an `account` belongs to exactly one `customer` (one-to-many).
- An `account` has many `transactions`; a `transaction` belongs to exactly one `account` (one-to-many).

```sql
CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE accounts (
    id          SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    account_no  TEXT NOT NULL UNIQUE,
    opened_at   TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE transactions (
    id          SERIAL PRIMARY KEY,
    account_id  INTEGER NOT NULL REFERENCES accounts(id),
    amount      NUMERIC NOT NULL CHECK (amount <> 0),
    type        TEXT NOT NULL CHECK (type IN ('deposit', 'withdrawal', 'transfer_in', 'transfer_out')),
    occurred_at TIMESTAMP NOT NULL DEFAULT now()
);
```

**Design reasoning:** this schema deliberately has **no** `balance` column anywhere — the current balance is always `SUM(amount)` over an account's `transactions`, computed on demand, rather than a stored value that could drift out of sync with its own history (an application of Module 15's normalization reasoning: a derivable value is redundant data, and redundant data is exactly what causes update anomalies if the two copies ever disagree). `amount` uses a signed `NUMERIC` (positive for money in, negative for money out) with a `CHECK (amount <> 0)` (Module 05) to reject meaningless zero-amount rows. Because every balance-affecting event becomes its own immutable, appended `transactions` row rather than an in-place update to a stored balance, this design also directly supports the atomicity and durability guarantees discussed under ACID (Module 14, Transactions & Concurrency) — a transfer between two accounts is naturally expressed as two `INSERT`s (a `transfer_out` on one account and a `transfer_in` on the other) wrapped in a single database transaction, succeeding or failing together.
