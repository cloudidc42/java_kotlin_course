# Part 51: Performance Tuning & JVM Optimization
## ขั้นตอนที่ 3471-3540: JVM Internals, GC, Profiling

---

## 51.1 JVM Memory Model

```
JVM Heap:
┌──────────────────────────────────────────────┐
│                   Heap                        │
│  ┌────────────────┐  ┌──────────────────────┐ │
│  │   Young Gen    │  │      Old Gen          │ │
│  │  ┌──┐  ┌────┐ │  │  (long-lived objects) │ │
│  │  │E0│  │ S0 │ │  │                       │ │
│  │  │E1│  │ S1 │ │  │                       │ │
│  │  └──┘  └────┘ │  │                       │ │
│  │  Eden  Survivor│  │                       │ │
│  └────────────────┘  └──────────────────────┘ │
└──────────────────────────────────────────────┘

Off-heap:
  Metaspace = class metadata (replaces PermGen in Java 8+)
  Direct memory = ByteBuffer.allocateDirect(), Netty

GC cycles:
  Minor GC = collect Young Gen (fast, milliseconds)
  Major GC = collect Old Gen (slow, seconds)
  Full GC   = both + compact (application pause = "stop-the-world")
```

---

## 51.2 G1GC Tuning

```bash
# JVM flags for production Spring Boot
java \
  # Heap size
  -Xms512m \
  -Xmx1g \
  
  # Use G1GC (default since Java 9)
  -XX:+UseG1GC \
  
  # Pause time goal (milliseconds)
  -XX:MaxGCPauseMillis=200 \
  
  # Region size (1-32MB, power of 2)
  -XX:G1HeapRegionSize=8m \
  
  # % of heap to trigger Mixed GC
  -XX:InitiatingHeapOccupancyPercent=45 \
  
  # GC logging for analysis
  -Xlog:gc*:file=/app/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=10m \
  
  # Container awareness
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  
  # Performance
  -XX:+UseStringDeduplication \
  -XX:+OptimizeStringConcat \
  
  # JIT compilation
  -XX:+TieredCompilation \
  -XX:ReservedCodeCacheSize=256m \
  
  # Thread stack size
  -Xss512k \
  
  -jar app.jar
```

---

## 51.3 Connection Pool Tuning (HikariCP)

```yaml
# application.yaml
spring:
  datasource:
    hikari:
      # Core sizing formula: connections = (core_count * 2) + effective_spindle_count
      # For 4 CPU, SSD: (4 * 2) + 1 = 9 ≈ 10
      maximum-pool-size: 10
      minimum-idle: 5
      
      # Connection lifecycle
      connection-timeout: 3000    # 3s: how long to wait for connection from pool
      idle-timeout: 600000        # 10min: remove idle connections
      max-lifetime: 1800000       # 30min: max connection age (< DB timeout)
      
      # Validation
      connection-test-query: SELECT 1
      keepalive-time: 60000       # 1min: keepalive ping
      
      # Performance
      auto-commit: false          # managed by Spring @Transactional
      pool-name: order-service-pool
      
      # Data source properties
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        rewriteBatchedStatements: true  # bulk inserts as single packet
```

---

## 51.4 Caching Strategy

```java
import org.springframework.cache.annotation.*;
import io.micrometer.core.instrument.*;
import org.springframework.stereotype.*;

// Multi-level caching: local (Caffeine) → distributed (Redis)
@Configuration
class MultiLevelCacheConfig {
    
    @Bean
    CacheManager cacheManager(RedisConnectionFactory redisFactory) {
        // L1: Caffeine (in-process, ~1μs)
        var caffeine = com.github.benmanes.caffeine.cache.Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(java.time.Duration.ofMinutes(5))
            .recordStats();  // enables metrics
        
        var l1 = new org.springframework.cache.caffeine.CaffeineCacheManager();
        l1.setCaffeine(caffeine);
        
        // L2: Redis (distributed, ~1ms)
        var redisCacheConfig = org.springframework.data.redis.cache.RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(java.time.Duration.ofMinutes(30))
            .serializeValuesWith(
                org.springframework.data.redis.serializer.RedisSerializationContext
                    .SerializationPair
                    .fromSerializer(new org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer())
            );
        
        var l2 = org.springframework.data.redis.cache.RedisCacheManager
            .builder(redisFactory)
            .cacheDefaults(redisCacheConfig)
            .build();
        
        // Chain: try L1, then L2, then source
        return new org.springframework.cache.support.CompositeCacheManager(l1, l2);
    }
}

// Cache-aside pattern for read-heavy data
@Service
class ProductCacheService {
    
    private final ProductRepository repo;
    private final MeterRegistry registry;
    private final Counter cacheHit;
    private final Counter cacheMiss;
    
    ProductCacheService(ProductRepository repo, MeterRegistry registry) {
        this.repo = repo;
        this.registry = registry;
        this.cacheHit = Counter.builder("cache.product.hit").register(registry);
        this.cacheMiss = Counter.builder("cache.product.miss").register(registry);
    }
    
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        cacheMiss.increment();
        return repo.findById(id).orElseThrow();
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    public Product update(Product product) {
        return repo.save(product);
    }
    
    // Cache warming on startup
    @org.springframework.scheduling.annotation.Scheduled(initialDelay = 10000, fixedRate = Long.MAX_VALUE)
    public void warmupTopProducts() {
        repo.findTop100ByOrderBySalesDesc().forEach(p -> findById(p.getId()));
        System.out.println("Cache warmed with top 100 products");
    }
}
```

