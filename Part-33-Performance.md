# Part 33: Performance Optimization
## ขั้นตอนที่ 2211-2280: Tuning ระดับ Production

---

## 33.1 JVM Performance Basics

```java
// ====== JVM Flags ======
/*
Recommended JVM flags for production Spring Boot:

java \
  -Xmx2g \                              # max heap
  -Xms2g \                              # initial heap (= max for predictability)
  -XX:+UseG1GC \                        # G1 Garbage Collector (default Java 9+)
  -XX:MaxGCPauseMillis=200 \            # target max GC pause
  -XX:+UnlockExperimentalVMOptions \
  -XX:+UseZGC \                         # ZGC (Java 15+, very low latency)
  -XX:+UseContainerSupport \            # Docker-aware heap sizing
  -XX:MaxRAMPercentage=75.0 \           # 75% of container RAM
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/heapdump.hprof \
  -XX:+ExitOnOutOfMemoryError \         # restart on OOM
  -Xlog:gc*:file=/app/gc.log:time,level:filecount=5,filesize=10m \
  -Djava.security.egd=file:/dev/./urandom \
  -jar app.jar

GC Types:
  Serial GC     = single-threaded, small apps
  Parallel GC   = throughput-focused (default pre-Java9)
  G1 GC         = balanced latency+throughput (default Java9+)
  ZGC           = ultra-low latency (<10ms pauses)
  Shenandoah    = low-pause, experimental
*/
```

---

## 33.2 Memory Optimization

```java
import java.lang.ref.*;
import java.util.*;
import java.util.concurrent.*;

public class MemoryOptimization {
    
    // ====== Use primitive arrays instead of boxed collections ======
    static void primitiveVsBoxed() {
        int n = 1_000_000;
        
        // BAD: ArrayList<Integer> uses ~20MB (boxing overhead)
        List<Integer> boxed = new ArrayList<>(n);
        for (int i = 0; i < n; i++) boxed.add(i);
        
        // GOOD: int[] uses ~4MB
        int[] primitive = new int[n];
        for (int i = 0; i < n; i++) primitive[i] = i;
        
        // Also consider: IntStream, IntBuffer for large numeric data
    }
    
    // ====== StringBuilder vs String concatenation ======
    static String badConcat(List<String> items) {
        String result = "";
        for (String item : items) {
            result += item + ", ";  // creates new String object each time!
        }
        return result;
    }
    
    static String goodConcat(List<String> items) {
        StringBuilder sb = new StringBuilder(items.size() * 10);  // pre-size
        for (String item : items) {
            sb.append(item).append(", ");
        }
        return sb.length() > 0 ? sb.substring(0, sb.length() - 2) : "";
    }
    
    // Even better: use Stream API
    static String bestConcat(List<String> items) {
        return String.join(", ", items);
    }
    
    // ====== Weak references for caches ======
    static final Map<String, WeakReference<byte[]>> imageCache = new ConcurrentHashMap<>();
    
    static byte[] getImage(String key) {
        WeakReference<byte[]> ref = imageCache.get(key);
        if (ref != null) {
            byte[] data = ref.get();
            if (data != null) return data;  // still in memory
            // GC collected it
        }
        // Load from disk/network
        byte[] data = loadImage(key);
        imageCache.put(key, new WeakReference<>(data));
        return data;
    }
    
    private static byte[] loadImage(String key) { return new byte[1024]; }
    
    // ====== Object pooling ======
    static class ConnectionPool {
        private final Queue<Connection> pool = new ConcurrentLinkedQueue<>();
        private final int maxSize;
        
        ConnectionPool(int maxSize) {
            this.maxSize = maxSize;
            for (int i = 0; i < maxSize / 2; i++) {
                pool.offer(new Connection());
            }
        }
        
        Connection acquire() {
            Connection conn = pool.poll();
            return conn != null ? conn : new Connection();
        }
        
        void release(Connection conn) {
            conn.reset();
            if (pool.size() < maxSize) {
                pool.offer(conn);
            }
            // else let GC collect it
        }
    }
    
    static class Connection {
        private String query;
        void reset() { query = null; }
        void executeQuery(String q) { this.query = q; }
    }
    
    public static void main(String[] args) {
        // Benchmark concat methods
        List<String> items = new ArrayList<>();
        for (int i = 0; i < 10000; i++) items.add("item" + i);
        
        long start = System.nanoTime();
        badConcat(items);
        System.out.printf("Bad: %.2fms%n", (System.nanoTime() - start) / 1e6);
        
        start = System.nanoTime();
        goodConcat(items);
        System.out.printf("Good: %.2fms%n", (System.nanoTime() - start) / 1e6);
        
        start = System.nanoTime();
        bestConcat(items);
        System.out.printf("Best: %.2fms%n", (System.nanoTime() - start) / 1e6);
    }
}
```

