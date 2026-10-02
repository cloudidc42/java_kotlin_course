# Part 95: Kafka Streams & Advanced Message-Driven Architecture
## ขั้นตอนที่ 6551-6620: Kafka Streams, Exactly-Once, Event Streaming Patterns

---

## 95.1 Kafka Streams Overview

```
Kafka Streams = library for stream processing on top of Kafka
ไม่ใช่ external system, runs inside your application

Concepts:
  KStream = unbounded stream of records (key, value)
  KTable = changelog stream → materialized as table (current state)
  GlobalKTable = replicated across all partitions
  
  Stateless ops: filter, map, flatMap, branch
  Stateful ops: groupBy, aggregate, join, windowing
  
Topology:
  Source Processor → read from Kafka topic
  Stream Processor → transform, filter, aggregate
  Sink Processor → write to Kafka topic

State Store:
  In-memory or RocksDB (persistent)
  Backed up to changelog Kafka topic
  Queryable via Interactive Queries

Exactly-Once Semantics:
  read-process-write atomically
  No duplicates even on reprocessing
  processing.guarantee = exactly_once_v2
```

---

## 95.2 Basic Kafka Streams Application

```kotlin
// build.gradle.kts
// implementation("org.apache.kafka:kafka-streams")
// implementation("org.springframework.kafka:spring-kafka")

// ====== Simple Word Count ======
@Configuration
class WordCountStreamConfig {
    
    @Bean
    fun wordCountTopology(): Topology {
        val builder = StreamsBuilder()
        
        builder
            .stream<String, String>("input-topic")
            .flatMapValues { line -> line.lowercase().split("\\s+".toRegex()) }
            .groupBy { _, word -> word }
            .count(Materialized.`as`("word-counts"))  // state store name
            .toStream()
            .to("word-count-output")
        
        return builder.build()
    }
    
    @Bean
    fun kafkaStreamsConfig(): KafkaStreamsConfiguration {
        return KafkaStreamsConfiguration(mapOf(
            StreamsConfig.APPLICATION_ID_CONFIG to "word-count-app",
            StreamsConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG to Serdes.String()::class.java,
            StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG to Serdes.String()::class.java,
            StreamsConfig.PROCESSING_GUARANTEE_CONFIG to StreamsConfig.EXACTLY_ONCE_V2,
            StreamsConfig.REPLICATION_FACTOR_CONFIG to 3,
            StreamsConfig.NUM_STREAM_THREADS_CONFIG to 4
        ))
    }
}
```

---

## 95.3 E-Commerce Order Processing Stream

```kotlin
// Domain events
data class OrderPlaced(
    val orderId: String,
    val customerId: String,
    val items: List<OrderItem>,
    val totalAmount: BigDecimal,
    val timestamp: Instant = Instant.now()
)

data class PaymentProcessed(
    val orderId: String,
    val paymentId: String,
    val status: PaymentStatus,
    val timestamp: Instant = Instant.now()
)

data class InventoryReserved(
    val orderId: String,
    val status: ReservationStatus,
    val failedItems: List<String> = emptyList(),
    val timestamp: Instant = Instant.now()
)

data class OrderStatus(
    val orderId: String,
    val status: String,
    val paymentStatus: PaymentStatus?,
    val inventoryStatus: ReservationStatus?,
    val lastUpdated: Instant = Instant.now()
)

// Complex stream topology
@Configuration
class OrderStreamTopology {
    
    @Bean
    fun orderProcessingTopology(): Topology {
        val builder = StreamsBuilder()
        
        val orderSerde = JsonSerde(OrderPlaced::class.java)
        val paymentSerde = JsonSerde(PaymentProcessed::class.java)
        val inventorySerde = JsonSerde(InventoryReserved::class.java)
        val statusSerde = JsonSerde(OrderStatus::class.java)
        
        // Source streams
        val orders: KStream<String, OrderPlaced> = builder.stream(
            "orders-placed",
            Consumed.with(Serdes.String(), orderSerde)
        )
        
        val payments: KTable<String, PaymentProcessed> = builder.table(
            "payments-processed",
            Consumed.with(Serdes.String(), paymentSerde),
            Materialized.`as`("payments-store")
        )
        
        val inventory: KTable<String, InventoryReserved> = builder.table(
            "inventory-reserved",
            Consumed.with(Serdes.String(), inventorySerde)
        )
        
        // ====== Route orders by value ======
        val (highValue, normalValue) = orders
            .split(Named.`as`("order-router"))
            .branch(
                { _, order -> order.totalAmount > BigDecimal(10000) },
                Branched.`as`("high-value")
            )
            .defaultBranch(Branched.`as`("normal"))
        
        // ====== Enrichment: join with customer data ======
        val customerTable: GlobalKTable<String, Customer> = builder.globalTable(
            "customers",
            Consumed.with(Serdes.String(), JsonSerde(Customer::class.java))
        )
        
        val enrichedOrders = orders.join(
            customerTable,
            { _, order -> order.customerId },  // key extractor
            { order, customer -> EnrichedOrder(order, customer) }
        )
        
        // ====== Aggregation: total sales per customer ======
        orders
            .groupBy(
                { _, order -> order.customerId },
                Grouped.with(Serdes.String(), orderSerde)
            )
            .aggregate(
                { CustomerSales("", BigDecimal.ZERO) },  // initializer
                { customerId, order, aggregate ->
                    aggregate.copy(
                        customerId = customerId,
                        totalSales = aggregate.totalSales + order.totalAmount
                    )
                },
                Materialized.`as`<String, CustomerSales>("customer-sales-store")
                    .withKeySerde(Serdes.String())
                    .withValueSerde(JsonSerde(CustomerSales::class.java))
            )
            .toStream()
            .to("customer-sales-summary")
        
        // ====== Join: combine order + payment + inventory status ======
        val orderWithPayment = orders.join(
            payments,
            { order, payment -> 
                OrderStatus(
                    orderId = order.orderId,
                    status = "PAYMENT_${payment.status}",
                    paymentStatus = payment.status,
                    inventoryStatus = null
                )
            },
            JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(30)),
            StreamJoined.with(Serdes.String(), orderSerde, paymentSerde)
        )
        
        orderWithPayment
            .join(
                inventory,
                { orderStatus, inv ->
                    orderStatus.copy(
                        inventoryStatus = inv.status,
                        status = if (orderStatus.paymentStatus == PaymentStatus.SUCCESS
                                   && inv.status == ReservationStatus.SUCCESS) "CONFIRMED"
                                 else "FAILED"
                    )
                }
            )
            .to("order-status-updates", Produced.with(Serdes.String(), statusSerde))
        
        return builder.build()
    }
}
```

