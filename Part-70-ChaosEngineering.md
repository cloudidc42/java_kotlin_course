# Part 70: Chaos Engineering & Resilience Testing
## ขั้นตอนที่ 4801-4870: Failure Injection, Netflix Principles, Resilience4j Testing

---

## 70.1 Chaos Engineering คืออะไร

```
Chaos Engineering (Netflix Chaos Monkey):
  "ทดสอบระบบโดยการสร้างความล้มเหลวโดยตั้งใจ
   เพื่อหาจุดอ่อนก่อนที่จะเกิดขึ้นใน production จริงๆ"

Principles of Chaos:
  1. สร้าง Hypothesis: "ระบบยังทำงานได้ถ้า DB ล่ม 30 วินาที"
  2. Inject Failure: simulate DB failure
  3. Observe: ดู metrics, error rate, recovery time
  4. Learn: แก้ไขจุดอ่อนที่พบ

Types of Failures to Test:
  Network:   latency, packet loss, partition, bandwidth limit
  Resources: CPU spike, memory pressure, disk full
  Process:   pod kill, graceful/ungraceful shutdown
  Dependency: DB slow, external API timeout, 500 errors
  Time:      clock skew

Blast Radius:
  ✓ เริ่มจาก local dev หรือ staging
  ✓ ควบคุม scope (เฉพาะ 1 pod, 1% traffic)
  ✓ มี kill switch
  ✗ ไม่ทำใน production โดยไม่มีแผน
```

---

## 70.2 Testing Resilience4j Patterns

```java
import io.github.resilience4j.circuitbreaker.*;
import io.github.resilience4j.retry.*;
import io.github.resilience4j.timelimiter.*;
import io.github.resilience4j.testing.*;

// Unit testing Circuit Breaker behavior
@ExtendWith(MockitoExtension.class)
class CircuitBreakerResilienceTest {
    
    // Circuit breaker config
    CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .slidingWindowSize(10)
        .failureRateThreshold(50.0f)     // open at 50% failure
        .waitDurationInOpenState(java.time.Duration.ofSeconds(5))
        .permittedNumberOfCallsInHalfOpenState(3)
        .build();
    
    CircuitBreaker circuitBreaker;
    
    @BeforeEach
    void setUp() {
        circuitBreaker = CircuitBreaker.of("test", config);
    }
    
    @Test
    void circuitBreaker_opensAfterFailureThreshold() {
        // Simulate 6 failures out of 10 (60% > 50% threshold)
        for (int i = 0; i < 10; i++) {
            int idx = i;
            try {
                circuitBreaker.executeCallable(() -> {
                    if (idx < 6) throw new RuntimeException("Service unavailable");
                    return "OK";
                });
            } catch (Exception ignored) {}
        }
        
        // Circuit should be OPEN
        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.OPEN);
        
        // Further calls fail fast (without calling the service)
        assertThatThrownBy(() ->
            circuitBreaker.executeCallable(() -> "should not reach here")
        ).isInstanceOf(io.github.resilience4j.circuitbreaker.CallNotPermittedException.class);
    }
    
    @Test
    void circuitBreaker_transitionsToHalfOpenAfterWaitDuration() throws Exception {
        // Force circuit open
        forceOpen();
        
        // Wait for open duration (use fake clock in tests)
        circuitBreaker.transitionToHalfOpenState();
        
        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.HALF_OPEN);
        
        // Successful calls in half-open → close the circuit
        for (int i = 0; i < 3; i++) {
            circuitBreaker.executeCallable(() -> "OK");
        }
        
        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.CLOSED);
    }
    
    private void forceOpen() {
        for (int i = 0; i < 10; i++) {
            try {
                circuitBreaker.executeCallable(() -> { throw new RuntimeException("fail"); });
            } catch (Exception ignored) {}
        }
    }
}

// Retry testing
@Test
void retry_retriesOnTransientFailure() throws Exception {
    var retryConfig = RetryConfig.custom()
        .maxAttempts(3)
        .waitDuration(java.time.Duration.ofMillis(100))
        .retryExceptions(RuntimeException.class)
        .build();
    
    var retry = Retry.of("test", retryConfig);
    
    // Mock that fails 2 times then succeeds
    var callCount = new java.util.concurrent.atomic.AtomicInteger(0);
    
    String result = retry.executeCallable(() -> {
        int attempt = callCount.incrementAndGet();
        if (attempt < 3) throw new RuntimeException("Transient failure " + attempt);
        return "Success on attempt " + attempt;
    });
    
    assertThat(result).isEqualTo("Success on attempt 3");
    assertThat(callCount.get()).isEqualTo(3);
}
```

