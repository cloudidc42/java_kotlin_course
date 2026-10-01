# Part 37: Apache Kafka
## ขั้นตอนที่ 2491-2560: Event Streaming ระดับ Enterprise

---

## 37.1 Kafka Overview

```
Kafka Architecture:

Producer → [Topic: orders] → Consumer Group
                ↕
           Broker Cluster
           (replicated)

Concepts:
  Topic    = category/feed of records
  Partition = ordered, immutable sequence (unit of parallelism)
  Offset   = position in partition
  Consumer Group = multiple consumers sharing load
  Broker   = Kafka server
  Zookeeper/KRaft = cluster coordination

Message retention: 7 days (default), can be forever
Each partition is read by exactly 1 consumer in a group
Partitions > consumers = some consumers idle
Partitions < consumers = optimal parallelism

Kafka vs RabbitMQ:
  Kafka:      high-throughput, replayable, ordered within partition
  RabbitMQ:   routing, priorities, simpler setup, acknowledgements
```

---

## 37.2 Spring Boot Kafka

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                          # wait for all replicas
      retries: 3
      properties:
        enable.idempotence: true         # exactly-once semantics
        max.in.flight.requests.per.connection: 5
    
    consumer:
      group-id: my-app-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest        # start from beginning if no offset
      enable-auto-commit: false          # manual commit for reliability
      properties:
        spring.json.trusted.packages: com.example.*
    
    listener:
      ack-mode: MANUAL_IMMEDIATE         # commit after processing
      concurrency: 3                     # 3 consumer threads

app:
  kafka:
    topics:
      orders: orders
      payments: payments
      notifications: notifications
```

---

## 37.3 Producer

```java
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.*;
import org.springframework.stereotype.*;

record OrderEvent(
    String orderId, String userId, String status,
    double total, java.time.Instant timestamp
) {}

record PaymentEvent(
    String paymentId, String orderId, double amount,
    String status, java.time.Instant timestamp
) {}

@Service
public class OrderEventProducer {
    
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public OrderEventProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void publishOrderCreated(String orderId, String userId, double total) {
        OrderEvent event = new OrderEvent(
            orderId, userId, "CREATED", total, java.time.Instant.now()
        );
        
        // Key = orderId ensures all events for same order go to same partition
        var future = kafkaTemplate.send("orders", orderId, event);
        
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                RecordMetadata meta = result.getRecordMetadata();
                System.out.printf("Published order event: partition=%d, offset=%d%n",
                    meta.partition(), meta.offset());
            } else {
                System.err.println("Failed to publish: " + ex.getMessage());
                // Handle: retry, dead letter queue, etc.
            }
        });
    }
    
    // Transactional producer
    public void publishOrderWithPayment(Order order, Payment payment) {
        kafkaTemplate.executeInTransaction(ops -> {
            ops.send("orders", order.getId(), new OrderEvent(
                order.getId(), order.getUserId(), "CONFIRMED", 
                order.getTotal(), java.time.Instant.now()
            ));
            ops.send("payments", payment.getId(), new PaymentEvent(
                payment.getId(), order.getId(), payment.getAmount(),
                "COMPLETED", java.time.Instant.now()
            ));
            return null;
        });
    }
}
```

---

## 37.4 Consumer

```java
import org.apache.kafka.clients.consumer.*;
import org.springframework.kafka.annotation.*;
import org.springframework.kafka.support.*;
import org.springframework.messaging.handler.annotation.*;

@org.springframework.stereotype.Component
public class OrderEventConsumer {
    
    // Simple consumer
    @KafkaListener(topics = "orders", groupId = "order-processor")
    public void handleOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {
        
        try {
            System.out.printf("[P%d|O%d] Processing order: %s status=%s%n",
                partition, offset, event.orderId(), event.status());
            
            processOrder(event);
            
            ack.acknowledge();  // commit only after successful processing
            
        } catch (Exception e) {
            System.err.println("Error processing: " + e.getMessage());
            // Don't ack → will be redelivered
            // Or send to DLQ
        }
    }
    