---

## 95.4 Windowed Aggregations

```kotlin
// Windowed aggregations for time-based analytics

@Configuration
class AnalyticsStreamConfig {
    
    @Bean
    fun salesAnalyticsTopology(): Topology {
        val builder = StreamsBuilder()
        
        val orderSerde = JsonSerde(OrderPlaced::class.java)
        
        val orders = builder.stream<String, OrderPlaced>(
            "orders-placed",
            Consumed.with(
                Serdes.String(),
                orderSerde
            ).withTimestampExtractor { record, _ ->
                (record.value() as OrderPlaced).timestamp.toEpochMilli()
            }
        )
        
        // ====== Tumbling Window: non-overlapping, fixed size ======
        // Sales per hour (tumbling)
        orders
            .groupBy({ _, order -> "hourly" })
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .aggregate(
                { SalesAggregate(0L, BigDecimal.ZERO) },
                { _, order, agg -> 
                    SalesAggregate(
                        count = agg.count + 1,
                        total = agg.total + order.totalAmount
                    )
                }
            )
            .toStream()
            .map { windowedKey, agg ->
                val window = windowedKey.window()
                KeyValue.pair(
                    "hourly:${Instant.ofEpochMilli(window.start())}",
                    agg
                )
            }
            .to("hourly-sales-stats")
        
        // ====== Hopping Window: overlapping ======
        // 5-min stats, updated every 1 min
        orders
            .groupBy({ _, order -> order.customerId })
            .windowedBy(
                TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1))
                           .advanceBy(Duration.ofMinutes(1))  // hop by 1 min
            )
            .count()
            .toStream()
            .filter { windowedKey, count -> count > 10 }  // suspicious: >10 orders in 5 min
            .mapValues { count -> FraudAlert(reason = "High order frequency: $count in 5 min") }
            .to("fraud-alerts")
        
        // ====== Session Window: activity-based ======
        // Group user activity into sessions (idle gap = 30 min)
        orders
            .groupBy({ _, order -> order.customerId })
            .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)))
            .aggregate(
                { UserSession("", mutableListOf(), Instant.now()) },
                { customerId, order, session ->
                    session.copy(
                        customerId = customerId,
                        orders = (session.orders + order.orderId).toMutableList()
                    )
                },
                { _, s1, s2 -> s1.merge(s2) }  // session merge function
            )
            .toStream()
            .to("user-sessions")
        
        return builder.build()
    }
}

data class SalesAggregate(val count: Long, val total: BigDecimal)
```

---

## 95.5 Exactly-Once Semantics

