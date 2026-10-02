# Part 97: System Design & Technical Interview Patterns
## ขั้นตอนที่ 6691-6760: System Design Questions, Architecture Decisions, Senior-Level Answers

---

## 97.1 System Design Framework

```
How to Approach System Design:

1. Clarify Requirements (5 min)
   Functional: what must the system do?
   Non-functional: scale, latency, availability, consistency
   
2. Estimate Scale (3 min)
   Users: 10M DAU
   Reads: 100M/day = ~1,200 RPS
   Writes: 1M/day = ~12 RPS
   Storage: 1KB per record × 1M records/day × 365 × 3yr = ~1TB

3. High-Level Design (10 min)
   Draw boxes: client → LB → API → DB
   Identify key components

4. Deep Dive (15 min)
   Database choice and schema
   Caching strategy
   API design
   Failure modes

5. Bottlenecks & Trade-offs (5 min)
   Single points of failure
   Scale-out strategy
   Data consistency trade-offs
```

---

## 97.2 Design: URL Shortener (bit.ly)

```
Requirements:
  Functional:
    - Shorten URL: POST /shorten → short URL
    - Redirect: GET /{code} → 301 redirect
    - Analytics: view count per URL
  
  Non-Functional:
    - 100M URLs created/day
    - 10B redirect requests/day = ~115,000 RPS
    - Low latency: < 10ms for redirect
    - 99.99% availability

Scale Estimation:
  Write: 100M/day = 1,157 RPS
  Read: 10B/day = 115,741 RPS (100:1 read:write)
  Storage: 500 bytes/URL × 100M/day × 5yr = 91TB

Short Code Generation:
  Options:
    1. Random 7 chars (base62: a-z A-Z 0-9)
       62^7 = 3.5 trillion combinations
    2. Hash (MD5/SHA256) → take first 7 chars
       Problem: collision possible
    3. Auto-increment ID → encode in base62
       ID 1000000 → base62 → "4c92"
       Better: no collision, deterministic
    
  Best: Snowflake-like ID → base62 encode
  
Architecture:
  
  Client → CDN (cache redirects) → LB → URL Service → Cache → DB
  
  URL Service:
    - POST /shorten: generate code, store in DB + cache
    - GET /{code}: check cache → DB → redirect
  
  Cache (Redis):
    Key: short-code, Value: long-URL
    TTL: 24h for popular URLs
    Eviction: LRU
    
  Database:
    - Write: single master (writes aren't that high)
    - Read: multiple read replicas
    - Schema:
      url_mappings(id BIGINT, short_code VARCHAR(8), long_url TEXT,
                   user_id BIGINT, created_at TIMESTAMP, expires_at TIMESTAMP)
    - Index: short_code (primary lookup)

Analytics:
  Don't count in write path (latency)
  Publish click event to Kafka
  Kafka → Flink/Kafka Streams → analytics DB
  
Redirect optimization:
  301 Moved Permanently: browser caches → less load but no analytics
  302 Found: no caching → more load but accurate analytics
  Use 302 with CDN caching at edge (configurable TTL)
```

---

## 97.3 Design: Rate Limiter

