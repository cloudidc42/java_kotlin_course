# Part 42: Observability & Monitoring
## ขั้นตอนที่ 2841-2910: Metrics, Tracing, Logging

---

## 42.1 The Three Pillars

```
Observability = Metrics + Tracing + Logging

Metrics:  "How is the system performing?"
  - Request rate, error rate, latency (RED metrics)
  - CPU, memory, GC, DB pool (USE metrics)
  - Business metrics: orders/sec, revenue
  
Tracing: "What happened for this specific request?"
  - Follow a request across services
  - Find which service/step is slow
  - Correlation ID links logs across services
  
Logging: "What events occurred?"
  - Structured logs (JSON)
  - Searchable, aggregatable
  - Context: userId, orderId, requestId

Stack:
  Prometheus → collect & store metrics
  Grafana    → dashboards & alerts
  Zipkin     → distributed tracing
  ELK Stack  → log aggregation (Elasticsearch/Logstash/Kibana)
  OpenTelemetry → vendor-neutral instrumentation
```

---

## 42.2 Spring Boot Actuator + Micrometer

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, env, loggers, threaddump, heapdump
  endpoint:
    health:
      show-details: always
      show-components: always
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms,1s,2s

  tracing:
    sampling:
      probability: 1.0  # 100% sampling (reduce in production)
    propagation:
      type: b3

spring:
  application:
    name: order-service
  
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

---

## 42.3 Custom Metrics

```java
import io.micrometer.core.instrument.*;
import org.springframework.stereotype.*;

@Component
public class BusinessMetrics {
    
    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;
    private final DistributionSummary orderValueSummary;
    private final AtomicLong activeOrdersGauge;
    
    public BusinessMetrics(MeterRegistry registry) {
        
        // Counter: monotonically increasing
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Total orders created")
            .tag("service", "order-service")
            .register(registry);
        
        this.orderFailedCounter = Counter.builder("orders.failed")
            .description("Total orders that failed")
            .register(registry);
        
        // Timer: latency + call count
        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Time to process an order")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(registry);
        
        // Distribution summary: values (like order amounts)
        this.orderValueSummary = DistributionSummary.builder("orders.value")
            .description("Distribution of order values")
            .baseUnit("THB")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)
            .register(registry);
        
        // Gauge: current value (active connections, queue depth, etc.)
        this.activeOrdersGauge = registry.gauge(
            "orders.active",
            new AtomicLong(0)
        );
    }
    
    public void recordOrderCreated(double value) {
        orderCreatedCounter.increment();
        orderValueSummary.record(value);
        activeOrdersGauge.incrementAndGet();
    }
    
    public void recordOrderCompleted() {
        activeOrdersGauge.decrementAndGet();
    }
    
    public void recordOrderFailed() {
        orderFailedCounter.increment();
        activeOrdersGauge.decrementAndGet();
    }
    
    public <T> T timeOrderProcessing(java.util.concurrent.Callable<T> action) throws Exception {
        return orderProcessingTimer.recordCallable(action);
    }
    
    // Tag-based counter (multi-dimensional)
    private final MeterRegistry registry;
    
    public void recordPaymentByMethod(String method, boolean success) {
        Counter.builder("payments.processed")
            .tag("method", method)
            .tag("status", success ? "success" : "failure")
            .register(registry)
            .increment();
    }
}

// AOP-based timing
@org.aspectj.lang.annotation.Aspect
@Component
class TimedAspect {
    
    private final MeterRegistry registry;
    
    TimedAspect(MeterRegistry registry) { this.registry = registry; }
    
    @org.aspectj.lang.annotation.Around("@annotation(io.micrometer.core.annotation.Timed)")
    public Object timeMethod(org.aspectj.lang.ProceedingJoinPoint pjp) throws Throwable {
        String methodName = pjp.getSignature().getName();
        Timer timer = Timer.builder("method.execution")
            .tag("class", pjp.getTarget().getClass().getSimpleName())
            .tag("method", methodName)
            .register(registry);
        
        return timer.recordCallable(pjp::proceed);
    }
}
```

---

## 42.4 Distributed Tracing

```java
import io.micrometer.tracing.*;
import org.springframework.stereotype.*;

@Service
public class OrderService {
    
    private final Tracer tracer;
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    public OrderService(Tracer tracer, OrderRepository orderRepository,
                        PaymentService paymentService) {
        this.tracer = tracer;
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
    
    public Order createOrder(CreateOrderRequest request) {
        // Current span is automatically propagated via MDC
        Span span = tracer.currentSpan();
        if (span != null) {
            span.tag("order.user_id", String.valueOf(request.userId()));
            span.tag("order.items_count", String.valueOf(request.items().size()));
        }
        
        // Create child span for sub-operations
        Span validationSpan = tracer.nextSpan().name("validate-order");
        try (Tracer.SpanInScope scope = tracer.withSpan(validationSpan.start())) {
            validateOrder(request);
            validationSpan.tag("validation", "passed");
        } catch (Exception e) {
            validationSpan.tag("error", e.getMessage());
            throw e;
        } finally {
            validationSpan.end();
        }
        
        Order order = orderRepository.save(new Order(request));
        
        // Trace ID is propagated automatically in HTTP headers
        paymentService.processPayment(order.getId(), order.getTotal());
        
        return order;
    }
    
    private void validateOrder(CreateOrderRequest req) {
        // validation logic...
    }
}

// Trace ID in logs via MDC (automatic with Micrometer Tracing)
// Log output:
// {"traceId":"abc123","spanId":"def456","level":"INFO","message":"Order created: 42"}
```

