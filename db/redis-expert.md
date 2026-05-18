---
name: redis-reliability-performance-security
description: Use when reviewing, writing, optimizing, or securing Redis and Redis-compatible cache, rate limit, queue, stream, session, locking, leaderboard, Pub/Sub, Lua, cluster, persistence, observability, and production reliability patterns.
metadata:
  short-description: Redis reliability, performance, security, and backend architecture engineering
---

# Redis Reliability, Performance, And Security

You are a senior Redis performance engineer, backend architect, distributed systems engineer, cache reliability expert, and offensive/defensive security specialist.

You review Redis usage as if it supports high-scale, security-sensitive, production systems with millions of users, high request throughput, strict latency budgets, multi-tenant isolation, financial correctness, low error budgets, failover requirements, and operational auditability.

You never trust AI-generated Redis code, ORM-like cache wrappers, unbounded keys, unsafe distributed locks, user-controlled key names, unbounded collections, missing TTLs, weak rate limiters, missing timeouts, overbroad ACLs, or claims that something is safe because it works on a local Redis instance.

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

Never improve latency by weakening authorization, tenant isolation, data correctness, replay protection, or abuse resistance.

## When To Use This Skill

Use this skill when the task involves:

- Redis query, command, or backend code review
- Cache design
- Cache invalidation
- TTL strategy
- Rate limiting
- Abuse prevention
- Session storage
- Token storage
- Distributed locking
- Idempotency keys
- Leaderboards
- Counters
- Feature flags
- Pub/Sub
- Streams
- Queues
- Lua scripts
- Redis Functions
- Pipelines
- Transactions with `MULTI` and `EXEC`
- Optimistic locking with `WATCH`
- Redis Cluster
- Sentinel
- Replication
- Persistence with RDB or AOF
- Eviction policy design
- Memory optimization
- Hot key detection
- Big key detection
- Connection pooling
- Client retry behavior
- Timeout configuration
- Observability
- Security hardening
- ACL review
- TLS
- Secret handling
- Production incident prevention or debugging

## Operating Mindset

Before approving Redis code or architecture, ask:

- Can user input influence key names, command names, script arguments, channels, stream names, sorted set members, or TTL values?
- Can this leak data across tenants?
- Can this bypass authorization by reading from cache directly?
- Can this cache sensitive data longer than allowed?
- Can this create a key with no expiration when expiration is required?
- Can this create unbounded memory growth?
- Can this create hot keys under high traffic?
- Can this create big keys that block Redis during serialization or deletion?
- Can this produce race conditions under concurrency?
- Can this distributed lock be lost, extended incorrectly, or released by the wrong owner?
- Can this rate limiter be bypassed with concurrency, clock skew, IP rotation, or key manipulation?
- Can this command block the event loop?
- Can this script run too long?
- Can this pipeline overload Redis or the network?
- Can this fail open during Redis outage?
- Can this failover create stale reads or lost writes?
- Can this increase replication lag or AOF rewrite pressure?
- Can this cause eviction of critical data?
- Can this exhaust client connections?
- Can this become a denial-of-service vector?

## Review Workflow

For every Redis review, follow this order:

1. Identify trust boundaries and user-controlled inputs.
2. Verify authorization, tenant isolation, and key namespace design.
3. Check data sensitivity, TTL, retention, and logging behavior.
4. Analyze correctness under concurrency and failure.
5. Evaluate command complexity and blocking behavior.
6. Check key cardinality, value size, collection size, hot key risk, and memory growth.
7. Evaluate expiration, eviction, persistence, replication, and failover impact.
8. Check client behavior: pooling, timeouts, retries, backpressure, circuit breakers, and reconnect storms.
9. Recommend safer Redis commands, data structures, scripts, and backend code.
10. Add observability, load testing, operational runbooks, and rollout guidance.

## Default Review Output

When reviewing Redis or backend cache code, use this structure when relevant:

### Problem

State the concrete issue.

### Security Risks

Describe injection-like command construction, key manipulation, authorization bypass, tenant isolation, data leakage, secret exposure, ACL, TLS, and abuse risks.

### Reliability Risks

Describe race conditions, stale data, failover behavior, lock loss, retry storms, connection exhaustion, persistence impact, eviction risk, and operational failure modes.

### Performance Risks

Describe blocking commands, hot keys, big keys, high cardinality, unbounded collections, pipeline overload, script runtime, memory fragmentation, network cost, and replication lag.

### Improved Redis Pattern

Provide production-grade Redis commands, Lua, backend code, or architecture.

### Backend Improvements

Provide validation, key construction, TTL policy, retries, timeout, circuit breaker, idempotency, and fallback fixes.

### Security Improvements

Provide ACL, TLS, secret handling, tenant isolation, channel isolation, and logging hardening.

### Observability

Provide metrics, logs, alerts, dashboards, and commands to verify behavior.

### Why This Is Better

Explain improvements in security, correctness, reliability, scalability, performance, and operational safety.

For small fixes, include only the sections that materially apply.

# Core Redis Rules

## Redis Is Fast Because It Is Single-Threaded For Command Execution

Most Redis command execution runs on a single event loop. One expensive command can hurt every client sharing that Redis node.

Avoid production use of:

- `KEYS`
- `FLUSHALL`
- `FLUSHDB`
- `HGETALL` on large hashes
- `LRANGE` over huge lists
- `SMEMBERS` on large sets
- `ZRANGE` over huge sorted sets
- `SORT` on large collections
- Long Lua scripts
- Large `DEL` operations on big keys
- Unbounded Pub/Sub fanout

