# Part 28: Microservices Architecture
## ขั้นตอนที่ 1861-1930: Design ระดับ Enterprise

---

## 28.1 Microservices Overview

```
Monolith vs Microservices:

Monolith:
  ┌─────────────────────────────────┐
  │  UserModule  OrderModule  ...   │
  │  One codebase, one deploy       │
  └─────────────────────────────────┘

Microservices:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  User    │  │  Order   │  │ Payment  │
  │ Service  │  │ Service  │  │ Service  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       └──────────────┴─────────────┘
              Message Bus / API Gateway

ข้อดี:
  ✓ Scale แต่ละ service แยกกัน
  ✓ Deploy อิสระ
  ✓ Technology per service
  ✓ Fault isolation
  
ข้อเสีย:
  ✗ Network overhead
  ✗ Distributed tracing ยาก
  ✗ Data consistency (eventual)
  ✗ Operational complexity
```

---

## 28.2 API Gateway Pattern

```java
// API Gateway ด้วย Spring Cloud Gateway
// pom.xml:
// spring-cloud-starter-gateway
// spring-cloud-starter-netflix-eureka-client

// application.yml
/*
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=0
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
        
        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**
        
        - id: product-service
          uri: lb://PRODUCT-SERVICE
          predicates:
            - Path=/api/products/**

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
*/

import org.springframework.web.bind.annotation.*;
import org.springframework.http.*;

@RestController
@RequestMapping("/fallback")
public class FallbackController {
    
    @GetMapping("/users")
    public ResponseEntity<?> usersFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "error", "User service unavailable",
                "message", "Please try again later"
            ));
    }
    
    @GetMapping("/orders")
    public ResponseEntity<?> ordersFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of("error", "Order service unavailable"));
    }
}
```

---

## 28.3 Service Discovery (Eureka)

```java
// Eureka Server
// pom.xml: spring-cloud-starter-netflix-eureka-server

@org.springframework.boot.autoconfigure.SpringBootApplication
@org.springframework.cloud.netflix.eureka.server.EnableEurekaServer
class EurekaServerApplication {
    public static void main(String[] args) {
        org.springframework.boot.SpringApplication.run(EurekaServerApplication.class, args);
    }
}

/*
# application.yml for Eureka Server
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
*/

// Service Registration (each microservice)
/*
spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
*/
```

---

## 28.4 Service-to-Service Communication

```java
// ====== Feign Client (declarative HTTP client) ======
// pom.xml: spring-cloud-starter-openfeign

import org.springframework.cloud.openfeign.*;
import org.springframework.web.bind.annotation.*;

@FeignClient(name = "user-service", fallback = UserClientFallback.class)
interface UserServiceClient {
    
    @GetMapping("/api/users/{id}")
    UserDto getUserById(@PathVariable Long id);
    
    @GetMapping("/api/users")
    List<UserDto> getAllUsers();
    
    @PostMapping("/api/users")
    UserDto createUser(@RequestBody CreateUserRequest request);
}

// Fallback implementation
@Component
class UserClientFallback implements UserServiceClient {
    
    @Override
    public UserDto getUserById(Long id) {
        return new UserDto(id, "Unknown", "unknown@example.com");
    }
    
    @Override
    public List<UserDto> getAllUsers() {
        return List.of();
    }
    
    @Override
    public UserDto createUser(CreateUserRequest request) {
        throw new RuntimeException("User service unavailable");
    }
}

record UserDto(Long id, String name, String email) {}
record CreateUserRequest(String name, String email) {}

// Order service using UserServiceClient
@org.springframework.stereotype.Service
class OrderService {
    
    private final UserServiceClient userClient;
    private final OrderRepository orderRepo;
    
    OrderService(UserServiceClient userClient, OrderRepository orderRepo) {
        this.userClient = userClient;
        this.orderRepo = orderRepo;
    }
    
    Order createOrder(Long userId, List<OrderItem> items) {
        // Get user from user-service
        UserDto user = userClient.getUserById(userId);
        
        Order order = new Order();
        order.setUserId(userId);
        order.setUserEmail(user.email());
        order.setItems(items);
        order.setStatus("PENDING");
        order.setTotal(items.stream().mapToDouble(OrderItem::total).sum());
        
        return orderRepo.save(order);
    }
}
```

---

## 28.5 Event-Driven Architecture (RabbitMQ)

