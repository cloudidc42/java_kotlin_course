# Part 49: Message-Driven Architecture
## ขั้นตอนที่ 3331-3400: Transactional Outbox, SAGA, Async Workflows

---

## 49.1 Message-Driven Architecture Patterns

```
Coupling levels:
  Tight coupling:   ServiceA calls ServiceB directly → if B is down, A fails
  Loose coupling:   ServiceA publishes event → ServiceB processes when ready

Patterns:
  1. Transactional Outbox  = guarantee message delivery
  2. SAGA                  = distributed transaction via events
  3. CQRS + Event Sourcing = separate read/write models (Part 36)
  4. Competing Consumers   = horizontal scaling of message processing

Guarantees:
  At-most-once  = might lose message (fastest, fire & forget)
  At-least-once = might duplicate (most common: ack after processing)
  Exactly-once  = hardest (Kafka transactions, idempotent consumers)
```

---

## 49.2 Transactional Outbox Pattern

```java
import jakarta.persistence.*;
import org.springframework.scheduling.annotation.*;
import org.springframework.transaction.annotation.*;

// Problem: save to DB and publish to Kafka in same "transaction"
// DB + Kafka can't share a transaction → data loss risk

// Solution: Outbox pattern
// 1. Write to DB + write to outbox table IN SAME TRANSACTION
// 2. Background job reads outbox, publishes to Kafka, marks as sent

@Entity
@Table(name = "outbox_events")
class OutboxEvent(
    @Id val id: String = java.util.UUID.randomUUID().toString(),
    val aggregateId: String,
    val aggregateType: String,
    val eventType: String,
    @Column(columnDefinition = "TEXT") val payload: String,
    @Enumerated(EnumType.STRING) var status: OutboxStatus = OutboxStatus.PENDING,
    val createdAt: java.time.Instant = java.time.Instant.now(),
    var processedAt: java.time.Instant? = null,
    var retryCount: Int = 0
)

enum class OutboxStatus { PENDING, PROCESSING, SENT, FAILED }

interface OutboxRepository : org.springframework.data.jpa.repository.JpaRepository<OutboxEvent, String> {
    @org.springframework.data.jpa.repository.Query(
        "SELECT e FROM OutboxEvent e WHERE e.status = 'PENDING' ORDER BY e.createdAt LIMIT :limit"
    )
    fun findPending(@org.springframework.data.repository.query.Param("limit") limit: Int): List<OutboxEvent>
}

// Save order + outbox in same transaction
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val outboxRepository: OutboxRepository,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) {
    
    @Transactional
    fun createOrder(request: CreateOrderRequest): Order {
        val order = orderRepository.save(Order(request.userId, request.items))
        
        // Write outbox event in SAME transaction
        outboxRepository.save(
            OutboxEvent(
                aggregateId = order.id,
                aggregateType = "Order",
                eventType = "ORDER_CREATED",
                payload = objectMapper.writeValueAsString(
                    OrderCreatedPayload(order.id, order.userId, order.total)
                )
            )
        )
        
        return order
    }
}

// Outbox relay job
@Component
class OutboxRelayJob(
    private val outboxRepository: OutboxRepository,
    private val kafkaTemplate: org.springframework.kafka.core.KafkaTemplate<String, String>
) {
    
    @Scheduled(fixedDelay = 1000)  // every second
    @Transactional
    fun relayPendingEvents() {
        val events = outboxRepository.findPending(100)
        
        events.forEach { event ->
            try {
                event.status = OutboxStatus.PROCESSING
                outboxRepository.save(event)
                
                kafkaTemplate.send(
                    "domain-events",
                    event.aggregateId,
                    event.payload
                ).get(5, java.util.concurrent.TimeUnit.SECONDS)
                
                event.status = OutboxStatus.SENT
                event.processedAt = java.time.Instant.now()
                
            } catch (e: Exception) {
                event.retryCount++
                event.status = if (event.retryCount >= 3) OutboxStatus.FAILED else OutboxStatus.PENDING
                System.err.println("Outbox relay failed for ${event.id}: ${e.message}")
            } finally {
                outboxRepository.save(event)
            }
        }
    }
}
```

---

## 49.3 Idempotent Consumer