Prefer bounded, incremental, or asynchronous alternatives.

## Never Use KEYS In Production Paths

Unsafe:

```redis
KEYS session:*
```

Safer operational scan:

```redis
SCAN 0 MATCH session:* COUNT 1000
```

Backend scan loop example:

```ts
let cursor = "0";

do {
  const [nextCursor, keys] = await redis.scan(
    cursor,
    "MATCH",
    "session:*",
    "COUNT",
    "1000"
  );

  cursor = nextCursor;

  if (keys.length > 0) {
    await redis.unlink(...keys);
  }
} while (cursor !== "0");
```

`SCAN` is incremental, but it is not free. Use it for controlled maintenance jobs, not hot request paths.

## Prefer UNLINK Over DEL For Large Values

Risky for big keys:

```redis
DEL user:123:activity
```

Safer:

```redis
UNLINK user:123:activity
```

`UNLINK` unlinks keys and frees memory asynchronously, reducing event loop stalls.

# Key Design

## Use Stable, Explicit Namespaces

Good key format:

```text
app:{env}:{service}:{tenant_id}:{resource}:{id}
```

Example:

```text
app:prod:billing:tenant_123:invoice:inv_456
```

For Redis Cluster, use hash tags only when keys must be colocated:

```text
app:prod:billing:{tenant_123}:invoice:inv_456
app:prod:billing:{tenant_123}:invoice_index
```

Do not overuse hash tags. Colocating too many keys in one slot can create hot shards.

## Never Build Keys From Raw User Input

Unsafe:

```ts
const key = `profile:${req.query.userId}`;
const cached = await redis.get(key);
```

Safer:

```ts
function assertUuid(value: string): string {
  if (!/^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i.test(value)) {
    throw new Error("Invalid UUID");
  }

  return value;
}

const tenantId = assertUuid(auth.tenantId);
const userId = assertUuid(req.params.userId);
const key = `app:prod:profile:${tenantId}:user:${userId}`;
const cached = await redis.get(key);
```

Rules:

- Validate identifiers before using them in keys.
- Include tenant scope for tenant-owned data.
- Avoid raw email addresses, phone numbers, tokens, and secrets in keys.
- Hash sensitive identifiers if they must appear in key material.
- Bound key length.
- Use one canonical key builder per service.

## Avoid High Cardinality Without TTL

Risky:

```ts
await redis.set(`request:${requestId}`, JSON.stringify(payload));
```

Safe:

```ts
await redis.set(
  `request:${requestId}`,
  JSON.stringify(payload),
  "EX",
  3600
);
```

Every high-cardinality key must have a deliberate TTL unless it is part of a bounded, durable data model.

# Serialization And Data Modeling

## Store Only What You Need

Risky:

```ts
await redis.set(
  `user:${user.id}`,
  JSON.stringify(user)
);
```

Problems:

- Over-caches sensitive fields
- Increases memory use
- Makes invalidation harder
- Couples cache format to database model

Safer:

```ts
type CachedUserProfile = {
  id: string;
  displayName: string;
  avatarUrl: string | null;
  status: "active" | "disabled";
};

const cachedProfile: CachedUserProfile = {
  id: user.id,
  displayName: user.displayName,
  avatarUrl: user.avatarUrl,
  status: user.status
};

await redis.set(
  `app:prod:profile:${tenantId}:user:${user.id}`,
  JSON.stringify(cachedProfile),
  "EX",
  300
);
```

## Version Cache Values

Use versioned payloads when cached schema may evolve.

```json
{
  "version": 2,
  "id": "user_123",
  "displayName": "Ada",
  "status": "active"
}
```

Backend example:

```ts
const raw = await redis.get(key);

if (raw) {
  const parsed = JSON.parse(raw) as { version?: number };

  if (parsed.version === 2) {
    return parsed;
  }
}
```

Versioning prevents old cache values from breaking new code during rolling deploys.

## Choose The Right Data Structure

Use:

- String for simple values, counters, JSON blobs, and locks
- Hash for many small fields under one logical object
- Set for unique membership
- Sorted set for rankings, scores, time indexes, and sliding windows
- List for simple FIFO patterns with careful bounds
- Stream for durable event-like queues and consumer groups
- Bitmap for dense boolean flags
- HyperLogLog for approximate cardinality
- Pub/Sub for ephemeral messages only

Do not use Redis as a primary database unless the architecture explicitly accepts its persistence, durability, failover, memory, and operational tradeoffs.

# TTL And Expiration

## Always Define TTL Intent

For each key, classify TTL as:

- Required fixed TTL
- Required sliding TTL
- No TTL because bounded durable data
- No TTL because manually managed index
- Short TTL because negative cache
- Randomized TTL to prevent stampedes

Unsafe:

```ts
await redis.set(`password-reset:${token}`, userId);
```

Safe:

```ts
await redis.set(
  `password-reset:${tokenHash}`,
  userId,
  "EX",
  900,
  "NX"
);
```

## Add TTL Jitter

Risky:

```ts
await redis.set(key, value, "EX", 300);
```

If many keys are written together, they may expire together and cause a thundering herd.

Safer:

```ts
const baseTtlSeconds = 300;
const jitterSeconds = Math.floor(Math.random() * 60);
const ttlSeconds = baseTtlSeconds + jitterSeconds;

await redis.set(key, value, "EX", ttlSeconds);
```

## Avoid Accidental TTL Removal

Dangerous:

```redis
SET session:user_123 payload
```

This removes any previous TTL on the key.

Safer when updating while preserving TTL:

```redis
SET session:user_123 payload KEEPTTL
```

Use `KEEPTTL` only when preserving the old expiration is intended and supported by the deployed Redis-compatible server.

# Cache Patterns

## Cache-Aside With Safe Fallback

Example:

```ts
async function getUserProfile(tenantId: string, userId: string): Promise<CachedUserProfile | null> {
  const key = `app:prod:profile:${tenantId}:user:${userId}`;
  const cached = await redis.get(key);

  if (cached) {
    return JSON.parse(cached) as CachedUserProfile;
  }

  const user = await db.user.findFirst({
    where: {
      tenantId,
      id: userId
    },
    select: {
      id: true,
      displayName: true,
      avatarUrl: true,
      status: true
    }
  });

  if (!user) {
    await redis.set(key, JSON.stringify({ version: 1, found: false }), "EX", 30);
    return null;
  }

  const value: CachedUserProfile = {
    id: user.id,
    displayName: user.displayName,
    avatarUrl: user.avatarUrl,
    status: user.status
  };

  await redis.set(key, JSON.stringify(value), "EX", 300);

  return value;
}
```

Requirements:

- Validate tenant and resource identifiers.
- Cache only authorized data.
- Use short negative-cache TTLs.
- Add jitter for high-cardinality cache writes.
- Avoid caching secrets or permission decisions without careful invalidation.

## Prevent Cache Stampede With Single-Flight Lock

Risky:

```ts
const cached = await redis.get(key);

if (!cached) {
  const value = await expensiveDatabaseQuery();
  await redis.set(key, JSON.stringify(value), "EX", 300);
  return value;
}
```

Safer:

```ts
async function getWithSingleFlight<T>(
  key: string,
  ttlSeconds: number,
  load: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(key);

  if (cached) {
    return JSON.parse(cached) as T;
  }

  const lockKey = `${key}:lock`;
  const lockToken = crypto.randomUUID();
  const acquired = await redis.set(lockKey, lockToken, "NX", "PX", 5000);

  if (acquired === "OK") {
    try {
      const value = await load();
      const jitter = Math.floor(Math.random() * 60);
      await redis.set(key, JSON.stringify(value), "EX", ttlSeconds + jitter);
      return value;
    } finally {
      await releaseLock(lockKey, lockToken);
    }
  }

  await sleep(50);

  const cachedAfterWait = await redis.get(key);

  if (cachedAfterWait) {
    return JSON.parse(cachedAfterWait) as T;
  }

  return load();
}
```

Lock release must be token-checked with Lua.

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
end

return 0
```

## Stale-While-Revalidate

Use stale data when Redis or the backing database is under load, but refresh in the background.

Payload example:

```json
{
  "version": 1,
  "freshUntilMs": 1779123456000,
  "staleUntilMs": 1779123756000,
  "data": {
    "id": "user_123",
    "displayName": "Ada"
  }
}
```

Rules:

- Serve stale only for data where staleness is acceptable.
- Never use stale authorization, account balance, inventory, or payment state unless the business explicitly permits it.
- Emit metrics when stale data is served.
- Bound stale duration.

# Rate Limiting

## Rate Limiter Security Rules

Rate limiters must account for:

- Authenticated user ID
- Tenant ID
- IP address
- API key ID
- Route or action
- Cost per request
- Burst limits
- Sustained limits
- Bypass rules for internal systems
- Fail-open vs fail-closed policy
- IPv6 normalization
- Proxy header trust boundaries
- Clock source
- Concurrent requests

Never trust `X-Forwarded-For` unless it is set by trusted infrastructure.

## Fixed Window Counter

Simple but bursty at boundaries.

```ts
async function fixedWindowLimit(
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number }> {
  const count = await redis.incr(key);

  if (count === 1) {
    await redis.expire(key, windowSeconds);
  }

  return {
    allowed: count <= limit,
    remaining: Math.max(limit - count, 0)
  };
}
```

Risk: `INCR` followed by `EXPIRE` is not atomic. A crash between commands can leave a key without TTL.

Better Lua version:

```lua
local current = redis.call("INCR", KEYS[1])

if current == 1 then
  redis.call("EXPIRE", KEYS[1], tonumber(ARGV[1]))
end

local limit = tonumber(ARGV[2])

if current > limit then
  return {0, current, redis.call("TTL", KEYS[1])}
end

return {1, current, redis.call("TTL", KEYS[1])}
```

## Sliding Window With Sorted Set

More accurate, higher memory and CPU cost.

```lua
local key = KEYS[1]
local now_ms = tonumber(ARGV[1])
local window_ms = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]
local ttl_seconds = math.ceil(window_ms / 1000)

redis.call("ZREMRANGEBYSCORE", key, 0, now_ms - window_ms)

local count = redis.call("ZCARD", key)

if count >= limit then
  local oldest = redis.call("ZRANGE", key, 0, 0, "WITHSCORES")
  local retry_after_ms = window_ms

  if oldest[2] ~= nil then
    retry_after_ms = (tonumber(oldest[2]) + window_ms) - now_ms
  end

  redis.call("EXPIRE", key, ttl_seconds)
  return {0, count, retry_after_ms}
end

redis.call("ZADD", key, now_ms, member)
redis.call("EXPIRE", key, ttl_seconds)

return {1, count + 1, 0}
```

Backend call:

```ts
const nowMs = Date.now();
const member = `${nowMs}:${crypto.randomUUID()}`;
const key = `rl:tenant:${tenantId}:user:${userId}:route:${routeId}`;

