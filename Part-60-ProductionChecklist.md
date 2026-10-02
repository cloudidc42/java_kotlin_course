# Part 60: Production Readiness Checklist
## ขั้นตอนที่ 4101-4170: Pre-launch Audit, Monitoring, SLO/SLA

---

## 60.1 Production Readiness Framework

```
Production Readiness = Can we trust this in prod?

Categories to assess:
  1. Reliability    = it works when needed
  2. Scalability    = it handles the load
  3. Operability    = we can operate it
  4. Security       = it's not a liability
  5. Performance    = it's fast enough
  6. Maintainability = we can change it safely

Use this as a pre-launch checklist for every service
```

---

## 60.2 Reliability Checklist

```yaml
# Spring Boot health checks
management:
  endpoints:
    web:
      exposure:
        include: health,info,readiness,liveness,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true  # /actuator/health/readiness + /actuator/health/liveness
  health:
    db:
      enabled: true
    redis:
      enabled: true
    kafka:
      enabled: true
```

```java
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.*;

// Custom health indicator
@Component
class ExternalApiHealthIndicator implements HealthIndicator {
    
    private final ExternalApiClient client;
    
    ExternalApiHealthIndicator(ExternalApiClient client) { this.client = client; }
    
    @Override
    public Health health() {
        try {
            client.ping();
            return Health.up()
                .withDetail("api", "reachable")
                .withDetail("latency", "< 100ms")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("api", "unreachable")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Graceful shutdown
// application.yaml:
// server.shutdown: graceful
// spring.lifecycle.timeout-per-shutdown-phase: 30s
// Allows in-flight requests to complete before shutdown

// Circuit breakers configured on all external calls
// Retry with exponential backoff on transient failures
// Fallback responses when dependencies are down
```

---

## 60.3 Scalability Checklist

```
Horizontal Scaling:
  ✓ Stateless: no server-side session (JWT or session in Redis)
  ✓ Connection pooling: HikariCP correctly sized
  ✓ Distributed cache: Redis, not local
  ✓ External state: DB + Redis (not in-memory)
  
Load Testing Results:
  ✓ Response time P50 < 100ms, P99 < 500ms under expected load
  ✓ Error rate < 0.1% under expected load
  ✓ Tested 2x expected load (capacity headroom)
  ✓ Verified autoscaling works correctly

Database:
  ✓ Indexes on all query patterns
  ✓ Connection pool sized correctly (HikariCP formula: cores*2+1)
  ✓ Query planner analyzed (EXPLAIN ANALYZE)
  ✓ No N+1 queries (verified with query counter in test)

Capacity Planning:
  Current load:  X req/s
  Peak load:     Y req/s (holiday, launch)
  Headroom:      3x peak provisioned
  Scale trigger: 70% CPU / 80% memory
```

---

## 60.4 Observability Checklist

```java
import io.micrometer.core.instrument.*;
import org.slf4j.*;

// Structured logging (every log line should be parseable)
@Service
class OrderService {
    
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    private final MeterRegistry registry;
    
    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;
    
    OrderService(MeterRegistry registry) {
        this.registry = registry;
        this.orderCreatedCounter = Counter.builder("orders.created")
            .tag("service", "order-service")
            .register(registry);
        this.orderFailedCounter = Counter.builder("orders.failed")
            .tag("service", "order-service")
            .register(registry);
        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    public Order createOrder(CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try (var mdc = MDC.putCloseable("userId", request.userId())) {
                log.info("Creating order userId={} itemCount={}", 
                    request.userId(), request.items().size());
                
                var order = processOrder(request);
                
                orderCreatedCounter.increment();
                log.info("Order created orderId={} total={}", order.getId(), order.getTotal());
                
                return order;
            } catch (Exception e) {
                orderFailedCounter.increment();
                log.error("Order creation failed userId={} error={}", 
                    request.userId(), e.getMessage(), e);
                throw e;
            }
        });
    }
    
    private Order processOrder(CreateOrderRequest request) { return null; }
}

// Required metrics for every service:
//   http_server_requests_seconds (auto from Spring Actuator)
//   jvm_memory_used_bytes, jvm_gc_pause_seconds
//   db_query_duration + connection pool metrics
//   Business metrics: orders.created, orders.failed, revenue.total
```

---

## 60.5 SLO/SLA Definition

