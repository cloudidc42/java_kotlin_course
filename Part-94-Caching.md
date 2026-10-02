# Part 94: Advanced Caching Strategies
## ขั้นตอนที่ 6481-6550: Multi-Level Cache, Cache Patterns, Redis Cluster

---

## 94.1 Caching Fundamentals

```
Caching Patterns:

Cache-Aside (Lazy Loading):
  Read: check cache → miss → load DB → store cache → return
  Write: write DB → invalidate cache
  Best for: read-heavy, tolerate stale data

Read-Through:
  Cache sits in front of DB
  On miss: cache loads from DB automatically
  App always talks to cache
  Best for: when DB is abstracted away

Write-Through:
  Write to cache → cache writes to DB synchronously
  Always consistent between cache and DB
  Higher write latency (both must succeed)
  Best for: write + read both frequent, consistency needed

Write-Behind (Write-Back):
  Write to cache → cache batches writes to DB asynchronously
  Low write latency, higher read performance
  Risk: data loss if cache fails before flushing
  Best for: high write volumes where some loss acceptable

Refresh-Ahead:
  Proactively refresh cache before expiry
  Prevents cache miss on popular items
  Best for: predictable access patterns, high-traffic items
```

---

## 94.2 Spring Cache Abstraction

```kotlin
// Spring Cache annotations

@Service
class ProductService(private val productRepository: ProductRepository) {
    
    // Cache result, use product id as key
    @Cacheable(
        cacheNames = ["products"],
        key = "#id",
        condition = "#id != null",          // only cache when id != null
        unless = "#result == null"          // don't cache null results
    )
    fun findById(id: String): Product? =
        productRepository.findById(id).orElse(null)
    
    // Cache all products with custom key
    @Cacheable(
        cacheNames = ["product-lists"],
        key = "#category + ':' + #pageable.pageNumber + ':' + #pageable.pageSize"
    )
    fun findByCategory(category: String, pageable: Pageable): Page<Product> =
        productRepository.findByCategory(category, pageable)
    
    // Update cache when product updated
    @CachePut(cacheNames = ["products"], key = "#result.id")
    fun update(id: String, request: UpdateProductRequest): Product {
        val product = productRepository.findById(id).orElseThrow()
        val updated = product.copy(name = request.name, price = request.price)
        return productRepository.save(updated)
    }
    
    // Evict from cache
    @CacheEvict(cacheNames = ["products"], key = "#id")
    fun delete(id: String) = productRepository.deleteById(id)
    
    // Evict all entries in cache
    @CacheEvict(cacheNames = ["product-lists"], allEntries = true)
    fun deleteAll() = productRepository.deleteAll()
    
    // Multiple cache operations
    @Caching(
        evict = [
            CacheEvict(cacheNames = ["products"], key = "#id"),
            CacheEvict(cacheNames = ["product-lists"], allEntries = true)
        ]
    )
    fun deleteAndClearLists(id: String) = productRepository.deleteById(id)
}
```

---

## 94.3 Redis Cache Configuration

```kotlin
// Redis cache with custom serialization and TTLs

@Configuration
@EnableCaching
class CacheConfig {
    
    @Bean
    fun cacheManager(connectionFactory: RedisConnectionFactory): CacheManager {
        val objectMapper = ObjectMapper().apply {
            findAndRegisterModules()
            configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            activateDefaultTyping(
                LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL,
                JsonTypeInfo.As.PROPERTY
            )
        }
        
        val serializer = GenericJackson2JsonRedisSerializer(objectMapper)
        
        val defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(serializer))
            .entryTtl(Duration.ofMinutes(10))  // default TTL
            .disableCachingNullValues()
        
        // Different TTLs per cache
        val cacheConfigs = mapOf(
            "products" to defaultConfig.entryTtl(Duration.ofHours(1)),
            "product-lists" to defaultConfig.entryTtl(Duration.ofMinutes(5)),
            "user-profiles" to defaultConfig.entryTtl(Duration.ofHours(24)),
            "sessions" to defaultConfig.entryTtl(Duration.ofHours(2)),
            "rate-limits" to defaultConfig.entryTtl(Duration.ofMinutes(1))
        )
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .transactionAware()  // participate in Spring transactions
            .build()
    }
}
```

---

## 94.4 Multi-Level Cache (L1 + L2)