```java
import jakarta.persistence.*;
import org.springframework.kafka.annotation.*;

// Problem: At-least-once delivery means we might process same message twice
// Solution: Track processed message IDs, skip duplicates

@Entity
@Table(name = "processed_events")
class ProcessedEvent(
    @Id val eventId: String,
    val processedAt: java.time.Instant = java.time.Instant.now()
)

interface ProcessedEventRepository : 
    org.springframework.data.jpa.repository.JpaRepository<ProcessedEvent, String> {
    fun existsById(id: String): Boolean
}

@Component
class IdempotentOrderConsumer(
    private val processedEventRepo: ProcessedEventRepository,
    private val orderService: OrderService
) {
    
    @KafkaListener(topics = ["domain-events"], groupId = "order-processor")
    @org.springframework.transaction.annotation.Transactional
    fun handleEvent(
        @org.springframework.messaging.handler.annotation.Payload payload: String,
        @org.springframework.messaging.handler.annotation.Header(
            org.springframework.kafka.support.KafkaHeaders.RECEIVED_MESSAGE_KEY
        ) eventId: String,
        acknowledgment: org.springframework.kafka.support.Acknowledgment
    ) {
        // Check if already processed
        if (processedEventRepo.existsById(eventId)) {
            acknowledgment.acknowledge()
            return
        }
        
        try {
            processEvent(payload)
            
            // Mark as processed (in same transaction)
            processedEventRepo.save(ProcessedEvent(eventId))
            
            acknowledgment.acknowledge()
            
        } catch (e: Exception) {
            System.err.println("Failed to process $eventId: ${e.message}")
            // Don't ack → will be redelivered
        }
    }
    
    private fun processEvent(payload: String) {
        // process...
    }
}
```

---

## 49.4 SAGA Pattern (Choreography)

```kotlin
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

// ====== Events ======
data class OrderCreated(val orderId: String, val userId: String, val total: Double, val items: List<String>)
data class PaymentProcessed(val orderId: String, val paymentId: String, val amount: Double)
data class PaymentFailed(val orderId: String, val reason: String)
data class InventoryReserved(val orderId: String)
data class InventoryFailed(val orderId: String, val reason: String)
data class OrderCompleted(val orderId: String)
data class OrderFailed(val orderId: String, val reason: String)

// ====== Payment Service ======
@Service
class PaymentSagaHandler(
    private val paymentService: PaymentService,
    private val kafka: KafkaTemplate<String, Any>
) {
    
    @KafkaListener(topics = ["order.created"], groupId = "payment-saga")
    @Transactional
    fun onOrderCreated(event: OrderCreated) {
        try {
            val payment = paymentService.charge(event.orderId, event.userId, event.total)
            kafka.send("payment.processed", event.orderId, 
                PaymentProcessed(event.orderId, payment.id, event.total))
        } catch (e: Exception) {
            kafka.send("payment.failed", event.orderId,
                PaymentFailed(event.orderId, e.message ?: "Payment failed"))
        }
    }
    
    // Compensation: reverse charge on inventory failure
    @KafkaListener(topics = ["inventory.failed"], groupId = "payment-saga")
    @Transactional
    fun onInventoryFailed(event: InventoryFailed) {
        paymentService.refund(event.orderId)
        kafka.send("order.failed", event.orderId, OrderFailed(event.orderId, event.reason))
    }
}

// ====== Inventory Service ======
@Service
class InventorySagaHandler(
    private val inventoryService: InventoryService,
    private val kafka: KafkaTemplate<String, Any>
) {
    
    @KafkaListener(topics = ["payment.processed"], groupId = "inventory-saga")
    @Transactional
    fun onPaymentProcessed(event: PaymentProcessed) {
        try {
            inventoryService.reserve(event.orderId)
            kafka.send("inventory.reserved", event.orderId, InventoryReserved(event.orderId))
        } catch (e: Exception) {
            kafka.send("inventory.failed", event.orderId,
                InventoryFailed(event.orderId, e.message ?: "Inventory failed"))
        }
    }
}

// ====== Order Orchestrator (tracks SAGA state) ======
@Service
class OrderSagaOrchestrator(
    private val orderRepository: OrderRepository,
    private val kafka: KafkaTemplate<String, Any>
) {
    
    @KafkaListener(topics = ["inventory.reserved"], groupId = "order-saga")
    @Transactional
    fun onInventoryReserved(event: InventoryReserved) {
        orderRepository.updateStatus(event.orderId, "CONFIRMED")
        kafka.send("order.completed", event.orderId, OrderCompleted(event.orderId))
    }
    
    @KafkaListener(topics = ["order.failed"], groupId = "order-saga")
    @Transactional
    fun onOrderFailed(event: OrderFailed) {
        orderRepository.updateStatus(event.orderId, "FAILED")
        // Notify user
    }
}

interface InventoryService { fun reserve(orderId: String) }
```

---

## 49.5 Dead Letter Queue Processing

