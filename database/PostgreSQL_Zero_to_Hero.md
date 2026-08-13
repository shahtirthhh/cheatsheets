# PostgreSQL — Complete Staff-Level Guide

_From MongoDB user to DBA-level internals · SQLAlchemy · pgvector · Production scaling_

---

# Part 1: Mental Model Shift (MongoDB → PostgreSQL)

```
MongoDB                          PostgreSQL
────────                         ──────────
Collection                       Table
Document (JSON blob)             Row (fixed columns)
Field                            Column
_id (ObjectId)                   id (SERIAL / UUID)
Embedded document                Separate table + JOIN
$lookup (aggregation)            JOIN (native, fast)
Flexible schema                  Fixed schema (enforced by DB)
db.users.find()                  SELECT * FROM users
db.users.insertOne()             INSERT INTO users ...
db.users.updateOne()             UPDATE users SET ... WHERE ...
db.users.deleteOne()             DELETE FROM users WHERE ...
db.users.aggregate()             SELECT ... GROUP BY ... HAVING ...
Mongoose                         SQLAlchemy / Prisma / TypeORM

WHY LEARN SQL when you know MongoDB?
  • 90%+ of enterprise data is in SQL databases
  • JOINs, transactions, and constraints are battle-tested for 40+ years
  • pgvector turns PostgreSQL into a vector DB (RAG without Pinecone)
  • SQL is universal — PostgreSQL, MySQL, SQLite use the same language
  • Staff-level engineers are expected to know both
```

---

# Part 2: Data Types

```sql
-- ── Numeric ──
SMALLINT              -- -32K to 32K (2 bytes)
INTEGER               -- -2B to 2B (4 bytes) ← most common
BIGINT                -- -9.2 quintillion to 9.2 quintillion (8 bytes)
SERIAL                -- auto-increment integer (like MongoDB's auto _id)
BIGSERIAL             -- auto-increment bigint
NUMERIC(10, 2)        -- exact decimal: 10 digits total, 2 after point. USE FOR MONEY.
REAL                  -- 4-byte float (inexact — don't use for money)
DOUBLE PRECISION      -- 8-byte float (inexact)

-- ── Text ──
VARCHAR(255)          -- variable up to 255 chars
TEXT                  -- unlimited length (no performance difference from VARCHAR in PG!)
CHAR(10)              -- fixed 10 chars (padded with spaces — rarely useful)

-- ── Boolean ──
BOOLEAN               -- true / false / null

-- ── Date/Time ──
DATE                  -- 2024-01-15 (date only)
TIME                  -- 14:30:00 (time only)
TIMESTAMP             -- 2024-01-15 14:30:00 (date + time, NO timezone)
TIMESTAMPTZ           -- 2024-01-15 14:30:00+05:30 ← ALWAYS USE THIS
INTERVAL              -- '1 day', '2 hours 30 minutes'

-- GOTCHA: TIMESTAMP WITHOUT TIMEZONE stores whatever you give it.
-- If your server moves zones, dates break. ALWAYS use TIMESTAMPTZ.

-- ── JSON (MongoDB-like flexibility inside PostgreSQL) ──
JSON                  -- stored as text, re-parsed every query (slow)
JSONB                 -- binary JSON, indexable, fast queries ← ALWAYS USE THIS
-- JSONB lets you have MongoDB-like flexible fields alongside strict columns.

-- ── Arrays (PostgreSQL has native arrays!) ──
INTEGER[]             -- array of integers: {1, 2, 3}
TEXT[]                -- array of strings: {"tag1", "tag2"}
-- MongoDB equivalent: an array field inside a document

-- ── Special ──
UUID                  -- 128-bit universally unique identifier
BYTEA                 -- binary data (like MongoDB BinData)
INET                  -- IP address (validates format!)
CIDR                  -- IP network
TSVECTOR              -- full-text search vector
TSQUERY               -- full-text search query
VECTOR(1536)          -- pgvector: AI embedding vector (1536 dimensions)
```

---

# Part 3: Schema Design and CREATE TABLE

