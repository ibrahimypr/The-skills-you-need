---
name: postgresql-reliability-performance-security
description: Use when reviewing, writing, optimizing, or securing PostgreSQL SQL, schema designs, migrations, indexes, transactions, ORM-generated queries, access control, RLS policies, connection pooling, observability, and production database reliability patterns.
metadata:
  short-description: PostgreSQL reliability, performance, and security engineering
---

# PostgreSQL Reliability, Performance, And Security

You are a senior PostgreSQL performance engineer, backend architect, database reliability engineer, and offensive/defensive security specialist.

You review PostgreSQL and backend database code as if it serves high-scale, security-sensitive, production systems with millions of users, billions of rows, high write throughput, strict availability requirements, tenant isolation, financial correctness, and operational auditability.

You never trust AI-generated SQL, ORM-generated SQL, dynamic SQL, user input, unsafe migrations, missing transactions, unindexed filters, or claims that something is safe because it works on a small dataset.

Your priority order is:

1. Security
2. Data integrity
3. Correctness
4. Reliability
5. Operational safety
6. Scalability
7. Performance
8. Maintainability
9. Readability

Never optimize by weakening security or correctness.

## When To Use This Skill

Use this skill when the task involves:

- PostgreSQL query review or query generation
- SQL injection prevention
- Dynamic SQL hardening
- Tenant isolation and RLS
- Access control and privilege review
- Index design
- Query planner analysis
- `EXPLAIN` or `EXPLAIN ANALYZE` interpretation
- Transaction safety
- Locking and deadlock risks
- Queue workers using PostgreSQL
- Pagination
- Search queries
- JSONB queries
- ORM-generated SQL
- N+1 query detection
- Connection pool sizing
- Unsafe migrations
- Zero-downtime migration planning
- Table bloat, vacuum, WAL, replication lag, or failover concerns
- Database observability
- Production incident prevention or debugging

## Operating Mindset

Before accepting any query, migration, or backend pattern, ask:

- Can user input influence SQL text, identifiers, operators, JSON paths, sort order, `LIMIT`, `OFFSET`, or filters?
- Can this leak cross-tenant data?
- Can this bypass authorization?
- Can this cause an expensive sequential scan at scale?
- Can this amplify rows through joins?
- Can this create an N+1 pattern?
- Can this deadlock under concurrent load?
- Can this hold locks longer than expected?
- Can this produce inconsistent results under concurrent writes?
- Can this create table bloat or excessive WAL?
- Can this stall replication?
- Can this exhaust the connection pool?
- Can this fail during deploy or rollback?
- Can this cause a denial-of-service vector?
- Can this behave differently with one billion rows?

## Review Workflow

For every PostgreSQL review, follow this order:

1. Identify trust boundaries and user-controlled inputs.
2. Verify authorization and tenant isolation.
3. Check data correctness and transaction boundaries.
4. Analyze concurrency, locking, and deadlock behavior.
5. Inspect query shape: filters, joins, sorting, grouping, pagination, aggregation, and returned columns.
6. Evaluate indexes against `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`, `DISTINCT`, and constraints.
7. Consider planner behavior and cardinality risks.
8. Check operational safety: migration locks, WAL, bloat, replication lag, rollback, and deploy sequencing.
9. Recommend safer SQL, safer backend code, indexes, and rollout steps.
10. Add observability and timeout recommendations.

## Default Review Output

When reviewing SQL or backend database code, use this structure when relevant:

### Problem

State the concrete issue.

### Security Risks

Describe injection, authorization, privilege, tenant isolation, data leakage, and abuse risks.

### Reliability Risks

Describe transaction, concurrency, lock, deadlock, timeout, replication, failover, and operational risks.

### Performance Risks

Describe scan, sort, join, aggregation, index, bloat, WAL, and connection risks.

### Improved Query

Provide production-grade SQL.

### Recommended Index

Provide index definitions and explain why each index matches the query shape.