---

## 33.3 Database Performance

```java
import org.springframework.data.jpa.*;
import java.util.*;

// ====== N+1 Problem ======
// BAD: N+1 queries
// for each user (1 query), loads orders (N queries)
// SELECT * FROM users
// SELECT * FROM orders WHERE user_id = 1
// SELECT * FROM orders WHERE user_id = 2
// ... (N more queries)

// GOOD: JOIN FETCH
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.active = true")
List<User> findActiveUsersWithOrders();

// ====== Pagination ======
@Query("SELECT p FROM Product p WHERE p.category = :category")
Page<Product> findByCategory(String category, Pageable pageable);

// Usage:
// Pageable pageable = PageRequest.of(0, 20, Sort.by("name").ascending());
// Page<Product> page = productRepo.findByCategory("electronics", pageable);

// ====== Projections (fetch only needed columns) ======
public interface ProductSummary {
    Long getId();
    String getName();
    Double getPrice();
    // No need to fetch description, images, etc.
}

@Query("SELECT p.id as id, p.name as name, p.price as price FROM Product p")
List<ProductSummary> findAllSummaries();

// ====== Batch insert ======
@Repository
class BatchProductRepository {
    
    @PersistenceContext
    private EntityManager em;
    
    @Transactional
    public void batchInsert(List<Product> products) {
        int batchSize = 50;
        for (int i = 0; i < products.size(); i++) {
            em.persist(products.get(i));
            if (i % batchSize == 0 && i > 0) {
                em.flush();
                em.clear();  // free memory
            }
        }
    }
}

// ====== Caching ======
@org.springframework.cache.annotation.EnableCaching
@org.springframework.context.annotation.Configuration
class CacheConfig {
    // Spring Boot autoconfigures with Caffeine/Redis
}

@org.springframework.stereotype.Service
class ProductService {
    
    @org.springframework.cache.annotation.Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        // called only if not in cache
        return productRepo.findById(id).orElseThrow();
    }
    
    @org.springframework.cache.annotation.CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepo.save(product);
    }
    
    @org.springframework.cache.annotation.CacheEvict(value = "products", allEntries = true)
    public void clearCache() {}
}
```

---

## 33.4 Async & Reactive Performance

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.time.*;

public class AsyncPerformance {
    
    // ====== Virtual Threads (Java 21) ======
    static void virtualThreadsDemo() throws InterruptedException {
        int tasks = 10_000;
        CountDownLatch latch = new CountDownLatch(tasks);
        AtomicLong total = new AtomicLong();
        
        long start = System.currentTimeMillis();
        
        // Virtual Threads: can handle millions of I/O-bound tasks
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < tasks; i++) {
                executor.submit(() -> {
                    try {
                        Thread.sleep(100);  // simulate I/O
                        total.incrementAndGet();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } finally {
                        latch.countDown();
                    }
                });
            }
        }
        
        latch.await();
        System.out.printf("Virtual threads: %d tasks in %dms%n",
            tasks, System.currentTimeMillis() - start);
    }
    
    // ====== CompletableFuture pipeline ======
    static CompletableFuture<String> buildPipeline(Long userId) {
        return CompletableFuture
            .supplyAsync(() -> fetchUser(userId))                 // async fetch
            .thenApplyAsync(user -> enrichWithProfile(user))     // enrich
            .thenCombineAsync(
                CompletableFuture.supplyAsync(() -> fetchOrders(userId)),
                (user, orders) -> buildResponse(user, orders)    // combine
            )
            .exceptionally(ex -> "Error: " + ex.getMessage())
            .orTimeout(5, TimeUnit.SECONDS);
    }
    
    private static String fetchUser(Long id) {
        simulateDelay(50); return "User" + id;
    }
    private static String enrichWithProfile(String user) {
        simulateDelay(30); return user + "+profile";
    }
    private static String fetchOrders(Long id) {
        simulateDelay(80); return "[order1, order2]";
    }
    private static String buildResponse(String user, String orders) {
        return "{user: " + user + ", orders: " + orders + "}";
    }
    private static void simulateDelay(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
    
    // ====== Throughput benchmark ======
    static void benchmarkThroughput() throws Exception {
        int requests = 1000;
        
        // Sequential
        long seqStart = System.currentTimeMillis();
        for (int i = 0; i < requests; i++) {
            fetchUser((long) i);
        }
        long seqTime = System.currentTimeMillis() - seqStart;
        
        // Parallel with virtual threads
        long parStart = System.currentTimeMillis();
        List<CompletableFuture<String>> futures = new ArrayList<>();
        for (int i = 0; i < requests; i++) {
            final int id = i;
            futures.add(CompletableFuture.supplyAsync(
                () -> fetchUser((long) id),
                Executors.newVirtualThreadPerTaskExecutor()
            ));
        }
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
        long parTime = System.currentTimeMillis() - parStart;
        
        System.out.printf("Sequential: %dms%n", seqTime);
        System.out.printf("Parallel:   %dms (%.1fx faster)%n",
            parTime, (double) seqTime / parTime);
    }
    
    public static void main(String[] args) throws Exception {
        virtualThreadsDemo();
        
        var result = buildPipeline(1L).get();
        System.out.println("Pipeline: " + result);
        
        benchmarkThroughput();
    }
}
```

---

## 33.5 JVM Profiling Tools

```bash
# ====== JVM Monitoring Tools ======