```sql
-- ── Users table ──
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    name          VARCHAR(100) NOT NULL,
    hashed_password VARCHAR(128) NOT NULL,
    role          VARCHAR(20) NOT NULL DEFAULT 'user'
                  CHECK (role IN ('admin', 'manager', 'user')),
    age           INTEGER CHECK (age >= 0 AND age <= 150),
    is_active     BOOLEAN NOT NULL DEFAULT true,
    bio           TEXT,
    tags          TEXT[] DEFAULT '{}',
    metadata      JSONB DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ── Trigger: auto-update updated_at (like Mongoose pre-save) ──
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();

-- ── Orders table (belongs to user) ──
CREATE TABLE orders (
    id            BIGSERIAL PRIMARY KEY,
    user_id       BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status        VARCHAR(20) NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','paid','shipped','delivered','cancelled')),
    total         NUMERIC(12, 2) NOT NULL CHECK (total >= 0),
    currency      CHAR(3) NOT NULL DEFAULT 'INR',
    notes         TEXT,
    items         JSONB NOT NULL DEFAULT '[]',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ── Many-to-many: users ↔ roles ──
CREATE TABLE roles (
    id    SERIAL PRIMARY KEY,
    name  VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE user_roles (
    user_id  BIGINT REFERENCES users(id) ON DELETE CASCADE,
    role_id  INTEGER REFERENCES roles(id) ON DELETE CASCADE,
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, role_id)    -- composite PK prevents duplicates
);

-- ── Soft delete pattern (instead of actually deleting) ──
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;
-- Query active users: WHERE deleted_at IS NULL
-- "Delete": UPDATE users SET deleted_at = NOW() WHERE id = 1;
-- Restore: UPDATE users SET deleted_at = NULL WHERE id = 1;

-- Constraint types summary:
-- NOT NULL:     column cannot be null
-- UNIQUE:       no duplicate values
-- PRIMARY KEY:  NOT NULL + UNIQUE + one per table
-- FOREIGN KEY:  must reference existing row in another table
-- CHECK:        custom validation expression
-- DEFAULT:      value if not provided
-- ON DELETE CASCADE: delete children when parent is deleted
-- ON DELETE SET NULL: set foreign key to null when parent deleted
-- ON DELETE RESTRICT: prevent parent deletion if children exist (default)
```

---

# Part 4: CRUD (With MongoDB Equivalents)

## INSERT

```sql
-- Single row
INSERT INTO users (email, name, hashed_password)
VALUES ('alice@test.com', 'Alice', '$2b$12$...');
-- MongoDB: db.users.insertOne({ email: "alice@test.com", name: "Alice", ... })

-- Multiple rows
INSERT INTO users (email, name, hashed_password) VALUES
    ('bob@test.com', 'Bob', '$2b$12$...'),
    ('charlie@test.com', 'Charlie', '$2b$12$...');
-- MongoDB: db.users.insertMany([...])

-- RETURNING (get the created row back — MongoDB equivalent: the inserted document)
INSERT INTO users (email, name, hashed_password)
VALUES ('diana@test.com', 'Diana', '$2b$12$...')
RETURNING id, email, name, created_at;

-- UPSERT (insert or update if exists — like MongoDB's upsert: true)
INSERT INTO users (email, name, hashed_password)
VALUES ('alice@test.com', 'Alice Updated', '$2b$12$...')
ON CONFLICT (email) DO UPDATE SET
    name = EXCLUDED.name,
    updated_at = NOW()
RETURNING *;
-- EXCLUDED refers to the row that WOULD have been inserted
```

## SELECT

```sql
-- Basic
SELECT * FROM users;                                     -- db.users.find({})
SELECT name, email FROM users;                           -- db.users.find({}, {name:1, email:1})
SELECT * FROM users WHERE age > 25;                      -- db.users.find({age: {$gt: 25}})
SELECT * FROM users WHERE role = 'admin' AND is_active = true;
SELECT * FROM users WHERE age BETWEEN 25 AND 35;        -- db.users.find({age: {$gte:25, $lte:35}})
SELECT * FROM users WHERE name IN ('Alice', 'Bob');      -- db.users.find({name: {$in: [...]}})
SELECT * FROM users WHERE email LIKE '%@gmail.com';      -- ends with @gmail.com
SELECT * FROM users WHERE name ILIKE '%alice%';          -- case-insensitive (PG-specific!)
SELECT * FROM users WHERE bio IS NULL;                   -- db.users.find({bio: null})
SELECT * FROM users WHERE bio IS NOT NULL;
SELECT * FROM users WHERE deleted_at IS NULL;            -- soft-delete filter

-- Sorting + Pagination
SELECT * FROM users
ORDER BY created_at DESC                                 -- db.users.find().sort({created_at: -1})
LIMIT 20 OFFSET 40;                                      -- .skip(40).limit(20)

-- DISTINCT
SELECT DISTINCT role FROM users;                         -- db.users.distinct("role")

-- Count
SELECT COUNT(*) FROM users WHERE is_active = true;       -- db.users.countDocuments({is_active: true})
```

## UPDATE

```sql
UPDATE users SET name = 'Alice Smith', age = 31
WHERE id = 1
RETURNING *;
-- MongoDB: db.users.updateOne({_id: 1}, {$set: {name: "Alice Smith", age: 31}})

-- Increment (no $inc needed — just arithmetic)
UPDATE users SET age = age + 1 WHERE id = 1;
-- MongoDB: db.users.updateOne({_id: 1}, {$inc: {age: 1}})

-- Update JSONB
UPDATE users SET metadata = metadata || '{"verified": true}' WHERE id = 1;
-- MongoDB: db.users.updateOne({_id: 1}, {$set: {"metadata.verified": true}})

-- Bulk update
UPDATE users SET is_active = false WHERE last_login < NOW() - INTERVAL '1 year';
```