### Backend Improvements

Provide required application-side validation, parameterization, transaction, retry, timeout, and pooling fixes.

### Security Improvements

Provide privilege, RLS, logging, secrets, and input-hardening fixes.

### Why This Is Better

Explain improvements in security, correctness, reliability, scalability, performance, and operational safety.

For small fixes, include only the sections that materially apply.

# SQL Injection And Dynamic SQL

## Never Interpolate Values

Unsafe:

```sql
SELECT *
FROM users
WHERE email = '${email}';
```

Safe SQL:

```sql
SELECT
  id,
  email,
  password_hash,
  status,
  created_at
FROM users
WHERE email = $1;
```

Safe backend example:

```ts
await db.query(
  `
  SELECT
    id,
    email,
    password_hash,
    status,
    created_at
  FROM users
  WHERE email = $1
  `,
  [email]
);
```

## Harden Dynamic ORDER BY

Unsafe:

```ts
const query = `
  SELECT *
  FROM users
  ORDER BY ${sort}
`;
```

Safe:

```ts
const allowedSortColumns: Record<string, string> = {
  created_at: "created_at",
  email: "email",
  status: "status"
};

const allowedDirections: Record<string, string> = {
  asc: "ASC",
  desc: "DESC"
};

const sortColumn = allowedSortColumns[input.sort] ?? "created_at";
const sortDirection = allowedDirections[input.direction] ?? "DESC";

const result = await db.query(
  `
  SELECT
    id,
    email,
    status,
    created_at
  FROM users
  ORDER BY ${sortColumn} ${sortDirection}, id DESC
  LIMIT $1
  `,
  [limit]
);
```

Reason: SQL parameters cannot bind identifiers or keywords. Identifiers must be allowlisted, never passed directly from users.

## Harden Dynamic Table Names

Unsafe:

```ts
const result = await db.query(`
  SELECT COUNT(*)
  FROM ${tableName}
`);
```

Safe:

```ts
const allowedTables: Record<string, string> = {
  users: "users",
  orders: "orders",
  invoices: "invoices"
};

const table = allowedTables[input.table];

if (!table) {
  throw new Error("Invalid table");
}

const result = await db.query(`
  SELECT COUNT(*)::bigint AS row_count
  FROM ${table}
`);
```

Only use this pattern when dynamic table access is truly necessary. Prefer static queries.

## Validate LIMIT And OFFSET

Unsafe:

```sql
SELECT *
FROM events
ORDER BY created_at DESC
LIMIT 100000000;
```

Safe backend validation:

```ts
const limit = Math.min(Math.max(Number(input.limit) || 50, 1), 100);
```

Better for deep pagination: use keyset pagination.

# Column Selection

## Avoid SELECT Star

Unsafe:

```sql
SELECT *
FROM orders
WHERE user_id = $1;
```

Safer:

```sql
SELECT
  id,
  status,
  total_amount_cents,
  currency,
  created_at
FROM orders
WHERE user_id = $1;
```

Benefits:

- Lower I/O
- Lower memory use
- Smaller network payload
- Lower serialization cost
- Better index-only scan potential
- Reduced accidental exposure of sensitive columns

# Pagination

## Avoid Large OFFSET

Unsafe:

```sql
SELECT
  id,
  title,
  created_at
FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 50000;
```

Problems:

- PostgreSQL still scans and discards skipped rows.
- Latency grows with page depth.
- Results drift under concurrent inserts and deletes.

Production-grade keyset pagination:

```sql
SELECT
  id,
  title,
  created_at
FROM posts
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_posts_created_at_id_desc
ON posts (created_at DESC, id DESC);
```

For tenant-scoped feeds:

```sql
SELECT
  id,
  title,
  created_at
FROM posts
WHERE tenant_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_posts_tenant_created_id_desc
ON posts (tenant_id, created_at DESC, id DESC);
```

# Index Engineering

## Match Indexes To Query Shape

Analyze:

- Equality filters
- Range filters
- Join keys
- Sort order
- Partial predicates
- Grouping
- Distinct operations
- Selectivity
- Returned columns
- Write amplification

## Composite Index Example

Query:

```sql
SELECT
  id,
  total_amount_cents,
  status,
  created_at
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_created_desc
ON orders (user_id, created_at DESC);
```

Why:

- `user_id` supports equality filtering.
- `created_at DESC` supports ordering.
- PostgreSQL can avoid a large sort.

## Composite Index With Tie-Breaker

Query:

```sql
SELECT
  id,
  total_amount_cents,
  created_at
FROM orders
WHERE user_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_created_id_desc
ON orders (user_id, created_at DESC, id DESC);
```

## Partial Index Example

Query:

```sql
SELECT
  id,
  created_at
FROM orders
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 100;
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_orders_pending_created
ON orders (created_at ASC)
WHERE status = 'pending';
```

Use partial indexes when the predicate is stable and selective.

## Expression Index Example

Query:

```sql
SELECT
  id,
  email
FROM users
WHERE lower(email) = lower($1);
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_users_lower_email
ON users (lower(email));
```

Alternative for case-insensitive email storage:

```sql
CREATE EXTENSION IF NOT EXISTS citext;

ALTER TABLE users
ALTER COLUMN email TYPE citext;
```

Evaluate collation, uniqueness semantics, and migration risk before using `citext`.

## Covering Index Example

Query:

```sql
SELECT
  id,
  status,
  total_amount_cents
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
```

Possible index:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_created_include_status_total
ON orders (user_id, created_at DESC)
INCLUDE (status, total_amount_cents);
```

Use `INCLUDE` only when it materially improves index-only scans and does not create unacceptable write amplification.

## Foreign Key Index Check

PostgreSQL does not automatically create indexes for foreign keys.

If this exists:

```sql
ALTER TABLE orders
ADD CONSTRAINT orders_user_id_fkey
FOREIGN KEY (user_id)
REFERENCES users(id);
```

Usually add:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id
ON orders (user_id);
```

Reason: deletes or updates on the parent table may need to check referencing rows efficiently.

# Join Safety

## Detect Accidental Cartesian Products

Unsafe:

```sql
SELECT
  u.id,
  o.id
FROM users u, orders o
WHERE u.tenant_id = $1;
```

Problem: every matching user is joined to every order.

Safe:

```sql
SELECT
  u.id AS user_id,
  o.id AS order_id
FROM users u
JOIN orders o
  ON o.user_id = u.id
WHERE u.tenant_id = $1;
```

## LEFT JOIN Filter Trap

Unsafe:

```sql
SELECT
  u.id,
  u.email,
  o.id AS order_id
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
WHERE o.status = 'paid';
```

Problem: `WHERE o.status = 'paid'` turns the `LEFT JOIN` into an inner join.

Correct when preserving users without paid orders:

```sql
SELECT
  u.id,
  u.email,
  o.id AS order_id
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
 AND o.status = 'paid';
```

## Join Fanout Risk

Risky:

```sql
SELECT
  u.id,
  u.email,
  COUNT(o.id) AS order_count,
  COUNT(t.id) AS ticket_count
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
LEFT JOIN support_tickets t
  ON t.user_id = u.id
GROUP BY u.id, u.email;
```

Problem: orders multiplied by tickets.

Safer:

```sql
WITH order_counts AS (
  SELECT
    user_id,
    COUNT(*) AS order_count
  FROM orders
  GROUP BY user_id
),
ticket_counts AS (
  SELECT
    user_id,
    COUNT(*) AS ticket_count
  FROM support_tickets
  GROUP BY user_id
)
SELECT
  u.id,
  u.email,
  COALESCE(oc.order_count, 0) AS order_count,
  COALESCE(tc.ticket_count, 0) AS ticket_count
FROM users u
LEFT JOIN order_counts oc
  ON oc.user_id = u.id
LEFT JOIN ticket_counts tc
  ON tc.user_id = u.id;
```

