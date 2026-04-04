---
name: postgres-pro
description: PostgreSQL mastery: advanced indexing (GIN, GiST, BRIN), JSONB, full-text search, partitioning, replication, pgvector for embeddings, and performance tuning.
---

# Postgres Pro

Expert-level PostgreSQL skill covering advanced indexing strategies, JSONB document patterns, full-text search, table partitioning, replication and high availability, pgvector for AI embeddings, and deep performance tuning. Use this when designing schemas, optimizing slow queries, setting up replication, building search features, or integrating vector similarity search into a Postgres-backed application.

## Table of Contents

- [Advanced Indexing](#advanced-indexing)
- [JSONB](#jsonb)
- [Full-Text Search](#full-text-search)
- [Partitioning](#partitioning)
- [Replication and High Availability](#replication-and-high-availability)
- [pgvector and Embeddings](#pgvector-and-embeddings)
- [Performance Tuning](#performance-tuning)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## Advanced Indexing

PostgreSQL supports multiple index types. Choosing the right one depends on the data type, query pattern, and selectivity.

### B-tree Indexes (Default)

B-tree handles equality (`=`) and range (`<`, `>`, `BETWEEN`) queries, plus `ORDER BY`.

```sql
-- Simple B-tree on a frequently filtered column
CREATE INDEX idx_orders_created_at ON orders (created_at);

-- Composite index: equality columns first, then range, then sort
-- Query: WHERE status = 'active' AND created_at > '2025-01-01' ORDER BY total DESC
CREATE INDEX idx_orders_status_date_total
  ON orders (status, created_at, total DESC);

-- Multi-column index is usable for leftmost prefix queries:
-- WHERE status = 'active'                          -> uses index
-- WHERE status = 'active' AND created_at > '...'   -> uses index
-- WHERE created_at > '...'                          -> does NOT use index (no leading column)
```

### GIN Indexes (JSONB, Arrays, Full-Text)

GIN (Generalized Inverted Index) indexes individual elements within composite values.

```sql
-- JSONB containment queries
CREATE INDEX idx_products_metadata ON products USING gin (metadata);
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
SELECT * FROM products WHERE metadata ? 'warranty';

-- Array containment
CREATE INDEX idx_posts_tags ON posts USING gin (tags);
SELECT * FROM posts WHERE tags @> ARRAY['postgres', 'performance'];

-- Full-text search (see FTS section for details)
CREATE INDEX idx_articles_fts ON articles USING gin (to_tsvector('english', title || ' ' || body));

-- GIN operator classes for JSONB:
-- jsonb_ops (default): supports @>, ?, ?|, ?&
-- jsonb_path_ops: supports only @>, smaller index, faster for containment
CREATE INDEX idx_products_metadata_path ON products USING gin (metadata jsonb_path_ops);
```

### GiST Indexes (Geometry, Ranges, Nearest-Neighbor)

GiST (Generalized Search Tree) supports overlapping, containment, and nearest-neighbor queries.

```sql
-- Range types: find overlapping time ranges
CREATE TABLE reservations (
  id SERIAL PRIMARY KEY,
  room_id INT NOT NULL,
  during TSTZRANGE NOT NULL,
  EXCLUDE USING gist (room_id WITH =, during WITH &&)  -- prevent double-booking
);

CREATE INDEX idx_reservations_during ON reservations USING gist (during);
SELECT * FROM reservations WHERE during && '[2025-06-01, 2025-06-07)';

-- PostGIS geometry
CREATE INDEX idx_locations_geom ON locations USING gist (geom);
SELECT * FROM locations
WHERE ST_DWithin(geom, ST_MakePoint(-73.99, 40.73)::geography, 1000);  -- within 1km

-- Nearest-neighbor with ORDER BY <-> operator (KNN-GiST)
SELECT name, geom <-> ST_MakePoint(-73.99, 40.73)::geometry AS dist
FROM locations
ORDER BY geom <-> ST_MakePoint(-73.99, 40.73)::geometry
LIMIT 10;
```

### BRIN Indexes (Time-Series, Append-Only Data)

BRIN (Block Range INdex) stores min/max summaries per block range. Tiny index size for naturally ordered data.

```sql
-- Ideal for monotonically increasing columns like timestamps on append-only tables
CREATE INDEX idx_events_created_brin ON events USING brin (created_at)
  WITH (pages_per_range = 32);

-- BRIN is effective when physical row order correlates with column value order
-- Check correlation (1.0 = perfectly ordered, 0.0 = random)
SELECT correlation FROM pg_stats
WHERE tablename = 'events' AND attname = 'created_at';
-- If correlation < 0.9, BRIN will be ineffective -- use B-tree instead

-- BRIN size comparison on a 100M row table:
-- B-tree: ~2 GB
-- BRIN:   ~50 KB
```

### Partial Indexes

Index only the rows that matter -- smaller, faster, cheaper to maintain.

```sql
-- Index only active orders (skip 90% of historical rows)
CREATE INDEX idx_orders_active ON orders (created_at)
  WHERE status = 'active';

-- Queries must include the matching WHERE clause to use the partial index
SELECT * FROM orders WHERE status = 'active' AND created_at > '2025-01-01';

-- Unique constraint only on non-deleted rows
CREATE UNIQUE INDEX idx_users_email_active ON users (email)
  WHERE deleted_at IS NULL;
```

### Expression Indexes

Index a computed expression so the planner can use it.

```sql
CREATE INDEX idx_users_lower_email ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- Index on JSONB field extraction
CREATE INDEX idx_products_brand ON products ((metadata->>'brand'));
SELECT * FROM products WHERE metadata->>'brand' = 'Acme';

-- Date truncation for time-bucket queries
CREATE INDEX idx_events_day ON events ((created_at::date));
SELECT created_at::date AS day, COUNT(*) FROM events
WHERE created_at::date = '2025-06-15' GROUP BY 1;
```

### Covering Indexes (INCLUDE)

Add non-key columns to enable index-only scans without bloating the B-tree structure.

```sql
-- The INCLUDE columns are stored in the leaf pages but not in the tree structure
CREATE INDEX idx_orders_status_covering
  ON orders (status)
  INCLUDE (customer_id, total);

-- This query never touches the heap (table)
EXPLAIN ANALYZE
SELECT customer_id, total FROM orders WHERE status = 'shipped';
-- -> Index Only Scan using idx_orders_status_covering
```

### Finding Unused Indexes

```sql
-- Indexes consuming disk and slowing writes but never scanned
SELECT
  schemaname, tablename, indexrelname,
  idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE 'pg_%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Anti-patterns to avoid:**
- Creating indexes on every column "just in case" -- each index slows writes and consumes disk.
- Using BRIN on randomly ordered data -- check `pg_stats.correlation` first.
- Applying functions on indexed columns in WHERE clauses without a matching expression index.
- Forgetting that composite indexes only work for leftmost-prefix queries.

---

## JSONB

PostgreSQL JSONB stores binary-parsed JSON with full indexing support. It bridges relational and document models.

### Operators and Path Navigation

```sql
-- Sample table
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'
);

INSERT INTO products (name, metadata) VALUES
  ('Widget', '{"color": "red", "weight": 1.2, "tags": ["sale", "new"], "dimensions": {"h": 10, "w": 5}}'),
  ('Gadget', '{"color": "blue", "weight": 0.8, "tags": ["premium"], "specs": {"voltage": 220}}');

-- -> returns JSONB (preserves type)
SELECT metadata->'dimensions' FROM products WHERE name = 'Widget';
-- {"h": 10, "w": 5}

-- ->> returns TEXT (casts to string)
SELECT metadata->>'color' FROM products WHERE name = 'Widget';
-- red

-- #> path navigation (JSONB result)
SELECT metadata#>'{dimensions,h}' FROM products WHERE name = 'Widget';
-- 10

-- #>> path navigation (TEXT result)
SELECT metadata#>>'{dimensions,h}' FROM products WHERE name = 'Widget';
-- "10" (text)

-- @> containment: does the left JSONB contain the right?
SELECT * FROM products WHERE metadata @> '{"color": "red"}';

-- ? key existence
SELECT * FROM products WHERE metadata ? 'specs';

-- ?| any of these keys exist
SELECT * FROM products WHERE metadata ?| ARRAY['specs', 'warranty'];

-- ?& all of these keys exist
SELECT * FROM products WHERE metadata ?& ARRAY['color', 'weight'];
```

### Indexing JSONB

```sql
-- GIN index for all operators (@>, ?, ?|, ?&)
CREATE INDEX idx_products_metadata ON products USING gin (metadata);

-- GIN with jsonb_path_ops: smaller index, only supports @>
CREATE INDEX idx_products_metadata_path ON products USING gin (metadata jsonb_path_ops);

-- B-tree on a specific extracted field (fast equality/range on one key)
CREATE INDEX idx_products_color ON products ((metadata->>'color'));

-- Expression index for numeric comparison
CREATE INDEX idx_products_weight ON products (((metadata->>'weight')::numeric));
SELECT * FROM products WHERE (metadata->>'weight')::numeric < 1.0;
```

### jsonb_path_query (SQL/JSON Path)

SQL/JSON path expressions (Postgres 12+) provide a standard way to query JSONB.

```sql
-- Find products where any tag equals 'sale'
SELECT * FROM products
WHERE jsonb_path_exists(metadata, '$.tags[*] ? (@ == "sale")');

-- Extract all tags as a set
SELECT jsonb_path_query(metadata, '$.tags[*]') AS tag FROM products;

-- Filter with variables
SELECT * FROM products
WHERE jsonb_path_exists(
  metadata,
  '$.weight ? (@ < $max_weight)',
  '{"max_weight": 1.0}'
);

-- jsonb_path_query_array: returns a JSONB array instead of a set
SELECT jsonb_path_query_array(metadata, '$.tags[*]') FROM products;
```

### Generated Columns from JSONB

Materialize frequently accessed JSONB fields as real columns for indexing and constraints.

```sql
ALTER TABLE products
  ADD COLUMN color TEXT GENERATED ALWAYS AS (metadata->>'color') STORED;

-- Now you can index and query the generated column directly
CREATE INDEX idx_products_color_gen ON products (color);
SELECT * FROM products WHERE color = 'red';
-- Generated column stays in sync with metadata automatically
```

### JSONB Modification Functions

```sql
-- Set or update a key
UPDATE products SET metadata = jsonb_set(metadata, '{color}', '"green"') WHERE id = 1;

-- Set a nested key (creates intermediate objects)
UPDATE products SET metadata = jsonb_set(metadata, '{dimensions,depth}', '3') WHERE id = 1;

-- Remove a key
UPDATE products SET metadata = metadata - 'weight' WHERE id = 1;

-- Remove a nested key
UPDATE products SET metadata = metadata #- '{dimensions,h}' WHERE id = 1;

-- Concatenate / merge
UPDATE products SET metadata = metadata || '{"warranty": true}' WHERE id = 1;
```

### When to Use JSONB vs Normalized Columns

| Use JSONB | Use Normalized Columns |
|---|---|
| Schema varies per row (user preferences, metadata) | Schema is stable and well-known |
| Rarely queried individually, mostly stored/retrieved as a blob | Frequently filtered, joined, or aggregated |
| Rapid prototyping, schema not yet finalized | Data integrity via CHECK, FK, NOT NULL constraints |
| Sparse attributes (most rows have different fields) | Every row has the same fields |
| Third-party data you cannot control | Performance-critical analytical queries |

**Anti-patterns to avoid:**
- Storing everything as JSONB to avoid schema migrations -- you lose constraints, foreign keys, and type safety.
- Querying deeply nested JSONB paths without indexes -- every row must be decompressed and traversed.
- Using `->>'field'` in WHERE without a matching expression index or GIN index.
- Storing large arrays in JSONB that grow unboundedly -- updates rewrite the entire JSONB value.

---

## Full-Text Search

PostgreSQL full-text search (FTS) provides ranking, stemming, phrase matching, and highlighting without an external search engine.

### tsvector and tsquery Basics

```sql
-- tsvector: normalized, stemmed, position-aware token list
SELECT to_tsvector('english', 'The quick brown foxes jumped over lazy dogs');
-- 'brown':3 'dog':8 'fox':4 'jump':5 'lazi':7 'quick':2

-- tsquery: search expression
SELECT to_tsquery('english', 'quick & foxes');
-- 'quick' & 'fox'

-- Basic search: @@ is the match operator
SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'database & performance');
```

### Stored tsvector Column with GIN Index

Pre-computing the tsvector avoids re-parsing on every query.

```sql
-- Add a dedicated search column
ALTER TABLE articles ADD COLUMN search_vector tsvector;

-- Populate it with weighted fields (A = highest relevance, D = lowest)
UPDATE articles SET search_vector =
  setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
  setweight(to_tsvector('english', coalesce(body, '')), 'B');

-- GIN index for fast lookups
CREATE INDEX idx_articles_search ON articles USING gin (search_vector);

-- Keep it in sync with a trigger
CREATE OR REPLACE FUNCTION articles_search_trigger() RETURNS trigger AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(NEW.body, '')), 'B');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_articles_search
  BEFORE INSERT OR UPDATE OF title, body ON articles
  FOR EACH ROW EXECUTE FUNCTION articles_search_trigger();
```

### Ranking and Relevance

```sql
-- ts_rank: frequency-based relevance score
SELECT
  title,
  ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgres & replication') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- ts_rank_cd: cover density ranking (rewards proximity of terms)
SELECT
  title,
  ts_rank_cd(search_vector, query) AS rank_cd
FROM articles, to_tsquery('english', 'postgres & replication') AS query
WHERE search_vector @@ query
ORDER BY rank_cd DESC;

-- Boost title matches (weight A) over body matches (weight B)
SELECT title,
  ts_rank('{0.1, 0.2, 0.4, 1.0}', search_vector, query) AS weighted_rank
FROM articles, to_tsquery('english', 'index & performance') AS query
WHERE search_vector @@ query
ORDER BY weighted_rank DESC;
```

### Phrase Search and Proximity

```sql
-- Phrase search: words must appear in order, adjacent
SELECT * FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'connection pooling');

-- Proximity: words within N words of each other (Postgres 9.6+)
-- <-> means "followed by" (distance 1)
-- <2> means "within 2 words"
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'database <2> performance');
```

### Highlighting Results

```sql
SELECT
  ts_headline('english', body,
    to_tsquery('english', 'replication & failover'),
    'StartSel=<b>, StopSel=</b>, MaxWords=35, MinWords=15, MaxFragments=3'
  ) AS snippet
FROM articles
WHERE search_vector @@ to_tsquery('english', 'replication & failover');
-- Returns: ...streaming <b>replication</b> enables automatic <b>failover</b>...
```

### Custom Dictionaries and Configurations

```sql
-- List available configurations
SELECT cfgname FROM pg_ts_config;

-- Create a custom configuration that adds synonyms
CREATE TEXT SEARCH DICTIONARY synonyms (
  TEMPLATE = synonym,
  SYNONYMS = my_synonyms  -- file: $SHAREDIR/tsearch_data/my_synonyms.syn
);

CREATE TEXT SEARCH CONFIGURATION custom_english (COPY = english);
ALTER TEXT SEARCH CONFIGURATION custom_english
  ALTER MAPPING FOR asciiword WITH synonyms, english_stem;

-- Use the custom configuration
SELECT to_tsvector('custom_english', 'PostgreSQL is a RDBMS');
```

**Anti-patterns to avoid:**
- Calling `to_tsvector()` in WHERE without a GIN index -- forces a sequential scan with per-row parsing.
- Not using weights when title matches should rank higher than body matches.
- Building search queries with string concatenation instead of parameterized `to_tsquery` -- SQL injection risk.
- Using `LIKE '%term%'` for search instead of FTS -- no stemming, no ranking, no index use.

---

## Partitioning

Partitioning splits large tables into smaller physical pieces while presenting a single logical table. It improves query performance (partition pruning), maintenance (vacuum per partition), and data lifecycle management.

### Range Partitioning

Best for time-series data where queries filter by date.

```sql
CREATE TABLE events (
  id BIGSERIAL,
  event_type TEXT NOT NULL,
  payload JSONB,
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE events_2025_01 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE events_2025_02 PARTITION OF events
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
CREATE TABLE events_2025_03 PARTITION OF events
  FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

-- Default partition catches rows that do not match any range
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Indexes on partitioned tables are created on each partition automatically
CREATE INDEX idx_events_created ON events (created_at);
-- This creates idx_events_2025_01_created_at, idx_events_2025_02_created_at, etc.
```

### List Partitioning

Best for categorical data with a finite set of values.

```sql
CREATE TABLE orders (
  id BIGSERIAL,
  region TEXT NOT NULL,
  total DECIMAL(10,2),
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('DE', 'FR', 'UK', 'ES');
CREATE TABLE orders_apac PARTITION OF orders FOR VALUES IN ('JP', 'AU', 'SG', 'IN');
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

### Hash Partitioning

Distributes rows evenly when there is no natural range or list boundary.

```sql
CREATE TABLE sessions (
  id UUID NOT NULL,
  user_id INT NOT NULL,
  data JSONB,
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_p0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_p1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_p2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_p3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Partition Pruning

The planner skips irrelevant partitions when the query references the partition key.

```sql
-- Verify pruning is active
SET enable_partition_pruning = on;  -- default since Postgres 11

EXPLAIN ANALYZE
SELECT * FROM events WHERE created_at >= '2025-02-01' AND created_at < '2025-03-01';
-- Scans only events_2025_02, all other partitions are pruned
```

### Partition Maintenance

```sql
-- Detach old partition (non-blocking in Postgres 14+)
ALTER TABLE events DETACH PARTITION events_2024_01 CONCURRENTLY;

-- Archive detached partition
ALTER TABLE events_2024_01 SET TABLESPACE archive_tablespace;

-- Drop old data instantly (much faster than DELETE)
DROP TABLE events_2024_01;

-- Attach a pre-loaded partition
CREATE TABLE events_2025_07 (LIKE events INCLUDING ALL);
-- Load data into events_2025_07 ...
ALTER TABLE events ATTACH PARTITION events_2025_07
  FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');
```

### Automating with pg_partman

```sql
-- Install extension
CREATE EXTENSION pg_partman;

-- Let pg_partman manage monthly partitions
SELECT partman.create_parent(
  p_parent_table := 'public.events',
  p_control := 'created_at',
  p_type := 'native',
  p_interval := '1 month',
  p_premake := 3  -- create 3 future partitions
);

-- Run maintenance regularly (cron or pg_cron)
SELECT partman.run_maintenance();
```

### Migrating an Existing Table to Partitioned

```sql
-- 1. Rename the original table
ALTER TABLE events RENAME TO events_old;

-- 2. Create the new partitioned table with the same schema
CREATE TABLE events (LIKE events_old INCLUDING ALL) PARTITION BY RANGE (created_at);

-- 3. Create partitions
CREATE TABLE events_2025_01 PARTITION OF events FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
-- ... create all needed partitions ...

-- 4. Migrate data
INSERT INTO events SELECT * FROM events_old;

-- 5. Swap sequences if using SERIAL/BIGSERIAL
-- 6. Recreate foreign keys pointing to this table
-- 7. Drop the old table once verified
DROP TABLE events_old;
```

**Anti-patterns to avoid:**
- Wrapping the partition key in a function in WHERE -- defeats partition pruning.
- Creating too many partitions (thousands) -- the planner slows down planning.
- Forgetting a DEFAULT partition -- inserts with unexpected values fail.
- Not automating partition creation -- manual partition management leads to outages when the next partition is missing.

---

## Replication and High Availability

### Streaming Replication (Physical)

The primary streams WAL (Write-Ahead Log) to one or more standbys that replay it in real time.

```ini
# postgresql.conf on PRIMARY
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB            # retain WAL for slow replicas
synchronous_standby_names = '' # async by default
```

```ini
# postgresql.conf on STANDBY
hot_standby = on               # allow read queries on standby
primary_conninfo = 'host=primary-host port=5432 user=replicator password=secret'
```

```bash
# Bootstrap a standby from the primary
pg_basebackup -h primary-host -D /var/lib/postgresql/data -U replicator -Fp -Xs -P
```

```sql
-- Check replication status on PRIMARY
SELECT client_addr, state, sent_lsn, write_lsn, replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;

-- Check lag on STANDBY
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
  pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS lag_bytes;
```

### Synchronous Replication

Guarantees data is written to at least one standby before committing on the primary.

```ini
# postgresql.conf on PRIMARY
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
# Options: FIRST N (...) -- wait for N standbys
#          ANY N (...)   -- wait for any N of the listed standbys

synchronous_commit = on        # default; remote_write or remote_apply for stricter
```

Trade-off: synchronous replication adds commit latency (network round-trip) but guarantees zero data loss on failover.

### Logical Replication

Replicates specific tables (not the entire cluster) at the SQL level. Allows different schemas, versions, and indexes on subscriber.

```sql
-- On PUBLISHER (source)
CREATE PUBLICATION my_pub FOR TABLE orders, customers;
-- Or replicate all tables:
-- CREATE PUBLICATION my_pub FOR ALL TABLES;

-- On SUBSCRIBER (destination)
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=publisher-host dbname=mydb user=replicator'
  PUBLICATION my_pub;

-- Monitor subscription status
SELECT * FROM pg_stat_subscription;

-- Add a table to an existing publication
ALTER PUBLICATION my_pub ADD TABLE products;
-- Subscriber must refresh:
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;
```

### Patroni (Automated HA)

Patroni manages PostgreSQL HA with automatic failover using a distributed consensus store (etcd, Consul, ZooKeeper).

```yaml
# patroni.yml (simplified)
scope: my-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008

etcd:
  host: etcd1:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    postgresql:
      parameters:
        max_connections: 200
        shared_buffers: 4GB

postgresql:
  listen: 0.0.0.0:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: replicator
      password: secret
```

```bash
# Check cluster status
patronictl -c /etc/patroni.yml list
# +--------+---------+---------+----+-----------+
# | Member | Host    | Role    | TL | Lag in MB |
# +--------+---------+---------+----+-----------+
# | node1  | 10.0.1.1| Leader  |  5 |           |
# | node2  | 10.0.1.2| Replica |  5 |         0 |
# | node3  | 10.0.1.3| Replica |  5 |         0 |
# +--------+---------+---------+----+-----------+

# Manual switchover
patronictl -c /etc/patroni.yml switchover --master node1 --candidate node2
```

### pgBouncer (Connection Pooling)

PostgreSQL forks a process per connection. pgBouncer pools connections to handle thousands of clients with a small backend pool.

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction       # best for web apps (release conn after each txn)
max_client_conn = 5000        # max frontend connections
default_pool_size = 25        # backend connections per database/user pair
reserve_pool_size = 5         # extra backends under load
reserve_pool_timeout = 3
server_idle_timeout = 600
```

Pool modes:
| Mode | Behavior | Use Case |
|---|---|---|
| `session` | Connection held for entire client session | Legacy apps using session features (LISTEN, prepared statements) |
| `transaction` | Connection released after each transaction | Most web applications |
| `statement` | Connection released after each statement | Simple, auto-commit workloads only |

### Read Replica Routing

```sql
-- Application-level routing pattern (pseudo-code)
-- Primary: all writes (INSERT, UPDATE, DELETE) and critical reads
-- Replica: read-heavy analytics, reports, search queries

-- Verify a node is in recovery (= replica)
SELECT pg_is_in_recovery();  -- true on standby, false on primary
```

**Anti-patterns to avoid:**
- Running without connection pooling in production -- each connection uses ~10 MB of RAM.
- Using synchronous replication across regions -- latency will cripple write throughput.
- Not monitoring replication lag -- a lagging replica serves stale data.
- Failing to test your failover procedure -- an untested failover is not a failover plan.

---

## pgvector and Embeddings

pgvector adds vector similarity search to PostgreSQL, enabling AI/ML embedding storage and retrieval alongside relational data.

### Setup and Vector Column

```sql
-- Install the extension
CREATE EXTENSION vector;

-- Add a vector column (1536 dimensions = OpenAI text-embedding-3-small)
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  embedding vector(1536)
);

-- Insert a vector (from your embedding API)
INSERT INTO documents (title, content, embedding)
VALUES ('Postgres Guide', 'Full-text search...', '[0.1, 0.02, -0.03, ...]'::vector);
```

### Similarity Search Operators

```sql
-- L2 (Euclidean) distance: <-> (lower = more similar)
SELECT title, embedding <-> '[0.1, 0.02, ...]'::vector AS distance
FROM documents
ORDER BY embedding <-> '[0.1, 0.02, ...]'::vector
LIMIT 10;

-- Cosine distance: <=> (lower = more similar; 0 = identical)
SELECT title, embedding <=> '[0.1, 0.02, ...]'::vector AS cosine_dist
FROM documents
ORDER BY embedding <=> '[0.1, 0.02, ...]'::vector
LIMIT 10;

-- Inner product (negative): <#> (lower = higher similarity)
SELECT title, (embedding <#> '[0.1, 0.02, ...]'::vector) * -1 AS similarity
FROM documents
ORDER BY embedding <#> '[0.1, 0.02, ...]'::vector
LIMIT 10;

-- Cosine distance is the most common choice for text embeddings
```

### HNSW vs IVFFlat Indexes

Two approximate nearest-neighbor (ANN) index types with different trade-offs.

```sql
-- HNSW (Hierarchical Navigable Small World) -- recommended for most use cases
-- Better recall, faster queries, but slower to build and more memory
CREATE INDEX idx_docs_embedding_hnsw ON documents
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Tune search accuracy vs speed at query time
SET hnsw.ef_search = 100;  -- higher = better recall, slower (default: 40)

-- IVFFlat (Inverted File with Flat compression)
-- Faster to build, less memory, but requires training on existing data
CREATE INDEX idx_docs_embedding_ivfflat ON documents
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);  -- sqrt(n) is a good starting point for lists

-- Tune probe count at query time
SET ivfflat.probes = 10;  -- higher = better recall, slower (default: 1)
```

| Feature | HNSW | IVFFlat |
|---|---|---|
| Build speed | Slower | Faster |
| Query speed | Faster | Slower |
| Recall at same speed | Higher | Lower |
| Memory usage | Higher | Lower |
| Needs training data | No | Yes (build after data is loaded) |
| Recommended for | Most workloads | Large datasets with memory constraints |

### Operator Classes for Different Distance Functions

```sql
-- Choose the operator class matching your distance function
-- Cosine distance (<=>)
CREATE INDEX ... USING hnsw (embedding vector_cosine_ops);

-- L2 distance (<->)
CREATE INDEX ... USING hnsw (embedding vector_l2_ops);

-- Inner product (<#>)
CREATE INDEX ... USING hnsw (embedding vector_ip_ops);
```

### Hybrid Search (Vector + Full-Text)

Combine semantic similarity (vector) with keyword matching (FTS) for better search quality.

```sql
-- Add FTS column alongside vector column
ALTER TABLE documents ADD COLUMN search_vector tsvector;
UPDATE documents SET search_vector =
  setweight(to_tsvector('english', title), 'A') ||
  setweight(to_tsvector('english', content), 'B');
CREATE INDEX idx_docs_fts ON documents USING gin (search_vector);

-- Reciprocal Rank Fusion (RRF): combine two ranked lists
WITH semantic AS (
  SELECT id, ROW_NUMBER() OVER (ORDER BY embedding <=> $1::vector) AS rank_sem
  FROM documents
  ORDER BY embedding <=> $1::vector
  LIMIT 50
),
keyword AS (
  SELECT id, ROW_NUMBER() OVER (ORDER BY ts_rank_cd(search_vector, query) DESC) AS rank_kw
  FROM documents, to_tsquery('english', $2) AS query
  WHERE search_vector @@ query
  LIMIT 50
)
SELECT
  d.id, d.title,
  COALESCE(1.0 / (60 + s.rank_sem), 0) +
  COALESCE(1.0 / (60 + k.rank_kw), 0) AS rrf_score
FROM documents d
LEFT JOIN semantic s ON d.id = s.id
LEFT JOIN keyword k ON d.id = k.id
WHERE s.id IS NOT NULL OR k.id IS NOT NULL
ORDER BY rrf_score DESC
LIMIT 20;
```

### Indexing Strategies for Large Datasets

```sql
-- For millions of vectors, build the index AFTER loading data
-- (especially IVFFlat which needs training data)

-- 1. Load data without index
\COPY documents (title, content, embedding) FROM 'embeddings.csv' WITH CSV;

-- 2. Build index with higher maintenance_work_mem for faster builds
SET maintenance_work_mem = '2GB';
CREATE INDEX idx_docs_embedding ON documents
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 100);

-- 3. For very large datasets (100M+), consider partitioning by a category
--    and building separate HNSW indexes per partition
CREATE TABLE documents_partitioned (
  id BIGSERIAL,
  category TEXT NOT NULL,
  embedding vector(1536)
) PARTITION BY LIST (category);

-- Each partition gets its own HNSW index, keeping index size manageable
```

**Anti-patterns to avoid:**
- Using exact nearest-neighbor search (no index) on large tables -- it scans every row.
- Building an IVFFlat index on an empty table -- it needs representative data for clustering.
- Setting `hnsw.ef_search` or `ivfflat.probes` too low -- fast but returns poor results.
- Storing very high-dimensional vectors (>2000) without dimensionality reduction -- index performance degrades.

---

## Performance Tuning

### Reading EXPLAIN ANALYZE

```sql
-- Always use ANALYZE (actually runs the query) and BUFFERS (shows I/O)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.name, o.total
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2025-01-01'
  AND c.region = 'US'
ORDER BY o.total DESC
LIMIT 20;

-- Sample output:
-- Limit  (cost=156.23..156.28 rows=20 width=52)
--        (actual time=2.145..2.152 rows=20 loops=1)
--   ->  Sort  (cost=156.23..157.48 rows=500 width=52)
--              (actual time=2.143..2.148 rows=20 loops=1)
--         Sort Key: o.total DESC
--         Sort Method: top-N heapsort  Memory: 27kB
--         ->  Hash Join  (cost=25.00..141.25 rows=500 width=52)
--                         (actual time=0.312..1.845 rows=487 loops=1)
--               Hash Cond: (o.customer_id = c.id)
--               ->  Index Scan using idx_orders_created on orders o
--                     (cost=0.42..95.18 rows=2500 width=20)
--                     (actual time=0.025..0.892 rows=2500 loops=1)
--                     Index Cond: (created_at > '2025-01-01')
--                     Buffers: shared hit=45
--               ->  Hash  (cost=20.00..20.00 rows=200 width=36)
--                          (actual time=0.265..0.266 rows=198 loops=1)
--                     Buckets: 1024  Batches: 1  Memory Usage: 18kB
--                     ->  Seq Scan on customers c
--                           (cost=0.00..20.00 rows=200 width=36)
--                           (actual time=0.008..0.198 rows=198 loops=1)
--                           Filter: (region = 'US')
--                           Rows Removed by Filter: 802
--                           Buffers: shared hit=12
-- Planning Time: 0.245 ms
-- Execution Time: 2.198 ms
```

**Key EXPLAIN fields to check:**

| Field | What to Look For |
|---|---|
| `actual time` vs `cost` | Large gap means bad estimate -- run `ANALYZE` |
| `rows` (estimated vs actual) | Off by 10x+ means stale statistics |
| `Rows Removed by Filter` | High value = missing index on filter column |
| `Buffers: shared hit` vs `read` | `read` = disk I/O, `hit` = cache |
| `Sort Method: external merge` | Not enough `work_mem`, spilling to disk |
| `Seq Scan` on large table | May need an index (unless selecting most rows) |
| `loops` | Nested loop with high loops count = potential N+1 problem |

### postgresql.conf Tuning

```ini
# Memory
shared_buffers = '4GB'            # 25% of total RAM (start here)
effective_cache_size = '12GB'     # 75% of total RAM (tells planner how much OS cache to expect)
work_mem = '64MB'                 # per-sort/hash operation; be careful with high connection counts
                                  # total memory = work_mem * max_connections * operations_per_query
maintenance_work_mem = '1GB'      # for VACUUM, CREATE INDEX, ALTER TABLE
huge_pages = try                  # reduce TLB misses on Linux

# WAL and Checkpoints
wal_buffers = '64MB'              # auto-tuned from shared_buffers, but set explicitly for large installs
checkpoint_completion_target = 0.9 # spread checkpoint writes over 90% of the interval
max_wal_size = '4GB'              # allow more WAL before forcing a checkpoint

# Query Planner
random_page_cost = 1.1            # SSD storage (default 4.0 is for spinning disks)
effective_io_concurrency = 200    # SSD: 200, HDD: 2
default_statistics_target = 100   # increase to 500+ for columns with skewed distributions

# Parallelism
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
parallel_tuple_cost = 0.01
```

### VACUUM and Autovacuum

VACUUM reclaims dead tuples left by UPDATE/DELETE (MVCC). Without it, tables bloat and queries slow down.

```sql
-- Manual vacuum (reclaims space, does not lock table)
VACUUM VERBOSE orders;

-- Full vacuum (rewrites table, locks exclusively -- use rarely)
VACUUM FULL orders;

-- Check table bloat
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  ROUND(n_dead_tup::numeric / GREATEST(n_live_tup, 1) * 100, 1) AS dead_pct,
  last_vacuum,
  last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

Autovacuum tuning for high-write tables:

```sql
-- Per-table autovacuum settings for a hot table
ALTER TABLE events SET (
  autovacuum_vacuum_scale_factor = 0.01,        -- trigger at 1% dead tuples (default 20%)
  autovacuum_vacuum_cost_delay = 2,              -- less aggressive throttling (default 2ms)
  autovacuum_vacuum_cost_limit = 1000            -- higher budget per round (default 200)
);

-- Global autovacuum settings (postgresql.conf)
-- autovacuum_max_workers = 5                    -- more parallel vacuum workers
-- autovacuum_naptime = 15s                      -- check more frequently (default 1min)
```

### pg_stat_statements

The single most important extension for query performance analysis.

```sql
-- Enable in postgresql.conf
-- shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION pg_stat_statements;

-- Top 10 queries by total time
SELECT
  LEFT(query, 80) AS query_preview,
  calls,
  ROUND(total_exec_time::numeric, 1) AS total_ms,
  ROUND(mean_exec_time::numeric, 1) AS mean_ms,
  ROUND((100 * total_exec_time / SUM(total_exec_time) OVER ())::numeric, 1) AS pct_total,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top queries by mean time (find individually slow queries)
SELECT LEFT(query, 80), calls,
  ROUND(mean_exec_time::numeric, 1) AS mean_ms,
  ROUND(stddev_exec_time::numeric, 1) AS stddev_ms
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Reset statistics after optimization to measure improvement
SELECT pg_stat_statements_reset();
```

### Query Optimization Patterns

```sql
-- 1. Replace correlated subquery with JOIN
-- SLOW: subquery runs once per outer row
SELECT o.id, (SELECT name FROM customers c WHERE c.id = o.customer_id) AS name
FROM orders o;

-- FAST: single hash join
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id;

-- 2. Use EXISTS instead of COUNT for existence checks
-- SLOW: counts all matching rows
SELECT * FROM customers c WHERE (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) > 0;

-- FAST: stops at first match
SELECT * FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- 3. Batch IN queries instead of N+1 lookups
-- SLOW: one query per ID from application code
-- for id in customer_ids: SELECT * FROM customers WHERE id = $1

-- FAST: single query with ANY
SELECT * FROM customers WHERE id = ANY($1::int[]);  -- pass array of IDs

-- 4. Use COPY for bulk loads (10-100x faster than INSERT)
\COPY orders FROM 'orders.csv' WITH (FORMAT csv, HEADER true);

-- 5. Avoid SELECT DISTINCT when GROUP BY achieves the same result
-- DISTINCT sorts the entire result set; GROUP BY with aggregation is usually more efficient
```

### Lock Monitoring

```sql
-- Find blocking queries
SELECT
  blocked_locks.pid AS blocked_pid,
  blocked_activity.query AS blocked_query,
  blocking_locks.pid AS blocking_pid,
  blocking_activity.query AS blocking_query,
  blocked_activity.wait_event_type
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Kill a blocking query (use with caution)
SELECT pg_terminate_backend(blocking_pid);
```

**Anti-patterns to avoid:**
- Tuning postgresql.conf without measuring -- always benchmark before and after.
- Setting `work_mem` too high globally -- multiply by `max_connections` to estimate worst case.
- Disabling autovacuum to "reduce overhead" -- table bloat will eventually cripple performance.
- Ignoring `pg_stat_statements` -- you cannot optimize what you do not measure.
- Running `VACUUM FULL` routinely -- it locks the table exclusively; use regular `VACUUM` or `pg_repack` for online table compaction.

---

## Common Patterns Quick Reference

| Pattern | When to Use | Key Detail |
|---|---|---|
| B-tree index | Equality, range, ORDER BY on scalar columns | Default; use composite index with equality cols first |
| GIN index | JSONB `@>`, arrays, full-text search | Use `jsonb_path_ops` for containment-only queries (smaller) |
| GiST index | Geometry, ranges, nearest-neighbor | Required for exclusion constraints and KNN `<->` queries |
| BRIN index | Time-series, append-only, high correlation | Check `pg_stats.correlation > 0.9` before using |
| Partial index | Filter on a fixed condition (e.g. `WHERE active`) | Query WHERE must match index WHERE exactly |
| Covering index (INCLUDE) | Enable index-only scans for specific queries | Non-key columns in leaf pages only |
| Expression index | Queries with functions on columns | Must match the expression exactly |
| JSONB `@>` with GIN | Flexible metadata filtering | Consider generated columns for hot paths |
| tsvector + GIN | Full-text search with ranking | Store pre-computed tsvector column with trigger |
| Range partitioning | Time-series tables > 100M rows | Automate with pg_partman; always add DEFAULT partition |
| List partitioning | Categorical separation (region, tenant) | Good for multi-tenant row-level isolation |
| Streaming replication | High availability, read replicas | Physical; entire cluster replicated |
| Logical replication | Selective table replication, cross-version | Schema can differ between publisher and subscriber |
| pgBouncer `transaction` mode | Web apps with many short-lived connections | Do not use session features (LISTEN, prepared statements) |
| Patroni | Automated failover with consensus | Requires etcd/Consul/ZooKeeper |
| pgvector HNSW | Semantic search, RAG, recommendations | Best recall/speed trade-off; set `ef_search` at query time |
| pgvector + FTS (RRF) | Hybrid search combining keywords and semantics | Reciprocal Rank Fusion merges two ranked lists |
| pg_stat_statements | Find slow and frequent queries | Reset after optimization to measure impact |
| Autovacuum tuning | High-write tables with bloat | Lower `scale_factor` on hot tables |
| EXPLAIN (ANALYZE, BUFFERS) | Diagnose any slow query | Compare estimated vs actual rows; check for Seq Scans |
| `COPY` for bulk loads | Loading CSVs, migrations, ETL | 10-100x faster than batched INSERTs |