---

## 70.3 WireMock for Dependency Failure Simulation

```java
import com.github.tomakehurst.wiremock.*;
import com.github.tomakehurst.wiremock.client.*;
import com.github.tomakehurst.wiremock.extension.responsetemplating.*;

@SpringBootTest
@AutoConfigureWireMock(port = 0)  // random port
class ExternalServiceResilienceTest {
    
    @Value("${wiremock.server.port}")
    int wireMockPort;
    
    @Autowired
    UserServiceClient userServiceClient;  // HTTP client pointing to WireMock
    
    @Test
    void shouldReturnFallback_whenExternalServiceTimeout() throws Exception {
        // Simulate slow response (timeout after 5 seconds)
        stubFor(WireMock.get(urlEqualTo("/api/users/user-1"))
            .willReturn(aResponse()
                .withFixedDelay(10_000)  // 10 second delay
                .withStatus(200)
                .withBody("{\"id\": \"user-1\"}")
            )
        );
        
        // Our client has 2s timeout → should fail fast and use fallback
        var stopwatch = com.google.common.base.Stopwatch.createStarted();
        var result = userServiceClient.getUser("user-1");  // uses fallback
        
        assertThat(stopwatch.elapsed(java.util.concurrent.TimeUnit.MILLISECONDS))
            .isLessThan(3000);  // failed fast, not waited 10 seconds
        assertThat(result.isFallback()).isTrue();
    }
    
    @Test
    void shouldRetry_onServerError() {
        // Fail 2 times then succeed
        stubFor(WireMock.get(urlEqualTo("/api/products/P001"))
            .inScenario("flaky-service")
            .whenScenarioStateIs(STARTED)
            .willReturn(aResponse().withStatus(500))
            .willSetStateTo("first-failure")
        );
        
        stubFor(WireMock.get(urlEqualTo("/api/products/P001"))
            .inScenario("flaky-service")
            .whenScenarioStateIs("first-failure")
            .willReturn(aResponse().withStatus(500))
            .willSetStateTo("second-failure")
        );
        
        stubFor(WireMock.get(urlEqualTo("/api/products/P001"))
            .inScenario("flaky-service")
            .whenScenarioStateIs("second-failure")
            .willReturn(aResponse()
                .withStatus(200)
                .withBody("{\"id\": \"P001\", \"name\": \"Product 1\"}")
            )
        );
        
        var product = productServiceClient.getProduct("P001");
        
        assertThat(product.getId()).isEqualTo("P001");
        verify(3, getRequestedFor(urlEqualTo("/api/products/P001")));
    }
    
    @Test
    void shouldHandlePartialFailure_andContinue() {
        // Product service works
        stubFor(WireMock.get(urlPathMatching("/api/products/.*"))
            .willReturn(aResponse().withStatus(200)
                .withBodyFile("product.json")
            )
        );
        
        // Recommendation service is down
        stubFor(WireMock.get(urlPathMatching("/api/recommendations/.*"))
            .willReturn(aResponse().withStatus(503))
        );
        
        // Should return product without recommendations (graceful degradation)
        var page = productPageService.getProductPage("P001");
        
        assertThat(page.product()).isNotNull();
        assertThat(page.recommendations()).isEmpty();  // fallback to empty
    }
}
```

---

## 70.4 Chaos Testing in Kubernetes