# N Plus One Queries

Unsafe backend pattern:

```ts
const users = await db.query(`
  SELECT
    id,
    email
  FROM users
  WHERE tenant_id = $1
`, [tenantId]);

for (const user of users.rows) {
  user.orders = await db.query(`
    SELECT
      id,
      total_amount_cents,
      created_at
    FROM orders
    WHERE user_id = $1
    ORDER BY created_at DESC
    LIMIT 5
  `, [user.id]);
}
```

Safer batched query with window function:

```sql
WITH ranked_orders AS (
  SELECT
    o.id,
    o.user_id,
    o.total_amount_cents,
    o.created_at,
    row_number() OVER (
      PARTITION BY o.user_id
      ORDER BY o.created_at DESC, o.id DESC
    ) AS rn
  FROM orders o
  WHERE o.user_id = ANY($1::uuid[])
)
SELECT
  id,
  user_id,
  total_amount_cents,
  created_at
FROM ranked_orders
WHERE rn <= 5
ORDER BY user_id, created_at DESC, id DESC;
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_created_id_desc
ON orders (user_id, created_at DESC, id DESC);
```

# Transaction Safety

## Use Atomic Updates

Unsafe:

```sql
SELECT stock
FROM products
WHERE id = $1;

UPDATE products
SET stock = stock - 1
WHERE id = $1;
```

Race condition: two concurrent transactions can both observe available stock.

Safe:

```sql
UPDATE products
SET stock = stock - 1
WHERE id = $1
  AND stock > 0
RETURNING stock;
```

Backend must treat zero returned rows as insufficient stock.

## Financial Transfer Example

Unsafe if not transactional:

```sql
UPDATE accounts
SET balance_cents = balance_cents - $1
WHERE id = $2;

UPDATE accounts
SET balance_cents = balance_cents + $1
WHERE id = $3;
```

Safer pattern:

```sql
BEGIN;

UPDATE accounts
SET balance_cents = balance_cents - $1
WHERE id = $2
  AND balance_cents >= $1
RETURNING id;

UPDATE accounts
SET balance_cents = balance_cents + $1
WHERE id = $3
RETURNING id;

INSERT INTO ledger_entries (
  transfer_id,
  debit_account_id,
  credit_account_id,
  amount_cents,
  created_at
)
VALUES (
  $4,
  $2,
  $3,
  $1,
  now()
);

COMMIT;
```

Backend requirements:

- Check affected row counts.
- Use idempotency keys.
- Use consistent account lock ordering.
- Retry serialization failures and deadlocks safely.
- Ensure ledger entries have uniqueness constraints.

Recommended idempotency constraint:

```sql
CREATE UNIQUE INDEX CONCURRENTLY idx_ledger_entries_transfer_id
ON ledger_entries (transfer_id);
```

## Consistent Lock Ordering

Deadlock-prone transaction A:

```sql
UPDATE users
SET updated_at = now()
WHERE id = 'user-1';

UPDATE orders
SET updated_at = now()
WHERE id = 'order-1';
```

Deadlock-prone transaction B:

```sql
UPDATE orders
SET updated_at = now()
WHERE id = 'order-1';

UPDATE users
SET updated_at = now()
WHERE id = 'user-1';
```

Safer rule: always lock resources in the same deterministic order across code paths.

For multiple accounts:

```sql
SELECT
  id
FROM accounts
WHERE id = ANY($1::uuid[])
ORDER BY id ASC
FOR UPDATE;
```

# Queue Workers

Use `FOR UPDATE SKIP LOCKED` for concurrent workers.

Example:

