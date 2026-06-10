# Engineering Decision Log

## Decision 1: PostgreSQL vs NoSQL

### Alternatives Considered
- PostgreSQL (RDBMS)
- MongoDB (Document store)
- DynamoDB (Key-value, AWS-native)

### Pros & Cons

**PostgreSQL:**
- ✅ ACID transactions (critical for bookings)
- ✅ Foreign keys prevent orphaned records
- ✅ Complex queries for analytics
- ✅ Mature tooling (pgAdmin, monitoring)
- ❌ Vertical scaling limits

**MongoDB:**
- ✅ Horizontal sharding
- ✅ Flexible schema
- ❌ No ACID across documents (before v4.2)
- ❌ Complex two-phase commit for consistency

**DynamoDB:**
- ✅ Serverless, infinite scale
- ✅ Single-digit millisecond latency
- ❌ No JOIN support
- ❌ Transaction cost (5x normal reads/writes)
- ❌ Expensive for scan-heavy queries

### Final Justification
**Choice: PostgreSQL**

Booking systems are transactional by nature. PostgreSQL's row-level locking, foreign key constraints, and CHECK constraints eliminate entire classes of bugs. The seat inventory fits in 32GB RAM (cache-friendly). NoSQL benefits (horizontal scaling) don't outweigh ACID guarantees for this use case.

---

## Decision 2: Redis for Distributed Locking

### Alternatives Considered
- PostgreSQL SELECT FOR UPDATE
- Redis SETNX
- Memcached (no lock primitive)
- ZooKeeper/etcd (coordination service)

### Pros & Cons

**PostgreSQL Locks:**
- ✅ Native to data store
- ✅ Perfect consistency
- ❌ Connection pool exhaustion (4,400 needed at peak)

**Redis SETNX:**
- ✅ 100K+ ops/sec throughput
- ✅ Automatic TTL expiry
- ✅ Lua scripts for atomic unlock
- ❌ Requires separate service

**ZooKeeper:**
- ✅ Leader election, strong consistency
- ❌ Overkill for this use case
- ❌ Additional ops complexity

### Final Justification
**Choice: Redis SETNX**

PostgreSQL locks scale to ~3K RPS before connection exhaustion. Redis handles 25K RPS with headroom. The hybrid approach (Redis locks + Postgres persistence) provides both throughput AND consistency. Lock failures are fast (NOWAIT semantics), enabling quick retries.

---

## Decision 3: Async Payment Processing

### Alternatives Considered
- Synchronous payment (block until gateway responds)
- Message queue (SQS + Lambda)
- Background job processor (Celery, Bull)

### Pros & Cons

**Synchronous:**
- ✅ Simple code flow
- ❌ Holds connections for 2-5s (kills throughput)
- ❌ No retry on failure
- ❌ Client timeout issues

**SQS + Lambda:**
- ✅ Auto-scaling workers
- ✅ Built-in retry + DLQ
- ✅ Pay-per-invocation (cost-efficient)
- ❌ Cold start latency (mitigated with reserved concurrency)

**Self-hosted Queue (Bull/Redis):**
- ✅ More control
- ❌ Need to manage workers
- ❌ Scaling complexity

### Final Justification
**Choice: SQS + Lambda**

Payment gateways are the slowest component (3s P99). Async processing decouples booking reservation from payment confirmation. SQS provides infinite buffer, Lambda scales automatically. Cost: $150/month for 5K payments (within budget). Synchronous would require 15K connections—infeasible.

---

## Decision 4: Read Replicas for Analytics

### Alternatives Considered
- Single primary for all queries
- Read replicas (async replication)
- Separate data warehouse (Redshift)

### Pros & Cons

**Single Primary:**
- ✅ No replication lag
- ❌ Analytics queries starve booking writes

**Read Replicas:**
- ✅ Offload read traffic (dashboards, reports)
- ✅ Cost-effective (t4g.large vs r5.xlarge)
- ❌ Replication lag (5-10s acceptable for analytics)

**Redshift:**
- ✅ Columnar storage for OLAP
- ❌ $500+/month (over budget)
- ❌ Overkill for this scale

### Final Justification
**Choice: 2x Read Replicas**

Booking writes target primary. Admin dashboards, user history, and event listings read from replicas. Saves $300/month vs scaling primary. Replication lag doesn't affect critical path (seat availability queries hit primary).

---

## Decision 5: Cache Strategy (Cache-Aside)

### Alternatives Considered
- No cache (direct DB queries)
- Write-through cache
- Cache-aside (lazy load)

### Pros & Cons

**No Cache:**
- ✅ Simple, no consistency issues
- ❌ DB overload (25K RPS × 20ms = 500 connections)

**Write-Through:**
- ✅ Cache always fresh
- ❌ Slower writes (update cache + DB)
- ❌ Cache pollution (one-time reads cached)

**Cache-Aside:**
- ✅ Only cache hot data
- ✅ Fast reads (Redis <5ms)
- ❌ Cache stampede risk (mitigated with locking)

### Final Justification
**Choice: Cache-Aside**

Event details are read 1000x more than written. Lazy loading avoids caching unpopular events. Short TTL (60s) for seat counts prevents stale data during rush. Individual seat status NOT cached (consistency nightmare). Redis reduces DB load by 90%.

---

## Decision 6: UUID vs SERIAL Primary Keys

### Alternatives Considered
- SERIAL (auto-increment integers)
- UUID v4 (random)

### Pros & Cons

**SERIAL:**
- ✅ Compact (8 bytes)
- ✅ Sequential (B-tree friendly)
- ❌ Enumeration attacks (user IDs guessable)
- ❌ Merge conflicts in distributed writes

**UUID:**
- ✅ Globally unique (safe for sharding)
- ✅ Non-guessable (security)
- ❌ 16 bytes (larger indexes)
- ❌ Random (less cache-friendly)

### Final Justification
**Choice: UUID (gen_random_uuid)**

Security matters for user-facing IDs. Prevents account enumeration. Future-proofs for multi-region sharding (single sequence generator becomes bottleneck). Index size cost (8GB vs 4GB) is acceptable at this scale.

---

## Decision 7: Node.js vs Other Runtimes

### Alternatives Considered
- Node.js (Express)
- Go (Gin)
- Java (Spring Boot)

### Pros & Cons

**Node.js:**
- ✅ Event-driven, non-blocking I/O
- ✅ Fast prototyping (large ecosystem)
- ❌ Single-threaded (CPU-bound tasks slow)

**Go:**
- ✅ Compiled, low memory
- ✅ Goroutines (excellent concurrency)
- ❌ Smaller ecosystem

**Java:**
- ✅ Battle-tested for high scale
- ❌ High memory usage (1GB+ per instance)
- ❌ Longer startup time

### Final Justification
**Choice: Node.js**

Booking API is I/O-bound (DB + Redis + payment gateway). Node.js async model fits perfectly. Rapid development (tight deadline). t3.medium instances handle 2K RPS each (sufficient). For CPU-heavy analytics, offload to Lambda (Python/Go).