# jstat: JVM statistics
jstat -gc <pid> 1s          # GC stats every 1 second
jstat -gcutil <pid> 1s      # GC utilization percentage

# Output columns:
# S0C S1C S0U S1U EC EU OC OU MC MU CCSC CCSU YGC YGCT FGC FGCT GCT
# S0/S1 = Survivor space 0/1
# E = Eden space
# O = Old space
# M = Metaspace
# YGC = Young GC count, YGCT = time
# FGC = Full GC count, FGCT = time

# jmap: heap analysis
jmap -heap <pid>                  # heap summary
jmap -dump:format=b,file=heap.hprof <pid>  # heap dump

# Analyze heap dump with:
# jhat heap.hprof               (built-in, basic)
# Eclipse MAT (Memory Analyzer)  (recommended, free)
# JProfiler, YourKit             (commercial)

# jstack: thread dump
jstack <pid>                      # print all threads
jstack -l <pid>                   # with lock info
# Look for: BLOCKED threads, deadlocks

# Detect deadlock:
jstack <pid> | grep -A5 "deadlock"

# async-profiler (best for production profiling)
# Download from: https://github.com/async-profiler/async-profiler
./profiler.sh start -e cpu,alloc,lock <pid>
./profiler.sh stop -f flamegraph.html <pid>

# Java Flight Recorder (built-in, low overhead)
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr app.jar
jcmd <pid> JFR.start duration=60s filename=recording.jfr
jcmd <pid> JFR.stop

# Analyze JFR with JDK Mission Control
jmc recording.jfr
```

---

## 33.6 HTTP Performance

```java
import java.net.http.*;
import java.net.*;
import java.time.*;
import java.util.concurrent.*;

// ====== HTTP/2 with Java HttpClient ======
public class HttpPerformance {
    
    private static final HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(5))
        .executor(Executors.newVirtualThreadPerTaskExecutor())
        .build();
    
    // Parallel requests
    static void parallelRequests() throws Exception {
        List<URI> urls = List.of(
            URI.create("https://api.example.com/users"),
            URI.create("https://api.example.com/products"),
            URI.create("https://api.example.com/orders")
        );
        
        long start = System.currentTimeMillis();
        
        List<CompletableFuture<HttpResponse<String>>> futures = urls.stream()
            .map(url -> client.sendAsync(
                HttpRequest.newBuilder(url).GET().build(),
                HttpResponse.BodyHandlers.ofString()
            ))
            .toList();
        
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        System.out.printf("All %d requests: %dms%n",
            urls.size(), System.currentTimeMillis() - start);
        
        futures.forEach(f -> {
            try {
                var resp = f.get();
                System.out.printf("  %s → %d (%d bytes)%n",
                    resp.uri(), resp.statusCode(), resp.body().length());
            } catch (Exception e) { /* handle */ }
        });
    }
    
    // Connection reuse with keep-alive
    static void benchmarkConnections() throws Exception {
        int requests = 100;
        
        // New client each time (bad): ~100 TCP handshakes
        long bad = benchmark(() -> {
            HttpClient.newHttpClient().send(
                HttpRequest.newBuilder(URI.create("https://httpbin.org/get")).GET().build(),
                HttpResponse.BodyHandlers.ofString()
            );
        }, requests);
        
        // Shared client (good): TCP/HTTP2 connection reused
        long good = benchmark(() -> {
            client.send(
                HttpRequest.newBuilder(URI.create("https://httpbin.org/get")).GET().build(),
                HttpResponse.BodyHandlers.ofString()
            );
        }, requests);
        
        System.out.printf("New client: %dms, Shared: %dms%n", bad, good);
    }
    
    static long benchmark(ThrowingRunnable task, int count) throws Exception {
        long start = System.nanoTime();
        for (int i = 0; i < count; i++) task.run();
        return (System.nanoTime() - start) / 1_000_000;
    }
    
    @FunctionalInterface interface ThrowingRunnable { void run() throws Exception; }
}
```

---

## 33.7 Spring Boot Performance

```java
// ====== Application startup optimization ======