```kotlin
// System Design: Rate Limiter (asked very frequently)

// Algorithms:
// Token Bucket: tokens refilled at rate R, max capacity C
// Leaky Bucket: process at constant rate, buffer excess
// Fixed Window: count per fixed interval (12:00-12:01)
// Sliding Window Log: exact but memory intensive
// Sliding Window Counter: approximate, efficient

// ====== Token Bucket (Redis Lua Script) ======
// Atomic: read + modify in one operation

val tokenBucketScript = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refillRate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    local requested = tonumber(ARGV[4])
    
    local data = redis.call("HMGET", key, "tokens", "lastRefill")
    local tokens = tonumber(data[1]) or capacity
    local lastRefill = tonumber(data[2]) or now
    
    -- Refill tokens based on elapsed time
    local elapsed = math.max(0, now - lastRefill)
    local newTokens = math.min(capacity, tokens + (elapsed * refillRate / 1000))
    
    if newTokens >= requested then
        -- Allow: consume tokens
        redis.call("HMSET", key, "tokens", newTokens - requested, "lastRefill", now)
        redis.call("EXPIRE", key, 3600)
        return {1, math.floor(newTokens - requested)}  -- allowed, remaining
    else
        -- Deny: not enough tokens
        redis.call("HMSET", key, "tokens", newTokens, "lastRefill", now)
        redis.call("EXPIRE", key, 3600)
        return {0, math.floor(newTokens)}  -- denied, remaining
    end
"""

// Multi-layer rate limiting
@Component
class RateLimiterService(private val redisTemplate: ReactiveStringRedisTemplate) {
    
    // Different limits per tier
    private val limits = mapOf(
        "free" to RateLimit(100, Duration.ofMinutes(1)),
        "basic" to RateLimit(1000, Duration.ofMinutes(1)),
        "premium" to RateLimit(10000, Duration.ofMinutes(1)),
        "internal" to RateLimit(1000000, Duration.ofMinutes(1))
    )
    
    data class RateLimit(val requests: Int, val window: Duration)
    data class RateLimitResult(val allowed: Boolean, val remaining: Int, val resetAt: Instant)
    
    fun checkLimit(
        identifier: String,  // userId or apiKey or IP
        tier: String = "free",
        endpoint: String? = null
    ): RateLimitResult {
        val limit = limits[tier] ?: limits["free"]!!
        
        // Per-endpoint limits (more granular)
        val key = if (endpoint != null) 
            "rl:$identifier:$endpoint"
        else 
            "rl:$identifier"
        
        val script = RedisScript.of(tokenBucketScript, List::class.java)
        val result = redisTemplate.execute(
            script,
            listOf(key),
            listOf(
                limit.requests.toString(),
                (limit.requests / limit.window.seconds).toString(),
                System.currentTimeMillis().toString(),
                "1"  // requesting 1 token
            )
        )
        
        val allowed = (result as List<*>)[0] == 1L
        val remaining = ((result)[1] as Long).toInt()
        
        return RateLimitResult(
            allowed = allowed,
            remaining = remaining,
            resetAt = Instant.now().plus(limit.window)
        )
    }
}

// Architecture for distributed rate limiting:
/*
Option 1: Centralized Redis
  + Simple, accurate
  - Single point of failure, latency

Option 2: Local counter + global sync
  Each node tracks locally
  Periodically sync to Redis
  + Low latency
  - Slightly inaccurate (brief race window)

Option 3: Sticky routing (consistent hash by user)
  + Each user goes to same server
  + No sync needed, local state
  - Less even load distribution
*/
```

---

## 97.4 Design: Notification Service

```
Requirements:
  Send notifications via: email, SMS, push notification
  10M notifications/day across all channels
  Retry failed notifications (up to 3 times)
  Track delivery status
  Template system for notification content
  
Components:

1. API Service (notification-service)
   POST /notifications - enqueue notification
   GET /notifications/{id}/status - check status
   
2. Message Queue (Kafka)
   Topic per channel: email-notifications, sms-notifications, push-notifications
   Partitioned by user_id for ordering
   
3. Channel Workers
   email-worker: email-notifications → SendGrid/SES
   sms-worker: sms-notifications → Twilio
   push-worker: push-notifications → FCM/APNS
   
4. Retry Service
   Dead Letter Queue for failed messages
   Exponential backoff: 1min, 5min, 30min
   
5. Template Service
   Notification templates (Mustache/Thymeleaf)
   Localization (i18n)
   
6. Status DB (PostgreSQL + Redis)
   Track: PENDING, SENT, DELIVERED, FAILED
   Real-time status in Redis, persist to Postgres

Flow:
  App → POST /notify → Validate + Persist → Kafka
  Kafka → Channel Worker → External API → Update status
  Worker fails → DLQ → Retry service → re-queue with backoff

Scale:
  10M/day = ~116/sec (peak 5x = 580/sec)
  Each channel worker: 3-5 replicas
  Email rate limits: SendGrid ~100/sec per API key
  → Partition Kafka by sender domain for email
```

