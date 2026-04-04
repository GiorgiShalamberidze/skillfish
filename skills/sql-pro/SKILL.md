---
name: sql-pro
description: Advanced SQL: window functions, CTEs, recursive queries, query optimization with EXPLAIN, indexing strategies, partitioning, and dialect differences.
---

# SQL Pro

Master advanced SQL techniques across PostgreSQL, MySQL, and SQLite. This skill covers window functions, recursive CTEs, query optimization with EXPLAIN plans, indexing strategies, partitioning, and dialect-specific differences -- everything beyond the basics that separates performant production queries from naive ones.

## Table of Contents

- [Window Functions](#window-functions)
- [Common Table Expressions](#common-table-expressions)
- [Query Optimization](#query-optimization)
- [Indexing Strategies](#indexing-strategies)
- [Advanced Joins and Set Operations](#advanced-joins-and-set-operations)
- [Data Modeling Patterns](#data-modeling-patterns)
- [Partitioning and Scaling](#partitioning-and-scaling)
- [Dialect Quick Reference](#dialect-quick-reference)

---

## Window Functions

Window functions perform calculations across a set of rows related to the current row without collapsing them into a single output row.

### Ranking Functions

```sql
-- Sample data
CREATE TABLE sales (
  id SERIAL PRIMARY KEY,
  rep_name TEXT,
  region TEXT,
  amount DECIMAL(10,2),
  sale_date DATE
);

INSERT INTO sales (rep_name, region, amount, sale_date) VALUES
  ('Alice', 'West',  1500, '2025-01-15'),
  ('Bob',   'West',  1500, '2025-01-20'),
  ('Carol', 'East',  2100, '2025-01-10'),
  ('Dave',  'East',  1800, '2025-02-05'),
  ('Alice', 'West',  2200, '2025-02-12'),
  ('Carol', 'East',  2100, '2025-02-18');

-- ROW_NUMBER: unique sequential integer, no ties
-- RANK: ties get the same rank, next rank is skipped
-- DENSE_RANK: ties get the same rank, next rank is NOT skipped
SELECT
  rep_name,
  amount,
  ROW_NUMBER() OVER (ORDER BY amount DESC) AS row_num,
  RANK()       OVER (ORDER BY amount DESC) AS rnk,
  DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rnk
FROM sales;

-- Result:
-- rep_name | amount  | row_num | rnk | dense_rnk
-- Alice    | 2200.00 |       1 |   1 |         1
-- Carol    | 2100.00 |       2 |   2 |         2
-- Carol    | 2100.00 |       3 |   2 |         2
-- Dave     | 1800.00 |       4 |   4 |         3
-- Alice    | 1500.00 |       5 |   5 |         4
-- Bob      | 1500.00 |       6 |   5 |         4
```

### LEAD, LAG, FIRST_VALUE, LAST_VALUE

```sql
-- Compare each sale to the previous and next sale per rep
SELECT
  rep_name,
  sale_date,
  amount,
  LAG(amount, 1)  OVER w AS prev_amount,
  LEAD(amount, 1) OVER w AS next_amount,
  amount - LAG(amount, 1) OVER w AS change_from_prev
FROM sales
WINDOW w AS (PARTITION BY rep_name ORDER BY sale_date);

-- FIRST_VALUE / LAST_VALUE within a partition
SELECT
  rep_name,
  sale_date,
  amount,
  FIRST_VALUE(amount) OVER w AS first_sale,
  LAST_VALUE(amount)  OVER w AS last_sale_so_far
FROM sales
WINDOW w AS (
  PARTITION BY rep_name ORDER BY sale_date
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

### NTILE, Running Totals, Moving Averages

```sql
-- NTILE: divide rows into N roughly equal buckets
SELECT rep_name, amount,
  NTILE(4) OVER (ORDER BY amount DESC) AS quartile
FROM sales;

-- Running total per region
SELECT
  region, sale_date, amount,
  SUM(amount) OVER (
    PARTITION BY region ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM sales;

-- 3-row moving average
SELECT
  sale_date, amount,
  AVG(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
  ) AS moving_avg_3
FROM sales;
```

### Frame Specification: ROWS vs RANGE vs GROUPS

```sql
-- ROWS: physical row offsets (exact row count)
SUM(amount) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- RANGE: logical value range (peers with same ORDER BY value included)
SUM(amount) OVER (ORDER BY sale_date RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW)

-- GROUPS (Postgres 11+): groups of peer rows
SUM(amount) OVER (ORDER BY sale_date GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING)
```

**Performance tip:** Window functions with `PARTITION BY` benefit from indexes on the partition column. If you see "Sort" nodes dominating your EXPLAIN output, add an index matching `(partition_col, order_col)`.

---

## Common Table Expressions

CTEs improve readability and enable recursive queries for hierarchical data.

### Basic WITH Clause

```sql
-- Break a complex report into digestible steps
WITH regional_totals AS (
  SELECT region, SUM(amount) AS total
  FROM sales
  GROUP BY region
),
overall AS (
  SELECT SUM(total) AS grand_total FROM regional_totals
)
SELECT
  r.region,
  r.total,
  ROUND(r.total / o.grand_total * 100, 1) AS pct
FROM regional_totals r, overall o
ORDER BY r.total DESC;
```

### Recursive CTEs for Hierarchical Data

```sql
-- Org chart: find all reports under a manager
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name TEXT,
  manager_id INT REFERENCES employees(id)
);

INSERT INTO employees VALUES
  (1, 'CEO',     NULL),
  (2, 'VP Eng',  1),
  (3, 'VP Sales', 1),
  (4, 'Dev Lead', 2),
  (5, 'Dev Sr',   4),
  (6, 'Dev Jr',   4),
  (7, 'Sales Mgr', 3);

-- All direct and indirect reports under VP Eng (id=2)
WITH RECURSIVE reports AS (
  -- Base case: the starting manager
  SELECT id, name, manager_id, 1 AS depth
  FROM employees
  WHERE id = 2

  UNION ALL

  -- Recursive step: find direct reports of current level
  SELECT e.id, e.name, e.manager_id, r.depth + 1
  FROM employees e
  JOIN reports r ON e.manager_id = r.id
)
SELECT id, name, depth
FROM reports
ORDER BY depth, name;

-- Result:
-- id | name     | depth
--  2 | VP Eng   |     1
--  4 | Dev Lead |     2
--  5 | Dev Sr   |     3
--  6 | Dev Jr   |     3
```

### CTE vs Subquery Performance

```sql
-- PostgreSQL: CTEs are optimization fences by default (< v12)
-- From Postgres 12+ the planner can inline non-recursive CTEs

-- Force inlining (Postgres 12+):
WITH regional AS NOT MATERIALIZED (
  SELECT region, SUM(amount) AS total FROM sales GROUP BY region
)
SELECT * FROM regional WHERE total > 3000;

-- Force materialization (useful if CTE is referenced multiple times):
WITH regional AS MATERIALIZED (
  SELECT region, SUM(amount) AS total FROM sales GROUP BY region
)
SELECT * FROM regional r1 JOIN regional r2 ON r1.total < r2.total;
```

**Performance tip:** In MySQL 8.0+, CTEs are always materialized into a temp table when referenced more than once. If your CTE is large and only used once, a subquery may be faster. In PostgreSQL 12+, the planner inlines single-use CTEs automatically.

---

## Query Optimization

### Reading EXPLAIN / EXPLAIN ANALYZE

```sql
-- Basic EXPLAIN shows the plan without executing
EXPLAIN
SELECT s.rep_name, COUNT(*)
FROM sales s
WHERE s.region = 'West'
GROUP BY s.rep_name;

-- EXPLAIN ANALYZE actually runs the query and shows real timings
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT s.rep_name, COUNT(*)
FROM sales s
WHERE s.region = 'West'
GROUP BY s.rep_name;

-- Sample Postgres output:
-- HashAggregate  (cost=15.20..15.22 rows=2 width=40)
--                (actual time=0.045..0.047 rows=2 loops=1)
--   Group Key: rep_name
--   Buffers: shared hit=1
--   ->  Seq Scan on sales s  (cost=0.00..15.10 rows=3 width=32)
--                             (actual time=0.012..0.015 rows=3 loops=1)
--         Filter: (region = 'West')
--         Rows Removed by Filter: 3
--         Buffers: shared hit=1
-- Planning Time: 0.085 ms
-- Execution Time: 0.078 ms
```

**Key EXPLAIN fields:**
| Field | Meaning |
|---|---|
| `cost=X..Y` | Estimated startup cost..total cost (arbitrary units) |
| `rows=N` | Estimated row count (compare to actual) |
| `actual time` | Real elapsed time in ms |
| `Buffers: shared hit` | Pages read from cache (no disk I/O) |
| `Buffers: shared read` | Pages read from disk |
| `Rows Removed by Filter` | Rows fetched but discarded -- high value means missing index |

### Join Algorithms

```sql
-- Nested Loop: best for small outer + indexed inner
-- The planner picks this when one side is very small
EXPLAIN ANALYZE
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE e.id = 5;  -- single row lookup -> nested loop

-- Hash Join: best for larger unsorted datasets
-- Builds a hash table from the smaller relation, probes with the larger
EXPLAIN ANALYZE
SELECT s.rep_name, r.region_name
FROM sales s
JOIN regions r ON s.region = r.code;

-- Merge Join: best when both sides are pre-sorted or indexed
-- Walks both sorted inputs in lockstep
EXPLAIN ANALYZE
SELECT a.id, b.id
FROM table_a a
JOIN table_b b ON a.sort_key = b.sort_key
ORDER BY a.sort_key;
```

### Statistics and Cardinality Estimation

```sql
-- Update statistics so the planner makes good choices
ANALYZE sales;

-- Check what the planner thinks about value distribution
SELECT
  attname,
  n_distinct,
  most_common_vals,
  most_common_freqs
FROM pg_stats
WHERE tablename = 'sales' AND attname = 'region';

-- If estimates are wildly off, consider:
-- 1. Running ANALYZE after bulk loads
-- 2. Increasing default_statistics_target for skewed columns
ALTER TABLE sales ALTER COLUMN region SET STATISTICS 1000;
ANALYZE sales;
```

### Common Anti-Patterns

```sql
-- BAD: function on indexed column prevents index use
SELECT * FROM sales WHERE YEAR(sale_date) = 2025;
-- GOOD: use a range
SELECT * FROM sales WHERE sale_date >= '2025-01-01' AND sale_date < '2026-01-01';

-- BAD: implicit type cast kills index
SELECT * FROM users WHERE phone = 5551234;  -- phone is VARCHAR
-- GOOD: match the column type
SELECT * FROM users WHERE phone = '5551234';

-- BAD: SELECT * fetches unnecessary columns, blocks index-only scans
SELECT * FROM sales WHERE region = 'West';
-- GOOD: select only what you need
SELECT rep_name, amount FROM sales WHERE region = 'West';

-- BAD: OR across different columns prevents single index use
SELECT * FROM sales WHERE rep_name = 'Alice' OR region = 'East';
-- GOOD: rewrite as UNION ALL
SELECT * FROM sales WHERE rep_name = 'Alice'
UNION ALL
SELECT * FROM sales WHERE region = 'East' AND rep_name != 'Alice';
```

---

## Indexing Strategies

### Index Types

```sql
-- B-tree (default): equality and range queries, ORDER BY
CREATE INDEX idx_sales_date ON sales (sale_date);

-- Hash: equality only, smaller than B-tree (Postgres 10+)
CREATE INDEX idx_sales_region_hash ON sales USING hash (region);

-- GIN (Generalized Inverted Index): arrays, JSONB, full-text search
CREATE INDEX idx_orders_tags ON orders USING gin (tags);
-- Supports: @>, ?, ?|, ?& operators for JSONB
SELECT * FROM orders WHERE tags @> '["urgent"]';

-- GiST (Generalized Search Tree): geometric, range, full-text
CREATE INDEX idx_locations_geo ON locations USING gist (coordinates);
-- Supports: &&, @>, <@, <<, >> operators for ranges/geometry
```

### Covering Indexes (Index-Only Scans)

```sql
-- Include non-key columns so the query never hits the heap
CREATE INDEX idx_sales_region_covering
  ON sales (region)
  INCLUDE (rep_name, amount);

-- This query can be satisfied entirely from the index
EXPLAIN ANALYZE
SELECT rep_name, amount FROM sales WHERE region = 'West';
-- -> Index Only Scan using idx_sales_region_covering on sales
```

### Partial Indexes

```sql
-- Index only the rows that matter -- smaller, faster, cheaper
CREATE INDEX idx_sales_active ON sales (sale_date)
  WHERE status = 'active';

-- Only useful when queries include the same WHERE clause
SELECT * FROM sales WHERE status = 'active' AND sale_date > '2025-01-01';
```

### Expression Indexes

```sql
-- Index a computed expression
CREATE INDEX idx_users_lower_email ON users (LOWER(email));

-- Now this query uses the index
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

### Composite Index Column Ordering

```sql
-- Rule of thumb: equality columns first, then range columns, then sort columns
-- Query: WHERE region = 'West' AND sale_date > '2025-01-01' ORDER BY amount DESC

-- GOOD: equality, range, sort
CREATE INDEX idx_sales_composite ON sales (region, sale_date, amount DESC);

-- BAD: range column before equality
CREATE INDEX idx_sales_bad ON sales (sale_date, region, amount DESC);
```

### When NOT to Index

- **Small tables** (< 1000 rows): sequential scan is faster than index overhead
- **High-write, low-read tables** (logs, event streams): indexes slow down inserts
- **Low-selectivity columns** (boolean, status with 2-3 values): index scan reads most of the table anyway
- **Frequently updated columns**: every UPDATE rewrites the index entry

**Performance tip:** Use `pg_stat_user_indexes` to find unused indexes consuming disk and slowing writes:

```sql
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Advanced Joins and Set Operations

### Lateral Joins

```sql
-- LATERAL lets the subquery reference columns from preceding FROM items
-- Get the top 2 sales per region
SELECT r.region, top.rep_name, top.amount
FROM (SELECT DISTINCT region FROM sales) r
CROSS JOIN LATERAL (
  SELECT rep_name, amount
  FROM sales s
  WHERE s.region = r.region
  ORDER BY amount DESC
  LIMIT 2
) top;
```

### Self Joins

```sql
-- Find employees who share the same manager
SELECT a.name AS emp1, b.name AS emp2, a.manager_id
FROM employees a
JOIN employees b ON a.manager_id = b.manager_id AND a.id < b.id;
```

### Anti-Joins and Semi-Joins

```sql
-- Anti-join: find customers with NO orders
-- Method 1: NOT EXISTS (usually best)
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Method 2: LEFT JOIN / IS NULL
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;

-- Semi-join: find customers who HAVE at least one order
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

### CROSS APPLY (SQL Server / translated to LATERAL in Postgres)

```sql
-- SQL Server syntax
SELECT d.dept_name, e.name, e.salary
FROM departments d
CROSS APPLY (
  SELECT TOP 3 name, salary FROM employees
  WHERE dept_id = d.id ORDER BY salary DESC
) e;

-- Postgres equivalent uses LATERAL (see above)
```

### EXCEPT, INTERSECT, UNION ALL

```sql
-- UNION ALL: all rows from both queries (duplicates kept, faster)
-- UNION: removes duplicates (requires sort/hash, slower)
SELECT rep_name FROM sales_2024
UNION ALL
SELECT rep_name FROM sales_2025;

-- EXCEPT: rows in first query but NOT in second
SELECT customer_id FROM customers_2024
EXCEPT
SELECT customer_id FROM churned_customers;

-- INTERSECT: rows that appear in BOTH queries
SELECT product_id FROM orders_web
INTERSECT
SELECT product_id FROM orders_mobile;
```

**Performance tip:** Always prefer `UNION ALL` over `UNION` unless you specifically need deduplication. `UNION` forces a sort or hash step.

---

## Data Modeling Patterns

### Normalization vs Denormalization

```sql
-- Normalized (3NF): no data redundancy, enforced via foreign keys
CREATE TABLE authors (id SERIAL PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE books   (id SERIAL PRIMARY KEY, title TEXT, author_id INT REFERENCES authors(id));

-- Denormalized: store author_name directly for read-heavy workloads
CREATE TABLE books_denorm (
  id SERIAL PRIMARY KEY,
  title TEXT,
  author_id INT,
  author_name TEXT  -- duplicated, but avoids JOIN on every read
);

-- Trade-off: normalized = easier writes, consistent data
--            denormalized = faster reads, risk of stale data
```

### Temporal Data (Slowly Changing Dimensions)

```sql
-- Type 2 SCD: keep full history with valid_from/valid_to
CREATE TABLE product_prices (
  product_id INT NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  valid_from TIMESTAMP NOT NULL DEFAULT NOW(),
  valid_to TIMESTAMP NOT NULL DEFAULT 'infinity',
  EXCLUDE USING gist (
    product_id WITH =,
    tsrange(valid_from, valid_to) WITH &&
  )
);

-- Current price query
SELECT * FROM product_prices
WHERE product_id = 42 AND NOW() BETWEEN valid_from AND valid_to;

-- Price at a specific point in time
SELECT * FROM product_prices
WHERE product_id = 42 AND '2025-06-15'::timestamp BETWEEN valid_from AND valid_to;
```

### Soft Deletes

```sql
-- Add a deleted_at column instead of actually deleting rows
ALTER TABLE customers ADD COLUMN deleted_at TIMESTAMP;

-- "Delete" a customer
UPDATE customers SET deleted_at = NOW() WHERE id = 99;

-- All active-record queries must filter
CREATE VIEW active_customers AS
  SELECT * FROM customers WHERE deleted_at IS NULL;

-- Partial index for fast lookups on active rows only
CREATE INDEX idx_customers_active ON customers (id) WHERE deleted_at IS NULL;
```

### Audit Trail

```sql
-- Generic audit log table
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  table_name TEXT NOT NULL,
  record_id INT NOT NULL,
  action TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
  old_data JSONB,
  new_data JSONB,
  changed_by TEXT NOT NULL,
  changed_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Trigger function (Postgres)
CREATE OR REPLACE FUNCTION audit_trigger() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_by)
  VALUES (
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    TG_OP,
    CASE WHEN TG_OP != 'INSERT' THEN to_jsonb(OLD) END,
    CASE WHEN TG_OP != 'DELETE' THEN to_jsonb(NEW) END,
    current_user
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_customers_audit
  AFTER INSERT OR UPDATE OR DELETE ON customers
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### Polymorphic Associations and EAV Alternatives

```sql
-- BAD: polymorphic foreign key (no real FK constraint)
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  body TEXT,
  commentable_type TEXT,  -- 'post' or 'photo'
  commentable_id INT      -- FK to posts.id OR photos.id
);

-- BETTER: separate join tables
CREATE TABLE post_comments  (post_id INT REFERENCES posts(id), comment_id INT REFERENCES comments(id));
CREATE TABLE photo_comments (photo_id INT REFERENCES photos(id), comment_id INT REFERENCES comments(id));

-- EAV pattern (Entity-Attribute-Value) -- avoid when possible
CREATE TABLE entity_attrs (
  entity_id INT, attr_name TEXT, attr_value TEXT
);
-- Problems: no type safety, no constraints, terrible query performance
-- Alternative: use JSONB column for flexible attributes
ALTER TABLE products ADD COLUMN metadata JSONB DEFAULT '{}';
CREATE INDEX idx_products_metadata ON products USING gin (metadata);
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
```

---

## Partitioning and Scaling

### Range Partitioning

```sql
-- Postgres declarative partitioning (10+)
CREATE TABLE events (
  id BIGSERIAL,
  event_type TEXT NOT NULL,
  payload JSONB,
  created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_q1 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
CREATE TABLE events_2025_q2 PARTITION OF events
  FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');

-- The planner automatically prunes irrelevant partitions
EXPLAIN ANALYZE
SELECT * FROM events WHERE created_at >= '2025-02-01' AND created_at < '2025-03-01';
-- -> Seq Scan on events_2025_q1 (only this partition is scanned)
```

### List and Hash Partitioning

```sql
-- List partitioning: discrete values
CREATE TABLE orders (
  id BIGSERIAL, region TEXT, total DECIMAL(10,2)
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('EU', 'UK');
CREATE TABLE orders_ap PARTITION OF orders FOR VALUES IN ('JP', 'AU', 'SG');

-- Hash partitioning: even distribution when no natural range/list
CREATE TABLE sessions (
  id UUID, user_id INT, data JSONB
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_p0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_p1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_p2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_p3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Partition Pruning and Maintenance

```sql
-- Verify partition pruning is active
SET enable_partition_pruning = on;  -- default in Postgres 11+

-- Detach old partitions for archival (non-blocking in PG 14+)
ALTER TABLE events DETACH PARTITION events_2024_q1 CONCURRENTLY;

-- Attach a pre-loaded partition
ALTER TABLE events ATTACH PARTITION events_2025_q3
  FOR VALUES FROM ('2025-07-01') TO ('2025-10-01');
```

### Sharding Strategies

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| Application-level sharding | App chooses shard by key hash | Full control, no middleware | Complex app logic, cross-shard queries hard |
| Proxy-based (Vitess, Citus) | Middleware routes queries | Transparent to app | Added latency, operational overhead |
| Native (Citus on Postgres) | Distributes tables across nodes | SQL-compatible, auto-rebalance | Limited cross-shard join support |

### Read Replicas and Connection Pooling

```sql
-- Read replica routing (application level)
-- Write queries -> primary
-- Read queries  -> replica

-- Connection pooling with PgBouncer config
-- [databases]
-- mydb = host=primary port=5432 dbname=mydb
-- mydb_ro = host=replica port=5432 dbname=mydb
-- [pgbouncer]
-- pool_mode = transaction    -- best for web apps
-- max_client_conn = 1000
-- default_pool_size = 25
```

**Performance tip:** Partition pruning only works when the WHERE clause directly references the partition key. Wrapping it in a function (e.g., `WHERE EXTRACT(YEAR FROM created_at) = 2025`) defeats pruning.

---

## Dialect Quick Reference

Common operations and their syntax across PostgreSQL, MySQL, and SQLite.

| Operation | PostgreSQL | MySQL | SQLite |
|---|---|---|---|
| Auto-increment PK | `SERIAL` / `GENERATED ALWAYS AS IDENTITY` | `AUTO_INCREMENT` | `INTEGER PRIMARY KEY` (implicit rowid) |
| Upsert | `INSERT ... ON CONFLICT DO UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` | `INSERT OR REPLACE` / `ON CONFLICT` (3.24+) |
| String concat | `'a' \|\| 'b'` | `CONCAT('a','b')` | `'a' \|\| 'b'` |
| Current timestamp | `NOW()` / `CURRENT_TIMESTAMP` | `NOW()` / `CURRENT_TIMESTAMP` | `datetime('now')` |
| LIMIT with offset | `LIMIT 10 OFFSET 20` | `LIMIT 10 OFFSET 20` | `LIMIT 10 OFFSET 20` |
| BOOLEAN type | Native `BOOLEAN` | `TINYINT(1)` | `INTEGER` (0/1) |
| JSON support | `JSONB` (indexed, binary) | `JSON` (text-based) | `json_extract()` (3.38+) |
| Full-text search | `tsvector` + `tsquery` | `FULLTEXT` index | `FTS5` virtual table |
| Window functions | Full support (8.4+) | Full support (8.0+) | Full support (3.25+) |
| Recursive CTEs | `WITH RECURSIVE` (8.4+) | `WITH RECURSIVE` (8.0+) | `WITH RECURSIVE` (3.8.3+) |
| Lateral joins | `LATERAL` (9.3+) | `LATERAL` (8.0.14+) | Not supported |
| Partitioning | Declarative (10+) | Native `PARTITION BY` | Not supported |
| EXPLAIN format | `EXPLAIN (ANALYZE, BUFFERS)` | `EXPLAIN ANALYZE` (8.0.18+) | `EXPLAIN QUERY PLAN` |
| CTE materialization | `MATERIALIZED` / `NOT MATERIALIZED` (12+) | Always materialized if multi-ref | Always materialized |

### Upsert Examples by Dialect

```sql
-- PostgreSQL
INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;

-- MySQL
INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice')
ON DUPLICATE KEY UPDATE name = VALUES(name);

-- SQLite (3.24+)
INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = excluded.name;
```

### Pagination Patterns

```sql
-- OFFSET pagination (simple, but slow on deep pages)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 1000;
-- Scans and discards 1000 rows

-- Keyset/cursor pagination (fast regardless of depth)
-- Postgres / MySQL / SQLite
SELECT * FROM products
WHERE id > 1020  -- last seen id
ORDER BY id
LIMIT 20;
```

**Performance tip:** Keyset pagination is O(1) for any page depth. OFFSET pagination is O(N) because the database must scan and discard N rows before returning results.
