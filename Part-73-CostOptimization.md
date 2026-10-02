# Part 73: Cost Optimization & FinOps
## ขั้นตอนที่ 5011-5080: Cloud Cost, JVM Memory, Connection Pooling

---

## 73.1 FinOps คืออะไร

```
FinOps = Financial + DevOps = วัฒนธรรมการจัดการ Cloud Cost

ปัญหาที่พบบ่อย:
  - Over-provisioning: เช่า resource มากเกินไป
  - Unused resources: EC2 / RDS ที่ไม่ได้ใช้แต่ยังจ่ายอยู่
  - Inefficient code: N+1 query ทำให้ DB ทำงานหนักเกิน
  - Wrong instance type: ซื้อ CPU-optimized แต่ workload เป็น memory

FinOps Cycle:
  Inform → Optimize → Operate
  
  Inform: เห็นว่าใครจ่ายเท่าไหร่ (cost allocation tags)
  Optimize: ลด waste, right-size, use commitments
  Operate: deploy ด้วย cost-awareness
```

---

## 73.2 JVM Memory Optimization

```java
// ปัญหา: application ใช้ memory มากเกิน

// 1. ลด heap size โดยใช้ off-heap
// Caffeine cache: in-heap by default → ใช้ RAM มาก
// Redis: off-heap → ลด JVM GC pressure

// 2. Object pooling (Netty ByteBuffer, Apache Commons Pool)
import org.apache.commons.pool2.*;
import org.apache.commons.pool2.impl.*;

class ExpensiveObjectPool {
    
    // Pool ของ object ที่ expensive ในการสร้าง (DB connection, HTTP client)
    private final GenericObjectPool<ExpensiveObject> pool;
    
    ExpensiveObjectPool() {
        var config = new GenericObjectPoolConfig<ExpensiveObject>();
        config.setMaxTotal(10);
        config.setMinIdle(2);
        config.setMaxIdle(5);
        config.setTestOnBorrow(true);
        config.setTimeBetweenEvictionRuns(java.time.Duration.ofMinutes(5));
        
        this.pool = new GenericObjectPool<>(new ExpensiveObjectFactory(), config);
    }
    
    public <T> T execute(java.util.function.Function<ExpensiveObject, T> action) throws Exception {
        var obj = pool.borrowObject();
        try {
            return action.apply(obj);
        } finally {
            pool.returnObject(obj);
        }
    }
    
    static class ExpensiveObjectFactory extends BasePooledObjectFactory<ExpensiveObject> {
        @Override
        public ExpensiveObject create() throws Exception {
            return new ExpensiveObject();  // heavyweight initialization
        }
        
        @Override
        public PooledObject<ExpensiveObject> wrap(ExpensiveObject obj) {
            return new DefaultPooledObject<>(obj);
        }
        
        @Override
        public boolean validateObject(PooledObject<ExpensiveObject> p) {
            return p.getObject().isHealthy();
        }
        
        @Override
        public void destroyObject(PooledObject<ExpensiveObject> p) throws Exception {
            p.getObject().close();
        }
    }
}

// 3. Lazy loading (load only what's needed)
// JPA: LAZY fetching (default for collections)
@Entity
class Order {
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "order")
    private java.util.List<OrderItem> items;  // not loaded until accessed
}

// 4. Weak references for cache (let GC reclaim when memory pressure)
java.util.WeakHashMap<String, ExpensiveData> cache = new java.util.WeakHashMap<>();
// GC can collect values when memory is low

// 5. GC tuning flags
/*
  # G1GC: balanced (default Java 9+)
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200      # target max pause
  -XX:G1HeapRegionSize=16m
  
  # ZGC: low latency (< 1ms pauses, Java 15+)
  -XX:+UseZGC
  -XX:SoftMaxHeapSize=800m      # ZGC will try to stay below
  
  # Container awareness
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=75.0     # use 75% of container RAM
  
  # Heap sizing
  -Xms256m -Xmx1g               # start small, grow to 1GB max
  
  # Monitoring
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=10m
*/

class MemoryMonitoring {
    
    @Scheduled(fixedRate = 30_000)
    void logMemoryUsage() {
        var runtime = Runtime.getRuntime();
        long usedMB = (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024);
        long maxMB = runtime.maxMemory() / (1024 * 1024);
        double usagePercent = (double) usedMB / maxMB * 100;
        
        if (usagePercent > 80) {
            log.warn("High memory usage: {}MB / {}MB ({:.1f}%)", usedMB, maxMB, usagePercent);
        }
    }
}
```

---

## 73.3 Database Query Optimization

```sql
-- Cost: DB query ที่ slow = more CPU time = higher cost

-- EXPLAIN ANALYZE: see actual execution plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'PENDING'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 20;

-- Look for:
-- Seq Scan → bad (reads whole table)
-- Index Scan → good
-- Hash Join → ok for medium tables
-- Nested Loop → bad if outer has many rows

-- Fix: add composite index
CREATE INDEX CONCURRENTLY idx_orders_status_created 
    ON orders(status, created_at DESC)
    INCLUDE (id, total, user_id);

-- After index: Index Scan instead of Seq Scan
-- Cost drops from 50,000 to 50 units (1000x improvement)

-- Reduce query load with materialized views
CREATE MATERIALIZED VIEW daily_order_stats AS
SELECT 
    DATE(created_at) as order_date,
    COUNT(*) as order_count,
    SUM(total) as total_revenue,
    AVG(total) as avg_order_value
FROM orders
WHERE status = 'COMPLETED'
GROUP BY DATE(created_at);

CREATE INDEX ON daily_order_stats(order_date DESC);

-- Refresh periodically (not on every query)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_order_stats;

-- Query is now instant:
SELECT * FROM daily_order_stats 
WHERE order_date > CURRENT_DATE - 30
ORDER BY order_date DESC;
```

