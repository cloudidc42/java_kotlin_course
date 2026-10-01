# Part 41: Spring WebFlux & Reactive Programming
## ขั้นตอนที่ 2771-2840: Non-Blocking Reactive APIs

---

## 41.1 Reactive Programming Concepts

```
Traditional (Blocking):
  Thread → Request → Block waiting for DB → Response
  1 request = 1 thread (thread pool limited)
  
Reactive (Non-Blocking):
  Thread → Request → Register callback → Free thread → DB responds → callback fires → Response
  1 thread = handle many requests
  
Reactor core types:
  Mono<T>  = 0 or 1 item (async Optional)
  Flux<T>  = 0 to N items (async Stream)

When to use WebFlux:
  ✓ High concurrency, many simultaneous connections
  ✓ Streaming data (SSE, WebSocket)
  ✓ Composing async operations
  ✗ CPU-bound work (reactive adds overhead)
  ✗ Teams unfamiliar with reactive thinking
  ✗ Many blocking dependencies (JPA, most JDBC)
```

---

## 41.2 Setup

```xml
<!-- pom.xml - replace spring-boot-starter-web with: -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: postgres
    password: postgres
    pool:
      initial-size: 5
      max-size: 20
```

---

## 41.3 Mono & Flux Operations

```java
import reactor.core.publisher.*;
import java.time.Duration;
import java.util.*;

public class ReactorExamples {
    
    // ====== Mono operations ======
    public static void monoExamples() {
        
        // Create
        Mono<String> just = Mono.just("Hello");
        Mono<String> empty = Mono.empty();
        Mono<String> error = Mono.error(new RuntimeException("Oops"));
        Mono<String> deferred = Mono.fromCallable(() -> "computed");
        
        // Transform
        Mono<Integer> mapped = just.map(String::length);
        Mono<String> flatMapped = just.flatMap(s -> Mono.just(s.toUpperCase()));
        
        // Error handling
        Mono<String> recovered = error
            .onErrorReturn("default")
            .doOnError(e -> System.err.println("Error: " + e.getMessage()));
        
        Mono<String> retried = error
            .retry(3)
            .onErrorResume(e -> Mono.just("fallback"));
        
        // Timeout
        Mono<String> withTimeout = deferred
            .timeout(Duration.ofSeconds(5))
            .onErrorReturn("timed-out");
        
        // Combine
        Mono<String> combined = Mono.zip(
            Mono.just("Hello"),
            Mono.just("World")
        ).map(tuple -> tuple.getT1() + " " + tuple.getT2());
        
        // Subscribe
        just.subscribe(
            value -> System.out.println("Got: " + value),
            error2 -> System.err.println("Error: " + error2),
            () -> System.out.println("Completed")
        );
    }
    
    // ====== Flux operations ======
    public static void fluxExamples() {
        
        // Create
        Flux<Integer> range = Flux.range(1, 10);
        Flux<String> fromList = Flux.fromIterable(List.of("a", "b", "c"));
        Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));
        
        // Transform
        Flux<String> processed = range
            .filter(n -> n % 2 == 0)
            .map(n -> "item-" + n)
            .take(3)
            .collectList()
            .flatMapMany(Flux::fromIterable);
        
        // FlatMap (async per element)
        Flux<String> async = fromList
            .flatMap(s -> Mono.fromCallable(() -> s.toUpperCase())
                .subscribeOn(reactor.core.scheduler.Schedulers.boundedElastic()));
        
        // Concurrency control
        Flux<String> concurrent = fromList
            .flatMap(s -> processAsync(s), 4);  // max 4 concurrent
        
        // Merge vs Concat
        Flux<Integer> merged = Flux.merge(
            Flux.just(1, 2, 3).delayElements(Duration.ofMillis(100)),
            Flux.just(4, 5, 6)
        );  // interleaved
        
        Flux<Integer> concatenated = Flux.concat(
            Flux.just(1, 2, 3),
            Flux.just(4, 5, 6)
        );  // 1,2,3 then 4,5,6
        
        // Backpressure
        range
            .onBackpressureBuffer(100)
            .subscribe(n -> System.out.println(n));
        
        // Batching
        range.buffer(3).subscribe(batch -> 
            System.out.println("Batch: " + batch));
        
        range.window(Duration.ofSeconds(1))
            .flatMap(window -> window.collectList())
            .subscribe(batch -> System.out.println("Window: " + batch));
    }
    
    private static Mono<String> processAsync(String input) {
        return Mono.fromCallable(() -> {
            Thread.sleep(100);
            return input.toUpperCase();
        }).subscribeOn(reactor.core.scheduler.Schedulers.boundedElastic());
    }
}
```

---

## 41.4 Reactive REST Controller