```kotlin
// Exactly-once: each message processed exactly once, even after failures

// ====== Producer: Transactional ======
@Configuration
class TransactionalProducerConfig {
    
    @Bean
    fun transactionalKafkaTemplate(producerFactory: ProducerFactory<String, String>): KafkaTemplate<String, String> {
        val factory = DefaultKafkaProducerFactory(mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.TRANSACTIONAL_ID_CONFIG to "order-producer-1",  // enables transactions
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to true,
            ProducerConfig.ACKS_CONFIG to "all",
            ProducerConfig.RETRIES_CONFIG to Int.MAX_VALUE,
            ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION to 5
        ))
        factory.setTransactionIdPrefix("order-tx-")
        return KafkaTemplate(factory)
    }
}

// Use @Transactional for atomic Kafka + DB operations
@Service
class OrderCommandService(
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val orderRepository: OrderRepository
) {
    
    @Transactional("kafkaTransactionManager")
    fun createOrder(request: CreateOrderRequest): Order {
        // Both DB write and Kafka publish happen atomically
        val order = orderRepository.save(Order.create(request))
        
        kafkaTemplate.send("orders-placed", order.id, objectMapper.writeValueAsString(order))
        
        return order
    }
}

// ====== Outbox Pattern: DB + Kafka atomically ======
// Avoids distributed transaction problem
@Service
class OutboxOrderService(
    private val orderRepository: OrderRepository,
    private val outboxRepository: OutboxRepository
) {
    
    @Transactional  // single DB transaction
    fun createOrder(request: CreateOrderRequest): Order {
        val order = orderRepository.save(Order.create(request))
        
        // Write to outbox IN SAME TRANSACTION as domain record
        outboxRepository.save(OutboxMessage(
            aggregateId = order.id,
            aggregateType = "Order",
            eventType = "OrderPlaced",
            payload = objectMapper.writeValueAsString(order)
        ))
        
        return order
    }
}

// Separate poller publishes outbox messages to Kafka
@Component
class OutboxPoller(
    private val outboxRepository: OutboxRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>
) {
    @Scheduled(fixedDelay = 1000)
    @Transactional
    fun pollAndPublish() {
        val messages = outboxRepository.findUnpublished(limit = 100)
        messages.forEach { msg ->
            kafkaTemplate.send(msg.aggregateType.lowercase() + "-events", msg.aggregateId, msg.payload)
                .get(5, TimeUnit.SECONDS)
            outboxRepository.markPublished(msg.id)
        }
    }
}
```

---

## 95.6 Interactive Queries

```kotlin
// Query Kafka Streams state stores directly (without Kafka)

@RestController
@RequestMapping("/analytics")
class AnalyticsController(private val kafkaStreams: KafkaStreams) {
    
    @GetMapping("/sales/hourly/{hour}")
    fun getHourlySales(@PathVariable hour: String): SalesAggregate? {
        val storeQuery = QueryableStoreTypes.keyValueStore<String, SalesAggregate>()
        val store = kafkaStreams.store(
            StoreQueryParameters.fromNameAndType("hourly-sales-store", storeQuery)
        )
        return store["hourly:$hour"]
    }
    
    @GetMapping("/customers/{customerId}/total")
    fun getCustomerTotal(@PathVariable customerId: String): CustomerSales? {
        val store = kafkaStreams.store(
            StoreQueryParameters.fromNameAndType(
                "customer-sales-store",
                QueryableStoreTypes.keyValueStore<String, CustomerSales>()
            )
        )
        return store[customerId]
    }
    
    @GetMapping("/word-count")
    fun getAllWordCounts(): Map<String, Long> {
        val store = kafkaStreams.store(
            StoreQueryParameters.fromNameAndType(
                "word-counts",
                QueryableStoreTypes.keyValueStore<String, Long>()
            )
        )
        
        val result = mutableMapOf<String, Long>()
        store.all().use { iterator ->
            while (iterator.hasNext()) {
                val entry = iterator.next()
                result[entry.key] = entry.value
            }
        }
        return result
    }
    
    // Range query
    @GetMapping("/customers/range")
    fun getCustomerRange(
        @RequestParam from: String,
        @RequestParam to: String
    ): List<CustomerSales> {
        val store = kafkaStreams.store(
            StoreQueryParameters.fromNameAndType(
                "customer-sales-store",
                QueryableStoreTypes.keyValueStore<String, CustomerSales>()
            )
        )
        
        return buildList {
            store.range(from, to).use { iterator ->
                iterator.forEach { add(it.value) }
            }
        }
    }
}
```

---

## สรุป Part 95

```
Kafka Streams:

Core Concepts:
  KStream = infinite event log (append-only)
  KTable = current state (latest value per key)
  GlobalKTable = replicated to all nodes
  Topology = DAG of processors

Stateless Operations:
  filter, map, flatMap, branch, merge

Stateful Operations:
  groupBy → aggregate/reduce/count
  join: KStream-KStream (window), KStream-KTable, KTable-KTable
  windowing: tumbling, hopping, session

Windowing:
  Tumbling: fixed size, no overlap (hourly, daily)
  Hopping: fixed size, overlapping (5min every 1min)
  Session: activity-based (idle gap = new session)

Exactly-Once:
  processing.guarantee = exactly_once_v2
  Transactional producer: TRANSACTIONAL_ID_CONFIG
  Outbox pattern: DB + Kafka in one DB transaction

State Stores:
  In-memory: fast, lost on restart
  RocksDB: persisted, backed by changelog topic
  Interactive Queries: HTTP API over local state

Best Practices:
  Use changelog topics for fault tolerance
  Size partitions based on parallelism needs
  Use Serdes for type safety
  Monitor: consumer lag, processing latency
  Handle late events with grace periods
  
vs. Spring Integration/Kafka:
  @KafkaListener: simple consume-process
  Kafka Streams: stateful processing, joins, aggregations
  Use Kafka Streams when: counts, joins, windowing needed
```

➡️ [Part 96: Cloud-Native Patterns & Kubernetes](./Part-96-CloudNative.md)