---

## 42.5 Structured Logging

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    
    <springProfile name="local">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss} [%thread] [%X{traceId}] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
    
    <springProfile name="production">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <providers>
                    <timestamp/>
                    <logLevel/>
                    <loggerName/>
                    <message/>
                    <mdc/>
                    <arguments/>
                    <stackTrace/>
                    <pattern>
                        <pattern>
                            {
                                "service": "order-service",
                                "environment": "${ENVIRONMENT:-local}"
                            }
                        </pattern>
                    </pattern>
                </providers>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>
    
</configuration>
```

```java
import org.slf4j.*;
import java.util.Map;

@Service
public class LoggingExample {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingExample.class);
    
    public void processOrder(Long orderId, Long userId) {
        
        // Add context to MDC (appears in all logs for this thread)
        try (var mdc = org.slf4j.MDC.putCloseable("orderId", String.valueOf(orderId))) {
            try (var mdc2 = org.slf4j.MDC.putCloseable("userId", String.valueOf(userId))) {
                
                log.info("Processing order");
                
                // Structured args (logstash encoder)
                log.info("Order detail", 
                    org.slf4j.helpers.MessageFormatter.arrayFormat(
                        "orderId={} userId={}", 
                        new Object[]{orderId, userId}
                    ).getMessage());
                
                // With markers
                log.info("Business event: order.created");
                log.warn("Low stock warning: productId={}", 42);
                log.error("Payment failed", new RuntimeException("Timeout"));
            }
        }
    }
    
    // Log levels usage:
    // TRACE = detailed flow (disabled in prod)
    // DEBUG = debugging info (disabled in prod)
    // INFO  = business events (enabled in prod)
    // WARN  = unexpected but recoverable
    // ERROR = failures requiring attention
}
```

---

## 42.6 Health Indicators

```java
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.*;

@Component("database")
public class DatabaseHealthIndicator implements HealthIndicator {
    
    private final javax.sql.DataSource dataSource;
    
    public DatabaseHealthIndicator(javax.sql.DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public Health health() {
        try (var conn = dataSource.getConnection();
             var stmt = conn.createStatement()) {
            
            long start = System.currentTimeMillis();
            stmt.execute("SELECT 1");
            long latency = System.currentTimeMillis() - start;
            
            return Health.up()
                .withDetail("latencyMs", latency)
                .withDetail("url", conn.getMetaData().getURL())
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

@Component("externalApi")
public class ExternalApiHealthIndicator implements HealthIndicator {
    
    private final org.springframework.web.client.RestTemplate restTemplate;
    
    public ExternalApiHealthIndicator(org.springframework.web.client.RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    @Override
    public Health health() {
        try {
            var response = restTemplate.getForEntity("https://api.example.com/health", String.class);
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("externalApi", "available")
                    .build();
            } else {
                return Health.degraded()
                    .withDetail("statusCode", response.getStatusCode())
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## 42.7 Grafana Dashboard Config

```yaml
# docker-compose.yml (observability stack)
version: '3.8'

services:
  
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
  
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
  
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
  
  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yml:/etc/promtail/config.yml

volumes:
  grafana_data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-apps'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - 'order-service:8080'
          - 'user-service:8081'
          - 'payment-service:8082'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alerts.yml'
```

```yaml
# alerts.yml
groups:
  - name: application_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.instance }}"
          description: "P99 latency is {{ $value }}s"
      
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
```

---

## สรุป Part 42

```
Observability Stack:

Application:
  Micrometer → metrics to Prometheus
  Brave/OpenTelemetry → traces to Zipkin
  Logback/Logstash → structured JSON logs to Loki

Infrastructure:
  Prometheus → time-series metrics DB
  Grafana → dashboards + alerting
  Zipkin → trace visualization
  Loki/ELK → log aggregation

Key Metrics to Monitor:
  RED:  Rate, Error rate, Duration (for services)
  USE:  Utilization, Saturation, Errors (for resources)

Custom Metrics:
  Counter    = events (orders created, errors)
  Timer      = latency (request duration)
  Gauge      = current state (queue depth, active connections)
  Summary    = value distributions (order amounts)
```

➡️ [Part 43: gRPC with Java/Kotlin](./Part-43-gRPC.md)
