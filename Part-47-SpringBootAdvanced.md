# Part 47: Spring Boot Advanced Features
## ขั้นตอนที่ 3191-3260: Auto-configuration, Events, AOP

---

## 47.1 Spring Boot Auto-configuration

```java
import org.springframework.boot.autoconfigure.*;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.*;

// Create custom auto-configuration
@AutoConfiguration
@ConditionalOnClass(name = "com.example.SomeLibrary")
@EnableConfigurationProperties(MyProperties.class)
public class MyLibraryAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean  // only if user hasn't defined their own
    public MyService myService(MyProperties properties) {
        return new MyService(properties.getEndpoint(), properties.getTimeout());
    }
    
    @Bean
    @ConditionalOnProperty(name = "my.library.cache.enabled", havingValue = "true", matchIfMissing = false)
    public MyServiceWithCache myServiceWithCache(MyProperties props) {
        return new MyServiceWithCache(props);
    }
    
    @Bean
    @ConditionalOnMissingBean(type = "javax.sql.DataSource")
    public InMemoryDataStore inMemoryDataStore() {
        return new InMemoryDataStore();
    }
}

// Configuration properties
@org.springframework.boot.context.properties.ConfigurationProperties(prefix = "my.library")
@org.springframework.boot.context.properties.ConfigurationPropertiesScan
public class MyProperties {
    private String endpoint = "http://localhost:8080";
    private int timeout = 30;
    private Cache cache = new Cache();
    
    public static class Cache {
        private boolean enabled = false;
        private int ttlSeconds = 300;
        // getters/setters...
    }
    // getters/setters...
}

// Register in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports:
// com.example.MyLibraryAutoConfiguration
```

---

## 47.2 Spring Events

```java
import org.springframework.context.*;
import org.springframework.context.event.*;
import org.springframework.stereotype.*;
import org.springframework.transaction.event.*;

// ====== Custom Events ======
record OrderCreatedEvent(String orderId, String userId, double total) {}
record OrderCancelledEvent(String orderId, String reason) {}

// ====== Publisher ======
@Service
public class OrderService {
    
    private final ApplicationEventPublisher eventPublisher;
    private final OrderRepository orderRepository;
    
    public OrderService(ApplicationEventPublisher eventPublisher,
                        OrderRepository orderRepository) {
        this.eventPublisher = eventPublisher;
        this.orderRepository = orderRepository;
    }
    
    @org.springframework.transaction.annotation.Transactional
    public Order createOrder(String userId, List<OrderItem> items) {
        Order order = orderRepository.save(new Order(userId, items));
        
        // Publish event AFTER transaction commits (AFTER_COMMIT)
        eventPublisher.publishEvent(
            new OrderCreatedEvent(order.getId(), userId, order.getTotal())
        );
        
        return order;
    }
}

// ====== Synchronous Listener ======
@Component
public class OrderEventHandler {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Called synchronously, in same thread
        System.out.println("Order created: " + event.orderId());
    }
    
    @EventListener
    @Order(1)  // Execution order when multiple listeners
    public void sendConfirmationEmail(OrderCreatedEvent event) {
        // emailService.send(...)
    }
    
    @EventListener
    @Order(2)
    public void updateInventory(OrderCreatedEvent event) {
        // inventoryService.reserve(...)
    }
    
    // Listen to multiple event types
    @EventListener({OrderCreatedEvent.class, OrderCancelledEvent.class})
    public void logOrderChange(Object event) {
        System.out.println("Order event: " + event);
    }
    
    // Conditional listener (SpEL expression)
    @EventListener(condition = "#event.total > 10000")
    public void handleHighValueOrder(OrderCreatedEvent event) {
        // Only fires when total > 10000
    }
}

// ====== Async Listener ======
@Component
@EnableAsync
public class AsyncOrderHandler {
    
    @Async
    @EventListener
    public java.util.concurrent.CompletableFuture<Void> processAsync(OrderCreatedEvent event) {
        // Runs in separate thread (won't block publisher)
        System.out.println("Async processing: " + event.orderId());
        return java.util.concurrent.CompletableFuture.completedFuture(null);
    }
}

// ====== Transactional Events ======
@Component
public class TransactionalOrderHandler {
    
    // Only fires AFTER transaction successfully commits
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterCommit(OrderCreatedEvent event) {
        // Safe to send emails, push to Kafka, etc.
    }
    
    // Fires if transaction rolls back
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void afterRollback(OrderCreatedEvent event) {
        System.err.println("Order creation failed: " + event.orderId());
    }
    
    // Fires after transaction completes (either way)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void afterCompletion(OrderCreatedEvent event) {
        // Cleanup, logging
    }
}

// ====== Spring lifecycle events ======
@Component
public class AppLifecycleListener {
    
    @EventListener(ContextRefreshedEvent.class)
    public void onContextRefreshed() {
        System.out.println("Context ready - warming up caches...");
    }
    
    @EventListener(ContextClosedEvent.class)
    public void onContextClosed() {
        System.out.println("Context closing - cleanup...");
    }
    
    @EventListener
    public void onReady(org.springframework.boot.context.event.ApplicationReadyEvent event) {
        System.out.println("App started! Running startup tasks...");
    }
}
```

