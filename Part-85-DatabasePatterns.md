# Part 85: Advanced Database Patterns
## ขั้นตอนที่ 5851-5920: Optimistic/Pessimistic Locking, Outbox, Saga Pattern

---

## 85.1 Database Locking Strategies

```
Concurrency Problem:
  Thread A reads balance = 1000
  Thread B reads balance = 1000
  Thread A: balance += 500 → writes 1500
  Thread B: balance += 300 → writes 1300  ← WRONG! Should be 1800

Solutions:
  1. Optimistic Locking: assume no conflict, verify at write time
  2. Pessimistic Locking: assume conflict, lock rows upfront
  3. Serializable Isolation: DB handles it automatically

Optimistic Locking:
  ✓ High throughput (no locks held during processing)
  ✓ Good for low-contention scenarios
  ✗ RetryableException on conflict (must handle)

Pessimistic Locking:
  ✓ Guaranteed no conflict
  ✗ Lower throughput (locks held)
  ✗ Deadlock risk
```

---

## 85.2 Optimistic Locking with JPA

```kotlin
// Add @Version field to entity
@Entity
@Table(name = "accounts")
data class Account(
    @Id val id: String,
    val customerId: String,
    val balanceCents: Long,
    
    @Version  // JPA manages this automatically
    val version: Long = 0
)

// Repository with optimistic locking
@Repository
interface AccountRepository : JpaRepository<Account, String> {
    @Query("SELECT a FROM Account a WHERE a.customerId = :customerId")
    fun findByCustomerIdForUpdate(customerId: String): Account?
}

// Service: retry on OptimisticLockException
@Service
class AccountService(private val accountRepository: AccountRepository) {
    
    @Transactional
    @Retryable(
        retryFor = [OptimisticLockingFailureException::class],
        maxAttempts = 3,
        backoff = Backoff(delay = 100, multiplier = 2.0)
    )
    fun transferFunds(fromId: String, toId: String, amount: Long) {
        val from = accountRepository.findById(fromId).orElseThrow()
        val to = accountRepository.findById(toId).orElseThrow()
        
        check(from.balanceCents >= amount) { "Insufficient funds" }
        
        // Update both accounts
        accountRepository.save(from.copy(balanceCents = from.balanceCents - amount))
        accountRepository.save(to.copy(balanceCents = to.balanceCents + amount))
        
        // If another transaction modified either account between read and save,
        // JPA throws OptimisticLockingFailureException → @Retryable retries
    }
    
    // Or handle manually:
    @Transactional
    fun updateBalance(accountId: String, delta: Long): Account {
        var retries = 0
        while (retries < 3) {
            try {
                val account = accountRepository.findById(accountId).orElseThrow()
                return accountRepository.save(account.copy(balanceCents = account.balanceCents + delta))
            } catch (e: OptimisticLockingFailureException) {
                if (++retries >= 3) throw e
                Thread.sleep(100 * retries.toLong())
            }
        }
        throw RuntimeException("Failed after 3 retries")
    }
}
```

---

## 85.3 Pessimistic Locking

```kotlin
// SELECT FOR UPDATE / SELECT FOR SHARE

@Repository
interface AccountRepository : JpaRepository<Account, String> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)  // SELECT FOR UPDATE
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    fun findByIdForWrite(id: String): Account?
    
    @Lock(LockModeType.PESSIMISTIC_READ)   // SELECT FOR SHARE
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    fun findByIdForRead(id: String): Account?
    
    // With timeout (avoid waiting forever)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    fun findByIdForWriteWithTimeout(id: String): Account?
}

// SKIP LOCKED: process only available rows (queue processing)
@Repository
interface JobRepository : JpaRepository<Job, String> {
    
    @Query(value = """
        SELECT * FROM jobs 
        WHERE status = 'PENDING'
        ORDER BY created_at
        LIMIT :batchSize
        FOR UPDATE SKIP LOCKED
    """, nativeQuery = true)
    fun claimNextBatch(batchSize: Int): List<Job>
}
```

---

## 85.4 Transactional Outbox Pattern