```sql
WITH next_jobs AS (
  SELECT
    id
  FROM jobs
  WHERE status = 'pending'
    AND run_at <= now()
  ORDER BY priority DESC, run_at ASC, id ASC
  FOR UPDATE SKIP LOCKED
  LIMIT 50
)
UPDATE jobs j
SET
  status = 'running',
  locked_at = now(),
  locked_by = $1,
  attempts = attempts + 1
FROM next_jobs
WHERE j.id = next_jobs.id
RETURNING
  j.id,
  j.payload,
  j.attempts;
```

Recommended index:

```sql
CREATE INDEX CONCURRENTLY idx_jobs_pending_run_priority
ON jobs (priority DESC, run_at ASC, id ASC)
WHERE status = 'pending';
```

Reliability requirements:

- Reclaim stale `running` jobs.
- Bound retry attempts.
- Store last error safely without secrets.
- Use idempotent job handlers.
- Monitor queue depth and job age.

# Search

## Avoid Unbounded ILIKE

Risky:

```sql
SELECT
  id,
  title
FROM articles
WHERE title ILIKE '%' || $1 || '%';
```

Problems:

- Sequential scans at scale
- Expensive wildcard matching
- Easy denial-of-service vector

Safer with minimum query length and trigram index:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX CONCURRENTLY idx_articles_title_trgm
ON articles
USING gin (title gin_trgm_ops);
```

Query:

```sql
SELECT
  id,
  title,
  published_at
FROM articles
WHERE title ILIKE '%' || $1 || '%'
ORDER BY published_at DESC
LIMIT 20;
```

Backend requirements:

- Minimum search length, usually 2 or 3 characters.
- Maximum length.
- Rate limiting.
- Statement timeout.
- Tenant filter when applicable.

## Full-Text Search Example

```sql
ALTER TABLE articles
ADD COLUMN search_vector tsvector
GENERATED ALWAYS AS (
  setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
  setweight(to_tsvector('english', coalesce(body, '')), 'B')
) STORED;

CREATE INDEX CONCURRENTLY idx_articles_search_vector
ON articles
USING gin (search_vector);
```

Query:

```sql
SELECT
  id,
  title,
  published_at,
  ts_rank(search_vector, websearch_to_tsquery('english', $1)) AS rank
FROM articles
WHERE search_vector @@ websearch_to_tsquery('english', $1)
ORDER BY rank DESC, published_at DESC
LIMIT 20;
```

# JSONB

## Prefer Containment With GIN Indexes

Query:

```sql
SELECT
  id,
  created_at
FROM events
WHERE metadata @> '{"source": "mobile"}'::jsonb
ORDER BY created_at DESC
LIMIT 50;
```

Index:

```sql
CREATE INDEX CONCURRENTLY idx_events_metadata_gin
ON events
USING gin (metadata);
```

If filtering on one stable key frequently, prefer expression indexes.

```sql
CREATE INDEX CONCURRENTLY idx_events_metadata_source_created
ON events ((metadata->>'source'), created_at DESC);
```

Query:

```sql
SELECT
  id,
  created_at
FROM events
WHERE metadata->>'source' = $1
ORDER BY created_at DESC
LIMIT 50;
```

## Validate JSON Paths

Never allow arbitrary user-controlled JSON paths to become raw SQL. Allowlist supported fields.

Safe backend example:

```ts
const allowedMetadataFields: Record<string, string> = {
  source: "metadata->>'source'",
  campaign: "metadata->>'campaign'",
  device: "metadata->>'device'"
};

const fieldExpression = allowedMetadataFields[input.field];

if (!fieldExpression) {
  throw new Error("Invalid metadata field");
}
```

# Tenant Isolation And RLS

For multi-tenant systems, application-only tenant filtering is fragile. Evaluate Row-Level Security.

Example:

```sql
ALTER TABLE invoices
ENABLE ROW LEVEL SECURITY;

CREATE POLICY invoices_tenant_isolation
ON invoices
USING (tenant_id = current_setting('app.tenant_id')::uuid)
WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

Application transaction setup:

```sql
BEGIN;

SET LOCAL app.tenant_id = '00000000-0000-0000-0000-000000000001';

SELECT
  id,
  amount_cents,
  status
FROM invoices
WHERE status = 'open';

COMMIT;
```

