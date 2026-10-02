# Part 96: Cloud-Native Patterns & Kubernetes Production
## ขั้นตอนที่ 6621-6690: Health Probes, Graceful Shutdown, Cloud Events, GKE/EKS

---

## 96.1 Cloud-Native Application Principles

```
12-Factor App Principles (updated for cloud):

1. Codebase: one repo, many deploys
2. Dependencies: explicitly declare (Maven/Gradle)
3. Config: store in environment (not code)
4. Backing services: treat as attached resources
5. Build/Release/Run: strictly separated stages
6. Processes: stateless, share-nothing processes
7. Port binding: export services via port binding
8. Concurrency: scale out via process model
9. Disposability: fast startup, graceful shutdown
10. Dev/Prod parity: keep environments similar
11. Logs: treat as event streams (stdout/stderr)
12. Admin processes: run as one-off processes

Additional (15-Factor):
13. API first: design API before implementation
14. Telemetry: metrics, tracing, logging
15. Auth/Sec: security at every layer
```

---

## 96.2 Health Probes & Readiness

```kotlin
// Spring Boot Actuator health probes for Kubernetes

// application.yaml
/*
management:
  endpoint:
    health:
      probes:
        enabled: true     # enables /actuator/health/liveness and /actuator/health/readiness
      show-details: always
      group:
        liveness:
          include: livenessState, db
        readiness:
          include: readinessState, db, redis, kafka
*/

// ====== Custom Readiness Indicator ======
@Component
class KafkaConsumerReadinessIndicator(
    private val kafkaListenerEndpointRegistry: KafkaListenerEndpointRegistry
) : HealthIndicator {
    
    override fun health(): Health {
        val allRunning = kafkaListenerEndpointRegistry.allListenerContainers
            .all { it.isRunning }
        
        return if (allRunning) {
            Health.up()
                .withDetail("consumers", "all running")
                .build()
        } else {
            Health.down()
                .withDetail("consumers", "some not running")
                .build()
        }
    }
}

// ====== Liveness vs Readiness ======
// Liveness: is the app alive? (dead → restart)
//   - Infinite loop detected
//   - Deadlock
//   - OOM (that didn't crash JVM)
//
// Readiness: can the app serve traffic? (not ready → remove from LB)
//   - DB connection not ready
//   - Warmup not complete
//   - Downstream service degraded
//   - Kafka consumer lag too high

@Component
class WarmupReadinessIndicator : HealthIndicator {
    
    @Volatile private var warmedUp = false
    
    @EventListener(ApplicationReadyEvent::class)
    fun onReady() {
        // Simulate warmup (cache loading, connection pool filling, etc.)
        warmedUp = true
    }
    
    override fun health(): Health =
        if (warmedUp) Health.up().build()
        else Health.down().withDetail("reason", "warmup not complete").build()
}

// Programmatic health state management
@Component  
class TrafficController(private val applicationAvailability: ApplicationAvailability) {
    
    // Call this to prevent traffic (maintenance mode)
    fun refuseTraffic(publisher: ApplicationEventPublisher) {
        publisher.publishEvent(AvailabilityChangeEvent(this, ReadinessState.REFUSING_TRAFFIC))
    }
    
    // Resume accepting traffic
    fun acceptTraffic(publisher: ApplicationEventPublisher) {
        publisher.publishEvent(AvailabilityChangeEvent(this, ReadinessState.ACCEPTING_TRAFFIC))
    }
}
```

---

## 96.3 Graceful Shutdown

```kotlin
// Graceful shutdown: finish in-flight requests, close connections

// application.yaml
/*
server:
  shutdown: graceful          # wait for in-flight requests to complete
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # max 30s per phase
*/

// Shutdown hook for cleanup
@Component
class GracefulShutdownManager(
    private val kafkaConsumerFactory: ConcurrentKafkaListenerContainerFactory<*, *>
) {
    
    @PreDestroy
    fun onShutdown() {
        log.info("Starting graceful shutdown...")
        // 1. Stop accepting new requests (handled by server.shutdown=graceful)
        // 2. Wait for in-flight to complete (timeout-per-shutdown-phase)
        // 3. Close connections
    }
    
    @EventListener
    fun onContextClosed(event: ContextClosedEvent) {
        log.info("Context closing - stopping Kafka consumers")
        kafkaConsumerFactory.stop()  // stop Kafka consumers gracefully
    }
}

// SIGTERM handler
@SpringBootApplication
class Application {
    companion object {
        @JvmStatic
        fun main(args: Array<String>) {
            val context = SpringApplication.run(Application::class.java, *args)
            
            // Register JVM shutdown hook
            Runtime.getRuntime().addShutdownHook(Thread {
                log.info("JVM shutdown hook triggered")
                context.close()
            })
        }
    }
}

// Kubernetes: SIGTERM → app has terminationGracePeriodSeconds to clean up
// Then SIGKILL if not done
```

---

## 96.4 Kubernetes Production Configuration

