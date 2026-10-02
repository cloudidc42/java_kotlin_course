# Part 56: Distributed Systems & Consistency
## ขั้นตอนที่ 3821-3890: CAP Theorem, Consensus, Distributed Locks

---

## 56.1 CAP Theorem

```
CAP Theorem: distributed system can guarantee only 2 of 3:

  C = Consistency
      All nodes see the same data at the same time
      Read always returns latest write
      
  A = Availability
      Every request gets a response (not necessarily latest)
      System always works, even if some nodes are down
      
  P = Partition Tolerance
      System continues working even if network partition occurs
      (some nodes can't communicate with others)

In practice, P is required (network partitions WILL happen)
So you choose: CP or AP

CP (Consistency + Partition Tolerance):
  - Strict consistency
  - May refuse requests during partition
  - Example: HBase, Zookeeper, etcd, Consul
  - Use for: financial transactions, inventory

AP (Availability + Partition Tolerance):
  - Eventually consistent
  - Always responds (might be stale)
  - Example: Cassandra, DynamoDB, Redis Cluster, CouchDB
  - Use for: user profiles, shopping cart, social media

PACELC extension:
  If Partition:    choose Availability or Consistency
  Else (normal):   choose Latency or Consistency
```

---

## 56.2 Consistency Models

```
Consistency Spectrum (strong → weak):

Linearizability (Strict):
  Operations appear instantaneous
  Real-time order preserved
  Most expensive
  Example: single-node DB, Zookeeper

Sequential Consistency:
  All nodes see operations in same order
  Not necessarily real-time
  Example: distributed locks

Causal Consistency:
  Causally related operations in order
  Concurrent operations can differ
  Example: collaborative editing

Eventual Consistency:
  All replicas eventually converge
  No ordering guarantee
  Cheapest, most available
  Example: DNS, shopping cart, Amazon S3

Monotonic Read Consistency:
  If you read X, future reads return X or newer
  Example: session-based reads

Read-Your-Writes:
  After you write X, your reads return X
  Other users might still see old value
  Example: profile update
```

---

## 56.3 Distributed Transactions

```java
import org.springframework.transaction.annotation.*;
import org.springframework.stereotype.*;

// Problem: update Order DB + Inventory DB atomically

// Option 1: Two-Phase Commit (2PC) - strong consistency
// Phase 1: Coordinator asks all participants "can you commit?"
// Phase 2: If all say yes → commit; if any say no → rollback
// Problem: slow, coordinator is SPOF, blocking protocol

// Option 2: Saga Pattern (eventual consistency)
// Break into local transactions, compensate on failure

// Option 3: Outbox + Idempotent Consumers
// Best for most use cases (eventual consistency without 2PC)

// Practical: use @Transactional within one service boundary
// Cross-service: use Saga or Outbox

@Service
class InventoryService {
    
    private final InventoryRepository inventoryRepo;
    private final OutboxRepository outboxRepo;
    
    InventoryService(InventoryRepository inventoryRepo, OutboxRepository outboxRepo) {
        this.inventoryRepo = inventoryRepo;
        this.outboxRepo = outboxRepo;
    }
    
    // Optimistic locking: detect concurrent modifications
    @Transactional
    public void reserveStock(String productId, int quantity) {
        var inventory = inventoryRepo.findByProductId(productId);
        
        // @Version field in entity → OptimisticLockException if concurrent update
        if (inventory.getAvailable() < quantity) {
            throw new InsufficientStockException("Not enough stock: " + productId);
        }
        
        inventory.setAvailable(inventory.getAvailable() - quantity);
        inventory.setReserved(inventory.getReserved() + quantity);
        inventoryRepo.save(inventory);  // will throw if version mismatch
        
        // Outbox for eventual notification
        outboxRepo.save(new OutboxEvent("StockReserved", 
            "{\"productId\":\"" + productId + "\",\"quantity\":" + quantity + "}"));
    }
}

@jakarta.persistence.Entity
class Inventory {
    @jakarta.persistence.Id
    String productId;
    
    int available;
    int reserved;
    
    @jakarta.persistence.Version
    long version;  // incremented on every update; if stale → OptimisticLockException
    
    // getters/setters...
    public String getProductId() { return productId; }
    public int getAvailable() { return available; }
    public void setAvailable(int available) { this.available = available; }
    public int getReserved() { return reserved; }
    public void setReserved(int reserved) { this.reserved = reserved; }
    public void save(Inventory inventory) {}
}
```

---

## 56.4 Distributed Lock (Redis)

```java
import org.springframework.data.redis.core.*;
import org.springframework.stereotype.*;
import java.time.Duration;
import java.util.UUID;
import java.util.function.Supplier;

// Redlock Algorithm (simplified single-node for illustration)
// For production multi-node: use Redisson RedLock

@Service
public class DistributedLockService {
    
    private final StringRedisTemplate redis;
    
    private static final String LUA_RELEASE = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """;
    
    DistributedLockService(StringRedisTemplate redis) { this.redis = redis; }
    
    public <T> T withLock(String lockKey, Duration ttl, Duration waitTimeout,
                          Supplier<T> action) throws InterruptedException {
        var token = UUID.randomUUID().toString();
        var deadline = System.currentTimeMillis() + waitTimeout.toMillis();
        
        while (System.currentTimeMillis() < deadline) {
            // SET key value NX PX ttl (atomic)
            boolean acquired = Boolean.TRUE.equals(
                redis.opsForValue().setIfAbsent(lockKey, token, ttl)
            );
            
            if (acquired) {
                try {
                    return action.get();
                } finally {
                    releaseLock(lockKey, token);
                }
            }
            
            Thread.sleep(50 + (long)(Math.random() * 50));  // jitter
        }
        
        throw new RuntimeException("Could not acquire lock: " + lockKey);
    }
    
    private void releaseLock(String key, String token) {
        redis.execute(
            new org.springframework.data.redis.core.script.DefaultRedisScript<>(LUA_RELEASE, Long.class),
            java.util.List.of(key),
            token
        );
    }
    
    // Usage
    public void processOrder(String orderId) throws InterruptedException {
        withLock("order:lock:" + orderId, Duration.ofSeconds(30), Duration.ofSeconds(5), () -> {
            System.out.println("Processing order exclusively: " + orderId);
            return null;
        });
    }
}
```