Security requirements:

- Use `SET LOCAL` inside transactions.
- Ensure background jobs set tenant context.
- Test admin bypass paths explicitly.
- Avoid table owners as application users.
- Understand `BYPASSRLS` and owner behavior.
- Use `FORCE ROW LEVEL SECURITY` when appropriate.

```sql
ALTER TABLE invoices
FORCE ROW LEVEL SECURITY;
```

# Privilege Hardening

Unsafe:

```sql
GRANT ALL PRIVILEGES
ON DATABASE app_db
TO app_user;
```

Safer:

```sql
GRANT CONNECT
ON DATABASE app_db
TO app_user;

GRANT USAGE
ON SCHEMA public
TO app_user;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA public
TO app_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO app_user;
```

Application users should not:

- Own tables
- Be superusers
- Create extensions
- Drop schemas
- Alter tables
- Bypass RLS
- Access unrelated schemas
- Execute unsafe `SECURITY DEFINER` functions

## Search Path Hardening

Unsafe function:

```sql
CREATE FUNCTION public.do_admin_task()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  INSERT INTO audit_log(action)
  VALUES ('admin_task');
END;
$$;
```

Safer:

```sql
CREATE FUNCTION public.do_admin_task()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
BEGIN
  INSERT INTO public.audit_log(action)
  VALUES ('admin_task');
END;
$$;

REVOKE ALL ON FUNCTION public.do_admin_task() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION public.do_admin_task() TO app_admin_role;
```

# Sensitive Data

Review sensitive columns for encryption, hashing, masking, tokenization, and logging exposure.

Sensitive data includes:

- Passwords
- Password reset tokens
- API keys
- OAuth tokens
- Session IDs
- JWTs
- Credit card data
- National IDs
- SSNs
- Emails
- Phone numbers
- Addresses
- Medical data
- Financial data

Password storage must use Argon2id or bcrypt. Never store plaintext passwords. Never use raw SHA-256 for passwords.

Do not log:

- Passwords
- Tokens
- JWTs
- Session IDs
- Authorization headers
- Secret keys
- Full payment details
- Raw SQL errors containing sensitive values

# Migrations

## Production-Safe Index Creation

Prefer:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_created_desc
ON orders (user_id, created_at DESC);
```

Important operational notes:

- `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block.
- Failed concurrent index builds may leave invalid indexes.
- Check and clean invalid indexes deliberately.
- Concurrent indexes still consume CPU, I/O, and memory.
- Large index builds can increase replication lag.

Check invalid indexes:

```sql
SELECT
  schemaname,
  tablename,
  indexname
FROM pg_indexes
WHERE indexname IN (
  SELECT c.relname
  FROM pg_class c
  JOIN pg_index i
    ON i.indexrelid = c.oid
  WHERE i.indisvalid = false
);
```

## Add Column Safely

Often safe:

```sql
ALTER TABLE users
ADD COLUMN last_seen_at timestamptz;
```

Riskier on large tables:

```sql
ALTER TABLE users
ADD COLUMN plan_name text NOT NULL DEFAULT 'free';
```

Safer rollout:

```sql
ALTER TABLE users
ADD COLUMN plan_name text;

UPDATE users
SET plan_name = 'free'
WHERE plan_name IS NULL
  AND id >= $1
  AND id < $2;

ALTER TABLE users
ALTER COLUMN plan_name SET DEFAULT 'free';

ALTER TABLE users
ADD CONSTRAINT users_plan_name_not_null
CHECK (plan_name IS NOT NULL) NOT VALID;

ALTER TABLE users
VALIDATE CONSTRAINT users_plan_name_not_null;
```

Then, if needed during a maintenance window:

```sql
ALTER TABLE users
ALTER COLUMN plan_name SET NOT NULL;
```

## Add Foreign Key Safely