---

## 97.5 Java/Kotlin Interview Questions

```kotlin
// ====== Concurrency ======

// Q: Difference between synchronized, ReentrantLock, volatile?
// A:
//   synchronized: JVM intrinsic lock, block-level, no timeout
//   ReentrantLock: explicit lock, tryLock(timeout), fair/unfair, interruptible
//   volatile: visibility guarantee (no atomicity), variable reads always from main memory
//   AtomicInteger: compare-and-swap (CAS), no lock, best for counters
//   
//   Use: ReentrantLock when you need tryLock() or lockInterruptibly()
//        synchronized for simple block protection
//        volatile for flags that one thread writes, many read
//        Atomic* for counters and simple compare-and-swap

// Q: How does HashMap work?
// Array + LinkedList (chaining) for collision
// Java 8+: treeify bucket if > 8 entries (O(n) → O(log n))
// Load factor 0.75: resize at 75% capacity
// Initial capacity 16, doubles on resize
// hashCode() → spread bits → index into array

// Q: What is CompletableFuture?
// Asynchronous computation with callbacks
// Non-blocking unlike Future.get()

val result: CompletableFuture<String> = CompletableFuture
    .supplyAsync { fetchFromDB() }           // async
    .thenApply { data -> process(data) }     // transform
    .thenCombine(fetchFromApi()) { d1, d2 -> combine(d1, d2) }
    .exceptionally { ex -> "fallback" }

// ====== Kotlin Coroutines Interview ======

// Q: Difference between launch and async?
//   launch: fire and forget, returns Job
//   async: concurrent computation, returns Deferred<T>
//   Both run concurrently with coroutineScope

// Q: What is structured concurrency?
// Parent coroutine scope encompasses child coroutines
// If parent cancels → all children cancel
// If child fails → parent fails (unless SupervisorJob)
// Children must complete before parent completes

// Q: Flow vs Channel?
//   Flow: cold stream (starts when collected), suspend fun
//   Channel: hot stream, can send/receive from different coroutines
//   StateFlow: always has value, replays latest to new collectors
//   SharedFlow: configurable replay, hot broadcast

// ====== Spring Boot Interview ======

// Q: How does @Transactional work?
// Spring creates a proxy around the bean
// Proxy: begin tx → call method → commit/rollback
// Only works on public methods called from outside the class!
// Self-invocation (this.method()) bypasses proxy!

// Q: What's the difference between @Component, @Service, @Repository?
// All register beans, @Service and @Repository are specializations
// @Repository: DataAccessException translation (wrap SQL exceptions)
// @Service: no extra behavior, semantic marker
// @Controller/@RestController: web layer, request mapping

// Q: Explain Bean Scopes
//   singleton (default): one instance per ApplicationContext
//   prototype: new instance each time requested
//   request: one per HTTP request (web only)
//   session: one per HTTP session (web only)
//   application: one per ServletContext

// ====== Database Questions ======

// Q: ACID properties?
//   Atomicity: all-or-nothing (transaction commits or rolls back)
//   Consistency: data always in valid state (constraints satisfied)
//   Isolation: concurrent transactions don't interfere
//   Durability: committed data persists even after crash

// Q: Isolation Levels (PostgreSQL default = READ COMMITTED)?
//   READ UNCOMMITTED: can read uncommitted data (dirty read)
//   READ COMMITTED: only committed data (default in most DBs)
//   REPEATABLE READ: same query returns same results within transaction
//   SERIALIZABLE: transactions as if serial (most strict, slowest)

// Q: Index internals?
//   B-Tree (default): good for range queries, equality
//   Hash: equality only, faster than B-Tree for exact match
//   GiST/GIN: complex types (geometry, text search, arrays)
//   Partial index: WHERE clause in index definition
//   Covering index: INCLUDE all needed columns (index-only scan)
```

---

## 97.6 Senior Engineer Answers