// 1. Lazy initialization
/*
spring.main.lazy-initialization=true
# Beans created on first use, not at startup
# Reduces startup time by 40-60%
# Downside: first request is slower
*/

// 2. Spring Native (GraalVM)
// pom.xml:
// <groupId>org.springframework.boot</groupId>
// <artifactId>spring-boot-starter-actuator</artifactId>

// Build native:
// mvn -Pnative native:compile
// Startup: ~100ms vs ~3s JVM mode

// 3. Class Data Sharing (CDS)
/*
# Generate CDS archive
java -XX:ArchiveClassesAtExit=app-cds.jsa -jar app.jar --spring.main.lazy-initialization=true
# Use archive
java -XX:SharedArchiveFile=app-cds.jsa -jar app.jar
# Startup 20-40% faster
*/

@org.springframework.boot.autoconfigure.SpringBootApplication
@org.springframework.boot.context.event.ApplicationReadyEvent.class
public class OptimizedApp {
    
    // @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        // Pre-warm caches on startup
        System.out.println("Application ready, pre-warming caches...");
    }
}

// ====== Connection pool tuning ======
/*
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # = num_cores * 2 + spindle_count
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      pool-name: HikariPool-1
      leak-detection-threshold: 60000  # warn on long-held connections
*/

// ====== Response compression ======
/*
server:
  compression:
    enabled: true
    mime-types: application/json,text/plain,text/html
    min-response-size: 1024  # only compress > 1KB
*/

// ====== Redis cache config ======
import org.springframework.data.redis.cache.*;
import java.time.*;

@org.springframework.context.annotation.Configuration
class RedisConfig {
    
    @org.springframework.context.annotation.Bean
    public org.springframework.cache.CacheManager cacheManager(
            org.springframework.data.redis.connection.RedisConnectionFactory factory) {
        
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                org.springframework.data.redis.serializer.RedisSerializationContext
                    .SerializationPair.fromSerializer(
                        new org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer()
                    )
            );
        
        Map<String, RedisCacheConfiguration> configs = Map.of(
            "products",  defaultConfig.entryTtl(Duration.ofHours(1)),
            "users",     defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "sessions",  defaultConfig.entryTtl(Duration.ofDays(1))
        );
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(configs)
            .build();
    }
}
```

---

## สรุป Part 33

```
Performance Checklist:

JVM:
  ✓ Use G1GC or ZGC
  ✓ Set -Xmx = -Xms (avoid heap resize)
  ✓ Enable container-aware sizing
  ✓ Profile before optimizing (premature optimization = root of all evil)

Memory:
  ✓ Use primitive arrays for numeric data
  ✓ Pre-size collections when size is known
  ✓ Use WeakReference for caches
  ✓ Pool expensive objects

Database:
  ✓ Avoid N+1 queries (use JOIN FETCH)
  ✓ Use projections (select only needed columns)
  ✓ Add indexes on frequently queried columns
  ✓ Use batch operations for bulk inserts
  ✓ Cache frequently-read, rarely-changed data

HTTP:
  ✓ Reuse HTTP connections
  ✓ Use HTTP/2
  ✓ Enable response compression
  ✓ Parallel requests with CompletableFuture

Profiling Tools:
  ✓ async-profiler for CPU/memory flamegraphs
  ✓ JFR for production monitoring
  ✓ Spring Boot Actuator + Micrometer + Grafana
```

➡️ [Part 34: Advanced Smali & APK Patching](./Part-34-Advanced-Smali.md)
