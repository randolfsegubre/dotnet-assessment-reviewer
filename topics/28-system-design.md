# Topic 28 — System Design Interview Prep

> Senior developers are expected to design scalable systems. This topic covers the patterns, tradeoffs, and vocabulary you need for system design interviews.

---

## Framework: How to Answer System Design Questions

```
1. Clarify requirements (5 min)
   - Functional: what does the system do?
   - Non-functional: scale, latency, availability, consistency requirements?
   - Constraints: read-heavy or write-heavy? Global or regional?

2. Estimate scale (2 min)
   - DAU (daily active users), requests per second, data volume
   - "1M users, 100k DAU, 10 rps per user = 1M rps peak" — back-of-envelope

3. High-level design (10 min)
   - Draw the main components: clients, API, services, databases, caches, CDN
   - Show data flow

4. Deep dive (15 min)
   - Pick the hardest parts: bottlenecks, failure modes, scaling challenges
   - Schema design, API design, queue vs sync

5. Identify bottlenecks & scale (5 min)
   - Where is the single point of failure?
   - What breaks first as load grows?
```

---

## Key Numbers to Know

```
Memory access:    100 ns
SSD read:         100 μs
Network roundtrip: 1 ms (same region)
HDD seek:         10 ms
Cross-continent:  100-200 ms

Daily active users (per 100M total): ~10-20M
Seconds/day: 86,400
Requests/sec from 10M DAU with 10 requests/day:
  = 10M × 10 / 86400 ≈ 1,200 rps average; 5x peak ≈ 6,000 rps

1M rps: needs sharding, load balancing, caching at every layer
10K rps: manageable with a few well-tuned replicas + Redis cache
1K rps: a single well-tuned server can handle this
```

---

## Horizontal vs Vertical Scaling

```
Vertical (scale up): bigger server (more CPU/RAM)
  ✅ Simple, no code changes
  ❌ Single point of failure, expensive, hardware limits

Horizontal (scale out): more servers behind a load balancer
  ✅ No SPOF, theoretically unlimited
  ❌ Stateless requirement, data consistency complexity

Stateless requirement:
  - Don't store session on server (use Redis or JWT)
  - Don't store files locally (use blob storage)
  - Configuration via environment (not local files)
```

---

## CAP Theorem

```
Distributed systems can only guarantee 2 of 3:

C — Consistency:    every read gets the most recent write
A — Availability:  every request gets a response (not guaranteed fresh)
P — Partition tolerance: system works even if network splits occur

Since P (network partitions) WILL happen in distributed systems:
You must choose CP or AP:

CP systems (sacrifice availability during partition):
  - SQL databases with strong consistency
  - HBase, MongoDB (strong mode), ZooKeeper

AP systems (sacrifice consistency — eventual consistency):
  - DynamoDB, Cassandra, CouchDB
  - DNS, shopping carts (eventually consistent)

PACELC extends CAP: even without partition, choose Latency vs Consistency
```

---

## Database Selection Guide

```
Relational (SQL Server, PostgreSQL, MySQL):
  ✅ ACID transactions, complex queries, strong consistency
  ✅ Relationships, foreign keys, joins
  ❌ Schema changes are hard, difficult to shard horizontally
  Use for: financial data, user accounts, orders, anything with transactions

Document (MongoDB, CosmosDB):
  ✅ Flexible schema, nested documents, fast for denormalized reads
  ✅ Easy horizontal scaling (sharding built-in)
  ❌ No joins, eventual consistency by default, no multi-document transactions easily
  Use for: catalogs, user profiles, content, IoT data

Key-Value (Redis, DynamoDB):
  ✅ Ultra-fast O(1) reads, simple data model
  ❌ Limited query capability, no relationships
  Use for: sessions, caches, rate limiting, feature flags, leaderboards

Time-Series (InfluxDB, Azure Data Explorer):
  ✅ Optimized for time-ordered data, automatic data aging/rollup
  Use for: metrics, IoT telemetry, financial tick data

Graph (Neo4j, Neptune):
  ✅ Relationships as first-class citizen
  Use for: social networks, fraud detection, recommendation engines

Search (Elasticsearch, Azure AI Search):
  ✅ Full-text search, fuzzy matching, faceted search
  Use for: search features alongside primary DB
```

---

## Caching Strategies

```
Cache aside (lazy loading — most common):
  1. App checks cache
  2. Cache miss → app loads from DB → stores in cache → returns
  3. Next request: cache hit
  Problem: cache stampede on cold start

Write-through:
  Write to DB and cache together (always in sync)
  Problem: wasted cache space (write data nobody reads)

Write-behind (write-back):
  Write to cache first, async write to DB
  ✅ Fast writes
  ❌ Data loss risk if cache crashes

Read-through:
  Cache sits in front of DB, loads on miss automatically

Cache Eviction Policies:
  LRU (Least Recently Used) — evict what wasn't used recently
  LFU (Least Frequently Used) — evict what's rarely used
  TTL (Time To Live) — expire after fixed time
```