```java
import org.springframework.kafka.annotation.*;
import org.springframework.stereotype.*;

@Component
public class DLQProcessor {
    
    private final OrderRepository orderRepository;
    private final AlertService alertService;
    private final org.springframework.kafka.core.KafkaTemplate<String, String> kafkaTemplate;
    
    public DLQProcessor(OrderRepository orderRepository, AlertService alertService,
                        org.springframework.kafka.core.KafkaTemplate<String, String> kafkaTemplate) {
        this.orderRepository = orderRepository;
        this.alertService = alertService;
        this.kafkaTemplate = kafkaTemplate;
    }
    
    @KafkaListener(topics = "orders.DLT", groupId = "dlq-processor")
    public void handleDeadLetter(
            org.apache.kafka.clients.consumer.ConsumerRecord<String, String> record,
            @org.springframework.messaging.handler.annotation.Header(
                org.springframework.kafka.support.KafkaHeaders.EXCEPTION_MESSAGE
            ) String exceptionMessage,
            org.springframework.kafka.support.Acknowledgment ack) {
        
        System.err.printf("DLQ record: key=%s exception=%s%n",
            record.key(), exceptionMessage);
        
        // Analyze failure
        if (isRetryable(exceptionMessage)) {
            // Wait and retry after delay
            scheduleRetry(record.key(), record.value(), 5);
        } else {
            // Permanent failure: alert and store for manual review
            alertService.alert("Permanent failure for order: " + record.key());
            storeForManualReview(record.key(), record.value(), exceptionMessage);
        }
        
        ack.acknowledge();
    }
    
    private boolean isRetryable(String exception) {
        return exception.contains("Timeout") || exception.contains("Connection");
    }
    
    private void scheduleRetry(String key, String payload, int delayMinutes) {
        // Schedule re-publish after delay (can use scheduler or separate job)
        java.util.concurrent.Executors.newSingleThreadScheduledExecutor()
            .schedule(() -> kafkaTemplate.send("orders", key, payload),
                delayMinutes, java.util.concurrent.TimeUnit.MINUTES);
    }
    
    private void storeForManualReview(String key, String payload, String error) {
        // Save to DB for manual review
        System.out.println("Stored for review: " + key);
    }
}
```

---

## 49.6 Competing Consumers & Partitioning

```
Kafka Partitioning Strategy:

Orders topic with 3 partitions:
  Partition 0: userId % 3 == 0
  Partition 1: userId % 3 == 1
  Partition 2: userId % 3 == 2

Consumer Group with 3 consumers:
  Consumer 1 → Partition 0
  Consumer 2 → Partition 1
  Consumer 3 → Partition 2

Benefits:
  - Ordered processing per user (all events for same user → same partition)
  - Horizontal scaling (add consumers up to partition count)
  - Load balanced

Partition key selection:
  Use: orderId, userId, sessionId (consistent routing)
  Avoid: random key (breaks ordering), too few values (hot partition)
```

```java
// Custom partition key
@Service
class OrderKafkaProducer {
    
    private final org.springframework.kafka.core.KafkaTemplate<String, Object> kafkaTemplate;
    
    OrderKafkaProducer(org.springframework.kafka.core.KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void publishOrderEvent(String userId, OrderEvent event) {
        // Use userId as partition key → consistent routing
        kafkaTemplate.send("orders", userId, event);
    }
}

// Concurrent consumers (1 per partition)
@Component
class OrderConsumer {
    
    @KafkaListener(
        topics = "orders",
        groupId = "order-processor",
        concurrency = "3"  // 3 concurrent consumers = 3 threads
    )
    public void handleOrder(
            @org.springframework.messaging.handler.annotation.Payload OrderEvent event,
            @org.springframework.messaging.handler.annotation.Header(
                org.springframework.kafka.support.KafkaHeaders.RECEIVED_PARTITION
            ) int partition,
            org.springframework.kafka.support.Acknowledgment ack) {
        
        System.out.printf("[P%d] Processing: %s%n", partition, event.orderId());
        // Each partition handled by dedicated consumer thread
        
        ack.acknowledge();
    }
}
```

---

## สรุป Part 49

```
Message-Driven Patterns:

Transactional Outbox:
  Problem: DB + Kafka can't share atomic transaction
  Solution: Write to outbox table in DB transaction
            → Background job publishes outbox to Kafka
  
  Guarantees: At-least-once delivery with no data loss

Idempotent Consumer:
  Problem: At-least-once → might process same message twice
  Solution: Track processed message IDs, skip duplicates
  
  With database: INSERT IGNORE / INSERT IF NOT EXISTS in same TX

SAGA (Choreography):
  Each service listens to events, does its work, publishes next event
  Compensation: reverse actions on failure
  
  Flow: OrderCreated → PaymentProcessed → InventoryReserved → OrderCompleted
                     ↓ (on failure)
                PaymentFailed → [compensate] → OrderFailed

SAGA (Orchestration): Central coordinator (Conductor, Temporal, Axon)
```

➡️ [Part 50: Service Mesh & Advanced Deployment](./Part-50-ServiceMesh.md)