```sql
ALTER TABLE orders
ADD CONSTRAINT orders_user_id_fkey
FOREIGN KEY (user_id)
REFERENCES users(id)
NOT VALID;

ALTER TABLE orders
VALIDATE CONSTRAINT orders_user_id_fkey;
```

Also verify the referencing column has an index:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id
ON orders (user_id);
```

## Batch Updates

Unsafe:

```sql
UPDATE events
SET processed = true
WHERE processed = false;
```

Safer batch pattern:

```sql
WITH batch AS (
  SELECT
    id
  FROM events
  WHERE processed = false
  ORDER BY id
  LIMIT 1000
)
UPDATE events e
SET processed = true
FROM batch
WHERE e.id = batch.id
RETURNING e.id;
```

Operational requirements:

- Run in bounded batches.
- Commit between batches.
- Monitor locks, WAL, replication lag, and bloat.
- Pause or slow down if production latency increases.
- Vacuum impact must be considered.

# EXPLAIN Analysis

Prefer:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  id,
  status
FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 20;
```

Inspect:

- Sequential scans on large tables
- Rows removed by filter
- Estimated rows vs actual rows mismatch
- Nested loops over large inputs
- Hash joins spilling to disk
- Sort method using disk
- Shared buffers hit/read ratio
- Temporary reads and writes
- Parallelism
- Index condition vs filter condition
- Planning time vs execution time

Danger signs:

- `Seq Scan` on a large table for selective filters
- `Rows Removed by Filter` extremely high
- `Sort Method: external merge Disk`
- Actual rows much higher than estimated rows
- Large nested-loop inner scans
- Temporary file usage
- High shared read blocks for hot endpoints

# Connection Pooling

Review:

- Max PostgreSQL connections
- App pool size per instance
- Number of app instances
- Background worker pools
- Migration clients
- Admin clients
- Serverless concurrency
- Connection leaks
- Idle clients
- Idle in transaction sessions
- PgBouncer mode

Example risk:

```text
50 app instances * pool size 20 = 1000 possible database connections
```

If PostgreSQL supports 300 safe active connections, this configuration can cause production failure.

Recommendations:

- Use bounded pools.
- Set acquisition timeouts.
- Monitor pool wait time.
- Use PgBouncer when appropriate.
- Prefer transaction pooling only when application behavior is compatible.
- Avoid session state when using transaction pooling.
- Set `idle_in_transaction_session_timeout`.

# Timeouts And Retries

Recommended baseline for OLTP workloads, tuned per system:

```sql
SET statement_timeout = '5s';
SET lock_timeout = '2s';
SET idle_in_transaction_session_timeout = '30s';
```

Use context-specific values. Reporting jobs, maintenance tasks, and batch workers may need different limits.

Retry safely for:

- Deadlocks: SQLSTATE `40P01`
- Serialization failures: SQLSTATE `40001`
- Transient connection failures when operation is idempotent

Never blindly retry non-idempotent operations without idempotency keys or uniqueness constraints.

# Observability

Recommend:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Track:

- Slow queries
- Query count
- Mean, p95, and p99 latency
- Rows read vs rows returned
- Shared blocks read and hit
- Temporary file usage
- Lock waits
- Deadlocks
- Idle in transaction sessions
- Connection pool saturation
- Replication lag
- Autovacuum activity
- Table and index bloat
- Checkpoint pressure
- WAL generation rate

Useful views:

```sql
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

```sql
SELECT
  pid,
  state,
  wait_event_type,
  wait_event,
  now() - query_start AS query_age,
  query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_age DESC;
```

```sql
SELECT
  locktype,
  relation::regclass,
  mode,
  granted,
  pid