    // Batch consumer
    @KafkaListener(topics = "orders", groupId = "batch-processor",
                   containerFactory = "batchListenerContainerFactory")
    public void handleBatch(
            List<ConsumerRecord<String, OrderEvent>> records,
            Acknowledgment ack) {
        
        System.out.println("Processing batch of " + records.size());
        
        try {
            List<OrderEvent> events = records.stream()
                .map(ConsumerRecord::value)
                .toList();
            
            batchProcess(events);
            ack.acknowledge();
            
        } catch (Exception e) {
            // Retry logic...
        }
    }
    
    // Multiple topics
    @KafkaListener(topics = {"orders", "payments"}, groupId = "analytics")
    public void handleAll(
            @Payload Object payload,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        System.out.println("From topic [" + topic + "]: " + payload);
    }
    
    // With filter
    @KafkaListener(
        topics = "orders",
        groupId = "confirmed-only",
        filter = "orderFilter"  // bean name
    )
    public void handleConfirmedOnly(OrderEvent event) {
        // Only receives CONFIRMED events (filtered by bean)
        processConfirmedOrder(event);
    }
    
    private void processOrder(OrderEvent e) { /* business logic */ }
    private void batchProcess(List<OrderEvent> events) { /* batch logic */ }
    private void processConfirmedOrder(OrderEvent e) { /* logic */ }
}

// Kafka configuration
@org.springframework.context.annotation.Configuration
class KafkaConsumerConfig {
    
    @org.springframework.context.annotation.Bean
    public org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
    batchListenerContainerFactory(
            org.springframework.kafka.core.ConsumerFactory<String, OrderEvent> consumerFactory) {
        
        var factory = new org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory<String, OrderEvent>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);
        factory.getContainerProperties().setAckMode(
            org.springframework.kafka.listener.ContainerProperties.AckMode.MANUAL_IMMEDIATE
        );
        return factory;
    }
    
    @org.springframework.context.annotation.Bean
    public org.springframework.kafka.listener.adapter.RecordFilterStrategy<String, OrderEvent>
    orderFilter() {
        return record -> !record.value().status().equals("CONFIRMED");
        // return true = filter out (skip), false = keep
    }
}
```

---

## 37.5 Dead Letter Queue

```java
import org.springframework.kafka.core.*;
import org.springframework.kafka.listener.*;

@org.springframework.context.annotation.Configuration
public class KafkaErrorConfig {
    
    @org.springframework.context.annotation.Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        
        // Send failed records to DLQ after 3 retries
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> {
                System.err.printf("Sending to DLQ: topic=%s key=%s%n",
                    record.topic(), record.key());
                return new org.apache.kafka.common.TopicPartition(
                    record.topic() + ".DLT",  // Dead Letter Topic
                    record.partition()
                );
            }
        );
        
        // Retry with backoff
        org.springframework.util.backoff.ExponentialBackOff backoff = 
            new org.springframework.util.backoff.ExponentialBackOff();
        backoff.setInitialInterval(1000);
        backoff.setMaxInterval(10000);
        backoff.setMaxElapsedTime(60000);  // give up after 1 min
        
        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backoff);
        
        // Don't retry these exceptions
        handler.addNotRetryableExceptions(
            IllegalArgumentException.class,
            ClassCastException.class
        );
        
        return handler;
    }
}

// DLQ Consumer - manual review/reprocessing
@org.springframework.stereotype.Component
class DLQConsumer {
    
    @KafkaListener(topics = "orders.DLT", groupId = "dlq-processor")
    public void handleDeadLetter(
            org.apache.kafka.clients.consumer.ConsumerRecord<String, ?> record,
            @org.springframework.messaging.handler.annotation.Header(
                org.springframework.kafka.support.KafkaHeaders.EXCEPTION_MESSAGE
            ) String exceptionMessage) {
        
        System.err.printf("DLQ record: key=%s error=%s%n",
            record.key(), exceptionMessage);
        
        // Alert, store in DB, manual review, etc.
        storeForManualReview(record.key(), record.value(), exceptionMessage);
    }
    