```yaml
# Complete K8s deployment for Spring Boot app

# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
  namespace: production
  labels:
    app: shop-api
    version: "2.1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shop-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # always keep min replicas available
  template:
    metadata:
      labels:
        app: shop-api
        version: "2.1.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: shop-api
      terminationGracePeriodSeconds: 60  # must be > timeout-per-shutdown-phase
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      # Init container: wait for DB
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command: ['sh', '-c', 
            'until nc -z postgres 5432; do echo waiting for postgres; sleep 2; done']
      
      containers:
        - name: shop-api
          image: registry.example.com/shop-api:2.1.0
          imagePullPolicy: IfNotPresent
          
          ports:
            - containerPort: 8080
              name: http
          
          # Resource limits
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"     # OOMKilled if exceeded
              cpu: "1000m"
          
          # Health probes
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1
          
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30   # allow 150s for startup (slow cold start)
          
          # Environment variables from secrets
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: shop-api-secrets
                  key: db-password
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: shop-api-secrets
                  key: redis-password
          
          # Volume mounts
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
          
          # Container security
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
      
      volumes:
        - name: config
          configMap:
            name: shop-api-config

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shop-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shop-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # scale up if avg CPU > 60%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60    # add max 4 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60   # remove max 1 pod per minute

---
# PodDisruptionBudget: guarantee availability during node maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shop-api-pdb
  namespace: production
spec:
  minAvailable: 2   # always keep at least 2 pods
  selector:
    matchLabels:
      app: shop-api
```

---

## 96.5 ConfigMap & Secret Management

```yaml
# ConfigMap: non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: shop-api-config
  namespace: production
data:
  application.yaml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/shopdb
        hikari:
          maximum-pool-size: 10
      redis:
        host: redis
        port: 6379
    management:
      endpoints:
        web:
          exposure:
            include: health,prometheus,info
      endpoint:
        health:
          probes:
            enabled: true

---
# Secret: sensitive data (base64 encoded, ideally from Vault/AWS SM)
apiVersion: v1
kind: Secret
metadata:
  name: shop-api-secrets
  namespace: production
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=  # base64: password123
  redis-password: cmVkaXNwYXNz    # base64: redispass
  jwt-secret: c2VjcmV0a2V5        # base64: secretkey
```

```kotlin
// RBAC: limit what the service account can do
// serviceaccount.yaml
/*
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shop-api
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shop-api-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shop-api-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: shop-api
roleRef:
  kind: Role
  name: shop-api-role
  apiGroup: rbac.authorization.k8s.io
*/
```

---

## 96.6 Cloud Events Standard

```kotlin
// CloudEvents: standard for event data across clouds
// Spec: https://cloudevents.io/

// CloudEvent structure:
/*
{
  "specversion": "1.0",
  "type": "com.example.shop.order.placed",
  "source": "https://shop.example.com/orders",
  "subject": "order/123",
  "id": "A234-1234-1234",
  "time": "2024-01-15T17:31:00Z",
  "datacontenttype": "application/json",
  "data": {
    "orderId": "123",
    "customerId": "456",
    "total": 25000
  }
}
*/

data class CloudEvent<T>(
    val specversion: String = "1.0",
    val id: String = UUID.randomUUID().toString(),
    val type: String,
    val source: String,
    val subject: String? = null,
    val time: String = Instant.now().toString(),
    val datacontenttype: String = "application/json",
    val data: T
)

@Service
class CloudEventPublisher(private val kafkaTemplate: KafkaTemplate<String, String>) {
    
    fun publish(event: Any, type: String, source: String) {
        val cloudEvent = CloudEvent(
            type = type,
            source = source,
            data = event
        )
        
        val headers = ProducerRecord<String, String>(
            deriveTopicFromType(type),
            cloudEvent.id,
            objectMapper.writeValueAsString(cloudEvent)
        ).also { record ->
            record.headers().apply {
                add("ce-specversion", "1.0".toByteArray())
                add("ce-type", type.toByteArray())
                add("ce-source", source.toByteArray())
                add("ce-id", cloudEvent.id.toByteArray())
                add("ce-time", cloudEvent.time.toByteArray())
            }
        }
        
        kafkaTemplate.send(headers)
    }
    
    private fun deriveTopicFromType(type: String): String {
        // com.example.shop.order.placed → shop-order-events
        val parts = type.split(".")
        return "${parts[2]}-${parts[3]}-events"
    }
}
```

---

## สรุป Part 96

```
Cloud-Native Patterns:

Health Probes:
  Liveness: is app alive? failure → restart pod
  Readiness: can serve traffic? failure → remove from LB
  Startup: allow slow startup without failing liveness
  
  Best practice:
    startupProbe: failureThreshold × periodSeconds > max startup time
    readinessProbe: check real dependencies (DB, cache)
    livenessProbe: check only internal health (avoid cascading restarts)

Graceful Shutdown:
  server.shutdown: graceful
  terminationGracePeriodSeconds > timeout-per-shutdown-phase
  PreDestroy: cleanup connections, stop consumers
  Order: stop accepting → drain requests → close connections

Kubernetes Production:
  Rolling update: maxUnavailable=0 (zero downtime)
  Resources: always set requests + limits
  HPA: scale on CPU/memory/custom metrics
  PDB: maintain minimum availability during disruptions
  Security: non-root, readOnlyRootFilesystem, drop capabilities

Secret Management:
  K8s Secrets: base64 (not secure alone!)
  External Secrets Operator: sync from Vault/AWS SM
  Sealed Secrets: encrypt in git
  Never: hardcode in Dockerfile or env files

CloudEvents:
  Standard event format across clouds
  Fields: specversion, id, type, source, subject, time, data
  Enables: event routing, filtering, replay
  Works with: Kafka, HTTP webhooks, event buses
  
Monitoring Checklist:
  ✓ Health probes configured
  ✓ Resources requests + limits set
  ✓ HPA enabled for stateless services
  ✓ PDB set for critical services
  ✓ Graceful shutdown configured
  ✓ Structured logs (JSON)
  ✓ Prometheus metrics exposed
```

➡️ [Part 97: System Design Interview Patterns](./Part-97-SystemDesign.md)
