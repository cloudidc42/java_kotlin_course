# Part 83: Monitoring & Observability (Prometheus, Grafana, Jaeger)
## ขั้นตอนที่ 5711-5780: Metrics, Tracing, Logging, Alerting

---

## 83.1 Three Pillars of Observability

```
Observability = ความสามารถในการเข้าใจ state ของระบบจาก output

Three Pillars:
  1. Metrics   = aggregate numbers over time (CPU, latency, error rate)
  2. Traces    = request journey across services (distributed tracing)
  3. Logs      = text events with context (what happened + when + context)

Tools:
  Metrics: Prometheus (collect) + Grafana (visualize)
  Traces:  Jaeger / Zipkin / Tempo
  Logs:    ELK Stack / Loki + Grafana
  
  OpenTelemetry: standard protocol for all three
  
The Golden Signals (Google SRE):
  1. Latency     = time to serve a request (P50, P95, P99)
  2. Traffic     = requests per second (RPS)
  3. Errors      = error rate (4xx, 5xx)
  4. Saturation  = how "full" the service is (CPU, memory, queue depth)

RED Method:
  Rate    = requests/second
  Errors  = errors/second
  Duration = latency distribution
```

---

## 83.2 Spring Boot Actuator + Micrometer

```kotlin
// build.gradle.kts
// implementation("org.springframework.boot:spring-boot-starter-actuator")
// implementation("io.micrometer:micrometer-registry-prometheus")
// implementation("io.micrometer:micrometer-tracing-bridge-otel")
// implementation("io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter")

// application.yaml
/*
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true  # /health/liveness + /health/readiness
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true  # histogram for percentiles
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
*/
```

```kotlin
// Custom metrics
import io.micrometer.core.instrument.*

@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val meterRegistry: MeterRegistry
) {
    // Counter: monotonically increasing
    private val ordersCreated = Counter.builder("orders.created")
        .description("Total orders created")
        .register(meterRegistry)
    
    // Gauge: current value
    private val pendingOrders = Gauge.builder("orders.pending") {
        orderRepository.countByStatus(OrderStatus.SUBMITTED).toDouble()
    }.register(meterRegistry)
    
    // Timer: latency
    private val orderProcessingTimer = Timer.builder("order.processing.duration")
        .description("Time to process an order")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(meterRegistry)
    
    // Distribution summary: values
    private val orderValueSummary = DistributionSummary.builder("order.value.cents")
        .description("Distribution of order values")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(meterRegistry)
    
    fun createOrder(customerId: String): Order {
        val order = Order(...)
        val saved = orderRepository.save(order)
        
        ordersCreated.increment()
        orderValueSummary.record(order.total.cents.toDouble())
        
        return saved
    }
    
    fun processOrder(orderId: String): Order =
        orderProcessingTimer.recordCallable {
            val order = orderRepository.findById(orderId)!!
            // ... processing ...
            orderRepository.save(order.confirm())
        }!!
    
    // Custom health indicator
    @Component
    class OrderQueueHealthIndicator(
        private val orderRepository: OrderRepository
    ) : HealthIndicator {
        override fun health(): Health {
            val pending = orderRepository.countByStatus(OrderStatus.SUBMITTED)
            return if (pending < 1000) {
                Health.up().withDetail("pendingOrders", pending).build()
            } else {
                Health.down()
                    .withDetail("pendingOrders", pending)
                    .withDetail("reason", "Queue overloaded")
                    .build()
            }
        }
    }
}
```

---

## 83.3 Distributed Tracing with OpenTelemetry

```kotlin
// application.yaml
/*
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, 0.1 (10%) in prod

otel:
  exporter:
    otlp:
      endpoint: http://jaeger:4318
  service:
    name: order-service
*/

// Auto-instrumentation: Spring Boot starter instruments all HTTP requests,
// DB queries, Redis calls, Kafka producers/consumers automatically

// Manual span for important operations
import io.opentelemetry.api.trace.*
import io.opentelemetry.api.GlobalOpenTelemetry

@Service
class PaymentService(
    private val stripeClient: StripeClient
) {
    private val tracer = GlobalOpenTelemetry.getTracer("payment-service")
    
    fun processPayment(orderId: String, amount: Long): PaymentResult {
        val span = tracer.spanBuilder("process-payment")
            .setAttribute("order.id", orderId)
            .setAttribute("payment.amount", amount)
            .startSpan()
        
        return span.makeCurrent().use {
            try {
                val result = stripeClient.charge(orderId, amount)
                span.setAttribute("payment.transaction_id", result.transactionId)
                span.setStatus(StatusCode.OK)
                PaymentResult.Success(result.paymentId, result.transactionId)
            } catch (e: Exception) {
                span.recordException(e)
                span.setStatus(StatusCode.ERROR, e.message ?: "Payment failed")
                PaymentResult.Failure(e.message ?: "Unknown error")
            } finally {
                span.end()
            }
        }
    }
}

// Correlation ID: propagate trace across services
@Component
class TraceIdFilter : OncePerRequestFilter() {
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        // Add trace ID to response headers for debugging
        val traceId = Span.current().spanContext.traceId
        response.addHeader("X-Trace-Id", traceId)
        filterChain.doFilter(request, response)
    }
}
```

---

## 83.4 Structured Logging