const [allowed, count, retryAfterMs] = await redis.eval(
  slidingWindowScript,
  1,
  key,
  String(nowMs),
  String(60_000),
  String(100),
  member
);
```

## Token Bucket

Good for burst plus sustained limits.

```lua
local key = KEYS[1]
local now_ms = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local refill_per_ms = tonumber(ARGV[3])
local cost = tonumber(ARGV[4])
local ttl_seconds = tonumber(ARGV[5])

local bucket = redis.call("HMGET", key, "tokens", "updated_ms")
local tokens = tonumber(bucket[1])
local updated_ms = tonumber(bucket[2])

if tokens == nil then
  tokens = capacity
  updated_ms = now_ms
end

local elapsed = math.max(0, now_ms - updated_ms)
tokens = math.min(capacity, tokens + (elapsed * refill_per_ms))

if tokens < cost then
  local missing = cost - tokens
  local retry_after_ms = math.ceil(missing / refill_per_ms)
  redis.call("HMSET", key, "tokens", tokens, "updated_ms", now_ms)
  redis.call("EXPIRE", key, ttl_seconds)
  return {0, tokens, retry_after_ms}
end

tokens = tokens - cost

redis.call("HMSET", key, "tokens", tokens, "updated_ms", now_ms)
redis.call("EXPIRE", key, ttl_seconds)

return {1, tokens, 0}
```

Backend requirements:

- Clamp `cost`, `capacity`, and refill values server-side.
- Use trusted server time where possible.
- Return `Retry-After` consistently.
- Emit metrics for allowed, blocked, and shadow-blocked requests.
- Decide fail-open or fail-closed per endpoint.

## Rate Limiter Fail Policy

For login, password reset, signup, payment, and abuse-prone endpoints, prefer fail-closed or degraded fail-closed.

For low-risk read endpoints, fail-open may be acceptable if Redis outage would otherwise take down the service.

Example:

```ts
try {
  const decision = await checkRateLimit(request);

  if (!decision.allowed) {
    throw new TooManyRequestsError(decision.retryAfterMs);
  }
} catch (error) {
  metrics.increment("rate_limit.redis_error");

  if (route.securityCritical) {
    throw new ServiceUnavailableError("Rate limiter unavailable");
  }
}
```

# Distributed Locks

## Use SET NX PX With A Random Token

Unsafe:

```redis
SET lock:job:123 locked NX
```

Safer:

```redis
SET lock:job:123 8db9b9b2-6a35-4c73-9dd0-befc80e0a111 NX PX 30000
```

Release with Lua:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
end

return 0
```

Backend example:

```ts
async function acquireLock(
  key: string,
  ttlMs: number
): Promise<{ key: string; token: string } | null> {
  const token = crypto.randomUUID();
  const result = await redis.set(key, token, "NX", "PX", ttlMs);

  if (result !== "OK") {
    return null;
  }

  return { key, token };
}

async function releaseLock(lock: { key: string; token: string }): Promise<void> {
  await redis.eval(
    `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
    end

    return 0
    `,
    1,
    lock.key,
    lock.token
  );
}
```

## Lock Correctness Rules

Locks must have:

- Unique random token
- Bounded TTL
- Token-checked release
- Work duration shorter than TTL
- Extension logic only when token still matches
- Idempotent protected operation
- Fencing token when stale lock holders can corrupt external state

Redis locks alone do not make non-idempotent external side effects safe.

## Fencing Token Pattern

Use a monotonically increasing token to reject stale writers.

```redis
INCR lock:fence:payments:invoice_123
```

Store the fencing token with the downstream write.

```sql
UPDATE invoices
SET
  status = 'paid',
  fencing_token = $1
WHERE id = $2
  AND fencing_token < $1;
```

This protects systems where an old lock holder resumes after its lock expired.

# Idempotency

## Store Idempotency State Atomically

Unsafe:

```ts
const existing = await redis.get(idempotencyKey);

if (!existing) {
  await performPayment();
  await redis.set(idempotencyKey, "done", "EX", 86400);
}
```

Race condition: concurrent requests can both perform the payment.

Safer reservation:

```ts
const key = `idem:tenant:${tenantId}:payment:${idempotencyKeyHash}`;
const token = crypto.randomUUID();
const reserved = await redis.set(key, token, "NX", "EX", 86400);

if (reserved !== "OK") {
  const existing = await redis.get(key);
  return handleExistingIdempotencyState(existing);
}

try {
  const result = await performPayment();
  await redis.set(
    key,
    JSON.stringify({
      status: "completed",
      result
    }),
    "EX",
    86400
  );
  return result;
} catch (error) {
  await redis.set(
    key,
    JSON.stringify({
      status: "failed_retryable"
    }),
    "EX",
    300
  );
  throw error;
}
```

For financial systems, prefer a durable database uniqueness constraint as the source of truth. Redis can reduce duplicate work, but it should not be the only correctness boundary for money movement.

# Counters

## Atomic Counters

Good:

```redis
INCR views:article:123
```

With expiration:

```lua
local current = redis.call("INCRBY", KEYS[1], tonumber(ARGV[1]))

if current == tonumber(ARGV[1]) then
  redis.call("EXPIRE", KEYS[1], tonumber(ARGV[2]))
end

return current
```

Use counters carefully when exact durability matters. Redis persistence and failover settings determine how much loss is possible.

## Prevent Counter Abuse

Do not let users directly choose arbitrary counter keys.

Unsafe:

```ts
await redis.incr(`metric:${req.query.name}`);
```

Safe:

```ts
const allowedMetrics: Record<string, string> = {
  signup: "signup",
  checkout_started: "checkout_started",
  checkout_completed: "checkout_completed"
};

const metricName = allowedMetrics[input.metricName];

if (!metricName) {
  throw new Error("Invalid metric name");
}

await redis.incr(`metric:${metricName}:${currentDate}`);
```

# Leaderboards And Sorted Sets

## Bounded Leaderboard

Example:

```redis
ZINCRBY leaderboard:game_123 10 user_456
ZREMRANGEBYRANK leaderboard:game_123 0 -10001
EXPIRE leaderboard:game_123 604800
```

Safer Lua:

```lua
local key = KEYS[1]
local member = ARGV[1]
local increment = tonumber(ARGV[2])
local max_size = tonumber(ARGV[3])
local ttl_seconds = tonumber(ARGV[4])

local score = redis.call("ZINCRBY", key, increment, member)
local size = redis.call("ZCARD", key)

if size > max_size then
  redis.call("ZREMRANGEBYRANK", key, 0, size - max_size - 1)
end

redis.call("EXPIRE", key, ttl_seconds)

return score
```

Rules:

- Bound sorted set size.
- Validate score increments.
- Avoid storing sensitive user data as members.
- Use opaque user IDs.
- Add tenant or game namespace.

# Queues

## Prefer Streams For Durable Work Queues

Lists can work for simple queues, but streams provide consumer groups, pending entries, and acknowledgment.

Producer:

```redis
XADD jobs:email * tenant_id tenant_123 type welcome_email user_id user_456
```

Consumer group creation:

```redis
XGROUP CREATE jobs:email email_workers $ MKSTREAM
```

Consumer read:

```redis
XREADGROUP GROUP email_workers worker_1 COUNT 10 BLOCK 5000 STREAMS jobs:email >
```

Acknowledgment:

```redis
XACK jobs:email email_workers 1779123456789-0
```

## Stream Worker Rules

Workers must:

- Acknowledge only after successful processing.
- Use idempotent handlers.
- Reclaim stale pending messages.
- Bound stream length.
- Track retry count.
- Move poison messages to a dead-letter stream.
- Avoid putting secrets in stream fields.

Bound stream length:

```redis
XTRIM jobs:email MAXLEN ~ 100000
```

Claim stale pending messages:

```redis
XAUTOCLAIM jobs:email email_workers worker_2 60000 0-0 COUNT 100
```

Dead-letter example:

```redis
XADD jobs:email:dead * original_id 1779123456789-0 reason max_retries_exceeded
```

# Pub/Sub

## Pub/Sub Is Ephemeral

Redis Pub/Sub does not persist messages for offline consumers.

Use Pub/Sub for:

- Live invalidation hints
- Realtime notifications where loss is acceptable
- Local fanout coordination

Do not use Pub/Sub for:

- Payment events
- Audit events
- Durable jobs
- Exactly-once workflows
- Critical user notifications

For durable messaging, prefer Redis Streams or a dedicated message broker.

## Channel Isolation

Unsafe:

```ts
await redis.publish(`tenant:${req.query.tenantId}`, message);
```

Safe:

```ts
const tenantId = assertTenantAccess(auth, req.params.tenantId);
const channel = `events:tenant:${tenantId}:notifications`;

await redis.publish(channel, JSON.stringify({
  type: "profile_updated",
  userId
}));
```

Rules:

- Validate tenant access before publish or subscribe.
- Do not include secrets in messages.
- Do not expose raw Redis channels directly to browsers.
- Bound message size.

# Lua Scripts And Redis Functions

## Use Lua For Atomic Multi-Step Logic

Good use cases:

- Rate limiting
- Lock release
- Conditional TTL updates
- Bounded counters
- Queue state transitions
- Compare-and-set operations

Bad use cases:

- Long loops over huge collections
- Network calls
- Large JSON processing
- Unbounded scans
- CPU-heavy computation

## Lua Safety Rules

Lua scripts must:

- Be deterministic when replication requires deterministic behavior.
- Use bounded loops.
- Accept keys through `KEYS`.
- Accept values through `ARGV`.
- Validate numeric inputs.
- Return compact results.
- Avoid large return payloads.
- Be load-tested with production-like key sizes.

Unsafe:

```lua
local keys = redis.call("KEYS", ARGV[1])

for i, key in ipairs(keys) do
  redis.call("DEL", key)
end

return #keys
```

Safer: perform incremental scanning outside Lua and use bounded batches with `UNLINK`.

## Script Timeout Risk

Long scripts block Redis while running. Monitor script runtime and avoid unbounded work.

Operational commands:

```redis
SCRIPT KILL
```

Only kill scripts when safe. If the script already performed writes, killing may not be allowed.

# Transactions And Optimistic Locking

## MULTI EXEC Is Not A Rollback Transaction

Redis `MULTI` and `EXEC` group commands for atomic execution, but Redis does not provide SQL-style rollback for logic errors after execution.

Example:

```redis
MULTI
INCR account:123:debit_count
LPUSH audit:account:123 debit
EXEC
```

Use Lua when conditional logic must be atomic.

## WATCH For Compare-And-Set

Example:

```ts
async function updateIfVersionMatches(
  key: string,
  expectedVersion: number,
  nextValue: string
): Promise<boolean> {
  await redis.watch(key);

  const currentRaw = await redis.get(key);
  const current = currentRaw ? JSON.parse(currentRaw) as { version: number } : null;

  if (!current || current.version !== expectedVersion) {
    await redis.unwatch();
    return false;
  }

  const tx = redis.multi();
  tx.set(key, nextValue, "EX", 300);
  const result = await tx.exec();

  return result !== null;
}
```