---

## 47.3 AOP (Aspect-Oriented Programming)

```java
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.*;
import org.springframework.stereotype.*;

// Custom annotations for AOP
@java.lang.annotation.Target(java.lang.annotation.ElementType.METHOD)
@java.lang.annotation.Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@interface Audited {
    String action() default "";
}

@java.lang.annotation.Target(java.lang.annotation.ElementType.METHOD)
@java.lang.annotation.Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@interface RateLimit {
    int maxRequests() default 100;
    int perSeconds() default 60;
}

@Aspect
@Component
public class AuditAspect {
    
    private final AuditLogger auditLogger;
    
    public AuditAspect(AuditLogger auditLogger) { this.auditLogger = auditLogger; }
    
    // Before: intercept @Audited methods
    @Before("@annotation(audited) && within(com.example.service..*)")
    public void auditMethodCall(JoinPoint jp, Audited audited) {
        String className = jp.getTarget().getClass().getSimpleName();
        String methodName = jp.getSignature().getName();
        Object[] args = jp.getArgs();
        
        auditLogger.log(String.format("[AUDIT] %s.%s action=%s args=%s",
            className, methodName, audited.action(), java.util.Arrays.toString(args)));
    }
    
    // Around: full control over method execution
    @Around("@annotation(com.example.RateLimit)")
    public Object rateLimit(ProceedingJoinPoint pjp) throws Throwable {
        String key = pjp.getTarget().getClass().getSimpleName() + "." + pjp.getSignature().getName();
        
        RateLimit annotation = ((org.aspectj.lang.reflect.MethodSignature) pjp.getSignature())
            .getMethod().getAnnotation(RateLimit.class);
        
        if (!rateLimiter.isAllowed(key, annotation.maxRequests())) {
            throw new RuntimeException("Rate limit exceeded for: " + key);
        }
        
        return pjp.proceed();
    }
    
    @Autowired RateLimiter rateLimiter;
    
    // After returning: log result
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..)) && @annotation(audited)",
                    returning = "result")
    public void logResult(JoinPoint jp, Audited audited, Object result) {
        System.out.printf("[AUDIT] %s returned: %s%n", jp.getSignature().getName(), result);
    }
    
    // After throwing: handle exceptions
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))",
                   throwing = "ex")
    public void handleException(JoinPoint jp, Exception ex) {
        System.err.printf("[ERROR] %s threw: %s%n", jp.getSignature().getName(), ex.getMessage());
    }
}

// Performance logging aspect
@Aspect
@Component
public class PerformanceAspect {
    
    private static final org.slf4j.Logger log = 
        org.slf4j.LoggerFactory.getLogger(PerformanceAspect.class);
    
    private static final long SLOW_THRESHOLD_MS = 500;
    
    @Around("within(com.example.service..*)")
    public Object logPerformance(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            return pjp.proceed();
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            String method = pjp.getSignature().toShortString();
            
            if (elapsed > SLOW_THRESHOLD_MS) {
                log.warn("SLOW method: {} took {}ms", method, elapsed);
            } else {
                log.debug("Method: {} took {}ms", method, elapsed);
            }
        }
    }
}

// Usage
@Service
public class UserService {
    
    @Audited(action = "CREATE_USER")
    @RateLimit(maxRequests = 10, perSeconds = 60)
    public UserDTO createUser(CreateUserRequest request) {
        // AOP automatically applies auditing and rate limiting
        return null;
    }
}
```

---

## 47.4 Spring Cache Abstraction