```java
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.*;
import org.springframework.http.*;

@RestController
@RequestMapping("/api/v1/products")
public class ProductController {
    
    private final ProductService productService;
    
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    
    @GetMapping
    public Flux<ProductDTO> findAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return productService.findAll(page, size);
    }
    
    @GetMapping("/{id}")
    public Mono<ResponseEntity<ProductDTO>> findById(@PathVariable Long id) {
        return productService.findById(id)
            .map(product -> ResponseEntity.ok(product))
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public Mono<ResponseEntity<ProductDTO>> create(
            @RequestBody Mono<CreateProductRequest> request) {
        return request
            .flatMap(productService::create)
            .map(product -> ResponseEntity
                .status(HttpStatus.CREATED)
                .body(product));
    }
    
    @PutMapping("/{id}")
    public Mono<ResponseEntity<ProductDTO>> update(
            @PathVariable Long id,
            @RequestBody Mono<UpdateProductRequest> request) {
        return request
            .flatMap(req -> productService.update(id, req))
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    @DeleteMapping("/{id}")
    public Mono<ResponseEntity<Void>> delete(@PathVariable Long id) {
        return productService.delete(id)
            .then(Mono.just(ResponseEntity.<Void>noContent().build()));
    }
    
    // Server-Sent Events (SSE)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ProductDTO> streamProducts() {
        return productService.findAll(0, Integer.MAX_VALUE)
            .delayElements(Duration.ofMillis(100));  // simulate streaming
    }
}

// Functional routing (alternative to annotations)
@org.springframework.context.annotation.Configuration
class ProductRouter {
    
    @org.springframework.context.annotation.Bean
    public org.springframework.web.reactive.function.server.RouterFunction<
        org.springframework.web.reactive.function.server.ServerResponse
    > productRoutes(ProductHandler handler) {
        
        return org.springframework.web.reactive.function.server.RouterFunctions
            .route()
            .GET("/functional/products", handler::findAll)
            .GET("/functional/products/{id}", handler::findById)
            .POST("/functional/products", handler::create)
            .DELETE("/functional/products/{id}", handler::delete)
            .build();
    }
}

@org.springframework.stereotype.Component
class ProductHandler {
    
    private final ProductService service;
    
    ProductHandler(ProductService service) { this.service = service; }
    
    public Mono<org.springframework.web.reactive.function.server.ServerResponse> findAll(
            org.springframework.web.reactive.function.server.ServerRequest req) {
        return org.springframework.web.reactive.function.server.ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(service.findAll(0, 20), ProductDTO.class);
    }
    
    public Mono<org.springframework.web.reactive.function.server.ServerResponse> findById(
            org.springframework.web.reactive.function.server.ServerRequest req) {
        Long id = Long.parseLong(req.pathVariable("id"));
        return service.findById(id)
            .flatMap(p -> org.springframework.web.reactive.function.server.ServerResponse.ok().bodyValue(p))
            .switchIfEmpty(org.springframework.web.reactive.function.server.ServerResponse.notFound().build());
    }
    
    public Mono<org.springframework.web.reactive.function.server.ServerResponse> create(
            org.springframework.web.reactive.function.server.ServerRequest req) {
        return req.bodyToMono(CreateProductRequest.class)
            .flatMap(service::create)
            .flatMap(p -> org.springframework.web.reactive.function.server.ServerResponse.created(
                java.net.URI.create("/functional/products/" + p.id())
            ).bodyValue(p));
    }
    
    public Mono<org.springframework.web.reactive.function.server.ServerResponse> delete(
            org.springframework.web.reactive.function.server.ServerRequest req) {
        Long id = Long.parseLong(req.pathVariable("id"));
        return service.delete(id)
            .then(org.springframework.web.reactive.function.server.ServerResponse.noContent().build());
    }
}
```

---

## 41.5 R2DBC (Reactive Database)