```kotlin
// Problem: Save to DB + publish Kafka event atomically
// If DB commit succeeds but Kafka publish fails → lost event

// Solution: Transactional Outbox
// 1. Save entity + outbox message in same transaction (same DB)
// 2. Separate process reads outbox and publishes to Kafka
// 3. Delete from outbox after confirmed publish

@Entity
@Table(name = "outbox_messages")
data class OutboxMessage(
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    val id: UUID? = null,
    val aggregateType: String,
    val aggregateId: String,
    val eventType: String,
    
    @Column(columnDefinition = "jsonb")
    val payload: String,
    
    val createdAt: java.time.Instant = java.time.Instant.now(),
    var processedAt: java.time.Instant? = null,
    var failureReason: String? = null,
    var retryCount: Int = 0
)

// Service: write to outbox in same transaction
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val outboxRepository: OutboxRepository,
    private val objectMapper: ObjectMapper
) {
    @Transactional  // Both order + outbox in same transaction
    fun createOrder(customerId: String): Order {
        val order = Order(
            id = UUID.randomUUID().toString(),
            customerId = customerId,
            status = OrderStatus.DRAFT,
            createdAt = java.time.Instant.now()
        )
        
        val saved = orderRepository.save(order)
        
        // Write to outbox (same transaction)
        outboxRepository.save(OutboxMessage(
            aggregateType = "Order",
            aggregateId = saved.id,
            eventType = "OrderCreated",
            payload = objectMapper.writeValueAsString(mapOf(
                "orderId" to saved.id,
                "customerId" to saved.customerId
            ))
        ))
        
        return saved
    }
}

// Outbox Poller: reads and publishes
@Component
class OutboxPoller(
    private val outboxRepository: OutboxRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val objectMapper: ObjectMapper
) {
    @Scheduled(fixedDelay = 1000)  // every 1 second
    @Transactional
    fun processOutbox() {
        val messages = outboxRepository.findUnprocessed(limit = 100)
        
        messages.forEach { message ->
            try {
                kafkaTemplate.send(
                    "${message.aggregateType}.${message.eventType}",
                    message.aggregateId,
                    message.payload
                ).get(5, TimeUnit.SECONDS)  // wait for acknowledgment
                
                message.processedAt = java.time.Instant.now()
                outboxRepository.save(message)
                
            } catch (e: Exception) {
                message.retryCount++
                message.failureReason = e.message
                outboxRepository.save(message)
                
                if (message.retryCount >= 5) {
                    // Move to dead letter / alert
                    log.error("Outbox message failed after 5 retries: ${message.id}")
                }
            }
        }
    }
    
    // Clean up old processed messages
    @Scheduled(cron = "0 0 3 * * *")  // 3 AM daily
    fun cleanupProcessed() {
        val cutoff = java.time.Instant.now().minus(30, ChronoUnit.DAYS)
        outboxRepository.deleteProcessedBefore(cutoff)
    }
}
```

---

## 85.5 Saga Pattern (Distributed Transactions)

```kotlin
// Saga: sequence of local transactions with compensating actions
// If step N fails → run compensation for steps N-1, N-2, ...

// Choreography-based Saga (using events)
// Each service listens to events and reacts

// Order Saga steps:
// 1. Order Created → Reserve Inventory
// 2. Inventory Reserved → Charge Payment
// 3. Payment Charged → Confirm Order
// 
// If Payment fails:
// → Release Inventory (compensation)
// → Reject Order (compensation)

// Order Service
@KafkaListener(topics = ["PaymentCharged"])
fun onPaymentCharged(event: PaymentChargedEvent) {
    orderService.confirm(event.orderId)
}

@KafkaListener(topics = ["PaymentFailed"])
fun onPaymentFailed(event: PaymentFailedEvent) {
    // Compensate
    orderService.cancel(event.orderId, "Payment failed: ${event.reason}")
    // Also publish OrderCancelled → Inventory Service will release
}

// Inventory Service
@KafkaListener(topics = ["OrderCreated"])
fun onOrderCreated(event: OrderCreatedEvent) {
    try {
        inventoryService.reserve(event.orderId, event.items)
        eventPublisher.publish(InventoryReservedEvent(event.orderId))
    } catch (e: InsufficientStockException) {
        eventPublisher.publish(InventoryReservationFailed(event.orderId, e.message))
    }
}

@KafkaListener(topics = ["OrderCancelled"])  // compensation
fun onOrderCancelled(event: OrderCancelledEvent) {
    inventoryService.release(event.orderId)
}

// Saga State Machine (Orchestration-based Saga)
enum class SagaState {
    STARTED,
    INVENTORY_RESERVED,
    INVENTORY_FAILED,
    PAYMENT_CHARGED,
    PAYMENT_FAILED,
    COMPLETED,
    COMPENSATING,
    COMPENSATED
}

@Entity
data class OrderSaga(
    @Id val sagaId: String,
    val orderId: String,
    
    @Enumerated(EnumType.STRING)
    var state: SagaState = SagaState.STARTED,
    
    var compensationStep: Int = 0,
    val createdAt: java.time.Instant = java.time.Instant.now(),
    var updatedAt: java.time.Instant = java.time.Instant.now()
)

@Service
class OrderSagaOrchestrator(
    private val sagaRepository: OrderSagaRepository,
    private val inventoryClient: InventoryClient,
    private val paymentClient: PaymentClient,
    private val orderService: OrderService
) {
    fun start(orderId: String): OrderSaga {
        val saga = OrderSaga(
            sagaId = UUID.randomUUID().toString(),
            orderId = orderId
        )
        sagaRepository.save(saga)
        
        step1_reserveInventory(saga)
        return saga
    }
    
    private fun step1_reserveInventory(saga: OrderSaga) {
        try {
            inventoryClient.reserve(saga.orderId)
            saga.state = SagaState.INVENTORY_RESERVED
            sagaRepository.save(saga)
            step2_chargePayment(saga)
        } catch (e: Exception) {
            saga.state = SagaState.INVENTORY_FAILED
            sagaRepository.save(saga)
            compensate(saga, from = 0)  // nothing to compensate yet
        }
    }
    
    private fun step2_chargePayment(saga: OrderSaga) {
        try {
            paymentClient.charge(saga.orderId)
            saga.state = SagaState.PAYMENT_CHARGED
            sagaRepository.save(saga)
            complete(saga)
        } catch (e: Exception) {
            saga.state = SagaState.PAYMENT_FAILED
            sagaRepository.save(saga)
            compensate(saga, from = 1)  // compensate step 1 (inventory)
        }
    }
    
    private fun compensate(saga: OrderSaga, from: Int) {
        saga.state = SagaState.COMPENSATING
        
        when {
            from >= 1 -> inventoryClient.release(saga.orderId)  // compensate step 1
        }
        
        orderService.cancel(saga.orderId, "Saga compensation")
        saga.state = SagaState.COMPENSATED
        sagaRepository.save(saga)
    }
    
    private fun complete(saga: OrderSaga) {
        orderService.confirm(saga.orderId)
        saga.state = SagaState.COMPLETED
        sagaRepository.save(saga)
    }
}
```