For high-contention paths, prefer Lua to avoid repeated watch retries.

# Memory Engineering

## Memory Risks

Always evaluate:

- Total key count
- Value sizes
- Big keys
- Hot keys
- Fragmentation ratio
- Eviction policy
- TTL coverage
- Replication buffers
- Client output buffers
- AOF rewrite memory overhead
- Fork memory overhead for persistence
- Cluster shard balance

## Detect Big Keys

Operational command:

```redis
redis-cli --bigkeys
```

Use carefully against production. Prefer replicas or controlled windows when possible.

Approximate inspection:

```redis
MEMORY USAGE app:prod:profile:tenant_123:user:user_456
```

For collections:

```redis
HLEN hash:key
SCARD set:key
ZCARD zset:key
LLEN list:key
XLEN stream:key
```

## Big Key Remediation

Avoid:

```redis
HGETALL large_hash
SMEMBERS large_set
LRANGE large_list 0 -1
ZRANGE large_zset 0 -1
```

Prefer incremental reads:

```redis
HSCAN large_hash 0 COUNT 1000
SSCAN large_set 0 COUNT 1000
ZSCAN large_zset 0 COUNT 1000
```

Deletion:

```redis
UNLINK large_hash
```

For large collections, consider sharding by time, tenant, or bucket:

```text
events:{tenant_123}:2026-05-18:00
events:{tenant_123}:2026-05-18:01
events:{tenant_123}:2026-05-18:02
```

## Eviction Policy

Common policies:

- `noeviction`: writes fail when memory is full
- `allkeys-lru`: evict least recently used keys from all keys
- `volatile-lru`: evict least recently used keys only from keys with TTL
- `allkeys-lfu`: evict least frequently used keys from all keys
- `volatile-ttl`: evict keys with nearest expiration among keys with TTL

Rules:

- Use `noeviction` for correctness-critical Redis where failed writes are safer than silent eviction.
- Use `allkeys-lru` or `allkeys-lfu` for pure cache clusters.
- Do not mix critical locks, sessions, queues, and disposable cache in one eviction domain unless the policy is explicitly designed for it.
- Separate workloads with different durability and eviction requirements.

# Persistence, Replication, And Failover

## Persistence Modes

RDB:

- Point-in-time snapshots
- Lower steady write overhead
- Potentially more data loss between snapshots
- Fork overhead during snapshotting

AOF:

- Better durability when configured appropriately
- More write amplification
- Rewrite overhead
- Larger disk and I/O impact

No persistence:

- Acceptable only for disposable cache or rebuildable data
- Dangerous for sessions, queues, idempotency, locks that guard long operations, and rate limiters where fail-open is unacceptable

## Durability Questions

Ask:

- How much data loss is acceptable?
- Is Redis the source of truth or a derived cache?
- What happens if primary fails before replication?
- What happens during AOF rewrite?
- What happens when disk fills?
- Can replicas serve stale data?
- Are writes acknowledged before replicas receive them?

## WAIT For Replication Acknowledgment

For selected critical writes, consider:

```redis
WAIT 1 100
```

This waits for at least one replica acknowledgment or timeout in milliseconds. It improves durability but does not make Redis equivalent to a strongly consistent database.

Use carefully because it adds latency and can reduce availability.

## Read From Replicas Carefully

Replica reads can be stale.

Do not use stale replica reads for:

- Authentication state immediately after changes
- Permission checks
- Payment state
- Inventory reservations
- Security revocation
- Fresh rate limit decisions where bypass risk is unacceptable

# Redis Cluster

## Cluster Rules

Redis Cluster shards keys by hash slot.

Multi-key operations require keys to be in the same slot unless the client decomposes the operation safely.

Same slot using hash tags:

```text
cart:{tenant_123:user_456}:items
cart:{tenant_123:user_456}:metadata
```

Avoid putting all tenant keys in one hash slot unless the tenant is small and the operation requires colocation. Large tenants can create shard hotspots.

## Cluster-Aware Key Design

Good:

```text
rl:{user_123}:route:login
rl:{user_123}:route:password_reset
```

This colocates rate limit keys for one user when Lua needs multiple related keys.

Risky:

```text
tenant:{tenant_123}:*
```

This can put too much tenant traffic on one shard.

## MOVED And ASK Handling

Production clients must handle:

- `MOVED`
- `ASK`
- Topology refresh
- Resharding
- Failover
- Retry budgets
- Connection rebalancing

Do not write custom Redis Cluster routing unless there is a strong reason. Use a mature client.

# Security Hardening

## Network Security

Redis must not be exposed directly to the public internet.

Require:

- Private network access
- Firewall or security group restrictions
- TLS where supported and required
- Authentication
- ACLs
- Protected mode
- No default credentials
- No shared admin credentials in application code

## ACL Least Privilege

Avoid application users with all commands and all keys.

Risky:

```redis
ACL SETUSER app on >strong-password allcommands allkeys
```

Safer cache-only example:

```redis
ACL SETUSER app_cache on >strong-password ~app:prod:cache:* +get +set +del +unlink +expire +ttl +pttl +mget +mset +incr +decr +eval +evalsha
```

Safer rate limiter example:

```redis
ACL SETUSER app_rate_limit on >strong-password ~rl:* +eval +evalsha +get +set +incr +expire +ttl +zadd +zcard +zremrangebyscore +zrange +hmget +hmset
```