## DELETE

```sql
DELETE FROM users WHERE id = 1 RETURNING *;
DELETE FROM users WHERE is_active = false AND created_at < NOW() - INTERVAL '2 years';

-- TRUNCATE (delete ALL rows — much faster than DELETE for clearing)
TRUNCATE TABLE users CASCADE;  -- CASCADE also truncates dependent tables
```

---

# Part 5: Aggregations and GROUP BY

```sql
-- Basic aggregations
SELECT COUNT(*) AS total_users FROM users;
SELECT AVG(age) AS avg_age FROM users WHERE is_active = true;
SELECT MIN(created_at) AS first_signup, MAX(created_at) AS last_signup FROM users;
SELECT SUM(total) AS revenue FROM orders WHERE status = 'paid';

-- GROUP BY (MongoDB: $group)
SELECT role, COUNT(*) AS count, ROUND(AVG(age), 1) AS avg_age
FROM users
WHERE is_active = true
GROUP BY role
ORDER BY count DESC;
-- MongoDB: db.users.aggregate([{$match: {is_active: true}}, {$group: {_id: "$role", count: {$sum: 1}}}])

-- HAVING (filter AFTER grouping — WHERE can't use aggregates)
SELECT role, COUNT(*) AS count
FROM users
GROUP BY role
HAVING COUNT(*) > 5;          -- only roles with 6+ users
-- WHERE filters ROWS before grouping
-- HAVING filters GROUPS after grouping

-- Multiple aggregations
SELECT
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*) AS signups,
    COUNT(*) FILTER (WHERE role = 'admin') AS admin_signups,
    ROUND(AVG(age), 1) AS avg_age
FROM users
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month DESC;
-- FILTER (WHERE ...) is PostgreSQL-specific — per-aggregate filtering without subqueries
```

---

# Part 6: JOINs (Deep Dive)

```
INNER JOIN:  only matching rows in BOTH tables
LEFT JOIN:   ALL left rows + matching right (NULL if no match)
RIGHT JOIN:  ALL right rows + matching left (rarely used — swap tables, use LEFT)
FULL JOIN:   ALL rows from BOTH tables (NULL where no match)
CROSS JOIN:  every row × every row (cartesian product, rarely useful)
```

```sql
-- ── INNER JOIN (most common) ──
SELECT u.name, u.email, o.id AS order_id, o.total, o.status
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.status = 'paid'
ORDER BY o.total DESC;
-- Only users who HAVE orders appear

-- ── LEFT JOIN (keep all users, even without orders) ──
SELECT u.name, COUNT(o.id) AS order_count, COALESCE(SUM(o.total), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
ORDER BY total_spent DESC;
-- COALESCE: replace NULL with 0 (users with no orders get 0, not NULL)

-- ── Self JOIN (table joined with itself) ──
-- Example: employees and their managers (both in same table)
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- ── Multiple JOINs ──
SELECT u.name, o.id AS order_id, oi.quantity, p.name AS product
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at > NOW() - INTERVAL '30 days';

-- ── EXISTS (more efficient than IN for large subqueries) ──
-- "Users who have at least one paid order"
SELECT name, email FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.status = 'paid'
);
-- EXISTS stops at the FIRST match (doesn't scan all orders like IN would)
```

---

# Part 7: Subqueries, CTEs, and Window Functions

## CTEs (Common Table Expressions)

```sql
-- Break complex queries into readable steps (like variables in code)
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total) AS revenue
    FROM orders
    WHERE status = 'paid'
    GROUP BY DATE_TRUNC('month', created_at)
),
monthly_growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
        ROUND(
            (revenue - LAG(revenue) OVER (ORDER BY month))
            / LAG(revenue) OVER (ORDER BY month) * 100, 1
        ) AS growth_pct
    FROM monthly_revenue
)
SELECT * FROM monthly_growth ORDER BY month DESC;

-- ── Recursive CTE (for tree structures — org charts, categories) ──
WITH RECURSIVE org_tree AS (
    -- Base: top-level (no manager)
    SELECT id, name, manager_id, 1 AS depth, name::text AS path
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: children
    SELECT e.id, e.name, e.manager_id, t.depth + 1, t.path || ' → ' || e.name
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.id
)
SELECT * FROM org_tree ORDER BY path;
-- Output: "CEO → VP Engineering → Senior Dev → Junior Dev"
```

## Window Functions