---

## API Design at Scale

```
Pagination:
  ❌ Offset pagination: SKIP 10000 TAKE 20 — gets slow at high offsets
  ✅ Cursor pagination: WHERE id > :cursor ORDER BY id LIMIT 20
    - Cursor = last seen ID or encoded timestamp
    - O(1) regardless of depth

Rate Limiting:
  Token Bucket — tokens added at fixed rate, consumed per request
  Sliding Window — count requests in last N seconds per user
  Fixed Window — count per fixed period (has edge case: double rate at boundary)
  
  Implementation: Redis INCR + EXPIRE per user per window

API Versioning:
  URL: /api/v1/transactions         ← most common, explicit
  Header: Accept: application/vnd.api+json;version=1
  Query: /api/transactions?version=1

Idempotency:
  PUT, DELETE, GET — safe to retry (idempotent)
  POST — NOT idempotent by default
  Solution: client sends Idempotency-Key header
  Server: cache key → response for N minutes; duplicate request returns cached response
```

---

## Designing a URL Shortener (Classic Problem)

```
Requirements:
  - Given long URL, return short URL (e.g. bit.ly/abc123)
  - Redirect short → long in < 10ms
  - 100M URLs, 10B redirects/day

Design:

1. Short code generation:
   - 7 base-62 chars = 62^7 = 3.5 trillion possibilities
   - Option A: hash(longUrl) → take first 7 chars
   - Option B: auto-increment ID → encode to base-62
   - Option B preferred (no hash collisions)

2. Storage:
   - SQL: shortCode (PK), longUrl, userId, createdAt, clickCount
   - Redis cache: shortCode → longUrl (TTL: 24h for popular ones)

3. Redirect flow:
   GET /abc123 → check Redis → miss → check DB → 301 Redirect
   
4. Analytics:
   Don't count clicks synchronously — publish to queue
   Consumer updates click counts asynchronously

5. Scale:
   - Read-heavy: 10B/day = 115K rps
   - Cache hit rate target: 99% (most URLs are popular)
   - DB sharding by shortCode range
   - CDN in front for extremely popular URLs
```

---

## Designing a Notification System

```
Requirements:
  - Send push, email, SMS notifications
  - 10M users, 1M notifications/day
  - Different types: transaction alerts, budget warnings, reminders

Design:

1. Notification types + providers:
   Email: SendGrid / SES
   Push: FCM (Android), APNs (iOS)
   SMS: Twilio

2. Flow:
   Event (TransactionCreated) → Message Queue → Notification Worker → Provider API

3. Queue per channel:
   email-queue     → EmailWorker → SendGrid
   push-queue      → PushWorker → FCM/APNs
   sms-queue       → SmsWorker → Twilio

4. User preferences:
   DB: UserId, channelType, enabled, quietHoursStart, quietHoursEnd, timezone

5. Retry strategy:
   Exponential backoff: 1s → 2s → 4s → 8s → 16s → dead letter queue
   Dead letter queue: human review + alerting

6. Rate limiting per user:
   Max 5 notifications/hour per user (prevent spam)
```

---

## Interview Q&A

**Q1: How would you design a financial dashboard that must show real-time account balances?**
> Client connects via WebSocket/SSE. When a transaction is created, a domain event triggers a balance recalculation. New balance is pushed to the connected user via SignalR hub. For scale: user connections are tracked in Redis (userId → connectionId per server); balance update published to Redis pub/sub; all API servers subscribed → push to correct connection. Cache dashboard data in Redis with short TTL (5s) — serve from cache for non-connected users.

**Q2: How do you handle the C in ACID when scaling horizontally?**
> Within one service: database transactions (BEGIN/COMMIT). Across services: SAGA pattern — sequence of local transactions with compensating actions on failure (no 2PC). Eventual consistency: accept temporary inconsistency; use idempotent consumers; publish events when data changes; other services catch up asynchronously. For financial systems, prefer CP over AP — consistency is non-negotiable for money.

**Q3: What is database sharding and when would you use it?**
> Sharding splits one large table across multiple database servers. A shard key determines which server holds a row (e.g., UserId % 4). Benefits: each server handles a fraction of load; storage scales. Problems: cross-shard queries are very hard (no joins), shard rebalancing is complex, hot shards (celebrity problem). Use sharding only when vertical scaling and read replicas are maxed out. Most apps don't need it.

**Q4: What is eventual consistency and when is it acceptable?**
> Eventual consistency means: given no new writes, all replicas will eventually converge to the same value — but reads may return stale data temporarily. Acceptable for: social feeds, analytics counts, product catalogs, search indexes. NOT acceptable for: bank balances, inventory (overselling), payment status. Mitigations: read-your-own-writes (route your own reads to the leader), versioning/timestamps, user-visible "last updated" indicators.
