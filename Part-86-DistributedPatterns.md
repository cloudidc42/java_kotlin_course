# Part 86: Distributed Systems Patterns
## ขั้นตอนที่ 5921-5990: Idempotency, Rate Limiting, Distributed Lock, Leader Election

---

## 86.1 Idempotency

```
Idempotency:
  การส่ง request เดิมหลายครั้ง ผลลัพธ์เหมือนกัน
  
  GET, PUT, DELETE = idempotent by definition (HTTP spec)
  POST, PATCH = NOT idempotent by default → ต้องทำให้ idempotent

ทำไมต้องทำ:
  Network retry → request ส่ง 2 ครั้ง
  Client timeout → ไม่รู้ว่าสำเร็จหรือเปล่า
  Load balancer retry
  
  ถ้าไม่ idempotent:
    POST /orders → create order → timeout
    Client retry → create SECOND order!

Idempotency Key:
  Client ส่ง unique key ใน header
  Server: if key exists → return cached response
           if key new → process + cache response
```

---

## 86.2 Idempotency Implementation

```kotlin
@Entity
data class IdempotencyRecord(
    @Id val idempotencyKey: String,
    val requestHash: String,
    
    @Column(columnDefinition = "jsonb")
    val responseBody: String,
    
    val responseStatus: Int,
    val createdAt: java.time.Instant = java.time.Instant.now(),
    val expiresAt: java.time.Instant = java.time.Instant.now().plus(24, ChronoUnit.HOURS)
)

@Component
class IdempotencyFilter(
    private val idempotencyRepository: IdempotencyRepository,
    private val objectMapper: ObjectMapper
) : OncePerRequestFilter() {
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val key = request.getHeader("Idempotency-Key")
        
        if (key == null || request.method !in listOf("POST", "PATCH")) {
            filterChain.doFilter(request, response)
            return
        }
        
        // Check if we've seen this key before
        idempotencyRepository.findById(key)?.let { record ->
            // Return cached response
            response.status = record.responseStatus
            response.contentType = "application/json"
            response.writer.write(record.responseBody)
            return
        }
        
        // Capture response
        val wrappedResponse = ContentCachingResponseWrapper(response)
        filterChain.doFilter(request, wrappedResponse)
        
        // Save idempotency record
        val responseBody = String(wrappedResponse.contentAsByteArray)
        idempotencyRepository.save(IdempotencyRecord(
            idempotencyKey = key,
            requestHash = hashRequest(request),
            responseBody = responseBody,
            responseStatus = wrappedResponse.status
        ))
        
        wrappedResponse.copyBodyToResponse()
    }
    
    private fun hashRequest(request: HttpServletRequest): String =
        // Hash: method + path + body
        java.security.MessageDigest.getInstance("SHA-256")
            .digest("${request.method}${request.requestURI}".toByteArray())
            .let { java.util.Base64.getEncoder().encodeToString(it) }
}
```

---

## 86.3 Rate Limiting