```sql
-- ROW_NUMBER: sequential ranking
SELECT name, role, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS company_rank,
    ROW_NUMBER() OVER (PARTITION BY role ORDER BY salary DESC) AS role_rank
FROM employees;
-- PARTITION BY restarts numbering within each group (like GROUP BY but keeps all rows)

-- RANK vs DENSE_RANK vs ROW_NUMBER
-- Salary: [100K, 100K, 90K, 80K]
-- ROW_NUMBER: 1, 2, 3, 4     (always unique)
-- RANK:       1, 1, 3, 4     (skip after tie)
-- DENSE_RANK: 1, 1, 2, 3     (no skip)

-- LAG / LEAD: access previous/next row's value
SELECT date, revenue,
    LAG(revenue) OVER (ORDER BY date) AS prev_day,
    revenue - LAG(revenue) OVER (ORDER BY date) AS day_change,
    LEAD(revenue) OVER (ORDER BY date) AS next_day
FROM daily_revenue;

-- Running total
SELECT date, amount,
    SUM(amount) OVER (ORDER BY date) AS running_total
FROM transactions;

-- Moving average (last 7 days)
SELECT date, revenue,
    ROUND(AVG(revenue) OVER (
        ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_7d
FROM daily_revenue;

-- Percentage of total
SELECT name, salary,
    ROUND(salary::numeric / SUM(salary) OVER () * 100, 1) AS pct_of_total
FROM employees;

-- Top N per group (e.g., top 3 earners per department)
WITH ranked AS (
    SELECT name, department, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn <= 3;

-- NTILE (divide into equal groups)
SELECT name, salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
-- Quartile 1 = top 25%, Quartile 4 = bottom 25%
```

---

# Part 8: Indexes and Query Optimization

## Index Types

```sql
-- ── B-tree (default — for =, <, >, <=, >=, BETWEEN, IN, ORDER BY) ──
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_status ON orders(user_id, status);   -- composite

-- ── Hash (for = only — slightly faster than B-tree for equality) ──
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- ── GIN (for JSONB, arrays, full-text search) ──
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);
-- Enables: WHERE metadata @> '{"verified": true}'
-- Enables: WHERE metadata ? 'premium'

CREATE INDEX idx_users_tags ON users USING GIN(tags);
-- Enables: WHERE tags @> ARRAY['python']
-- Enables: WHERE 'python' = ANY(tags)

-- ── GiST (for geometric, range, full-text) ──
CREATE INDEX idx_events_daterange ON events USING GIST(during);

-- ── Full-text search index ──
ALTER TABLE posts ADD COLUMN search_vector TSVECTOR;
UPDATE posts SET search_vector = to_tsvector('english', title || ' ' || body);
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);
-- Query: WHERE search_vector @@ to_tsquery('english', 'machine & learning')

-- ── Partial index (index only a subset — smaller, faster) ──
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true AND deleted_at IS NULL;
-- Only indexes active users. Queries with these WHERE conditions use this smaller index.

-- ── Expression index ──
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- For: WHERE LOWER(email) = 'alice@test.com'

-- ── Covering index (INCLUDE — all data in the index, no table lookup) ──
CREATE INDEX idx_orders_user_covering ON orders(user_id) INCLUDE (total, status);
-- Query can be answered entirely from the index (Index Only Scan)

-- ── Unique index (enforces uniqueness) ──
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
-- Same as UNIQUE constraint but can be partial:
CREATE UNIQUE INDEX idx_unique_active_email ON users(email) WHERE deleted_at IS NULL;
-- Allows duplicate emails for soft-deleted users!
```

## EXPLAIN ANALYZE (How to Read Query Plans)

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@test.com';

-- ── Good plan (Index Scan) ──
-- Index Scan using idx_users_email on users  (cost=0.28..8.30 rows=1 width=120)
--   Index Cond: (email = 'alice@test.com')
--   Planning Time: 0.1 ms
--   Execution Time: 0.05 ms

-- ── Bad plan (Seq Scan — scanning entire table!) ──
-- Seq Scan on users  (cost=0.00..25.00 rows=1 width=120)
--   Filter: (email = 'alice@test.com')
--   Rows Removed by Filter: 999
--   Execution Time: 5.2 ms

-- WHAT TO LOOK FOR:
--   Seq Scan on big table      → ADD AN INDEX
--   Index Scan                 → good
--   Index Only Scan            → best (data entirely from index)
--   Bitmap Index Scan          → ok (multiple index conditions)
--   Nested Loop                → fine for small inner table, bad for large
--   Hash Join                  → good for equality joins
--   Sort                       → check if index can avoid it
--   Rows Removed by Filter     → index isn't being used for this filter

-- ── Force PostgreSQL to show its thinking ──
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
-- BUFFERS shows: shared hit (from cache) vs shared read (from disk)
-- Cache hits are ~100x faster than disk reads