```kotlin
// application.yaml
/*
logging:
  structured:
    format:
      console: ecs    # Elastic Common Schema JSON format
  level:
    root: INFO
    com.example: DEBUG

# Or use Logback JSON encoder:
# logback-spring.xml
*/

// Use SLF4J with MDC for context
import org.slf4j.LoggerFactory
import org.slf4j.MDC

private val log = LoggerFactory.getLogger(OrderService::class.java)

// Add context to all logs in this scope
fun processOrder(orderId: String, userId: String) {
    MDC.put("orderId", orderId)
    MDC.put("userId", userId)
    MDC.put("operation", "process-order")
    
    try {
        log.info("Starting order processing")  // Includes orderId, userId in JSON
        
        // ... processing ...
        
        log.info("Order processed successfully", 
            kv("totalAmount", order.total.cents),  // structured log
            kv("itemCount", order.items.size)
        )
    } catch (e: Exception) {
        log.error("Order processing failed", e)  // exception included
        throw e
    } finally {
        MDC.clear()  // IMPORTANT: always clear!
    }
}

// Logbook for HTTP request/response logging
@Configuration
class LogbookConfig {
    @Bean
    fun logbook(): Logbook = Logbook.builder()
        .sink(DefaultSink(
            HttpLogFormatter(),
            DefaultHttpLogWriter()
        ))
        .condition(Conditions.exclude(
            Conditions.requestTo("/actuator/**"),
            Conditions.contentTypeToString("**/image**")
        ))
        .requestFilter(
            RequestFilters.replaceBody(BodyReplacers.defaultValue("FILTERED"))
                .onBody(ContentType.of("application/octet-stream"))
        )
        .responseFilter(
            ResponseFilters.replaceBody(BodyReplacers.defaultValue("FILTERED"))
                .onBody(ContentType.of("text/html"))
        )
        .build()
}
```

---

## 83.5 Prometheus Query Language (PromQL)

```promql
# ====== Basic Metrics ======

# Request rate (per second, 5-minute window)
rate(http_server_requests_seconds_count[5m])

# Error rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
  /
rate(http_server_requests_seconds_count[5m])

# P99 latency
histogram_quantile(0.99, 
  rate(http_server_requests_seconds_bucket[5m])
)

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}

# ====== Aggregations ======

# Total requests by endpoint and status
sum by(uri, status) (
  rate(http_server_requests_seconds_count[5m])
)

# Average latency per service
avg by(application) (
  rate(http_server_requests_seconds_sum[5m])
  /
  rate(http_server_requests_seconds_count[5m])
)

# Orders per minute
increase(orders_created_total[1m])

# ====== Alerts (in Prometheus rules file) ======
# alerting_rules.yaml
groups:
  - name: shop-api
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{status=~"5.."}[5m])
          / 
          rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket[5m])
          ) > 2.0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "P99 latency above 2s"
      
      - alert: PodDown
        expr: up{job="shop-api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Pod is down"
```

---

## 83.6 Grafana Dashboard

```json
// Grafana Dashboard JSON (simplified)
{
  "title": "Shop API Dashboard",
  "panels": [
    {
      "title": "Request Rate (RPS)",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count[5m]))",
          "legendFormat": "Total RPS"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count[5m]))",
          "legendFormat": "Error Rate"
        }
      ]
    },
    {
      "title": "P99 Latency",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))",
          "legendFormat": "P99"
        }
      ]
    },
    {
      "title": "JVM Heap Usage",
      "type": "gauge",
      "targets": [
        {
          "expr": "sum(jvm_memory_used_bytes{area=\"heap\"}) / sum(jvm_memory_max_bytes{area=\"heap\"})"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "max": 1,
          "thresholds": {
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 0.7, "color": "yellow"},
              {"value": 0.9, "color": "red"}
            ]
          }
        }
      }
    }
  ]
}
```

---

## 83.7 Docker Compose: Local Observability Stack

```yaml
# docker-compose.observability.yml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./monitoring/alerting_rules.yaml:/etc/prometheus/alerting_rules.yaml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "4318:4318"    # OTLP HTTP receiver

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./monitoring/promtail.yml:/etc/promtail/config.yml

volumes:
  grafana-data:
```

---

## สรุป Part 83

```
Observability Summary:

Three Pillars:
  Metrics  → Prometheus → Grafana
  Traces   → OpenTelemetry → Jaeger
  Logs     → Structured JSON → Loki → Grafana

Spring Boot Auto-Instrumentation:
  Actuator:    /actuator/health, /actuator/prometheus
  Micrometer:  auto-instruments HTTP, DB, cache, messaging
  OTel:        auto-traces HTTP, JDBC, Redis, Kafka, Mongo

Key Metrics to Watch:
  Error Rate    < 1%
  P99 Latency   < 1000ms
  CPU           < 70%
  JVM Heap      < 80%
  Active Threads < limit

Alerting Best Practices:
  Alert on symptoms (high latency, errors)
  Not causes (CPU usage alone)
  Avoid alert fatigue: only actionable alerts
  PagerDuty/OpsGenie for on-call rotation

Tracing:
  Each request = one trace with unique trace ID
  Trace has spans (one per service/component)
  W3C TraceContext headers propagated between services
  Sample 100% in dev, 1-10% in prod

Structured Logging:
  Always JSON format in production
  Include: timestamp, level, message, service, traceId, userId
  MDC for request-scoped context
  Never log PII (passwords, tokens, full credit card)
```

➡️ [Part 84: API Design & Documentation (OpenAPI)](./Part-84-APIDesign.md)