```java
import org.springframework.cache.annotation.*;
import org.springframework.cache.*;
import org.springframework.stereotype.*;

@Service
@CacheConfig(cacheNames = "products")  // default cache name
public class ProductService {
    
    @Cacheable(key = "#id")
    public ProductDTO findById(Long id) {
        // Only executed on cache miss
        return repository.findById(id).map(this::toDTO).orElseThrow();
    }
    
    @Cacheable(key = "#page + '_' + #size + '_' + #search",
               condition = "#search == null || #search.length() > 2",
               unless = "#result.isEmpty()")
    public List<ProductDTO> search(int page, int size, String search) {
        return repository.findAll(page, size, search);
    }
    
    @CachePut(key = "#result.id")  // always execute + update cache
    public ProductDTO update(Long id, UpdateProductRequest req) {
        Product product = repository.findById(id).orElseThrow();
        // update...
        return toDTO(repository.save(product));
    }
    
    @CacheEvict(key = "#id")
    public void delete(Long id) {
        repository.deleteById(id);
    }
    
    @CacheEvict(allEntries = true)
    @org.springframework.scheduling.annotation.Scheduled(cron = "0 0 * * * *")  // every hour
    public void evictAllCache() {
        // scheduled cache invalidation
    }
    
    // Multiple cache operations
    @Caching(
        evict = {
            @CacheEvict(cacheNames = "products", key = "#result.id"),
            @CacheEvict(cacheNames = "product-pages", allEntries = true)
        },
        put = @CachePut(cacheNames = "products", key = "#result.id")
    )
    public ProductDTO createProduct(CreateProductRequest req) {
        return toDTO(repository.save(new Product(req)));
    }
    
    @Autowired ProductRepository repository;
    
    ProductDTO toDTO(Product p) { return new ProductDTO(p.getId(), p.getName(), p.getPrice()); }
}

// Programmatic cache management
@Service
class CacheManagerService {
    
    private final CacheManager cacheManager;
    
    CacheManagerService(CacheManager cacheManager) { this.cacheManager = cacheManager; }
    
    public void clearProductCache(Long productId) {
        Cache cache = cacheManager.getCache("products");
        if (cache != null) {
            cache.evict(productId);
        }
    }
    
    public void clearAllCaches() {
        cacheManager.getCacheNames().forEach(name -> {
            Cache cache = cacheManager.getCache(name);
            if (cache != null) cache.clear();
        });
    }
    
    @SuppressWarnings("unchecked")
    public <T> T getFromCache(String cacheName, Object key) {
        Cache cache = cacheManager.getCache(cacheName);
        return cache != null ? (T) cache.get(key, Object.class) : null;
    }
}
```

---

## 47.5 Scheduling & Async

```java
import org.springframework.scheduling.annotation.*;
import org.springframework.stereotype.*;
import java.util.concurrent.CompletableFuture;

@Configuration
@EnableAsync
@EnableScheduling
class AsyncConfig {
    
    @Bean
    public java.util.concurrent.Executor taskExecutor() {
        var executor = new org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}

@Service
class ScheduledTaskService {
    
    // Fixed rate: every 5 minutes
    @Scheduled(fixedRate = 5 * 60 * 1000)
    public void syncInventory() {
        System.out.println("Syncing inventory at " + java.time.LocalDateTime.now());
    }
    
    // Fixed delay: 1 minute AFTER previous completes
    @Scheduled(fixedDelay = 60000)
    public void processQueue() {
        System.out.println("Processing queue...");
    }
    
    // Cron expression
    @Scheduled(cron = "0 0 9 * * MON-FRI")  // 9 AM weekdays
    public void sendDailyReport() {
        System.out.println("Sending daily report...");
    }
    
    // With initial delay
    @Scheduled(initialDelay = 30000, fixedRate = 60000)
    public void warmupCache() {
        System.out.println("Cache warmup...");
    }
}

@Service
class AsyncService {
    
    @Async
    public CompletableFuture<String> processAsync(String input) {
        // Runs in separate thread from taskExecutor
        Thread.sleep(1000);
        return CompletableFuture.completedFuture("Processed: " + input);
    }
    
    @Async
    public CompletableFuture<Void> fireAndForget(String input) {
        // Fire and forget
        System.out.println("Processing: " + input);
        return CompletableFuture.completedFuture(null);
    }
    
    // Combine async results
    public CompletableFuture<String> combineResults(String a, String b) {
        var futureA = processAsync(a);
        var futureB = processAsync(b);
        
        return futureA.thenCombine(futureB, (ra, rb) -> ra + " + " + rb);
    }
}
```

---

## สรุป Part 47

| Feature | คำอธิบาย |
|---------|---------|
| `@AutoConfiguration` | Custom starter/library config |
| `@ConditionalOn*` | Conditional bean creation |
| `ApplicationEventPublisher` | Publish domain events |
| `@EventListener` | Handle events |
| `@TransactionalEventListener` | Events tied to transactions |
| `@Aspect` + `@Around` | Cross-cutting concerns |
| `@Cacheable` / `@CacheEvict` | Declarative caching |
| `@Scheduled` | Periodic tasks |
| `@Async` | Async method execution |

**AOP Use Cases:**
- Logging & performance monitoring
- Transaction management (Spring does this internally)
- Security checks
- Rate limiting
- Auditing

➡️ [Part 48: Reactive Kotlin with Coroutines in Spring](./Part-48-CoroutinesSpring.md)