---

## 51.5 Async Processing & Thread Pools

```java
import java.util.concurrent.*;
import org.springframework.scheduling.concurrent.*;

@Configuration
class ThreadPoolConfig {
    
    // CPU-bound work: #threads = CPU cores
    @Bean("cpuTaskExecutor")
    ThreadPoolTaskExecutor cpuTaskExecutor() {
        var exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(Runtime.getRuntime().availableProcessors());
        exec.setMaxPoolSize(Runtime.getRuntime().availableProcessors());
        exec.setQueueCapacity(1000);
        exec.setThreadNamePrefix("cpu-");
        exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.initialize();
        return exec;
    }
    
    // IO-bound work: #threads = CPU cores * 10 (blocking on I/O most of the time)
    @Bean("ioTaskExecutor")
    ThreadPoolTaskExecutor ioTaskExecutor() {
        int cores = Runtime.getRuntime().availableProcessors();
        var exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(cores * 5);
        exec.setMaxPoolSize(cores * 10);
        exec.setQueueCapacity(5000);
        exec.setThreadNamePrefix("io-");
        exec.setKeepAliveSeconds(60);
        exec.initialize();
        return exec;
    }
}

// Batch processing with CompletableFuture
@Service
class BatchProcessor {
    
    @Autowired @Qualifier("ioTaskExecutor")
    Executor executor;
    
    public List<String> processInParallel(List<String> items) {
        var futures = items.stream()
            .map(item -> CompletableFuture
                .supplyAsync(() -> processItem(item), executor)
                .exceptionally(e -> "ERROR: " + e.getMessage())
            )
            .toList();
        
        return CompletableFuture
            .allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .toList()
            )
            .join();
    }
    
    // Semaphore-based rate limiting for concurrent calls
    private final Semaphore semaphore = new Semaphore(100);
    
    public String processWithLimit(String item) throws InterruptedException {
        semaphore.acquire();
        try {
            return processItem(item);
        } finally {
            semaphore.release();
        }
    }
    
    private String processItem(String item) { return item.toUpperCase(); }
}
```

---

## 51.6 JVM Profiling

```
Profiling Tools:

1. VisualVM (free, GUI)
   - CPU profiling: which methods are hot?
   - Memory profiling: which objects are taking memory?
   - Thread view: deadlock detection

2. async-profiler (production safe, low overhead)
   ./profiler.sh -d 30 -f /tmp/profile.html <PID>
   → Wall-clock, CPU, alloc, lock profiling
   → Flame graph output

3. JFR + JMC (Java Flight Recorder)
   Start recording:
   java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar
   
   Analyze:
   jmc recording.jfr
   → Method profiling, memory leaks, GC details

4. Spring Boot Actuator metrics:
   GET /actuator/metrics/jvm.memory.used?tag=area:heap
   GET /actuator/metrics/jvm.gc.pause
   GET /actuator/metrics/http.server.requests
```

```java
// Custom metrics for performance monitoring
@Component
class PerformanceMetrics {
    
    private final MeterRegistry registry;
    
    // Track object allocation rate
    private final DistributionSummary orderSize;
    
    // Track slow queries
    private final Timer dbQueryTimer;
    
    PerformanceMetrics(MeterRegistry registry) {
        this.registry = registry;
        this.orderSize = DistributionSummary.builder("order.size")
            .baseUnit("items")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        this.dbQueryTimer = Timer.builder("db.query.duration")
            .publishPercentileHistogram()
            .register(registry);
    }
    
    public void recordOrderSize(int items) {
        orderSize.record(items);
    }
    
    public <T> T timeQuery(String queryName, java.util.function.Supplier<T> query) {
        return Timer.builder("db.query.duration")
            .tag("query", queryName)
            .register(registry)
            .record(query);
    }
}
```

---

## สรุป Part 51

```
Performance Checklist:

JVM:
  ✓ -XX:+UseContainerSupport (respect k8s limits)
  ✓ -XX:MaxRAMPercentage=75.0 (leave room for OS)
  ✓ -XX:MaxGCPauseMillis=200 (G1GC target)
  ✓ GC logging (-Xlog:gc*) for diagnostics

Connection Pool (HikariCP):
  ✓ max-pool-size = (cores * 2) + 1
  ✓ connection-timeout: 3s (fail fast > wait forever)
  ✓ max-lifetime: < DB server timeout
  ✓ rewriteBatchedStatements = true for bulk inserts

Caching:
  ✓ L1: Caffeine (in-process, nanoseconds)
  ✓ L2: Redis (distributed, milliseconds)
  ✓ Cache-aside: @Cacheable / @CacheEvict
  ✓ Warm cache on startup for critical data

Thread Pools:
  ✓ CPU tasks: threads = CPU cores
  ✓ IO tasks: threads = CPU cores * 5-10
  ✓ CallerRunsPolicy for backpressure (CPU pool)

Profile before optimizing:
  async-profiler → flame graph → find actual bottleneck
  Don't guess where the slow part is
```

➡️ [Part 52: Database Performance & Query Optimization](./Part-52-DatabasePerformance.md)