```java
import org.springframework.data.repository.*;
import org.springframework.data.r2dbc.repository.*;
import reactor.core.publisher.*;

@org.springframework.data.relational.core.mapping.Table("products")
public class Product {
    @org.springframework.data.annotation.Id
    private Long id;
    private String name;
    private String description;
    private Double price;
    private Integer stock;
    
    // constructors, getters, setters...
}

// Reactive repository
public interface ProductRepository extends ReactiveCrudRepository<Product, Long> {
    
    Flux<Product> findByStockGreaterThan(int minStock);
    
    @Query("SELECT * FROM products WHERE price BETWEEN :min AND :max ORDER BY price")
    Flux<Product> findByPriceRange(Double min, Double max);
    
    Mono<Long> countByStockLessThan(int threshold);
}

// Reactive service with transactions
@org.springframework.stereotype.Service
public class ProductService {
    
    private final ProductRepository repository;
    private final org.springframework.r2dbc.core.DatabaseClient databaseClient;
    
    public ProductService(ProductRepository repository,
                           org.springframework.r2dbc.core.DatabaseClient databaseClient) {
        this.repository = repository;
        this.databaseClient = databaseClient;
    }
    
    public Flux<ProductDTO> findAll(int page, int size) {
        return repository.findAll()
            .skip((long) page * size)
            .take(size)
            .map(this::toDTO);
    }
    
    public Mono<ProductDTO> findById(Long id) {
        return repository.findById(id).map(this::toDTO);
    }
    
    @org.springframework.transaction.annotation.Transactional
    public Mono<ProductDTO> create(CreateProductRequest req) {
        Product product = new Product();
        product.setName(req.name());
        product.setPrice(req.price());
        product.setStock(req.stock());
        return repository.save(product).map(this::toDTO);
    }
    
    @org.springframework.transaction.annotation.Transactional
    public Mono<ProductDTO> update(Long id, UpdateProductRequest req) {
        return repository.findById(id)
            .flatMap(product -> {
                if (req.name() != null) product.setName(req.name());
                if (req.price() != null) product.setPrice(req.price());
                return repository.save(product);
            })
            .map(this::toDTO);
    }
    
    public Mono<Void> delete(Long id) {
        return repository.deleteById(id);
    }
    
    // Complex query with DatabaseClient
    public Flux<ProductDTO> findLowStock(int threshold) {
        return databaseClient
            .sql("SELECT p.*, c.name as category_name FROM products p " +
                 "JOIN categories c ON c.id = p.category_id " +
                 "WHERE p.stock < :threshold ORDER BY p.stock")
            .bind("threshold", threshold)
            .map(row -> new ProductDTO(
                row.get("id", Long.class),
                row.get("name", String.class),
                row.get("description", String.class),
                row.get("price", Double.class),
                row.get("stock", Integer.class),
                row.get("category_name", String.class)
            ))
            .all();
    }
    
    private ProductDTO toDTO(Product p) {
        return new ProductDTO(p.getId(), p.getName(), p.getDescription(), p.getPrice(), p.getStock(), null);
    }
}
```

---

## 41.6 WebClient (Reactive HTTP Client)

```java
import org.springframework.web.reactive.function.client.*;
import reactor.core.publisher.*;
import java.time.Duration;

@org.springframework.stereotype.Service
public class ExternalApiService {
    
    private final WebClient webClient;
    
    public ExternalApiService() {
        this.webClient = WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader("Accept", "application/json")
            .codecs(config -> config.defaultCodecs().maxInMemorySize(10 * 1024 * 1024))
            .filter(ExchangeFilterFunction.ofRequestProcessor(req -> {
                System.out.println("Request: " + req.method() + " " + req.url());
                return Mono.just(req);
            }))
            .build();
    }
    
    public Mono<UserDTO> getUser(Long userId) {
        return webClient.get()
            .uri("/users/{id}", userId)
            .retrieve()
            .onStatus(status -> status.is4xxClientError(), response ->
                response.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(new RuntimeException("4xx: " + body))))
            .onStatus(status -> status.is5xxServerError(), response ->
                Mono.error(new RuntimeException("Server error")))
            .bodyToMono(UserDTO.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(reactor.util.retry.Retry.backoff(3, Duration.ofMillis(500)));
    }
    
    public Flux<ProductDTO> streamProducts() {
        return webClient.get()
            .uri("/products/stream")
            .accept(org.springframework.http.MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(ProductDTO.class);
    }
    
    // Parallel calls
    public Mono<OrderWithUserDTO> getOrderWithUser(Long orderId) {
        Mono<OrderDTO> orderMono = webClient.get()
            .uri("/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(OrderDTO.class);
        
        Mono<UserDTO> userMono = orderMono.flatMap(order ->
            webClient.get()
                .uri("/users/{id}", order.userId())
                .retrieve()
                .bodyToMono(UserDTO.class));
        
        return Mono.zip(orderMono, userMono)
            .map(tuple -> new OrderWithUserDTO(tuple.getT1(), tuple.getT2()));
    }
}
```

---

## สรุป Part 41

```
Reactive Key Concepts:

Mono<T>  = single async value (0..1)
Flux<T>  = stream of values (0..N)

Operators:
  map()         = transform each element
  flatMap()     = async transform (returns Mono/Flux)
  filter()      = keep elements matching predicate
  zip()         = combine multiple Publishers
  merge()       = interleave multiple Flux
  concat()      = sequential Flux  
  timeout()     = fail after duration
  retry()       = retry on error
  onErrorReturn() = fallback value

Schedulers:
  Schedulers.boundedElastic() = blocking I/O (DB, file)
  Schedulers.parallel()       = CPU work
  Schedulers.single()         = single thread

When reactive pays off:
  - Many concurrent connections (C10K problem)
  - Streaming large datasets
  - Aggregating multiple service calls
```

➡️ [Part 42: Observability & Monitoring](./Part-42-Observability.md)