---

## 56.5 Leader Election

```java
import org.springframework.integration.zookeeper.leader.*;
import org.springframework.integration.leader.*;

// Zookeeper-based leader election
@org.springframework.context.annotation.Configuration
class LeaderElectionConfig {
    
    @org.springframework.context.annotation.Bean
    LeaderInitiator leaderInitiator(
            org.apache.curator.framework.CuratorFramework curatorFramework) {
        return new LeaderInitiator(curatorFramework, new ClusterLeaderCandidate(), "/leader/my-app");
    }
}

@org.springframework.stereotype.Component
class ClusterLeaderCandidate implements Candidate {
    
    private volatile boolean isLeader = false;
    
    @Override
    public void onGranted(org.springframework.integration.leader.Context context) {
        isLeader = true;
        System.out.println("This node is now the LEADER");
        // Start background tasks that should only run on one node
        startLeaderTasks();
    }
    
    @Override
    public void onRevoked(org.springframework.integration.leader.Context context) {
        isLeader = false;
        System.out.println("Leadership REVOKED - stopping leader tasks");
        stopLeaderTasks();
    }
    
    @Override
    public String getRole() { return "master"; }
    
    @Override
    public String getId() {
        return java.net.InetAddress.getLoopbackAddress().getHostName();
    }
    
    public boolean isLeader() { return isLeader; }
    
    private void startLeaderTasks() { /* e.g. start @Scheduled jobs */ }
    private void stopLeaderTasks() { /* e.g. stop @Scheduled jobs */ }
}

// Only run scheduled task on leader
@org.springframework.stereotype.Component
class ClusterAwareScheduler {
    
    private final ClusterLeaderCandidate leaderCandidate;
    
    ClusterAwareScheduler(ClusterLeaderCandidate leaderCandidate) {
        this.leaderCandidate = leaderCandidate;
    }
    
    @org.springframework.scheduling.annotation.Scheduled(cron = "0 0 * * * *")
    public void hourlyCleanup() {
        if (!leaderCandidate.isLeader()) {
            return;  // Only leader does this
        }
        System.out.println("Running hourly cleanup (leader only)");
    }
}
```

---

## 56.6 Read-Your-Writes with Primary Routing

```java
import org.springframework.jdbc.datasource.lookup.*;

// Route reads to replica, writes to primary
// After write: route reads to primary temporarily (sticky session)
@org.springframework.context.annotation.Configuration
class DataSourceRoutingConfig {
    
    @org.springframework.context.annotation.Bean
    @org.springframework.boot.autoconfigure.condition.ConditionalOnProperty("db.replication.enabled")
    javax.sql.DataSource routingDataSource(
            @org.springframework.beans.factory.annotation.Qualifier("primaryDataSource") javax.sql.DataSource primary,
            @org.springframework.beans.factory.annotation.Qualifier("replicaDataSource") javax.sql.DataSource replica) {
        
        var routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                var context = DataSourceContext.get();
                // After write: use primary for read-your-writes
                return context.isRecentWrite() ? "primary" : "replica";
            }
        };
        
        routing.setTargetDataSources(java.util.Map.of(
            "primary", primary,
            "replica", replica
        ));
        routing.setDefaultTargetDataSource(replica);
        return routing;
    }
}

// Thread-local routing context
class DataSourceContext {
    
    private static final ThreadLocal<DataSourceContext> context = new ThreadLocal<>();
    private boolean recentWrite = false;
    private long writeTime = 0;
    
    static DataSourceContext get() {
        var ctx = context.get();
        if (ctx == null) {
            ctx = new DataSourceContext();
            context.set(ctx);
        }
        return ctx;
    }
    
    void markWrite() {
        this.recentWrite = true;
        this.writeTime = System.currentTimeMillis();
    }
    
    boolean isRecentWrite() {
        if (!recentWrite) return false;
        // Route to primary for 5 seconds after write
        if (System.currentTimeMillis() - writeTime > 5000) {
            recentWrite = false;
        }
        return recentWrite;
    }
    
    static void clear() { context.remove(); }
}
```

---

## สรุป Part 56

```
Distributed Systems Key Concepts:

CAP Theorem:
  Network partitions happen → choose CP or AP
  CP: banking, inventory (consistency required)
  AP: social media, shopping cart (availability required)

Consistency Models (strong → weak):
  Linearizable → Sequential → Causal → Eventual
  
  Most microservices use eventual consistency
  Use strong consistency only where required

Distributed Transactions:
  2PC = strong but slow and fragile
  Saga = eventual + compensations
  Outbox = at-least-once delivery without 2PC

Optimistic Locking:
  @Version in JPA entity
  Detect conflicts at commit time
  Good for low-contention writes

Distributed Lock (Redis):
  SET key value NX PX ttl (atomic)
  Release with Lua script (atomic check+delete)
  
Leader Election (Zookeeper/etcd):
  Only one node runs single-writer tasks
  Others stand by, take over on failure
```

➡️ [Part 57: Kotlin Multiplatform & Native](./Part-57-KotlinMultiplatform.md)
