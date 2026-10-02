# Part 54: Microservices Communication Patterns
## ขั้นตอนที่ 3681-3750: Service Discovery, Circuit Breaker, API Gateway

---

## 54.1 Service Discovery

```yaml
# Eureka Server
# build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-server")
}

# application.yaml (Eureka Server)
spring:
  application:
    name: service-registry
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false
```

```java
// Eureka Server main class
@org.springframework.boot.autoconfigure.SpringBootApplication
@org.springframework.cloud.netflix.eureka.server.EnableEurekaServer
public class ServiceRegistryApplication {
    public static void main(String[] args) {
        org.springframework.boot.SpringApplication.run(ServiceRegistryApplication.class, args);
    }
}

// Client service (registers with Eureka)
// application.yaml
spring:
  application:
    name: order-service
eureka:
  client:
    service-url:
      defaultZone: http://registry:8761/eureka
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    health-check-url-path: /actuator/health
```

---

## 54.2 Load-Balanced HTTP Client

```java
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.web.client.RestClient;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
class ServiceClientConfig {
    
    // Load-balanced RestClient (uses Eureka to resolve service name)
    @Bean
    @LoadBalanced
    RestClient.Builder restClientBuilder() {
        return RestClient.builder();
    }
    
    @Bean
    @LoadBalanced
    WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

@Service
class UserServiceClient {
    
    private final RestClient restClient;
    
    UserServiceClient(RestClient.Builder builder) {
        // "user-service" resolves via Eureka to actual IP:port
        this.restClient = builder.baseUrl("http://user-service").build();
    }
    
    public UserDTO getUser(Long userId) {
        return restClient.get()
            .uri("/api/v1/users/{id}", userId)
            .retrieve()
            .body(UserDTO.class);
    }
    
    public List<UserDTO> getUsersByIds(List<Long> ids) {
        return restClient.post()
            .uri("/api/v1/users/batch")
            .body(new BatchRequest(ids))
            .retrieve()
            .body(new org.springframework.core.ParameterizedTypeReference<>() {});
    }
}

record BatchRequest(List<Long> ids) {}
```

---

## 54.3 Circuit Breaker (Resilience4j)

```java
import io.github.resilience4j.circuitbreaker.*;
import io.github.resilience4j.retry.*;
import io.github.resilience4j.bulkhead.*;
import io.github.resilience4j.timelimiter.*;

@Configuration
class ResilienceConfig {
    
    @Bean
    CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)          // open if 50% of calls fail
            .waitDurationInOpenState(java.time.Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(5)
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)
            .recordExceptions(Exception.class)
            .ignoreExceptions(IllegalArgumentException.class)  // don't count validation errors
            .build();
        
        return CircuitBreakerRegistry.of(config);
    }
    
    @Bean
    RetryRegistry retryRegistry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(java.time.Duration.ofMillis(500))
            .retryExceptions(java.net.ConnectException.class,
                            java.util.concurrent.TimeoutException.class)
            .ignoreExceptions(IllegalArgumentException.class)
            .build();
        
        return RetryRegistry.of(config);
    }
    
    @Bean
    BulkheadRegistry bulkheadRegistry() {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(25)     // max concurrent calls to this service
            .maxWaitDuration(java.time.Duration.ofMillis(100))
            .build();
        
        return BulkheadRegistry.of(config);
    }
}

@Service
class ResilientUserServiceClient {
    
    private final RestClient restClient;
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final Bulkhead bulkhead;
    
    ResilientUserServiceClient(
            RestClient.Builder builder,
            CircuitBreakerRegistry cbRegistry,
            RetryRegistry retryRegistry,
            BulkheadRegistry bulkheadRegistry) {
        this.restClient = builder.baseUrl("http://user-service").build();
        this.circuitBreaker = cbRegistry.circuitBreaker("user-service");
        this.retry = retryRegistry.retry("user-service");
        this.bulkhead = bulkheadRegistry.bulkhead("user-service");
    }
    
    public UserDTO getUser(Long userId) {
        // Apply: Bulkhead → CircuitBreaker → Retry → actual call
        var decorated = Bulkhead.decorateSupplier(bulkhead,
            CircuitBreaker.decorateSupplier(circuitBreaker,
                Retry.decorateSupplier(retry,
                    () -> callUserService(userId)
                )
            )
        );
        
        return io.github.resilience4j.core.SupplierUtils
            .recover(decorated, this::fallbackUser)
            .get();
    }
    
    private UserDTO callUserService(Long userId) {
        return restClient.get()
            .uri("/api/v1/users/{id}", userId)
            .retrieve()
            .body(UserDTO.class);
    }
    
    // Fallback when circuit is open or all retries failed
    private UserDTO fallbackUser(Throwable t) {
        System.err.println("Fallback for user service: " + t.getMessage());
        return UserDTO.unknown();  // return minimal data
    }
}

record UserDTO(Long id, String name, String email) {
    static UserDTO unknown() { return new UserDTO(-1L, "Unknown", ""); }
}
```

