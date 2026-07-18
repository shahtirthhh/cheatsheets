# PostgreSQL — Zero to Hero
*For MongoDB users who've never written SQL*

---

# Part 1: What Is SQL and Why Learn It?

## MongoDB vs PostgreSQL — The Mental Shift

```
MongoDB thinks in:              PostgreSQL thinks in:
  Documents (JSON blobs)          Tables (spreadsheets)
  Collections                     Tables
  Documents                       Rows
  Fields                          Columns
  Embedded documents              JOINs between tables
  Flexible schema                 Fixed schema (enforced)
  _id                             Primary key (usually id)

MongoDB:                         PostgreSQL:
{                                users table:
  _id: ObjectId("..."),          | id | name  | email        | age |
  name: "Alice",                 |----|-------|--------------|-----|
  email: "alice@test.com",       | 1  | Alice | alice@t.com  | 30  |
  age: 30,                       | 2  | Bob   | bob@t.com    | 25  |
  addresses: [                   
    { city: "NYC", zip: "10001" } addresses table:
  ]                               | id | user_id | city | zip   |
}                                 |----|---------|------|-------|
                                  | 1  | 1       | NYC  | 10001 |
MongoDB embeds addresses          PostgreSQL uses a SEPARATE table
inside the user document.         linked by user_id (foreign key).
```

**Why PostgreSQL when you know MongoDB?**
- Most companies use SQL databases (90%+ of enterprise data is in SQL)
- SQL is a universal skill (PostgreSQL, MySQL, SQLite — same language)
- JOINs, transactions, and complex queries are PostgreSQL's strength
- pgvector makes PostgreSQL a vector database too (RAG without Pinecone)

---

# Part 2: Creating Tables (Schema)

## Data Types

```sql
-- Numbers
INTEGER          -- whole numbers (-2B to 2B)
BIGINT           -- large integers
SERIAL           -- auto-incrementing integer (like MongoDB's auto _id)
BIGSERIAL        -- auto-incrementing big integer
NUMERIC(10, 2)   -- exact decimal (10 digits total, 2 after decimal) — use for money
REAL             -- floating point (inexact)
DOUBLE PRECISION -- 8-byte floating point

-- Text
VARCHAR(255)     -- variable-length string up to 255 chars
TEXT             -- unlimited length string (like MongoDB's String)
CHAR(10)         -- fixed-length string (padded with spaces)

-- Boolean
BOOLEAN          -- true/false

-- Date/Time
DATE             -- date only (2024-01-15)
TIME             -- time only (14:30:00)
TIMESTAMP        -- date + time (2024-01-15 14:30:00)
TIMESTAMPTZ      -- timestamp with timezone (ALWAYS use this)
INTERVAL         -- duration ('1 day', '2 hours')

-- JSON (MongoDB-like flexibility in PostgreSQL!)
JSON             -- stored as text, validated on input
JSONB            -- binary JSON, indexable, faster queries (prefer this)

-- Arrays (PostgreSQL supports arrays natively!)
INTEGER[]        -- array of integers
TEXT[]           -- array of strings

-- Special
UUID             -- universally unique identifier
BYTEA            -- binary data (like MongoDB's BinData)
VECTOR(1536)     -- pgvector extension for AI embeddings
```

## CREATE TABLE

```sql
-- Users table
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,          -- auto-increment, unique, not null
    name        VARCHAR(100) NOT NULL,       -- required
    email       VARCHAR(255) UNIQUE NOT NULL, -- required, must be unique
    age         INTEGER CHECK (age >= 0 AND age <= 150),  -- validation
    role        VARCHAR(20) DEFAULT 'user',  -- default value
    is_active   BOOLEAN DEFAULT true,
    bio         TEXT,                         -- optional (nullable by default)
    tags        TEXT[] DEFAULT '{}',          -- array of strings
    metadata    JSONB DEFAULT '{}',           -- flexible JSON data
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Orders table (related to users)
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    --          ↑ FOREIGN KEY: must match an existing users.id
    --          ON DELETE CASCADE: if user is deleted, delete their orders too
    total       NUMERIC(10, 2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled')),
    items       JSONB NOT NULL,              -- [{product_id, quantity, price}]
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Many-to-many: users can have many roles, roles can have many users
CREATE TABLE roles (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE user_roles (
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    role_id INTEGER REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)  -- composite primary key
);
```