```java
// pom.xml: spring-boot-starter-amqp

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.amqp.rabbit.core.*;
import org.springframework.stereotype.*;

// ====== Configuration ======
@org.springframework.context.annotation.Configuration
class RabbitMQConfig {
    
    public static final String ORDER_QUEUE = "order.queue";
    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_ROUTING_KEY = "order.created";
    public static final String ORDER_DEAD_LETTER_QUEUE = "order.dlq";
    
    @org.springframework.context.annotation.Bean
    Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", ORDER_DEAD_LETTER_QUEUE)
            .withArgument("x-message-ttl", 86400000)  // 24h TTL
            .build();
    }
    
    @org.springframework.context.annotation.Bean
    Queue deadLetterQueue() {
        return QueueBuilder.durable(ORDER_DEAD_LETTER_QUEUE).build();
    }
    
    @org.springframework.context.annotation.Bean
    TopicExchange orderExchange() {
        return new TopicExchange(ORDER_EXCHANGE);
    }
    
    @org.springframework.context.annotation.Bean
    Binding binding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue)
            .to(orderExchange)
            .with(ORDER_ROUTING_KEY);
    }
}

// ====== Publisher (Order Service) ======
record OrderCreatedEvent(
    Long orderId,
    Long userId,
    String userEmail,
    double total,
    java.time.Instant createdAt
) {}

@Service
class OrderEventPublisher {
    
    private final RabbitTemplate rabbitTemplate;
    
    OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }
    
    void publishOrderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), order.getUserId(), order.getUserEmail(),
            order.getTotal(), java.time.Instant.now()
        );
        
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_ROUTING_KEY,
            event
        );
        
        System.out.println("Published: " + event);
    }
}

// ====== Consumer (Notification Service) ======
@Service
class NotificationService {
    
    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE)
    void handleOrderCreated(OrderCreatedEvent event) {
        System.out.printf("Sending email to %s for order #%d (%.2f)%n",
            event.userEmail(), event.orderId(), event.total());
        
        // Send email, push notification, etc.
        sendOrderConfirmationEmail(event.userEmail(), event.orderId(), event.total());
    }
    
    private void sendOrderConfirmationEmail(String email, Long orderId, double total) {
        System.out.printf("[EMAIL] To: %s | Order #%d confirmed | Total: %.2f%n",
            email, orderId, total);
    }
}
```

---

## 28.6 Saga Pattern (Distributed Transactions)

```java
// Choreography-based Saga
// Each service listens to events and reacts

// ====== Order Service ======
@Service
class OrderSagaService {
    
    private final OrderRepository orderRepo;
    private final OrderEventPublisher publisher;
    
    OrderSagaService(OrderRepository orderRepo, OrderEventPublisher publisher) {
        this.orderRepo = orderRepo;
        this.publisher = publisher;
    }
    
    // Step 1: Create order (PENDING state)
    Order startOrderSaga(Long userId, List<OrderItem> items) {
        Order order = new Order(userId, items, "PENDING");
        orderRepo.save(order);
        
        // Publish event to trigger inventory check
        publisher.publishOrderCreated(order);
        return order;
    }
    
    // Step 4: Payment confirmed → complete order
    @RabbitListener(queues = "payment.confirmed.queue")
    void handlePaymentConfirmed(PaymentConfirmedEvent event) {
        orderRepo.findById(event.orderId()).ifPresent(order -> {
            order.setStatus("CONFIRMED");
            orderRepo.save(order);
            System.out.println("Order " + event.orderId() + " confirmed!");
        });
    }
    
    // Compensation: Payment failed → cancel order
    @RabbitListener(queues = "payment.failed.queue")
    void handlePaymentFailed(PaymentFailedEvent event) {
        orderRepo.findById(event.orderId()).ifPresent(order -> {
            order.setStatus("CANCELLED");
            orderRepo.save(order);
            // Publish inventory.restore event
            System.out.println("Order " + event.orderId() + " cancelled due to payment failure");
        });
    }
}

// ====== Inventory Service ======
@Service
class InventorySagaService {
    
    // Step 2: Reserve inventory
    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE)
    void handleOrderCreated(OrderCreatedEvent event) {
        boolean reserved = checkAndReserveInventory(event.orderId());
        
        if (reserved) {
            // Trigger payment
            publishInventoryReserved(event);
        } else {
            // Compensation: cancel order
            publishInventoryFailed(event);
        }
    }
    
    private boolean checkAndReserveInventory(Long orderId) {
        // Check stock and reserve
        return Math.random() > 0.1;  // 90% success
    }
    
    private void publishInventoryReserved(OrderCreatedEvent event) {
        System.out.println("Inventory reserved for order " + event.orderId());
        // publish to payment service
    }
    
    private void publishInventoryFailed(OrderCreatedEvent event) {
        System.out.println("Inventory failed for order " + event.orderId());
        // publish order cancellation
    }
}
```

---

## 28.7 Distributed Tracing

```java
// Spring Boot 3 uses Micrometer Tracing (Brave/OTel)
// pom.xml:
// micrometer-tracing-bridge-brave
// zipkin-reporter-brave

/*
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # trace all requests (production: 0.1)
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

spring:
  application:
    name: order-service
*/

// Trace automatically propagates through:
// - HTTP headers (B3 or W3C Trace Context)
// - RabbitMQ message headers
// - Logs include trace ID

import io.micrometer.tracing.*;
import org.springframework.stereotype.*;

@Service
class TracedOrderService {
    
    private final Tracer tracer;
    
    TracedOrderService(Tracer tracer) { this.tracer = tracer; }
    
    void processOrder(Long orderId) {
        // Create custom span
        Span span = tracer.nextSpan().name("process-order").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
            span.tag("order.id", orderId.toString());
            
            // Business logic
            validateOrder(orderId);
            calculateTotal(orderId);
            
            span.event("order-processed");
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
    
    private void validateOrder(Long orderId) { /* ... */ }
    private void calculateTotal(Long orderId) { /* ... */ }
}
```