---

## 54.4 API Gateway (Spring Cloud Gateway)

```yaml
# Gateway application.yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true  # auto-create routes from Eureka
      routes:
        - id: order-service
          uri: lb://order-service  # lb:// = load balanced
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - RewritePath=/api/v1/orders/(?<segment>.*), /api/v1/orders/${segment}
            - AddRequestHeader=X-Source-Service, api-gateway
            - AddResponseHeader=X-Response-Time, ${responseTime}
            - RequestRateLimiter=10, 20  # 10 req/s, burst 20
            - CircuitBreaker=name=order-service,fallbackUri=/fallback/orders
        
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/v1/users/**
          filters:
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 50ms
                  maxBackoff: 500ms
                  factor: 2
        
        # Health check aggregation
        - id: health
          uri: http://localhost:8080
          predicates:
            - Path=/health
          filters:
            - SetStatus=200
```

```java
// Gateway filters
@Component
class AuthFilter implements org.springframework.cloud.gateway.filter.GlobalFilter,
                           org.springframework.core.Ordered {
    
    private final JwtService jwtService;
    
    AuthFilter(JwtService jwtService) { this.jwtService = jwtService; }
    
    @Override
    public reactor.core.publisher.Mono<Void> filter(
            org.springframework.web.server.ServerWebExchange exchange,
            org.springframework.cloud.gateway.filter.GatewayFilterChain chain) {
        
        var request = exchange.getRequest();
        
        // Skip auth for public endpoints
        if (isPublicPath(request.getPath().toString())) {
            return chain.filter(exchange);
        }
        
        var token = extractToken(request);
        if (token == null) {
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        var claims = jwtService.validateToken(token);
        if (claims == null) {
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // Propagate user info downstream
        var mutatedRequest = exchange.getRequest().mutate()
            .header("X-User-Id", claims.getSubject())
            .header("X-User-Role", claims.get("role", String.class))
            .build();
        
        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }
    
    private boolean isPublicPath(String path) {
        return path.startsWith("/api/v1/auth/") || path.equals("/health");
    }
    
    private String extractToken(org.springframework.http.server.reactive.ServerHttpRequest request) {
        var auth = request.getHeaders().getFirst("Authorization");
        if (auth != null && auth.startsWith("Bearer ")) {
            return auth.substring(7);
        }
        return null;
    }
    
    @Override
    public int getOrder() { return -100; }
}
```

---

## 54.5 Service Mesh Communication Patterns

```
Inter-service communication:

Synchronous (request-response):
  REST/HTTP  = simple, universal, good for queries
  gRPC       = fast, typed, good for internal services
  
  Problem: caller waits → latency chain
  Use when: result needed immediately

Asynchronous (fire-and-forget):
  Kafka      = durable, high-throughput, replay
  RabbitMQ   = routing, fanout, transient messages
  Redis Pub/Sub = ephemeral, low-latency
  
  Use when: eventually consistent is OK

Hybrid pattern:
  Request → sync response (acknowledgment)
         → async processing (actual work)
         → webhook/SSE/polling for result

Pattern selection guide:
  "Create order" → sync (need order ID back)
  "Send email after order" → async (eventual)
  "Update inventory" → async SAGA
  "Get product details" → sync REST/gRPC with cache
  "Stream live updates" → WebSocket or SSE
```

---

## สรุป Part 54

```
Microservices Communication:

Service Discovery:
  Eureka = services register themselves
  @LoadBalanced RestClient = resolve by service name

Circuit Breaker (Resilience4j):
  CLOSED → normal operation
  OPEN   → fails fast (50% failure rate)
  HALF-OPEN → test with small traffic
  
  Stack: Bulkhead → CircuitBreaker → Retry

API Gateway:
  Single entry point for all services
  Handles: auth, rate limit, routing, SSL termination
  lb://service-name = load balanced

When to use sync vs async:
  Sync  = need result immediately (GET, create+return ID)
  Async = fire-and-forget, eventual consistency (email, analytics)
```

➡️ [Part 55: Authentication & Authorization Patterns](./Part-55-AuthPatterns.md)