```kotlin
// Token Bucket Algorithm with Redis

@Component
class RateLimiter(private val redisTemplate: StringRedisTemplate) {
    
    // Token bucket: allow burst but maintain average rate
    fun isAllowed(key: String, maxRequests: Int, windowSeconds: Long): Boolean {
        val script = """
            local key = KEYS[1]
            local max = tonumber(ARGV[1])
            local window = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            
            -- Get current count and last reset time
            local count = tonumber(redis.call('GET', key .. ':count') or '0')
            local resetAt = tonumber(redis.call('GET', key .. ':reset') or '0')
            
            -- Reset if window expired
            if now > resetAt then
                count = 0
                resetAt = now + window
                redis.call('SET', key .. ':reset', resetAt, 'EX', window)
            end
            
            -- Check limit
            if count >= max then
                return {0, resetAt - now}
            end
            
            -- Increment
            redis.call('INCR', key .. ':count')
            redis.call('EXPIRE', key .. ':count', window)
            return {1, 0}
        """
        
        val result = redisTemplate.execute(
            DefaultRedisScript(script, List::class.java),
            listOf(key),
            maxRequests.toString(),
            windowSeconds.toString(),
            (System.currentTimeMillis() / 1000).toString()
        ) as List<*>
        
        return result[0] == 1L
    }
    
    // Sliding window with sorted sets
    fun isAllowedSlidingWindow(key: String, maxRequests: Int, windowMillis: Long): Boolean {
        val now = System.currentTimeMillis()
        val windowStart = now - windowMillis
        
        return redisTemplate.execute { conn ->
            conn.multi()
            // Remove old entries
            conn.zRemRangeByScore(key.toByteArray(), 0.0, windowStart.toDouble())
            // Add current request
            conn.zAdd(key.toByteArray(), now.toDouble(), now.toString().toByteArray())
            // Count requests in window
            conn.zCount(key.toByteArray(), windowStart.toDouble(), now.toDouble())
            // Set expiry
            conn.expire(key.toByteArray(), windowMillis / 1000)
            conn.exec()
        }?.let { results ->
            val count = results[2] as Long
            count <= maxRequests
        } ?: true
    }
}

// Rate limit interceptor
@Component
class RateLimitInterceptor(
    private val rateLimiter: RateLimiter
) : HandlerInterceptor {
    
    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any
    ): Boolean {
        val apiKey = request.getHeader("X-API-Key") ?: "anonymous"
        val clientIp = getClientIp(request)
        
        // Per API key: 1000 requests/minute
        if (!rateLimiter.isAllowed("api:$apiKey", 1000, 60)) {
            response.status = 429
            response.addHeader("Retry-After", "60")
            response.addHeader("X-RateLimit-Limit", "1000")
            response.writer.write("""{"error":"Rate limit exceeded"}""")
            return false
        }
        
        // Per IP: 100 requests/minute (anonymous protection)
        if (!rateLimiter.isAllowed("ip:$clientIp", 100, 60)) {
            response.status = 429
            response.writer.write("""{"error":"Rate limit exceeded"}""")
            return false
        }
        
        return true
    }
    
    private fun getClientIp(request: HttpServletRequest): String =
        request.getHeader("X-Forwarded-For")?.split(",")?.first()?.trim()
            ?: request.remoteAddr
}
```

---

## 86.4 Distributed Lock with Redis

```kotlin
// Redisson-based distributed lock
import org.redisson.api.RedissonClient

@Service
class DistributedLockService(private val redissonClient: RedissonClient) {
    
    fun <T> withLock(
        lockName: String,
        waitTimeSec: Long = 10,
        leaseTimeSec: Long = 30,
        block: () -> T
    ): T {
        val lock = redissonClient.getLock(lockName)
        
        val acquired = lock.tryLock(waitTimeSec, leaseTimeSec, TimeUnit.SECONDS)
        if (!acquired) {
            throw LockNotAcquiredException("Could not acquire lock: $lockName")
        }
        
        return try {
            block()
        } finally {
            if (lock.isHeldByCurrentThread) {
                lock.unlock()
            }
        }
    }
    
    // Fair lock: first-come-first-served
    fun <T> withFairLock(lockName: String, block: () -> T): T {
        val lock = redissonClient.getFairLock(lockName)
        lock.lock(30, TimeUnit.SECONDS)
        return try { block() } finally { lock.unlock() }
    }
    
    // Read-Write lock: multiple readers, exclusive writer
    fun <T> withReadLock(lockName: String, block: () -> T): T {
        val rwLock = redissonClient.getReadWriteLock(lockName)
        val readLock = rwLock.readLock()
        readLock.lock(30, TimeUnit.SECONDS)
        return try { block() } finally { readLock.unlock() }
    }
    
    fun <T> withWriteLock(lockName: String, block: () -> T): T {
        val rwLock = redissonClient.getReadWriteLock(lockName)
        val writeLock = rwLock.writeLock()
        writeLock.lock(30, TimeUnit.SECONDS)
        return try { block() } finally { writeLock.unlock() }
    }
}

// Annotation-based distributed lock
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class DistributedLock(
    val key: String,           // SpEL expression: "#orderId"
    val waitTimeSec: Long = 5,
    val leaseTimeSec: Long = 30
)

@Aspect
@Component
class DistributedLockAspect(
    private val distributedLockService: DistributedLockService,
    private val spelParser: SpelExpressionParser = SpelExpressionParser()
) {
    @Around("@annotation(lock)")
    fun around(joinPoint: ProceedingJoinPoint, lock: DistributedLock): Any? {
        val key = resolveKey(lock.key, joinPoint)
        
        return distributedLockService.withLock(
            lockName = "lock:$key",
            waitTimeSec = lock.waitTimeSec,
            leaseTimeSec = lock.leaseTimeSec
        ) {
            joinPoint.proceed()
        }
    }
    
    private fun resolveKey(expression: String, joinPoint: ProceedingJoinPoint): String {
        val context = MethodBasedEvaluationContext(
            joinPoint.target,
            (joinPoint.signature as MethodSignature).method,
            joinPoint.args,
            ParameterNameDiscoverer()
        )
        return spelParser.parseExpression(expression).getValue(context, String::class.java)!!
    }
}

// Usage
@Service
class FlashSaleService {
    
    @DistributedLock(key = "'flash-sale:' + #productId")
    fun purchaseFlashSale(productId: String, userId: String): Order {
        val stock = flashSaleRepository.getStock(productId)
        if (stock <= 0) throw OutOfStockException(productId)
        
        flashSaleRepository.decrementStock(productId)
        return orderService.createOrder(userId, productId, 1)
    }
}
```