---

## 28.8 Config Server

```java
// Spring Cloud Config Server
// pom.xml: spring-cloud-config-server

@org.springframework.boot.autoconfigure.SpringBootApplication
@org.springframework.cloud.config.server.EnableConfigServer
class ConfigServerApplication {
    public static void main(String[] args) {
        org.springframework.boot.SpringApplication.run(ConfigServerApplication.class, args);
    }
}

/*
# Config server application.yml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/yourorg/config-repo
          clone-on-start: true
          default-label: main
        # OR local filesystem for dev:
        native:
          search-locations: file:./config

# Service client bootstrap.yml
spring:
  application:
    name: user-service
  config:
    import: "configserver:http://localhost:8888"

# Config file in repo: user-service.yml
server:
  port: 8081
spring:
  datasource:
    url: jdbc:postgresql://db:5432/users
*/
```

---

## 28.9 Health & Metrics

```java
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.*;

// Custom Health Indicator
@Component
class ExternalServiceHealthIndicator implements HealthIndicator {
    
    private final ExternalServiceClient client;
    
    ExternalServiceHealthIndicator(ExternalServiceClient client) {
        this.client = client;
    }
    
    @Override
    public Health health() {
        try {
            boolean available = client.ping();
            if (available) {
                return Health.up()
                    .withDetail("service", "External API")
                    .withDetail("status", "reachable")
                    .build();
            } else {
                return Health.down()
                    .withDetail("service", "External API")
                    .withDetail("status", "unreachable")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Micrometer custom metrics
import io.micrometer.core.instrument.*;

@Service
class OrderMetricsService {
    
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;
    private final MeterRegistry registry;
    
    OrderMetricsService(MeterRegistry registry) {
        this.registry = registry;
        this.orderCounter = Counter.builder("orders.created.total")
            .description("Total orders created")
            .tag("service", "order-service")
            .register(registry);
        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Order processing duration")
            .register(registry);
    }
    
    void recordOrderCreated(String status) {
        registry.counter("orders.created.total",
            "status", status, "service", "order-service"
        ).increment();
    }
    
    void recordProcessingTime(Runnable task) {
        orderProcessingTimer.record(task);
    }
    
    Gauge getActiveOrdersGauge(java.util.function.Supplier<Number> supplier) {
        return Gauge.builder("orders.active", supplier)
            .description("Active orders count")
            .register(registry);
    }
}
```

---

## 28.10 Docker Compose for Microservices

```yaml
# docker-compose.yml
version: '3.8'

services:
  
  # Service Discovery
  eureka-server:
    image: my-eureka:latest
    ports: ["8761:8761"]
    environment:
      SPRING_PROFILES_ACTIVE: docker
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
  
  # API Gateway
  api-gateway:
    image: my-gateway:latest
    ports: ["8080:8080"]
    depends_on:
      eureka-server:
        condition: service_healthy
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
  
  # User Service
  user-service:
    image: my-user-service:latest
    ports: ["8081:8081"]
    depends_on:
      - eureka-server
      - postgres
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/users
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: secret
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
  
  # Order Service
  order-service:
    image: my-order-service:latest
    ports: ["8082:8082"]
    depends_on:
      - eureka-server
      - rabbitmq
      - mongo
    environment:
      SPRING_DATA_MONGODB_URI: mongodb://mongo:27017/orders
      SPRING_RABBITMQ_HOST: rabbitmq
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
  
  # Infrastructure
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: users
      POSTGRES_PASSWORD: secret
    volumes: [postgres_data:/var/lib/postgresql/data]
  
  mongo:
    image: mongo:7
    volumes: [mongo_data:/data/db]
  
  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672", "15672:15672"]
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
  
  # Monitoring
  zipkin:
    image: openzipkin/zipkin
    ports: ["9411:9411"]
  
  prometheus:
    image: prom/prometheus
    ports: ["9090:9090"]
    volumes: [./prometheus.yml:/etc/prometheus/prometheus.yml]
  
  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes: [grafana_data:/var/lib/grafana]

volumes:
  postgres_data:
  mongo_data:
  grafana_data:
```

---

## สรุป Part 28

```
Microservices Patterns ที่ใช้บ่อย:

1. API Gateway       = single entry point, routing, auth
2. Service Discovery = Eureka, Consul
3. Circuit Breaker   = Resilience4J, fail fast
4. Event-Driven      = RabbitMQ, Kafka, eventual consistency
5. Saga Pattern      = distributed transactions without 2PC
6. Config Server     = centralized configuration
7. Distributed Trace = Zipkin, Jaeger, correlation IDs
8. Health Check      = /actuator/health endpoints

การเลือก Communication:
  Sync  = REST/gRPC (ต้องการ immediate response)
  Async = Message Queue (ทนต่อ failure ได้ดีกว่า)
```

➡️ [Part 29: Docker & Kubernetes](./Part-29-Docker-Kubernetes.md)