---

## 73.4 Kubernetes Resource Right-Sizing

```yaml
# ปัญหา: ตั้ง resource limits สูงเกิน → จ่ายค่า node มากเกิน

# หาค่า actual usage ก่อน (Prometheus + Grafana)
# แล้วตั้ง requests = P99 usage + 20% buffer
# limits = requests * 1.5 (allow burst)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: myrepo/order-service:latest
          
          resources:
            requests:
              # Set based on actual p99 usage from Prometheus:
              # histogram_quantile(0.99, rate(container_memory_usage_bytes[1h]))
              memory: "256Mi"   # was 512Mi (over-provisioned)
              cpu: "100m"       # was 500m (over-provisioned)
            limits:
              memory: "512Mi"   # 2x requests for burst
              cpu: "500m"       # 5x requests for CPU burst
          
          # JVM flags for container
          env:
            - name: JAVA_OPTS
              value: >-
                -XX:+UseContainerSupport
                -XX:MaxRAMPercentage=75.0
                -XX:+UseG1GC
                -Xms64m

---
# HPA: scale down when load is low (save cost)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 1    # was 3 (over-provisioned for nights)
  maxReplicas: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5min before scale down
      policies:
        - type: Pods
          value: 1               # remove 1 pod at a time
          periodSeconds: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 73.5 Cost Monitoring

```java
// Tag resources for cost allocation
// AWS: tag EC2/RDS/S3 with Environment, Team, Service

// Application-level cost metrics
@Service
class CostAwarenessService {
    
    private final MeterRegistry registry;
    
    CostAwarenessService(MeterRegistry registry) {
        this.registry = registry;
    }
    
    // Track expensive operations
    public <T> T trackDbQuery(String queryName, java.util.function.Supplier<T> query) {
        return io.micrometer.core.instrument.Timer.builder("db.query.duration")
            .tag("query", queryName)
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry)
            .record(query);
    }
    
    // Track external API calls (these cost money)
    public <T> T trackExternalApi(String service, java.util.function.Supplier<T> call) {
        io.micrometer.core.instrument.Counter.builder("external.api.calls")
            .tag("service", service)
            .register(registry)
            .increment();
        
        return io.micrometer.core.instrument.Timer.builder("external.api.duration")
            .tag("service", service)
            .register(registry)
            .record(call);
    }
    
    // Alert when cost-sensitive resources spike
    // alert rule:
    // increase(external.api.calls_total{service="openai"}[1h]) > 1000
    // → "AI API calls spiking, check for runaway usage"
}

// Cost-efficient caching to reduce external API calls
@Service
class CostEfficientTranslationService {
    
    private final TranslationApiClient apiClient;
    private final com.github.benmanes.caffeine.cache.Cache<String, String> cache;
    
    CostEfficientTranslationService(TranslationApiClient apiClient) {
        this.apiClient = apiClient;
        this.cache = com.github.benmanes.caffeine.cache.Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(java.time.Duration.ofDays(30))  // translations don't change
            .recordStats()
            .build();
    }
    
    public String translate(String text, String targetLanguage) {
        String cacheKey = targetLanguage + ":" + text.hashCode();
        
        return cache.get(cacheKey, k -> {
            // Only call expensive API when cache miss
            return apiClient.translate(text, targetLanguage);
        });
    }
    
    // Expose cache hit rate metric
    @Scheduled(fixedRate = 60_000)
    void logCacheStats() {
        var stats = cache.stats();
        log.info("Translation cache: hit_rate={:.2f}%, size={}", 
            stats.hitRate() * 100, cache.estimatedSize());
        // Target: > 80% hit rate (means < 20% of API calls are paid)
    }
}
```

---

## สรุป Part 73

```
Cost Optimization Strategies:

JVM:
  ✓ MaxRAMPercentage=75 (not hard -Xmx)
  ✓ ZGC or G1GC based on latency needs
  ✓ Object pooling for expensive objects
  ✓ Cache off-heap (Redis) to reduce GC pressure

Database:
  ✓ EXPLAIN ANALYZE before every new query pattern
  ✓ Index on WHERE and ORDER BY columns
  ✓ Materialized views for heavy aggregations
  ✓ Connection pool sized correctly (not too large)
  ✓ Read replicas for heavy read workloads

Kubernetes:
  ✓ Set requests based on P99 actual usage
  ✓ limits = requests * 1.5-2x
  ✓ HPA scale-down to reduce pods at night
  ✓ Spot/preemptible instances for non-critical workloads

External APIs:
  ✓ Cache aggressively (translations, exchange rates)
  ✓ Batch API calls when possible
  ✓ Alert on unexpected API call spikes
  ✓ Use compression to reduce bandwidth costs

Quick Wins (low effort, high impact):
  1. Right-size over-provisioned VMs (often 2-5x savings)
  2. Delete unused resources (old test environments)
  3. Reserved instances for stable baseline
  4. S3 lifecycle policies (move old logs to Glacier)
```

➡️ [Part 74: Hexagonal Architecture](./Part-74-HexagonalArchitecture.md)