```kotlin
// ====== Architect-Level Thinking ======

// Q: "How would you migrate our monolith to microservices?"
/*
Answer framework:

1. Assessment first (don't just start splitting)
   - Map current monolith: bounded contexts, team ownership
   - Identify coupling hotspots (what changes together?)
   - Identify scale bottlenecks (what's the slowest, most loaded part?)

2. Start with Strangler Fig, not big bang rewrite
   - Extract services incrementally
   - Priority: highest value, lowest coupling first
   
3. Common extraction order:
   User auth → product catalog → search → order processing → payment
   
4. Each extraction needs:
   - API gateway routing rule
   - Data ownership decision (DB per service)
   - Eventual consistency strategy
   - Monitoring for the new boundary

5. Only split if:
   - Different scaling needs
   - Different deployment cycles
   - Team autonomy (Conway's Law!)
   - Technical necessity
   
6. Avoid:
   - Nano-services (too fine-grained)
   - Shared databases (tight coupling)
   - Synchronous chains (distributed monolith)
*/

// Q: "Our API is slow, how do you investigate?"
/*
Systematic approach:

1. Measure first (before guessing)
   - Distributed traces: Jaeger/Zipkin → where is time spent?
   - Flame graphs: CPU profiling → what code is hot?
   - Database: pg_stat_statements → which queries are slow?
   - Thread dump: jstack → are threads waiting?

2. Check in order:
   - Network: is it latency to caller? Check from same data center
   - Application: profiling shows CPU-bound code?
   - Database: N+1 queries? Missing index? Long transactions?
   - External calls: third-party APIs slow?
   - GC pauses: long GC causing latency spikes?

3. Common fixes:
   - N+1: use JOIN or @BatchSize
   - Missing index: EXPLAIN ANALYZE shows Seq Scan
   - Cache: add Redis for frequently read data  
   - Async: move slow ops to background job
   - Connection pool: increase if "waiting for connection" in traces
*/

// Q: "How do you ensure data consistency across microservices?"
/*
Answer: depends on consistency requirement

1. Strong consistency (e.g., bank transfers):
   - Saga pattern with compensating transactions
   - Or: keep in same service/DB (don't split if ACID needed!)
   
2. Eventual consistency (e.g., update search index after product save):
   - Transactional Outbox: DB write + outbox in same transaction
   - Saga choreography via events
   - Accept brief inconsistency
   
3. Read-your-writes (user sees own updates):
   - Route user to same replica temporarily
   - Or: cache write in session, read from cache first
   
4. Monitoring:
   - Reconciliation jobs: detect + fix inconsistencies
   - Idempotent retry: safe to retry failed events
*/
```

---

## สรุป Part 97

```
System Design Interview:

Framework:
  1. Clarify (functional + non-functional)
  2. Estimate (users, RPS, storage)
  3. High-level design
  4. Deep dive (DB, cache, API)
  5. Trade-offs & bottlenecks

Common Systems:
  URL Shortener: cache, hash, 301 vs 302
  Rate Limiter: token bucket, Redis Lua, distributed
  Notification: Kafka channels, retry, templates
  Social Feed: fanout on write vs read

Java Interview Topics:
  Concurrency: synchronized vs Lock vs volatile vs Atomic
  HashMap: array + chaining, treeify, load factor
  Spring: @Transactional proxy, bean scopes, DI internals
  JVM: GC algorithms, heap/stack, class loading

Architecture Questions:
  Monolith → Microservices: Strangler Fig, not big bang
  Performance: measure first, systematic investigation  
  Data consistency: Saga, Outbox, eventual consistency
  CAP theorem: pick 2 of 3 (CP or AP, never CA with partition)

Key Trade-offs to Know:
  SQL vs NoSQL: consistency vs scale/flexibility
  Cache-aside vs write-through: simplicity vs consistency
  Sync vs async: latency vs reliability
  Microservices vs monolith: autonomy vs simplicity
  Strong vs eventual consistency: correctness vs availability
```

➡️ [Part 98: Code Review Best Practices & Anti-patterns](./Part-98-CodeReview.md)