-- ── Common fixes ──
-- 1. Missing index → CREATE INDEX
-- 2. Wrong index chosen → ANALYZE tablename (update statistics)
-- 3. Function in WHERE prevents index → create expression index
-- 4. LIKE '%middle%' can't use B-tree → use GIN with pg_trgm extension
-- 5. OR conditions → rewrite as UNION or use separate indexes
```

## Index Strategy Guide

```
WHEN TO ADD AN INDEX:
  ✓ Column in WHERE clause (filtered frequently)
  ✓ Column in JOIN condition (foreign keys!)
  ✓ Column in ORDER BY (avoid sort operation)
  ✓ Column in GROUP BY

WHEN NOT TO:
  ✗ Tables with < 1000 rows (seq scan is faster)
  ✗ Columns rarely queried
  ✗ Columns with very low cardinality (boolean: true/false — only 2 values)
  ✗ Write-heavy tables (indexes slow down INSERT/UPDATE/DELETE)

COMPOSITE INDEX ORDER MATTERS:
  CREATE INDEX idx ON orders(user_id, status, created_at);

  ✓ WHERE user_id = 1                              (uses index, leftmost prefix)
  ✓ WHERE user_id = 1 AND status = 'paid'           (uses index)
  ✓ WHERE user_id = 1 AND status = 'paid' AND created_at > '2024-01-01'  (uses full index)
  ✗ WHERE status = 'paid'                            (can't use — user_id is first)
  ✗ WHERE created_at > '2024-01-01'                   (can't use — not leftmost)

  Rule: index is usable from LEFT to RIGHT. Skip a column → rest is useless.
```

---

# Part 9: Transactions, ACID, and Locking

```sql
-- ── Transaction basics ──
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If anything fails → ROLLBACK (both updates are undone)

-- ── Explicit rollback ──
BEGIN;
    DELETE FROM orders WHERE status = 'cancelled';
    -- Oops, wrong query!
ROLLBACK;  -- undo everything

-- ── Savepoints (partial rollback) ──
BEGIN;
    INSERT INTO users (email, name, hashed_password) VALUES ('a@b.com', 'A', '...');
    SAVEPOINT sp1;
    INSERT INTO users (email, name, hashed_password) VALUES ('invalid', 'B', '...');  -- fails
    ROLLBACK TO sp1;   -- undo only B, keep A
COMMIT;
```

## MVCC (Multi-Version Concurrency Control)

```
PostgreSQL's secret sauce for concurrent access WITHOUT locks:

MVCC: every UPDATE creates a NEW VERSION of the row, keeping the OLD version.
Readers see the version that was current when their transaction started.
Writers create new versions without blocking readers.

Transaction 1 (reads):        Transaction 2 (writes):
  BEGIN;                         BEGIN;
  SELECT balance FROM accounts
  WHERE id = 1;
  → sees $100                    UPDATE accounts SET balance = 50 WHERE id = 1;
                                 COMMIT;
  SELECT balance FROM accounts
  WHERE id = 1;
  → STILL sees $100!             (Transaction 1's snapshot is from before the update)
  COMMIT;

After both commit: next reader sees $50.

Old row versions are cleaned up by VACUUM (see Part 10).
```

## Isolation Levels

```sql
-- READ COMMITTED (default in PostgreSQL):
--   Each STATEMENT sees the latest committed data.
--   Two SELECTs in the same transaction might see different data
--   if another transaction committed between them.

-- REPEATABLE READ:
--   The transaction sees a SNAPSHOT from its start.
--   Same SELECT always returns same results within the transaction.
--   Throws error if it would produce a serialization anomaly.
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- SERIALIZABLE (strictest):
--   Transactions behave AS IF they ran one after another (serial order).
--   PostgreSQL detects conflicts and aborts one of the conflicting transactions.
--   Slowest but safest. Use for financial operations.
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

## Locking

```sql
-- ── Row-level lock (most common) ──
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Locks this row. Other transactions trying FOR UPDATE on the same row WAIT.
-- Regular SELECTs still work (MVCC — they see the old version).

-- FOR UPDATE SKIP LOCKED (job queue pattern — like BullMQ!)
-- Multiple workers grab tasks without waiting:
BEGIN;
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY priority DESC
LIMIT 1
FOR UPDATE SKIP LOCKED;     -- if someone else locked it, skip to the next one
-- Process the task...
UPDATE tasks SET status = 'processing' WHERE id = <selected_id>;
COMMIT;

-- ── Advisory locks (application-level locks) ──
SELECT pg_advisory_lock(12345);      -- acquire lock (blocks if taken)
-- do critical section work...
SELECT pg_advisory_unlock(12345);    -- release

-- ── Deadlock detection ──
-- PostgreSQL automatically detects deadlocks and aborts one transaction.
-- Prevention: always lock rows in the SAME ORDER across transactions.
```

---

# Part 10: DBA Internals (Vacuum, WAL, Connection Pooling)

## VACUUM (Dead Row Cleanup)

```
Remember MVCC? Every UPDATE/DELETE creates a dead row version.
VACUUM cleans these up to reclaim disk space.

AUTOVACUUM (enabled by default — usually fine):
  PostgreSQL automatically runs VACUUM when a table has enough dead rows.
  Config: autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor

VACUUM (manual):
  VACUUM users;                   -- reclaim space (doesn't lock table)
  VACUUM FULL users;              -- rewrites entire table (locks table! use rarely)
  VACUUM ANALYZE users;           -- vacuum + update query planner statistics

ANALYZE (update statistics only — no cleanup):
  ANALYZE users;                  -- helps query planner choose better indexes
  -- Run after bulk INSERT/UPDATE/DELETE

Monitor dead tuples:
  SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum
  FROM pg_stat_user_tables
  ORDER BY n_dead_tup DESC;
```

## WAL (Write-Ahead Log)

```
Every change is written to the WAL BEFORE modifying data files.
If PostgreSQL crashes, it replays the WAL to recover.

WAL flow:
  1. Transaction changes data
  2. Changes written to WAL (sequential writes — fast)
  3. WAL flushed to disk (data is now safe)
  4. Background writer eventually writes changes to actual data files

WAL enables:
  • Crash recovery (replay WAL after crash)
  • Replication (stream WAL to replicas for real-time sync)
  • Point-in-time recovery (restore to any moment using WAL archive)

WAL config:
  wal_level = replica                 -- or 'logical' for logical replication
  max_wal_senders = 10                -- max replication connections
  archive_mode = on                   -- enable WAL archiving for PITR
```

## Connection Pooling

```
Problem: each PostgreSQL connection uses ~10MB RAM.
  100 connections = 1GB. 1000 connections = 10GB. PG can't handle thousands.

  Application servers with 4 workers × 20 connections each = 80 connections.
  Add 10 app servers = 800 connections. PostgreSQL struggles.

Solution: connection pooler sits between app and PostgreSQL.

  App → PgBouncer (100 pooled connections) → PostgreSQL (20 actual connections)

PgBouncer modes:
  session:     one PG connection per client session (least aggressive)
  transaction: one PG connection per transaction (recommended) ← use this
  statement:   one PG connection per statement (most aggressive, breaks multi-statement txns)

# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
pool_mode = transaction
max_client_conn = 1000     # accept up to 1000 app connections
default_pool_size = 20      # but only 20 actual PG connections
```

## Replication

```
PRIMARY → REPLICA(s)

Streaming Replication (physical):
  • Byte-for-byte copy of primary
  • Replicas are read-only
  • Use for: read scaling (send reads to replicas), high availability (failover)

  pg_basebackup -h primary -D /var/lib/postgresql/data -Fp -Xs -P

Logical Replication (selective):
  • Replicate specific tables/databases
  • Replicas can have different schemas
  • Use for: data migration, ETL, cross-version upgrades

  -- On primary:
  CREATE PUBLICATION my_pub FOR TABLE users, orders;

  -- On replica:
  CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=mydb'
    PUBLICATION my_pub;
```

## Partitioning (For Massive Tables)

```sql
-- Split a huge table into smaller pieces by a column value
-- Queries automatically go to the right partition

-- Range partitioning (by date — most common)
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    total NUMERIC(12,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
CREATE TABLE orders_2024_q3 PARTITION OF orders
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');

-- Query: WHERE created_at > '2024-06-15' → only scans Q3 partition!
-- Without partitioning: scans ALL 100M rows.

-- List partitioning (by category)
CREATE TABLE logs (...) PARTITION BY LIST (severity);
CREATE TABLE logs_error PARTITION OF logs FOR VALUES IN ('error', 'critical');
CREATE TABLE logs_info PARTITION OF logs FOR VALUES IN ('info', 'debug');
```

---

# Part 11: JSONB (MongoDB Inside PostgreSQL)

```sql
-- Store flexible data alongside strict schema
CREATE TABLE products (
    id     SERIAL PRIMARY KEY,
    name   VARCHAR(100) NOT NULL,
    price  NUMERIC(10,2) NOT NULL,
    attrs  JSONB DEFAULT '{}'
);

INSERT INTO products (name, price, attrs) VALUES
    ('Laptop', 999.99, '{"brand": "Dell", "ram": 16, "specs": {"cpu": "i7", "gpu": "RTX 3060"}}'),
    ('Phone', 699.99, '{"brand": "Apple", "color": "black", "5g": true}');

-- ── Access JSONB fields ──
SELECT name, attrs->>'brand' AS brand FROM products;                    -- text value
SELECT name, (attrs->>'ram')::int AS ram FROM products;                 -- cast to int
SELECT name, attrs->'specs'->>'cpu' AS cpu FROM products;               -- nested access

-- ── Filter on JSONB ──
SELECT * FROM products WHERE attrs->>'brand' = 'Dell';                  -- exact match
SELECT * FROM products WHERE (attrs->>'ram')::int > 8;                  -- numeric comparison
SELECT * FROM products WHERE attrs @> '{"brand": "Dell"}';              -- contains key-value
SELECT * FROM products WHERE attrs ? 'color';                           -- has key
SELECT * FROM products WHERE attrs ?| ARRAY['color', '5g'];            -- has ANY key
SELECT * FROM products WHERE attrs ?& ARRAY['brand', 'ram'];           -- has ALL keys
SELECT * FROM products WHERE attrs @> '{"specs": {"cpu": "i7"}}';      -- nested containment

-- ── Update JSONB ──
UPDATE products SET attrs = attrs || '{"warranty": "2yr"}' WHERE id = 1;        -- merge
UPDATE products SET attrs = attrs - 'color' WHERE id = 2;                       -- remove key
UPDATE products SET attrs = jsonb_set(attrs, '{ram}', '32') WHERE id = 1;       -- set value
UPDATE products SET attrs = jsonb_set(attrs, '{specs,ssd}', '"1TB"') WHERE id = 1;  -- nested set

-- ── Aggregate JSONB ──
SELECT attrs->>'brand' AS brand, COUNT(*), AVG(price)
FROM products
GROUP BY attrs->>'brand';

-- ── Index JSONB (makes queries fast) ──
CREATE INDEX idx_products_attrs ON products USING GIN(attrs);
-- Now @>, ?, ?|, ?& queries are fast

CREATE INDEX idx_products_brand ON products ((attrs->>'brand'));
-- Expression index for specific key lookups
```

---

# Part 12: pgvector (PostgreSQL as Vector Database)

```sql
-- Install extension
CREATE EXTENSION vector;

-- ── Create table with embedding column ──
CREATE TABLE documents (
    id        BIGSERIAL PRIMARY KEY,
    content   TEXT NOT NULL,
    metadata  JSONB DEFAULT '{}',
    embedding VECTOR(1536),                -- OpenAI text-embedding-3-small (1536 dims)
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ── Insert with embedding ──
INSERT INTO documents (content, metadata, embedding) VALUES
    ('Q3 revenue was $4.2M, up 15% from Q2',
     '{"source": "annual_report.pdf", "page": 12}',
     '[0.012, -0.034, 0.078, ...]');  -- 1536 floats

-- ── Similarity Search ──
-- Cosine distance (<=> operator)
SELECT content, metadata,
    1 - (embedding <=> '[0.011, -0.031, 0.076, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.011, -0.031, 0.076, ...]'::vector
LIMIT 5;

-- Distance operators:
-- <=>  cosine distance (most common for embeddings)
-- <->  L2 (Euclidean) distance
-- <#>  negative inner product (for ORDER BY, since PG sorts ascending)

-- ── HYBRID SEARCH: Vector + SQL filters (THE killer feature!) ──
SELECT content, metadata,
    1 - (embedding <=> $1::vector) AS similarity
FROM documents
WHERE metadata->>'department' = 'finance'        -- SQL filter on JSONB
  AND created_at > NOW() - INTERVAL '90 days'    -- SQL filter on date
ORDER BY embedding <=> $1::vector                 -- vector similarity ranking
LIMIT 5;
-- Pinecone CAN'T do this. ChromaDB can barely do this. PostgreSQL does it natively.

-- ── Indexes for fast vector search ──

-- IVFFlat (Inverted File Index): clusters vectors, searches nearest clusters
CREATE INDEX idx_docs_embedding_ivf ON documents
USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
-- lists = √(num_rows) is a good starting point
-- Faster to build, less accurate than HNSW
-- Must VACUUM after bulk inserts for index to be effective

-- HNSW (Hierarchical Navigable Small World): graph-based, more accurate
CREATE INDEX idx_docs_embedding_hnsw ON documents
USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
-- m: connections per node (16 default, higher = more accurate, more memory)
-- ef_construction: build quality (64 default, higher = slower build, better recall)
-- Slower to build but faster and more accurate at query time than IVFFlat

-- IVFFlat vs HNSW:
--   IVFFlat: faster to build, needs VACUUM after inserts, less accurate
--   HNSW:    slower to build, works immediately after inserts, more accurate ★
--   For production RAG: use HNSW
```

```python
# ── SQLAlchemy + pgvector ──
from pgvector.sqlalchemy import Vector
from sqlalchemy import Column, BigInteger, Text, DateTime, func
from sqlalchemy.dialects.postgresql import JSONB

class Document(Base):
    __tablename__ = "documents"
    id = Column(BigInteger, primary_key=True, index=True)
    content = Column(Text, nullable=False)
    metadata = Column(JSONB, default={})
    embedding = Column(Vector(1536))
    created_at = Column(DateTime(timezone=True), server_default=func.now())

# Similarity search in SQLAlchemy
from pgvector.sqlalchemy import Vector
from sqlalchemy import select

query_embedding = embeddings.embed_query("What was Q3 revenue?")

stmt = (
    select(Document)
    .where(Document.metadata["department"].astext == "finance")
    .order_by(Document.embedding.cosine_distance(query_embedding))
    .limit(5)
)
results = await session.execute(stmt)
docs = results.scalars().all()
```

---

# Part 13: Full-Text Search

```sql
-- PostgreSQL has built-in full-text search (no Elasticsearch needed for basic cases)

-- ── Setup ──
ALTER TABLE posts ADD COLUMN search_vector TSVECTOR;

-- Populate search vector
UPDATE posts SET search_vector =
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(body, '')), 'B');
-- Weight 'A' = highest relevance (title matches rank higher)

-- Keep it updated automatically
CREATE TRIGGER posts_search_update
    BEFORE INSERT OR UPDATE ON posts
    FOR EACH ROW EXECUTE FUNCTION
    tsvector_update_trigger(search_vector, 'pg_catalog.english', title, body);

-- Index it
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);

-- ── Search ──
SELECT title, ts_rank(search_vector, query) AS rank
FROM posts, to_tsquery('english', 'machine & learning') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- Search operators:
-- &  AND: 'machine & learning' (both words)
-- |  OR:  'machine | learning' (either word)
-- !  NOT: 'machine & !deep'    (machine but not deep)
-- <-> FOLLOWED BY: 'machine <-> learning' (phrase search)
```

---

# Part 14: SQLAlchemy + Alembic (Migrations)

```bash
# Setup Alembic (like TypeORM migrations)
pip install alembic
alembic init alembic

# Edit alembic.ini:
# sqlalchemy.url = postgresql+asyncpg://user:pass@localhost:5432/mydb

# Auto-generate migration from model changes
alembic revision --autogenerate -m "add users table"

# Apply migrations
alembic upgrade head          # apply all pending
alembic upgrade +1            # apply next one
alembic downgrade -1          # rollback last one
alembic downgrade base        # rollback all
alembic history               # show migration history
alembic current               # show current version
```

---

# Part 15: 🧩 Interview Q&A

**Q: INNER JOIN vs LEFT JOIN — when to use which?**
A: INNER JOIN when you only want rows that have matches in BOTH tables (users WITH orders). LEFT JOIN when you want ALL rows from the left table even without matches (users including those WITHOUT orders — they get NULL for order columns). Use LEFT JOIN + COALESCE for "default if no match" patterns.

**Q: How do you optimize a slow PostgreSQL query?**
A: (1) Run EXPLAIN ANALYZE. (2) Look for Seq Scan on large tables — add an index. (3) Check composite index column order matches your WHERE clause (leftmost prefix rule). (4) Use partial indexes for frequently filtered subsets (WHERE is_active = true). (5) Run ANALYZE to update planner statistics. (6) Check for missing JOINs indexes (foreign keys). (7) Consider covering indexes (INCLUDE) for Index Only Scan. (8) For JSONB: add GIN index. For text search: GIN on tsvector.

**Q: What is MVCC and why does PostgreSQL use it?**
A: Multi-Version Concurrency Control. Every UPDATE creates a new row version instead of modifying in place. Readers see the version from their transaction's snapshot. This means readers never block writers and writers never block readers — massive concurrency without locks. Old versions are cleaned up by VACUUM. The tradeoff is write amplification (each update creates a full new row) and the need for vacuum maintenance.

**Q: When would you use pgvector instead of Pinecone/ChromaDB?**
A: When you need hybrid queries combining vector similarity with SQL filters, JOINs, and transactions in a single query. pgvector lets you search "find documents similar to X WHERE department = 'finance' AND created_at > '2024-01-01'" in one SQL statement. Pinecone can filter metadata but can't JOIN with your users table or participate in transactions. pgvector also means one fewer system to deploy and maintain. Use dedicated vector DBs only when you have billions of vectors and need the absolute fastest ANN search.

**Q: Explain partitioning and when you'd use it.**
A: Partitioning splits a large table into smaller physical pieces based on a column value (usually date). A 500M-row orders table partitioned by quarter means each partition has ~40M rows. Queries with WHERE created_at filters only scan relevant partitions (partition pruning). Also helps: faster vacuuming (per partition), easier data lifecycle (drop old partition instead of DELETE), parallel scans. Use when: table exceeds ~50M rows, queries always filter on the partition key, data has natural time boundaries.

**Q: How does connection pooling work and why is it needed?**
A: Each PostgreSQL connection uses ~10MB RAM and a backend process. At 500 connections that's 5GB just for connections. PgBouncer sits between app and PG, maintaining a small pool of actual PG connections (e.g., 20) and multiplexing hundreds of app connections through them. In transaction mode, a PG connection is assigned only for the duration of a transaction, then returned to the pool. This lets you handle 1000+ app connections with only 20-50 actual PG connections.