---

## 86.5 Leader Election

```kotlin
// Leader election using Redis (for scheduled tasks)
@Component
class LeaderElection(private val redisTemplate: StringRedisTemplate) {
    
    private val instanceId = java.util.UUID.randomUUID().toString()
    
    fun tryAcquireLeadership(electionKey: String, ttlSeconds: Long = 30): Boolean {
        val result = redisTemplate.opsForValue()
            .setIfAbsent(electionKey, instanceId, Duration.ofSeconds(ttlSeconds))
        return result == true
    }
    
    fun isLeader(electionKey: String): Boolean =
        redisTemplate.opsForValue().get(electionKey) == instanceId
    
    fun renewLeadership(electionKey: String, ttlSeconds: Long = 30): Boolean {
        val script = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('EXPIRE', KEYS[1], ARGV[2])
            else
                return 0
            end
        """
        val result = redisTemplate.execute(
            DefaultRedisScript(script, Long::class.java),
            listOf(electionKey),
            instanceId,
            ttlSeconds.toString()
        )
        return result == 1L
    }
    
    fun releaseLeadership(electionKey: String): Boolean {
        val script = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end
        """
        val result = redisTemplate.execute(
            DefaultRedisScript(script, Long::class.java),
            listOf(electionKey),
            instanceId
        )
        return result == 1L
    }
}

// Scheduled task that only runs on leader
@Component
class LeaderOnlyScheduler(
    private val leaderElection: LeaderElection,
    private val reportService: ReportService
) {
    @Scheduled(fixedDelay = 25000)   // renew every 25s (TTL = 30s)
    fun renewLeaderLease() {
        leaderElection.renewLeadership("scheduler-leader")
    }
    
    @Scheduled(cron = "0 0 * * * *")   // every hour
    fun generateHourlyReport() {
        val ELECTION_KEY = "hourly-report-leader"
        
        if (leaderElection.tryAcquireLeadership(ELECTION_KEY, 300)) {
            try {
                reportService.generateHourlyReport()
            } finally {
                leaderElection.releaseLeadership(ELECTION_KEY)
            }
        }
        // Non-leaders skip this
    }
}
```

---

## สรุป Part 86

```
Distributed Systems Patterns:

Idempotency:
  Client: send Idempotency-Key header
  Server: check key → return cached response OR process + cache
  Expiry: clear old records after 24h
  Use: POST payment, order creation

Rate Limiting:
  Token Bucket: allow burst, then steady rate
  Sliding Window: most accurate, more Redis ops
  Fixed Window: simplest, boundary spike issue
  
  Implementation: Redis + Lua script (atomic)
  Headers: X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After

Distributed Lock:
  Redis SET NX EX (setIfAbsent + TTL)
  Redisson: handles edge cases (node failure, lease renewal)
  Always: try-lock with timeout (don't wait forever)
  Always: release in finally block
  Read-Write lock: concurrent reads, exclusive writes

Leader Election:
  Redis SET NX: first writer wins
  Renew before TTL expires
  Release on shutdown
  
  Use for:
    ✓ Scheduled jobs (run once across replicas)
    ✓ Singletons in distributed environment
    ✓ Consumer group leader

Idempotency Key Generation (client-side):
  UUID.randomUUID() per request attempt
  Store locally, reuse on retry
  Clear after confirmed success
```

➡️ [Part 87: Kotlin Multiplatform](./Part-87-KotlinMultiplatform.md)