FROM pg_locks
ORDER BY granted, pid;
```

# ORM Review Rules

Never trust ORM-generated SQL.

Check for:

- N+1 queries
- Hidden `SELECT *`
- Over-fetching
- Missing tenant filters
- Hidden joins
- Join fanout
- Inefficient eager loading
- Inefficient counts
- Transaction boundaries
- Unbounded result sets
- Unbounded nested includes
- Missing indexes for generated filters
- Unsafe raw SQL escape hatches

Example ORM risk:

```ts
const users = await prisma.user.findMany({
  include: {
    orders: true,
    sessions: true,
    auditLogs: true
  }
});
```

Potential problems:

- Huge result payloads
- Sensitive data exposure
- Join or query fanout
- Memory pressure
- Missing pagination
- Missing tenant scope

Safer pattern:

```ts
const users = await prisma.user.findMany({
  where: {
    tenantId
  },
  select: {
    id: true,
    email: true,
    status: true,
    createdAt: true
  },
  orderBy: [
    { createdAt: "desc" },
    { id: "desc" }
  ],
  take: 50
});
```

# Common Findings

## SQL Injection

Problem:

```ts
await db.query(`
  SELECT
    id,
    email
  FROM users
  WHERE email = '${email}'
`);
```

Fix:

```ts
await db.query(
  `
  SELECT
    id,
    email
  FROM users
  WHERE email = $1
  `,
  [email]
);
```

## Missing Tenant Filter

Problem:

```sql
SELECT
  id,
  amount_cents,
  status
FROM invoices
WHERE id = $1;
```

Fix:

```sql
SELECT
  id,
  amount_cents,
  status
FROM invoices
WHERE tenant_id = $1
  AND id = $2;
```

Recommended index:

```sql
CREATE UNIQUE INDEX CONCURRENTLY idx_invoices_tenant_id_id
ON invoices (tenant_id, id);
```

## Unsafe Count

Problem:

```sql
SELECT COUNT(*)
FROM events;
```

Risk: expensive on huge tables.

Alternatives depend on correctness needs.

Exact scoped count:

```sql
SELECT COUNT(*)::bigint
FROM events
WHERE tenant_id = $1
  AND created_at >= $2
  AND created_at < $3;
```

Approximate table estimate:

```sql
SELECT reltuples::bigint AS estimated_rows
FROM pg_class
WHERE oid = 'public.events'::regclass;
```

## Unbounded Export

Problem:

```sql
SELECT
  id,
  payload,
  created_at
FROM events
WHERE tenant_id = $1;
```

Fix:

```sql
SELECT
  id,
  payload,
  created_at
FROM events
WHERE tenant_id = $1
  AND created_at >= $2
  AND created_at < $3
ORDER BY created_at ASC, id ASC
LIMIT $4;
```

Backend requirements:

- Enforce maximum date range.
- Use cursor-based pagination.
- Stream results.
- Rate limit exports.
- Audit access.

# Final Checklist

Before approving PostgreSQL code, verify:

- Values are parameterized.
- Dynamic identifiers are allowlisted.
- Tenant isolation is enforced.
- Sensitive columns are not accidentally returned.
- Queries avoid `SELECT *`.
- Pagination is bounded.
- Deep pagination uses keyset pagination where appropriate.
- Joins cannot create accidental fanout.
- N+1 query patterns are removed.
- Indexes match filters, joins, and ordering.
- Index write amplification is acceptable.
- Transactions are short and explicit.
- Critical updates are atomic.
- Lock ordering is deterministic.
- Deadlocks and serialization failures are retried safely.
- Migrations avoid long blocking locks.
- Large updates are batched.
- Concurrent indexes are used where appropriate.
- Failed concurrent indexes are handled.
- Vacuum, bloat, WAL, and replication lag are considered.
- Application roles follow least privilege.
- RLS is evaluated for multi-tenant data.
- `SECURITY DEFINER` functions harden `search_path`.
- Logs do not contain secrets.
- Connection pools are bounded.
- Timeouts are configured.
- Slow queries, locks, deadlocks, pool saturation, and replication lag are observable.

Always produce recommendations that are secure, correct, operationally safe, and scalable before they are merely fast.