```
MongoDB equivalent of the users table:
  db.createCollection("users", {
    validator: { $jsonSchema: {
      required: ["name", "email"],
      properties: {
        name: { bsonType: "string", maxLength: 100 },
        email: { bsonType: "string" },
        age: { bsonType: "int", minimum: 0, maximum: 150 },
      }
    }}
  });

The BIG difference: PostgreSQL enforces schema at the DATABASE level.
MongoDB enforces it optionally. In PostgreSQL, you CANNOT insert a row
that violates the schema. Period.
```

---

# Part 3: CRUD Operations

## INSERT (MongoDB: insertOne / insertMany)

```sql
-- Single insert
INSERT INTO users (name, email, age)
VALUES ('Alice', 'alice@test.com', 30);

-- Multiple insert
INSERT INTO users (name, email, age) VALUES
    ('Bob', 'bob@test.com', 25),
    ('Charlie', 'charlie@test.com', 35),
    ('Diana', 'diana@test.com', 28);

-- Insert and return the created row (like MongoDB returning the _id)
INSERT INTO users (name, email, age)
VALUES ('Eve', 'eve@test.com', 22)
RETURNING id, name, email, created_at;
-- Returns: | 5 | Eve | eve@test.com | 2024-01-15 ... |
```

## SELECT (MongoDB: find)

```sql
-- Select all columns
SELECT * FROM users;
-- MongoDB: db.users.find({})

-- Select specific columns
SELECT name, email FROM users;
-- MongoDB: db.users.find({}, { name: 1, email: 1 })

-- With condition
SELECT * FROM users WHERE age > 25;
-- MongoDB: db.users.find({ age: { $gt: 25 } })

-- Multiple conditions
SELECT * FROM users WHERE age > 25 AND is_active = true;
SELECT * FROM users WHERE age < 20 OR age > 60;
SELECT * FROM users WHERE age BETWEEN 25 AND 35;
SELECT * FROM users WHERE name IN ('Alice', 'Bob', 'Charlie');
SELECT * FROM users WHERE email LIKE '%@gmail.com';     -- ends with @gmail.com
SELECT * FROM users WHERE name ILIKE '%alice%';          -- case-insensitive LIKE
SELECT * FROM users WHERE bio IS NULL;                   -- check for null
SELECT * FROM users WHERE bio IS NOT NULL;

-- Sorting
SELECT * FROM users ORDER BY age ASC;            -- ascending (default)
SELECT * FROM users ORDER BY created_at DESC;    -- descending (newest first)
SELECT * FROM users ORDER BY age DESC, name ASC; -- multi-column sort

-- Pagination
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;
-- Skip 20, take 10 (page 3 if page size = 10)
-- MongoDB: db.users.find().skip(20).limit(10)

-- Distinct values
SELECT DISTINCT role FROM users;
-- MongoDB: db.users.distinct("role")
```

## UPDATE (MongoDB: updateOne / updateMany)

```sql
-- Update one (by primary key)
UPDATE users SET name = 'Alice Smith', age = 31 WHERE id = 1;
-- MongoDB: db.users.updateOne({ _id: 1 }, { $set: { name: "Alice Smith", age: 31 } })

-- Update many
UPDATE users SET is_active = false WHERE age < 18;

-- Update with RETURNING
UPDATE users SET age = age + 1 WHERE id = 1 RETURNING *;

-- Increment (no $inc needed — just use arithmetic)
UPDATE users SET age = age + 1 WHERE id = 1;

-- Update with JSONB
UPDATE users SET metadata = metadata || '{"verified": true}' WHERE id = 1;
-- || merges JSONB objects (like MongoDB's $set on embedded docs)
```

## DELETE (MongoDB: deleteOne / deleteMany)

```sql
DELETE FROM users WHERE id = 1;
DELETE FROM users WHERE is_active = false;
DELETE FROM users WHERE id = 1 RETURNING *;   -- return deleted row

-- TRUNCATE: delete ALL rows (much faster than DELETE for clearing a table)
TRUNCATE TABLE users CASCADE;
```

---

# Part 4: Aggregate Functions and GROUP BY