---

## 85.6 Database Connection Pooling Tuning

```yaml
# HikariCP tuning (application.yaml)
spring:
  datasource:
    hikari:
      # Pool sizing formula: (core_count * 2) + effective_spindle_count
      # For 4-core, SSD: (4*2) + 1 = 9 → round to 10
      maximum-pool-size: 10
      minimum-idle: 5
      
      # Timeout settings
      connection-timeout: 30000    # 30s: time to get connection from pool
      idle-timeout: 600000         # 10min: max idle time
      max-lifetime: 1800000        # 30min: max connection lifetime
      keepalive-time: 60000        # 1min: ping idle connections
      
      # Connection validation
      connection-test-query: SELECT 1
      validation-timeout: 5000     # 5s
      
      # Performance
      auto-commit: false           # explicit commit (with @Transactional)
      
      pool-name: ShopHikariPool
      register-mbeans: true        # JMX monitoring
```

```kotlin
// Monitor HikariCP metrics via Micrometer
@Component
class HikariPoolMonitor(
    private val dataSource: HikariDataSource,
    private val meterRegistry: MeterRegistry
) {
    @PostConstruct
    fun registerMetrics() {
        Gauge.builder("db.pool.active") { dataSource.hikariPoolMXBean.activeConnections }
            .register(meterRegistry)
        Gauge.builder("db.pool.idle") { dataSource.hikariPoolMXBean.idleConnections }
            .register(meterRegistry)
        Gauge.builder("db.pool.waiting") { dataSource.hikariPoolMXBean.threadsAwaitingConnection }
            .register(meterRegistry)
    }
}
```

---

## สรุป Part 85

```
Database Concurrency Patterns:

Optimistic Locking (@Version):
  Read → process → write (check version)
  If version mismatch → throw OptimisticLockingFailureException
  Use: low contention, short transactions
  Handle: @Retryable(maxAttempts=3)

Pessimistic Locking (SELECT FOR UPDATE):
  Lock row immediately on read
  Use: high contention, must prevent concurrent modification
  Risk: deadlock (always acquire locks in same order)
  SKIP LOCKED: skip locked rows (queue processing)

Transactional Outbox:
  Problem: DB update + event publish (two different systems)
  Solution: write event to outbox table in same transaction
  Poller: reads outbox → publishes → marks processed
  Idempotent consumer: handle duplicate events

Saga Pattern:
  Choreography: services react to events
  Orchestration: central coordinator manages flow
  Both: compensating transactions for rollback
  
  Saga State Machine:
    Track saga progress
    Resume on crash
    Execute compensation

HikariCP Sizing:
  (CPU cores * 2) + effective_spindle_count
  max-lifetime < DB's wait_timeout
  Monitor: active, idle, pending threads
```

➡️ [Part 86: Distributed Systems Patterns](./Part-86-DistributedPatterns.md)