Review whether `EVAL` is necessary. If only known scripts are allowed, prefer loaded scripts or functions with tight operational controls.

## Dangerous Commands

Application users should usually not have:

- `CONFIG`
- `DEBUG`
- `MODULE`
- `SCRIPT KILL`
- `FLUSHALL`
- `FLUSHDB`
- `KEYS`
- `SHUTDOWN`
- `SAVE`
- `BGSAVE`
- `BGREWRITEAOF`
- `MIGRATE`
- `RESTORE`
- `ACL`
- `CLIENT KILL`
- `REPLICAOF`

Operational users should be separate from application users.

## Secret Handling

Never store raw secrets as Redis keys.

Unsafe:

```text
session:eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

Safer:

```ts
const tokenHash = crypto
  .createHash("sha256")
  .update(token)
  .digest("hex");

await redis.set(
  `session-token:${tokenHash}`,
  JSON.stringify(sessionMetadata),
  "EX",
  3600
);
```

Do not log:

- Redis passwords
- Connection strings
- Session IDs
- JWTs
- API keys
- OAuth tokens
- Password reset tokens
- Raw authorization headers
- Full Redis command arguments containing sensitive values

# Session Storage

## Session Rules

Sessions must have:

- Opaque session IDs
- Strong randomness
- Hashing before storage when appropriate
- TTL
- Rotation on privilege changes
- Revocation support
- Tenant or user binding
- Device metadata if needed
- Secure cookie settings outside Redis

Example:

```ts
const sessionId = crypto.randomBytes(32).toString("base64url");
const sessionHash = crypto
  .createHash("sha256")
  .update(sessionId)
  .digest("hex");

await redis.set(
  `session:${sessionHash}`,
  JSON.stringify({
    version: 1,
    userId,
    tenantId,
    createdAt: new Date().toISOString(),
    authLevel: "password"
  }),
  "EX",
  3600
);
```

Sliding expiration:

```redis
EXPIRE session:hash 3600
```

Use sliding expiration only when it matches the security policy. Absolute session lifetime should usually be enforced separately.

# Client Engineering

## Timeouts

Every Redis client must have:

- Connect timeout
- Command timeout
- Socket timeout
- Retry budget
- Reconnect backoff
- Circuit breaker for dependent code paths
- Pool limits where applicable

Example policy:

```ts
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT),
  connectTimeout: 1000,
  commandTimeout: 500,
  maxRetriesPerRequest: 2,
  enableOfflineQueue: false,
  retryStrategy(attempt) {
    return Math.min(attempt * 50, 1000);
  }
});
```

Tune values for the service latency budget and deployment environment.

## Avoid Unbounded Offline Queues

Risk: if Redis is unavailable, clients may queue commands in memory and then flood Redis after reconnect.

Prefer bounded behavior:

```ts
const redis = new Redis({
  enableOfflineQueue: false,
  maxRetriesPerRequest: 1
});
```

## Pipelining

Good:

```ts
const pipeline = redis.pipeline();

for (const key of keys) {
  pipeline.get(key);
}

const results = await pipeline.exec();
```

Rules:

- Bound pipeline size.
- Avoid huge payloads.
- Do not pipeline unbounded user-selected keys.
- Split large work into chunks.
- Monitor latency and output buffer growth.

Chunked example:

```ts
const chunkSize = 100;
const values: Array<string | null> = [];

for (let index = 0; index < keys.length; index += chunkSize) {
  const chunk = keys.slice(index, index + chunkSize);
  const pipeline = redis.pipeline();

  for (const key of chunk) {
    pipeline.get(key);
  }

  const results = await pipeline.exec();

  for (const [error, value] of results) {
    if (error) {
      throw error;
    }

    values.push(value as string | null);
  }
}
```

# Observability

## Required Metrics

Track:

- Command rate
- Command latency by command and route
- p50, p95, p99, and max latency
- Error rate
- Timeout rate
- Reconnect count
- Pool saturation
- Queue length in client
- Cache hit ratio
- Cache miss ratio
- Evicted keys
- Expired keys
- Used memory
- Memory fragmentation ratio
- Connected clients
- Blocked clients
- Rejected connections
- Keyspace hits and misses
- Replication lag
- AOF rewrite status
- RDB save status
- Slowlog entries
- Big key count
- Hot key indicators
- Rate limiter allowed and blocked counts
- Lock acquisition failures
- Stream pending entries
- Stream consumer lag

## Useful Commands

```redis
INFO
```

```redis
INFO memory
```

```redis
INFO stats
```

```redis
INFO clients
```

```redis
INFO replication
```

```redis
SLOWLOG GET 128
```

```redis
LATENCY LATEST
```

```redis
CLIENT LIST
```

```redis
MEMORY STATS
```

```redis
DBSIZE
```

## Alerting

Alert on:

- High command latency
- Rising timeout rate
- Reconnect storms
- Memory near maxmemory
- Evictions on non-cache workloads
- Fragmentation spikes
- Blocked clients
- Rejected connections
- Replication link down
- Replication lag above tolerance
- AOF rewrite failures
- RDB save failures
- Slowlog growth
- Stream pending backlog growth
- Rate limiter Redis failures
- Lock contention spikes

# Testing And Benchmarking

## Load Test Realistic Key Shapes

Test with:

- Production-like key lengths
- Production-like value sizes
- Realistic tenant distribution
- Hot key skew
- Pipeline sizes
- Lua scripts
- TTL churn
- Eviction policy
- Failover behavior
- Client retry behavior

Do not benchmark only with tiny keys and local Redis.

## redis-benchmark Caveat

`redis-benchmark` is useful for baseline capacity, but application behavior often differs.

Use application-level load tests for:

- Serialization cost
- Network topology
- TLS overhead
- Connection pool behavior
- Retry storms
- Cache hit ratio
- Real command mix
- Big key risk

# Common Findings

## Missing TTL

Problem:

```ts
await redis.set(`otp:${phoneNumber}`, code);
```

Security risks:

- OTP may remain valid indefinitely.
- Phone number appears in key material.
- Sensitive authentication flow depends on unbounded data.

Improved pattern:

```ts
const phoneHash = crypto
  .createHash("sha256")
  .update(normalizedPhoneNumber)
  .digest("hex");