```sql
-- Count
SELECT COUNT(*) FROM users;                       -- total rows
SELECT COUNT(*) FROM users WHERE is_active = true; -- with filter
SELECT COUNT(DISTINCT role) FROM users;           -- count unique roles

-- Sum, Average, Min, Max
SELECT SUM(total) FROM orders;
SELECT AVG(age) FROM users;
SELECT MIN(age), MAX(age) FROM users;
SELECT ROUND(AVG(total), 2) AS avg_order_value FROM orders;

-- GROUP BY (MongoDB: $group in aggregation)
SELECT role, COUNT(*) AS user_count
FROM users
GROUP BY role;
-- | role  | user_count |
-- |-------|------------|
-- | admin | 3          |
-- | user  | 47         |

-- MongoDB equivalent:
-- db.users.aggregate([{ $group: { _id: "$role", count: { $sum: 1 } } }])

-- GROUP BY with HAVING (filter AFTER grouping)
SELECT role, COUNT(*) AS user_count
FROM users
GROUP BY role
HAVING COUNT(*) > 5;    -- only show roles with more than 5 users

-- WHERE filters rows BEFORE grouping
-- HAVING filters groups AFTER grouping

-- Multi-column GROUP BY
SELECT role, is_active, COUNT(*), AVG(age)
FROM users
GROUP BY role, is_active
ORDER BY role, is_active;
```

---

# Part 5: JOINs — The Superpower MongoDB Doesn't Have

## What Is a JOIN?

A JOIN combines rows from two or more tables based on a related column. This is the #1 reason SQL databases exist — they model relationships between data naturally.

```
users table:              orders table:
| id | name  |            | id | user_id | total  |
|----|-------|            |----|---------|--------|
| 1  | Alice |            | 1  | 1       | 99.99  |
| 2  | Bob   |            | 2  | 1       | 49.99  |
| 3  | Charlie|           | 3  | 2       | 199.99 |

"Give me each user's name WITH their order totals"

SELECT users.name, orders.total
FROM users
JOIN orders ON users.id = orders.user_id;

Result:
| name  | total  |
|-------|--------|
| Alice | 99.99  |
| Alice | 49.99  |
| Bob   | 199.99 |

Charlie has no orders → NOT included (INNER JOIN excludes non-matching rows)
```

## JOIN Types

```
INNER JOIN:  only rows that match in BOTH tables
             Alice ✓ (has orders), Bob ✓ (has orders), Charlie ✗ (no orders)

LEFT JOIN:   ALL rows from left table + matching from right
             Alice ✓, Bob ✓, Charlie ✓ (with NULL for order columns)

RIGHT JOIN:  ALL rows from right table + matching from left
             (rarely used — just swap the tables and use LEFT JOIN)

FULL JOIN:   ALL rows from BOTH tables
             Includes unmatched rows from both sides (NULLs for missing)

CROSS JOIN:  every row from A × every row from B (cartesian product)
             3 users × 3 orders = 9 rows. Rarely useful.
```

```sql
-- INNER JOIN (most common)
SELECT u.name, o.total, o.status
FROM users u                          -- alias: users AS u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN (keep all users even if they have no orders)
SELECT u.name, COALESCE(SUM(o.total), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.name
ORDER BY total_spent DESC;

-- Multiple JOINs
SELECT u.name, o.total, p.name AS product_name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;

-- Self JOIN (table joined with itself)
-- Example: find employees and their managers (both in the same table)
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

```
MongoDB equivalent of a JOIN:
  db.orders.aggregate([
    { $lookup: {
        from: "users",
        localField: "user_id",
        foreignField: "_id",
        as: "user"
    }},
    { $unwind: "$user" }
  ]);

SQL JOIN is cleaner and faster — it's what relational databases are built for.
```

---

# Part 6: Subqueries and CTEs

## Subqueries

```sql
-- Subquery in WHERE (find users who have placed orders)
SELECT name FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- Correlated subquery (subquery references outer query)
SELECT name, email,
    (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.id) AS order_count
FROM users;

-- Subquery in FROM (derived table)
SELECT avg_totals.role, avg_totals.avg_order
FROM (
    SELECT u.role, AVG(o.total) AS avg_order
    FROM users u JOIN orders o ON u.id = o.user_id
    GROUP BY u.role
) AS avg_totals
WHERE avg_totals.avg_order > 100;