```kotlin
// L1: In-process cache (Caffeine) — ultra-fast, local
// L2: Redis — shared across instances, survives restarts
// L1 miss → check L2 → miss → load DB → populate L2 → populate L1

@Component
class MultiLevelCache(
    private val redisTemplate: RedisTemplate<String, Any>,
    private val productRepository: ProductRepository
) {
    // L1: Caffeine (local, in-memory)
    private val l1Cache: Cache<String, Product> = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(1, TimeUnit.MINUTES)   // short TTL for local
        .recordStats()
        .build()
    
    fun getProduct(id: String): Product? {
        // 1. Check L1 cache
        l1Cache.getIfPresent(id)?.let { return it }
        
        // 2. Check L2 cache (Redis)
        val redisKey = "product:$id"
        val cached = redisTemplate.opsForValue().get(redisKey) as? Product
        if (cached != null) {
            l1Cache.put(id, cached)  // populate L1
            return cached
        }
        
        // 3. Load from database
        val product = productRepository.findById(id).orElse(null) ?: return null
        
        // Populate both caches
        redisTemplate.opsForValue().set(redisKey, product, Duration.ofHours(1))
        l1Cache.put(id, product)
        
        return product
    }
    
    fun invalidateProduct(id: String) {
        l1Cache.invalidate(id)
        redisTemplate.delete("product:$id")
    }
    
    // Cache stats
    fun getStats(): Map<String, Any> {
        val l1Stats = l1Cache.stats()
        return mapOf(
            "l1_hit_rate" to l1Stats.hitRate(),
            "l1_eviction_count" to l1Stats.evictionCount(),
            "l1_size" to l1Cache.estimatedSize()
        )
    }
}

// Caffeine setup
@Configuration
class CaffeineConfig {
    
    @Bean
    fun caffeineSpec(): CaffeineSpec =
        CaffeineSpec.parse("maximumSize=500,expireAfterWrite=60s,recordStats")
    
    @Bean
    fun caffeineCacheManager(caffeineSpec: CaffeineSpec): CacheManager =
        CaffeineCacheManager().apply {
            setCaffeineSpec(caffeineSpec)
        }
}
```

---

## 94.5 Cache Aside Pattern Implementation

```kotlin
// Generic Cache-Aside helper
@Component
class CacheAsideService(
    private val redisTemplate: RedisTemplate<String, String>,
    private val objectMapper: ObjectMapper
) {
    
    inline fun <reified T : Any> getOrLoad(
        key: String,
        ttl: Duration = Duration.ofMinutes(10),
        loader: () -> T?
    ): T? {
        // 1. Try cache
        val cached = redisTemplate.opsForValue().get(key)
        if (cached != null) {
            return objectMapper.readValue(cached, T::class.java)
        }
        
        // 2. Load from source
        val value = loader() ?: return null
        
        // 3. Store in cache
        val serialized = objectMapper.writeValueAsString(value)
        redisTemplate.opsForValue().set(key, serialized, ttl)
        
        return value
    }
    
    fun invalidate(key: String) {
        redisTemplate.delete(key)
    }
    
    fun invalidatePattern(pattern: String) {
        val keys = redisTemplate.keys(pattern)
        if (keys.isNotEmpty()) {
            redisTemplate.delete(keys)
        }
    }
}

// Usage
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val cache: CacheAsideService
) {
    fun findOrder(id: String): Order? =
        cache.getOrLoad("order:$id", Duration.ofMinutes(30)) {
            orderRepository.findById(id).orElse(null)
        }
    
    fun updateOrder(id: String, update: UpdateOrderRequest): Order {
        val updated = orderRepository.save(/* ... */)
        cache.invalidate("order:$id")  // invalidate on write
        return updated
    }
}
```

---

## 94.6 Redis Cluster & High Availability

```yaml
# Redis Cluster: data sharding across multiple nodes
# 3 master + 3 replica = minimum production setup

# application.yaml - Redis Cluster
spring:
  redis:
    cluster:
      nodes:
        - redis-node-1:7001
        - redis-node-2:7002
        - redis-node-3:7003
        - redis-node-1:7004  # replicas
        - redis-node-2:7005
        - redis-node-3:7006
      max-redirects: 3
    lettuce:
      cluster:
        refresh:
          adaptive: true          # auto-detect topology changes
          period: 60s
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 1000ms
```