```yaml
# Chaos Mesh (CNCF): inject failures at infrastructure level

# Kill random pods in a namespace
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-test
spec:
  action: pod-kill
  mode: random-max-percent
  value: "30"     # kill up to 30% of pods
  selector:
    namespaces:
      - production
    labelSelectors:
      app: order-service
  scheduler:
    cron: "0 */4 * * *"  # every 4 hours

---
# Network delay injection
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-test
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: payment-service
  delay:
    latency: "500ms"
    correlation: "25"
    jitter: "200ms"
  duration: "5m"

---
# Stress CPU/memory
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: memory-stress-test
spec:
  mode: one
  selector:
    namespaces:
      - staging
    labelSelectors:
      app: product-service
  stressors:
    memory:
      workers: 4
      size: "512MB"
  duration: "2m"
```

---

## 70.5 Resilience Testing Checklist

```java
// Integration test: verify system behavior under failure conditions

@SpringBootTest
@Testcontainers
class SystemResilienceIntegrationTest {
    
    @Container
    static org.testcontainers.containers.PostgreSQLContainer<?> postgres =
        new org.testcontainers.containers.PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb");
    
    @Container
    static org.testcontainers.containers.GenericContainer<?> redis =
        new org.testcontainers.containers.GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    
    @Autowired
    ProductService productService;
    
    @Test
    void shouldUseCache_whenDatabaseDown() {
        // 1. Prime cache
        productService.getProducts();  // loads from DB, caches in Redis
        
        // 2. Simulate DB failure
        postgres.stop();
        
        // 3. Request should still work (from cache)
        assertThatNoException().isThrownBy(() -> productService.getProducts());
        
        // 4. Restart DB
        postgres.start();
    }
    
    @Test
    void shouldGracefullyDegrade_whenCacheDown() {
        // Cache down → should fall back to DB (slower but correct)
        redis.stop();
        
        var products = productService.getProducts();
        
        assertThat(products).isNotEmpty();  // DB still works
        
        redis.start();
    }
    
    @Test
    void shouldHandleCircuitBreakerRecovery() throws InterruptedException {
        // Cause circuit to open (6 failures out of 10)
        for (int i = 0; i < 10; i++) {
            try { productService.callExternalApi(); }
            catch (Exception ignored) {}
        }
        
        // Circuit should be open
        var metricsEndpoint = restTemplate.getForObject(
            "/actuator/circuitbreakers", String.class);
        assertThat(metricsEndpoint).contains("\"state\":\"OPEN\"");
        
        // Wait for reset
        Thread.sleep(6000);  // waitDurationInOpenState = 5s
        
        // Circuit should try half-open, then close after successful calls
        productService.callExternalApi();
        assertThat(metricsEndpoint).contains("\"state\":\"CLOSED\"");
    }
}
```

---

## สรุป Part 70

```
Chaos Engineering Mindset:

Before Chaos:
  ✓ Define steady state (normal metrics)
  ✓ Create hypothesis: "System X-functional during Y failure"
  ✓ Minimize blast radius (staging first, 1 pod, 1%)
  ✓ Have kill switch ready

Failure Types to Test:
  Dependency: external service timeout, 500 error, DNS failure
  Resource:   CPU 95%, memory 90%, disk full, thread pool exhaustion
  Network:    latency 500ms, packet loss 10%, partition
  Data:       corrupt message, schema mismatch

Tools:
  WireMock   = mock HTTP dependencies with programmable behavior
  Testcontainers = real DB/Redis in tests (start/stop on demand)
  Chaos Mesh = Kubernetes-native chaos injection
  Gatling    = load testing to find breaking points

Resilience Patterns to Verify:
  ✓ Circuit breaker opens + recovers
  ✓ Retry only on transient errors (not 4xx)
  ✓ Timeout + fallback
  ✓ Cache fallback when DB down
  ✓ Graceful degradation (serve partial data vs total failure)
  ✓ Health check accuracy (report DOWN when actually down)
  ✓ Graceful shutdown (finish in-flight requests)
```

➡️ [Part 71: Advanced Spring Security](./Part-71-SpringSecurity.md)