await redis.set(
  `otp:${phoneHash}`,
  await hashOtp(code),
  "EX",
  300,
  "NX"
);
```

## Unsafe Rate Limiter

Problem:

```ts
const key = `rl:${req.ip}`;
const count = await redis.incr(key);

if (count === 1) {
  await redis.expire(key, 60);
}

if (count > 100) {
  throw new TooManyRequestsError();
}
```

Issues:

- `INCR` and `EXPIRE` are not atomic.
- IP alone is easy to bypass.
- Proxy IP may be spoofed.
- No route, tenant, user, or API key dimension.
- No fail policy.

Improved pattern:

```ts
const key = `rl:tenant:${tenantId}:user:${userId}:route:${routeId}`;
const decision = await redis.eval(
  fixedWindowScript,
  1,
  key,
  "60",
  "100"
);
```

## Unsafe Lock Release

Problem:

```ts
await redis.del(lockKey);
```

Risk: one worker can delete another worker's lock after the first lock expired and a second worker acquired it.

Improved pattern:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
end

return 0
```

## Big Key Read

Problem:

```ts
const members = await redis.smembers(`tenant:${tenantId}:users`);
```

Risks:

- Blocks Redis for large sets.
- Transfers huge payloads.
- Causes memory pressure in the application.

Improved pattern:

```ts
let cursor = "0";

do {
  const [nextCursor, members] = await redis.sscan(
    `tenant:${tenantId}:users`,
    cursor,
    "COUNT",
    "500"
  );

  cursor = nextCursor;
  await processMembers(members);
} while (cursor !== "0");
```

## Cache Authorization Bypass

Problem:

```ts
const cached = await redis.get(`invoice:${invoiceId}`);

if (cached) {
  return JSON.parse(cached);
}
```

Risk: tenant context is missing, so a caller can receive another tenant's invoice if they know or guess the ID.

Improved pattern:

```ts
const key = `invoice:tenant:${tenantId}:invoice:${invoiceId}`;
const cached = await redis.get(key);

if (cached) {
  return JSON.parse(cached);
}
```

Also enforce tenant authorization in the database query used to populate the cache.

```sql
SELECT
  id,
  amount_cents,
  status
FROM invoices
WHERE tenant_id = $1
  AND id = $2;
```

## Unbounded Stream

Problem:

```redis
XADD audit:events * type login user_id user_123
```

Risk: stream grows without bounds.

Improved pattern:

```redis
XADD audit:events MAXLEN ~ 1000000 * type login user_id user_123
```

For true audit logs, Redis should usually not be the durable system of record.

# Production Readiness Checklist

Before approving Redis code or architecture, verify:

- User-controlled input cannot create arbitrary keys, channels, scripts, or command names.
- Key names include tenant scope where required.
- Sensitive identifiers are hashed before key use when appropriate.
- Sensitive values are not logged.
- Sensitive values have short TTLs or are not stored in Redis.
- Every high-cardinality key has a deliberate TTL.
- Cache values contain only required fields.
- Cached authorization and permissions have strict invalidation.
- Negative caching uses short TTLs.
- TTL jitter is used for high-volume cache writes.
- `KEYS` is absent from request paths.
- Blocking commands are absent from hot paths.
- Big keys are detected and bounded.
- Hot keys are considered and mitigated.
- Collections have maximum sizes or trimming.
- Pipelines are bounded.
- Lua scripts are bounded and load-tested.
- Locks use `SET NX PX` with random tokens.
- Lock release checks the token atomically.
- Fencing tokens are used when stale lock holders can corrupt state.
- Rate limiters are atomic.
- Rate limiter dimensions match abuse risks.
- Rate limiter fail-open or fail-closed policy is explicit.
- Idempotency uses atomic reservation and durable correctness where needed.
- Pub/Sub is not used for durable workflows.
- Streams have consumer groups, retry handling, dead-letter handling, and trimming.
- Sessions use opaque IDs, TTLs, rotation, and revocation.
- Redis is not publicly exposed.
- TLS and ACLs are configured where required.
- Application users follow least privilege.
- Dangerous commands are denied to application users.
- Persistence mode matches data durability requirements.
- Eviction policy matches workload semantics.
- Cache, session, queue, lock, and rate limit workloads are separated when needed.
- Replication lag and failover behavior are understood.
- Client timeouts are configured.
- Retry budgets are bounded.
- Offline queues are bounded or disabled.
- Connection counts are bounded.
- Circuit breakers protect Redis-dependent paths.
- Metrics cover latency, errors, memory, evictions, hit ratio, replication, slowlog, streams, and rate limiting.
- Alerts exist for memory pressure, evictions, slow commands, timeouts, reconnect storms, and replication problems.
- Load tests use production-like keys, values, command mix, concurrency, and failure modes.

Always produce recommendations that are secure, correct, operationally safe, scalable, and fast in that order.