```kotlin
// Redis Cluster configuration
@Configuration
class RedisClusterConfig {
    
    @Bean
    fun lettuceConnectionFactory(
        clusterProperties: RedisClusterProperties
    ): LettuceConnectionFactory {
        val clusterConfig = RedisClusterConfiguration(clusterProperties.nodes)
        clusterConfig.maxRedirects = 3
        
        val clientConfig = LettuceClientConfiguration.builder()
            .readFrom(ReadFrom.REPLICA_PREFERRED)  // read from replica when possible
            .clientOptions(
                ClusterClientOptions.builder()
                    .topologyRefreshOptions(
                        ClusterTopologyRefreshOptions.builder()
                            .enableAllAdaptiveRefreshTriggers()
                            .adaptiveRefreshTriggersTimeout(Duration.ofSeconds(30))
                            .build()
                    )
                    .build()
            )
            .build()
        
        return LettuceConnectionFactory(clusterConfig, clientConfig)
    }
}

// Cache Stampede Prevention (Mutex/Probabilistic)
@Service  
class StampedeProtectedCache(
    private val redisTemplate: RedisTemplate<String, String>
) {
    
    // Mutex: only one thread fetches, others wait
    fun <T> getWithMutex(
        key: String,
        lockKey: String = "$key:lock",
        loader: () -> T?
    ): T? {
        // Check cache first
        val cached = redisTemplate.opsForValue().get(key)
        if (cached != null) return deserialize<T>(cached)
        
        // Try to acquire lock
        val lockAcquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(30)) ?: false
        
        if (lockAcquired) {
            try {
                val value = loader()
                if (value != null) {
                    redisTemplate.opsForValue().set(key, serialize(value), Duration.ofMinutes(10))
                }
                return value
            } finally {
                redisTemplate.delete(lockKey)
            }
        } else {
            // Another instance is loading - wait and retry
            Thread.sleep(100)
            return redisTemplate.opsForValue().get(key)?.let { deserialize(it) }
        }
    }
    
    // Probabilistic Early Expiration: refresh before expiry
    // Prevents all instances hitting DB at the same expiry moment
    fun <T> getWithEarlyRefresh(
        key: String,
        beta: Double = 1.0,
        ttl: Duration = Duration.ofMinutes(10),
        loader: () -> T
    ): T {
        data class CacheEntry<V>(val value: V, val expiry: Long, val delta: Long)
        
        val cached = redisTemplate.opsForValue().get("$key:v2")
        if (cached != null) {
            val entry = deserialize<CacheEntry<T>>(cached)
            val ttlRemaining = entry.expiry - System.currentTimeMillis()
            val shouldRefresh = -beta * entry.delta * ln(Math.random()) > ttlRemaining
            
            if (!shouldRefresh) return entry.value
        }
        
        val start = System.currentTimeMillis()
        val value = loader()
        val delta = System.currentTimeMillis() - start
        
        val entry = CacheEntry(value, System.currentTimeMillis() + ttl.toMillis(), delta)
        redisTemplate.opsForValue().set("$key:v2", serialize(entry), ttl)
        
        return value
    }
    
    private fun <T> serialize(value: T): String = objectMapper.writeValueAsString(value)
    private fun <T> deserialize(json: String): T = objectMapper.readValue(json)
}
```

---

## สรุป Part 94

```
Caching Strategies:

Patterns:
  Cache-Aside: app manages cache, lazy load on miss
  Read-Through: cache loads from DB on miss automatically
  Write-Through: write cache + DB synchronously
  Write-Behind: write cache, async write DB (risky)
  Refresh-Ahead: proactively refresh before expiry

Multi-Level Cache:
  L1 (Caffeine): in-process, ~ns latency, small capacity
  L2 (Redis): shared across instances, ms latency, large
  L1 miss → L2 → DB; on write: invalidate both

Redis Cache Config:
  Different TTLs per cache name
  Jackson serialization with type info
  Lettuce client pool settings
  
Redis Cluster:
  3 masters + 3 replicas minimum
  ReadFrom.REPLICA_PREFERRED for read scaling
  Adaptive topology refresh
  maxRedirects for failover

Cache Stampede:
  Problem: cache expires → many threads hit DB simultaneously
  Solutions:
    Mutex: only one fetches, others wait
    Probabilistic: random early refresh before expiry
    Background refresh: refresh async before TTL hits

Best Practices:
  Set appropriate TTL (not forever, not too short)
  Cache serializable objects only
  Include version in cache key for schema changes
  Monitor: hit rate, eviction rate, memory usage
  Warm cache on startup for critical data
  Never cache user-specific aggregates at global level

Common Mistakes:
  Caching mutable shared state without invalidation
  Caching too long → stale data visible to users
  Not handling cache miss gracefully
  Caching sensitive data (PII) without encryption
```

➡️ [Part 95: Kafka Streams & Message-Driven Architecture](./Part-95-KafkaStreams.md)