-- EXISTS (more efficient than IN for large datasets)
SELECT name FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 100);
```

## CTEs (Common Table Expressions) — WITH Clause

CTEs make complex queries readable by breaking them into named steps. Think of them as "temporary variables" for SQL.

```sql
-- Simple CTE
WITH active_users AS (
    SELECT * FROM users WHERE is_active = true
),
recent_orders AS (
    SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT au.name, COUNT(ro.id) AS recent_order_count
FROM active_users au
LEFT JOIN recent_orders ro ON au.id = ro.user_id
GROUP BY au.name
ORDER BY recent_order_count DESC;

-- Recursive CTE (for tree structures — org charts, categories)
WITH RECURSIVE category_tree AS (
    -- Base case: top-level categories (no parent)
    SELECT id, name, parent_id, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive case: children of previous level
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

---

# Part 7: Window Functions — The Advanced SQL Superpower

Window functions perform calculations across rows RELATED to the current row, without collapsing them into groups (unlike GROUP BY).

```sql
-- ROW_NUMBER: assign a sequential number to each row
SELECT name, age,
    ROW_NUMBER() OVER (ORDER BY age DESC) AS rank
FROM users;
-- | name    | age | rank |
-- |---------|-----|------|
-- | Charlie | 35  | 1    |
-- | Alice   | 30  | 2    |
-- | Diana   | 28  | 3    |

-- PARTITION BY: restart numbering within groups
SELECT name, role, age,
    ROW_NUMBER() OVER (PARTITION BY role ORDER BY age DESC) AS rank_in_role
FROM users;
-- | name    | role  | age | rank_in_role |
-- |---------|-------|-----|--------------|
-- | Charlie | admin | 35  | 1            |
-- | Alice   | admin | 30  | 2            |
-- | Diana   | user  | 28  | 1            |
-- | Bob     | user  | 25  | 2            |

-- RANK vs DENSE_RANK vs ROW_NUMBER
-- If two people have the same age:
--   ROW_NUMBER: 1, 2, 3, 4    (always unique)
--   RANK:       1, 1, 3, 4    (skip after tie)
--   DENSE_RANK: 1, 1, 2, 3    (no skip after tie)

-- Running total
SELECT name, total,
    SUM(total) OVER (ORDER BY created_at) AS running_total
FROM orders;

-- LAG / LEAD: access previous/next row
SELECT date, revenue,
    LAG(revenue) OVER (ORDER BY date) AS prev_day_revenue,
    revenue - LAG(revenue) OVER (ORDER BY date) AS day_over_day_change
FROM daily_revenue;

-- Percentage of total
SELECT name, total,
    ROUND(total / SUM(total) OVER () * 100, 2) AS percentage
FROM orders;

-- Top N per group (e.g., top 3 orders per user)
WITH ranked AS (
    SELECT user_id, total,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rn
    FROM orders
)
SELECT * FROM ranked WHERE rn <= 3;
```

---

# Part 8: Indexes and Performance

## What Indexes Do

```
Without index:  scan EVERY row to find matching ones → O(n)
With index:     jump directly to matching rows via B-tree → O(log n)

Like a book's index: instead of reading every page to find "PostgreSQL",
look it up in the index → page 142. Jump directly there.
```

```sql
-- Create indexes
CREATE INDEX idx_users_email ON users(email);           -- single column
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);  -- composite
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);  -- unique index

-- Partial index (index only a subset of rows)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
-- Smaller index, faster for queries that filter by is_active = true

-- GIN index for JSONB (like MongoDB's wildcard index)
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);
-- Enables: WHERE metadata @> '{"verified": true}'

-- GIN index for arrays
CREATE INDEX idx_users_tags ON users USING GIN (tags);
-- Enables: WHERE tags @> ARRAY['python']

-- GIN index for full-text search
CREATE INDEX idx_posts_search ON posts USING GIN (to_tsvector('english', title || ' ' || body));

-- pgvector index for AI embeddings
CREATE INDEX idx_documents_embedding ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
-- Or HNSW:
CREATE INDEX idx_documents_embedding ON documents USING hnsw (embedding vector_cosine_ops);
```

## EXPLAIN ANALYZE

```sql
-- See how PostgreSQL executes your query
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@test.com';

-- Output:
-- Index Scan using idx_users_email on users  (cost=0.28..8.30 rows=1)
--   Index Cond: (email = 'alice@test.com')
--   Planning Time: 0.1 ms
--   Execution Time: 0.05 ms

-- Without index:
-- Seq Scan on users  (cost=0.00..12.50 rows=1)     ← BAD: sequential scan
--   Filter: (email = 'alice@test.com')
--   Execution Time: 2.3 ms

-- Key things to look for:
--   Seq Scan        → missing index (add one!)
--   Index Scan      → good, using an index
--   Index Only Scan → best, answer entirely from index (covered query)
--   Nested Loop     → could be slow for large datasets
--   Hash Join       → efficient for equality joins
--   Sort            → might need an index with matching ORDER BY
```

---

# Part 9: Transactions and ACID

```sql
-- Transaction: all-or-nothing. Either ALL statements succeed, or NONE do.
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If anything fails between BEGIN and COMMIT, both updates are rolled back.

-- Explicit rollback
BEGIN;
    DELETE FROM users WHERE is_active = false;
    -- Oops, wrong condition! Undo everything:
ROLLBACK;

-- Savepoints (partial rollback)
BEGIN;
    INSERT INTO users (name, email) VALUES ('Alice', 'alice@test.com');
    SAVEPOINT sp1;
    INSERT INTO users (name, email) VALUES ('Bob', 'INVALID-EMAIL');  -- fails
    ROLLBACK TO sp1;   -- undo only Bob's insert, keep Alice's
COMMIT;
```

```
ACID Properties:
  A - Atomicity:    all or nothing (transaction succeeds or rolls back entirely)
  C - Consistency:  data always satisfies constraints (foreign keys, checks)
  I - Isolation:    concurrent transactions don't interfere with each other
  D - Durability:   committed data survives crashes (written to disk)

MongoDB has ACID for single-document operations by default.
Multi-document transactions were added in 4.0 but are less mature than PostgreSQL's.
PostgreSQL has had battle-tested ACID transactions for 25+ years.
```

## Isolation Levels

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- default in PostgreSQL

-- READ UNCOMMITTED: can see other transactions' uncommitted changes (dirty reads)
-- READ COMMITTED:   only sees committed data (default, good for most apps)
-- REPEATABLE READ:  same query returns same results within a transaction
-- SERIALIZABLE:     transactions execute as if they were sequential (slowest, safest)
```

---

# Part 10: JSONB — MongoDB Inside PostgreSQL

```sql
-- Store flexible data alongside structured data
CREATE TABLE products (
    id        SERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    price     NUMERIC(10,2) NOT NULL,
    attrs     JSONB DEFAULT '{}'        -- flexible attributes
);

INSERT INTO products (name, price, attrs) VALUES
    ('Laptop', 999.99, '{"brand": "Dell", "ram": 16, "storage": "512GB SSD"}'),
    ('Phone', 699.99, '{"brand": "Apple", "color": "black", "5g": true}');

-- Query JSONB fields
SELECT name, attrs->>'brand' AS brand FROM products;
-- | Laptop | Dell  |
-- | Phone  | Apple |

-- -> returns JSON, ->> returns text
SELECT name FROM products WHERE attrs->>'brand' = 'Apple';
SELECT name FROM products WHERE (attrs->>'ram')::int > 8;

-- Containment (like MongoDB's $elemMatch equivalent)
SELECT * FROM products WHERE attrs @> '{"brand": "Dell"}';  -- contains this key-value
SELECT * FROM products WHERE attrs ? 'color';                 -- has this key
SELECT * FROM products WHERE attrs ?| ARRAY['color', '5g'];  -- has ANY of these keys
SELECT * FROM products WHERE attrs ?& ARRAY['brand', 'ram']; -- has ALL of these keys

-- Update JSONB
UPDATE products SET attrs = attrs || '{"warranty": "2 years"}' WHERE id = 1;  -- merge
UPDATE products SET attrs = attrs - 'color' WHERE id = 2;                      -- remove key
UPDATE products SET attrs = jsonb_set(attrs, '{ram}', '32') WHERE id = 1;     -- set nested
```

---

# Part 11: pgvector — PostgreSQL as a Vector Database

```sql
-- Install the extension
CREATE EXTENSION vector;

-- Create a table with vector column
CREATE TABLE documents (
    id        SERIAL PRIMARY KEY,
    content   TEXT NOT NULL,
    metadata  JSONB DEFAULT '{}',
    embedding VECTOR(1536)          -- 1536-dimensional vector (OpenAI)
);

-- Insert with embedding
INSERT INTO documents (content, embedding) VALUES
    ('Q3 revenue was $4.2M', '[0.12, -0.34, 0.78, ...]');

-- Similarity search (cosine distance)
SELECT content, metadata,
    1 - (embedding <=> '[0.11, -0.31, 0.76, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.11, -0.31, 0.76, ...]'::vector
LIMIT 5;

-- <=>  cosine distance
-- <->  L2 (Euclidean) distance
-- <#>  inner product (negative, for ORDER BY)

-- HNSW index for fast approximate search
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Combine vector search with SQL filters (the killer feature!)
SELECT content, metadata
FROM documents
WHERE metadata->>'department' = 'finance'          -- SQL filter
  AND created_at > NOW() - INTERVAL '90 days'      -- date filter
ORDER BY embedding <=> query_vector                 -- vector similarity
LIMIT 5;

-- This is why pgvector is powerful: full SQL + vector search in ONE query.
-- Pinecone can't do SQL JOINs. PostgreSQL can do both.
```

---

# Part 12: Useful Patterns and Functions

```sql
-- COALESCE: first non-null value (like MongoDB's $ifNull)
SELECT COALESCE(nickname, name, 'Anonymous') AS display_name FROM users;

-- CASE: conditional logic (like if/else)
SELECT name,
    CASE
        WHEN age < 18 THEN 'minor'
        WHEN age < 65 THEN 'adult'
        ELSE 'senior'
    END AS age_group
FROM users;

-- String functions
SELECT UPPER(name), LOWER(email), LENGTH(name),
    CONCAT(name, ' (', role, ')'),
    TRIM(name), LEFT(name, 3), RIGHT(email, 10),
    REPLACE(email, '@', ' [at] '),
    SPLIT_PART(email, '@', 2) AS domain
FROM users;

-- Date functions
SELECT NOW(),
    CURRENT_DATE,
    EXTRACT(YEAR FROM created_at) AS year,
    EXTRACT(MONTH FROM created_at) AS month,
    DATE_TRUNC('month', created_at) AS month_start,
    AGE(NOW(), created_at) AS account_age,
    created_at + INTERVAL '30 days' AS expires_at
FROM users;

-- Generate series (useful for filling gaps in time series)
SELECT generate_series('2024-01-01'::date, '2024-12-31'::date, '1 month') AS month;

-- UPSERT (INSERT or UPDATE if exists — like MongoDB's upsert)
INSERT INTO users (email, name) VALUES ('alice@test.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name, updated_at = NOW();

-- Bulk upsert
INSERT INTO products (sku, name, price) VALUES
    ('A001', 'Widget', 9.99),
    ('A002', 'Gadget', 19.99)
ON CONFLICT (sku) DO UPDATE SET price = EXCLUDED.price;

-- RETURNING (get the affected rows back)
UPDATE users SET age = age + 1 WHERE id = 1 RETURNING *;
DELETE FROM users WHERE is_active = false RETURNING id, name;
```

---

# Part 13: 🧩 Interview Q&A

**Q: What's the difference between WHERE and HAVING?**
A: WHERE filters individual rows BEFORE grouping. HAVING filters groups AFTER GROUP BY. You can't use aggregate functions in WHERE (e.g., `WHERE COUNT(*) > 5` is invalid — use `HAVING COUNT(*) > 5`).

**Q: What's the difference between INNER JOIN and LEFT JOIN?**
A: INNER JOIN returns only rows that have a match in BOTH tables. LEFT JOIN returns ALL rows from the left table, plus matching rows from the right table (NULLs if no match). Use LEFT JOIN when you want to include items that might not have a match (e.g., users who haven't placed orders).

**Q: Explain window functions with an example.**
A: Window functions compute values across a set of related rows without collapsing them. For example, `ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)` assigns a rank to each employee within their department. Unlike GROUP BY, the original rows are preserved — you see every employee with their rank alongside.

**Q: How do you optimize a slow query?**
A: Step 1: Run `EXPLAIN ANALYZE` to see the execution plan. Step 2: Look for Seq Scans on large tables — add an index on the filtered/joined columns. Step 3: Check if the right index type is used (B-tree for equality/range, GIN for JSONB/arrays/full-text). Step 4: Ensure composite indexes follow the query's filter order. Step 5: Consider partial indexes for frequently filtered subsets. Step 6: Use `pg_stat_statements` to find the most expensive queries in production.

**Q: What are the ACID properties?**
A: Atomicity (transaction is all-or-nothing), Consistency (data always satisfies constraints after a transaction), Isolation (concurrent transactions don't see each other's intermediate states), Durability (committed data survives crashes). PostgreSQL guarantees all four. The default isolation level is READ COMMITTED, which prevents dirty reads but allows non-repeatable reads.

**Q: When would you use PostgreSQL JSONB instead of a NoSQL database?**
A: When you need BOTH structured and flexible data in the same system. JSONB gives you MongoDB-like flexibility for specific fields while keeping relational integrity (foreign keys, transactions, JOINs) for the rest. Also when you need vector search (pgvector) alongside SQL queries — one database instead of PostgreSQL + MongoDB + Pinecone.