```
Service Level Objectives (SLOs):

For Order Service:
  Availability SLO:   99.9% uptime (= 8.7 hours downtime/year)
  Latency SLO P50:    < 100ms
  Latency SLO P99:    < 500ms
  Error Rate SLO:     < 0.1%

SLO Alerting (Prometheus rules):
  Alert when:
    Error rate > 1% for 5 minutes
    P99 latency > 1 second for 5 minutes
    Availability < 99% in rolling 1 hour

Error Budget:
  99.9% SLO = 0.1% error budget
  Monthly: 0.1% * 43200 minutes = 43.2 minutes of allowed downtime
  
  If error budget is consumed:
    1. Stop feature work
    2. Focus on reliability fixes
    3. Reduce deployment frequency

Runbook (required for every alert):
  What: Alert fired for high error rate
  Why: Check DB connections, dependency health
  How: Check /actuator/health, Grafana dashboard
  Fix: Restart pods, check upstream services
```

```yaml
# Prometheus alert rules
groups:
  - name: order-service
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{
            job="order-service",
            status=~"5.."
          }[5m])
          /
          rate(http_server_requests_seconds_count{
            job="order-service"
          }[5m])
          > 0.01
        for: 5m
        labels:
          severity: critical
          service: order-service
        annotations:
          summary: "High error rate in order-service"
          description: "Error rate is {{ $value | humanizePercentage }}"
          runbook: "https://wiki.example.com/runbooks/order-service"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket{
              job="order-service"
            }[5m])
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency in order-service"
          description: "P99 latency is {{ $value }}s"
      
      - alert: ServiceDown
        expr: up{job="order-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Order service is down"
```

---

## 60.6 Security Checklist

```
Pre-deployment Security Audit:

Authentication & Authorization:
  ✓ All endpoints require auth except /health, /metrics, /auth/**
  ✓ JWT validated on every request (signature + expiry)
  ✓ Role-based access: admin endpoints protected
  ✓ No sensitive data in JWT payload (only IDs and roles)

Input Validation:
  ✓ All request bodies validated (@Valid + @NotBlank)
  ✓ All SQL uses parameterized queries (no string concat)
  ✓ File uploads: type validation, size limits
  ✓ No SQL/NoSQL injection possible

Secrets Management:
  ✓ No secrets in code or config files (use env vars / Vault)
  ✓ No secrets in logs
  ✓ Secrets rotated regularly

Network:
  ✓ HTTPS only (HTTP → redirect to HTTPS)
  ✓ HSTS header with long max-age
  ✓ CORS configured explicitly (not *)
  ✓ CSP headers configured

Dependencies:
  ✓ OWASP dependency-check passes (no HIGH/CRITICAL CVEs)
  ✓ Base image up to date (monthly refresh)
  ✓ Dependency lock files committed

Data:
  ✓ PII encrypted at rest
  ✓ Passwords hashed with bcrypt strength 12
  ✓ DB connection over SSL
  ✓ Backup tested (restore drill)
```

---

## 60.7 Deployment Checklist

```
Pre-deployment:
  ✓ All tests pass (unit + integration + contract)
  ✓ Code reviewed and approved
  ✓ Security scan passed
  ✓ Load test passed
  ✓ Rollback plan documented

Deployment:
  ✓ Deploy to staging first
  ✓ Smoke test staging
  ✓ Deploy to production (rolling update, no downtime)
  ✓ Verify health checks pass
  ✓ Monitor error rates and latency for 30 minutes

Post-deployment:
  ✓ Verify SLOs not breached
  ✓ Check business metrics (orders/minute, revenue)
  ✓ Update documentation
  ✓ Notify stakeholders

Rollback Trigger:
  Error rate > 5% for 2 minutes → automatic rollback
  kubectl rollout undo deployment/order-service
```

---

## สรุป Part 60

```
Production Readiness = Reliability + Scalability + Operability + Security

Required for Every Service:
  Health checks: /actuator/health/readiness + /actuator/health/liveness
  Structured logs: JSON with traceId, userId, requestId
  Metrics: http requests, latency percentiles, business metrics
  Alerts: high error rate, high latency, service down
  Runbook: how to debug and fix each alert

SLO Targets (adjust per service criticality):
  Availability: 99.9% to 99.99%
  P99 Latency: < 500ms to < 1s
  Error Rate: < 0.1%

The 3 Questions Before Deploy:
  1. Can we detect when it's broken? (monitoring)
  2. Can we fix it when it's broken? (runbook + rollback)
  3. Can we prevent data loss? (backup + transactions)
```

➡️ [Part 61: Java Virtual Threads (Project Loom)](./Part-61-VirtualThreads.md)