    private void storeForManualReview(Object key, Object value, String error) {
        // Persist to DB for manual review
    }
}
```

---

## 37.6 Kafka Streams

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.springframework.context.annotation.*;

@Configuration
public class KafkaStreamsConfig {
    
    @Bean
    public KStream<String, OrderEvent> orderStream(StreamsBuilder builder) {
        
        KStream<String, OrderEvent> orders = builder.stream("orders");
        
        // Real-time analytics: count orders by status
        KTable<String, Long> orderCountByStatus = orders
            .groupBy((key, event) -> event.status())
            .count(Materialized.as("order-counts-store"));
        
        orderCountByStatus.toStream()
            .foreach((status, count) ->
                System.out.printf("Status %s: %d orders%n", status, count));
        
        // Windowed aggregation: revenue per minute
        KTable<Windowed<String>, Double> revenuePerMinute = orders
            .filter((key, event) -> "CONFIRMED".equals(event.status()))
            .groupBy((key, event) -> "revenue",
                Grouped.with(
                    org.apache.kafka.common.serialization.Serdes.String(),
                    new org.springframework.kafka.support.serializer.JsonSerde<>(OrderEvent.class)
                ))
            .windowedBy(TimeWindows.ofSizeWithNoGrace(java.time.Duration.ofMinutes(1)))
            .aggregate(
                () -> 0.0,
                (key, event, aggregate) -> aggregate + event.total(),
                Materialized.with(
                    org.apache.kafka.common.serialization.Serdes.String(),
                    org.apache.kafka.common.serialization.Serdes.Double()
                )
            );
        
        revenuePerMinute.toStream()
            .map((windowedKey, revenue) -> 
                KeyValue.pair(windowedKey.key(), revenue))
            .to("revenue-per-minute");
        
        // Join: enrich orders with user info
        KTable<String, UserEvent> users = builder.table("users");
        
        KStream<String, String> enriched = orders.join(
            users,
            (order, user) -> String.format(
                "Order %s by %s: %.2f", order.orderId(), user.name(), order.total()
            )
        );
        
        enriched.to("enriched-orders");
        
        return orders;
    }
}
```

---

## 37.7 Docker Compose for Kafka

```yaml
# docker-compose-kafka.yml
version: '3.8'

services:
  
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9101
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka:29092,CONTROLLER://kafka:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
    healthcheck:
      test: kafka-topics --bootstrap-server localhost:9092 --list
      interval: 30s
      timeout: 10s
      retries: 5
  
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    depends_on:
      kafka:
        condition: service_healthy
```

```bash
# Useful Kafka CLI commands:
# Create topic
kafka-topics --create --topic orders --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092

# List topics
kafka-topics --list --bootstrap-server localhost:9092

# Describe topic
kafka-topics --describe --topic orders --bootstrap-server localhost:9092

# Produce messages
kafka-console-producer --topic orders --bootstrap-server localhost:9092

# Consume messages (from beginning)
kafka-console-consumer --topic orders --from-beginning --bootstrap-server localhost:9092

# Consumer group info
kafka-consumer-groups --list --bootstrap-server localhost:9092
kafka-consumer-groups --describe --group my-app-group --bootstrap-server localhost:9092
```

---

## สรุป Part 37

| Concept | คำอธิบาย |
|---------|---------|
| Topic | หมวดหมู่ของ messages |
| Partition | หน่วยของ parallelism |
| Consumer Group | share load among consumers |
| `@KafkaListener` | Spring annotation สำหรับ consume |
| `KafkaTemplate` | Spring สำหรับ produce |
| Dead Letter Queue | รับ messages ที่ fail |
| Kafka Streams | real-time stream processing |

**When to use Kafka:**
- High-throughput event streaming (millions/sec)
- Event replay / audit log
- Multiple consumers for same events
- Decoupled microservices

➡️ [Part 38: Redis & Caching](./Part-38-Redis.md)
