# Part 80: Reactive Programming with Spring WebFlux
## ขั้นตอนที่ 5501-5570: Non-blocking I/O, Reactor, Reactive Streams

---

## 80.1 Reactive Programming คืออะไร

```
Traditional (Blocking):
  Thread → Request → DB Query → Wait → Response
  Thread ถูก block ระหว่างรอ
  100 concurrent → 100 threads → ใช้ RAM เยอะ

Reactive (Non-blocking):
  Thread → Request → DB Query (non-blocking) → Thread ทำอื่น
  เมื่อ DB ตอบ → thread ใดก็ได้รับและตอบกลับ
  100 concurrent → 4-8 threads (event loop) → ประหยัด RAM

Reactive Streams Spec:
  Publisher  = ส่งข้อมูล (0..N items, แล้วจบ/error)
  Subscriber = รับข้อมูล
  Processor  = ทั้ง Publisher และ Subscriber
  Subscription = ควบคุม backpressure

Project Reactor (Spring's implementation):
  Mono<T>  = 0 หรือ 1 item (เหมือน Optional แต่ async)
  Flux<T>  = 0..N items (stream)

เมื่อใช้ WebFlux แทน MVC:
  ✓ High concurrency, low thread count
  ✓ Streaming responses (SSE, WebSocket)
  ✓ Reactive database drivers (R2DBC, MongoDB reactive)
  ✗ Learning curve สูง
  ✗ Debugging/tracing ยากกว่า
  ✗ JDBC ไม่รองรับ (blocking I/O)
```

---

## 80.2 Mono และ Flux Basics

```kotlin
import reactor.core.publisher.*
import reactor.kotlin.core.publisher.*

// ====== Mono: 0 or 1 item ======
fun createMono() {
    // Create
    val mono1 = Mono.just("hello")
    val mono2 = Mono.empty<String>()
    val mono3 = Mono.error<String>(RuntimeException("oops"))
    val mono4 = Mono.fromCallable { expensiveComputation() }  // lazy
    val mono5 = Mono.fromSupplier { "value" }
    
    // Transform
    mono1
        .map { it.uppercase() }          // transform value
        .flatMap { fetchDataFor(it) }    // flatMap = mono.map().flatten()
        .filter { it.isNotEmpty() }
        .defaultIfEmpty("default")
        .switchIfEmpty(Mono.just("fallback"))
        .doOnNext { println("Got: $it") }
        .doOnError { println("Error: $it") }
        .onErrorReturn("fallback")
        .onErrorResume { Mono.just("recovered") }
        .block()  // ONLY in tests/main - never in reactive chain!
}

// ====== Flux: 0..N items ======
fun createFlux() {
    // Create
    val flux1 = Flux.just(1, 2, 3, 4, 5)
    val flux2 = Flux.fromIterable(listOf("a", "b", "c"))
    val flux3 = Flux.range(1, 100)  // 1 to 100
    val flux4 = Flux.interval(java.time.Duration.ofSeconds(1))  // tick every second
    
    // Transform
    flux1
        .filter { it % 2 == 0 }          // keep even
        .map { it * 10 }                  // transform
        .flatMap { fetchAsync(it) }       // async for each (concurrent)
        .concatMap { fetchAsync(it) }     // async for each (sequential order kept)
        .take(3)                          // only first 3
        .skip(1)                          // skip first
        .distinct()                       // remove duplicates
        .collectList()                    // Flux → Mono<List>
        .block()
}

// ====== Combine ======
fun combine() {
    val flux1 = Flux.just(1, 2, 3)
    val flux2 = Flux.just(10, 20, 30)
    
    // merge = interleave based on timing
    Flux.merge(flux1, flux2)
    
    // concat = flux1 then flux2
    Flux.concat(flux1, flux2)
    
    // zip = pair items
    Flux.zip(flux1, flux2) { a, b -> a + b }  // [11, 22, 33]
    
    // combine latest
    Flux.combineLatest(flux1, flux2) { a, b -> "$a-$b" }
}
```

---

## 80.3 Spring WebFlux Controller

```kotlin
import org.springframework.web.bind.annotation.*
import org.springframework.http.*
import reactor.core.publisher.*

@RestController
@RequestMapping("/api/products")
class ProductController(
    private val productService: ProductService
) {
    
    // Returns Mono = single product
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: String): Mono<ResponseEntity<ProductDto>> =
        productService.findById(id)
            .map { ResponseEntity.ok(it.toDto()) }
            .defaultIfEmpty(ResponseEntity.notFound().build())
    
    // Returns Flux = list of products
    @GetMapping
    fun getProducts(
        @RequestParam(required = false) category: String?
    ): Flux<ProductDto> =
        productService.findAll(category).map { it.toDto() }
    
    // Server-Sent Events: stream updates
    @GetMapping(value = ["/stream"], produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun streamProducts(): Flux<ProductDto> =
        productService.streamLiveProducts()
            .map { it.toDto() }
    
    // POST: create
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createProduct(@RequestBody request: CreateProductRequest): Mono<ProductDto> =
        productService.create(request).map { it.toDto() }
    
    // DELETE: returns Mono<Void>
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun deleteProduct(@PathVariable id: String): Mono<Void> =
        productService.delete(id)
    
    // Streaming with backpressure: client controls rate
    @GetMapping(value = ["/export"], produces = [MediaType.APPLICATION_NDJSON_VALUE])
    fun exportProducts(): Flux<ProductDto> =
        productService.findAll(null)
            .map { it.toDto() }
            .delayElements(java.time.Duration.ofMillis(10))  // rate limit
}
```

---

## 80.4 Reactive Service Layer

```kotlin
@Service
class ProductService(
    private val productRepository: ReactiveProductRepository,
    private val cacheManager: ReactiveCacheManager,
    private val eventBus: EventBus
) {
    
    fun findById(id: String): Mono<Product> =
        cacheManager.get("product:$id")
            .switchIfEmpty(
                productRepository.findById(id)
                    .flatMap { product ->
                        cacheManager.set("product:$id", product, Duration.ofMinutes(10))
                            .thenReturn(product)
                    }
            )
    
    fun findAll(category: String?): Flux<Product> =
        if (category != null) productRepository.findByCategory(category)
        else productRepository.findAll()
    
    fun create(request: CreateProductRequest): Mono<Product> =
        Mono.just(request)
            .map { it.toDomain() }
            .flatMap { productRepository.save(it) }
            .flatMap { product ->
                eventBus.publish(ProductCreatedEvent(product.id))
                    .thenReturn(product)
            }
    
    fun delete(id: String): Mono<Void> =
        productRepository.findById(id)
            .switchIfEmpty(Mono.error(ProductNotFoundException(id)))
            .flatMap { productRepository.deleteById(id) }
            .then(cacheManager.evict("product:$id"))
    
    // Stream live price updates
    fun streamLiveProducts(): Flux<Product> =
        Flux.interval(Duration.ofSeconds(5))
            .flatMap { findAll(null) }
            .distinctUntilChanged()
}
```

---

## 80.5 R2DBC (Reactive Database)

```kotlin
// R2DBC = Reactive Relational Database Connectivity

// build.gradle.kts
// implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
// implementation("io.r2dbc:r2dbc-postgresql")

// Entity
@Table("products")
data class ProductEntity(
    @Id val id: String?,
    val name: String,
    val price: Double,
    val category: String,
    val inStock: Boolean
)

// Repository extends ReactiveCrudRepository
interface ProductR2dbcRepository : ReactiveCrudRepository<ProductEntity, String> {
    
    fun findByCategory(category: String): Flux<ProductEntity>
    
    @Query("SELECT * FROM products WHERE price BETWEEN :min AND :max")
    fun findByPriceRange(min: Double, max: Double): Flux<ProductEntity>
    
    @Query("SELECT COUNT(*) FROM products WHERE category = :category")
    fun countByCategory(category: String): Mono<Long>
}

// Custom queries with DatabaseClient
@Component
class ProductCustomRepository(private val databaseClient: DatabaseClient) {
    
    fun searchProducts(query: String, minPrice: Double): Flux<ProductEntity> =
        databaseClient.sql("""
            SELECT * FROM products 
            WHERE name ILIKE :query 
              AND price >= :minPrice
              AND in_stock = true
            ORDER BY price ASC
        """)
        .bind("query", "%$query%")
        .bind("minPrice", minPrice)
        .map { row, _ ->
            ProductEntity(
                id = row.get("id", String::class.java),
                name = row.get("name", String::class.java)!!,
                price = row.get("price", Double::class.java)!!,
                category = row.get("category", String::class.java)!!,
                inStock = row.get("in_stock", Boolean::class.java)!!
            )
        }
        .all()
}

// application.yaml
/*
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/shopdb
    username: postgres
    password: password
  sql:
    init:
      mode: always
      schema-locations: classpath:schema.sql
*/
```

---

## 80.6 Error Handling & Retry

```kotlin
@Service
class ResilientProductService(
    private val productRepository: ProductR2dbcRepository,
    private val externalPriceService: ExternalPriceService
) {
    
    fun getProductWithCurrentPrice(id: String): Mono<Product> =
        productRepository.findById(id)
            .switchIfEmpty(Mono.error(ProductNotFoundException(id)))
            // Retry transient errors
            .flatMap { product ->
                externalPriceService.getCurrentPrice(product.id)
                    .retryWhen(
                        Retry.backoff(3, Duration.ofMillis(100))
                            .filter { it is TransientException }
                            .maxBackoff(Duration.ofSeconds(5))
                    )
                    .onErrorReturn(product.price)  // fallback to stored price
                    .map { price -> product.copy(currentPrice = price) }
            }
    
    // Timeout
    fun getProductWithTimeout(id: String): Mono<Product> =
        productRepository.findById(id)
            .timeout(Duration.ofSeconds(3))
            .onErrorResume(TimeoutException::class.java) {
                Mono.error(ServiceUnavailableException("Product service timeout"))
            }
    
    // Global error handler via @ControllerAdvice
    // (WebFlux uses same @ControllerAdvice as MVC)
}

@ControllerAdvice
class GlobalErrorHandler {
    
    @ExceptionHandler(ProductNotFoundException::class)
    fun handleNotFound(ex: ProductNotFoundException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(404)
            .body(ErrorResponse("NOT_FOUND", ex.message ?: "Product not found"))
    
    @ExceptionHandler(ServiceUnavailableException::class)
    fun handleUnavailable(ex: ServiceUnavailableException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(503)
            .body(ErrorResponse("SERVICE_UNAVAILABLE", ex.message ?: "Service unavailable"))
}

data class ErrorResponse(val code: String, val message: String)
```

---

## 80.7 Reactive Security

```kotlin
// Security config for WebFlux (different from MVC)
@Configuration
@EnableWebFluxSecurity
@EnableReactiveMethodSecurity
class SecurityConfig {
    
    @Bean
    fun securityWebFilterChain(http: ServerHttpSecurity): SecurityWebFilterChain =
        http
            .csrf { it.disable() }
            .cors { it.configurationSource(corsConfig()) }
            .authorizeExchange { auth ->
                auth
                    .pathMatchers("/api/public/**").permitAll()
                    .pathMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                    .pathMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyExchange().authenticated()
            }
            .oauth2ResourceServer { oauth2 ->
                oauth2.jwt { jwt -> jwt.jwtAuthenticationConverter(jwtConverter()) }
            }
            .build()
    
    @Bean
    fun jwtConverter(): ReactiveJwtAuthenticationConverter =
        ReactiveJwtAuthenticationConverter().apply {
            setJwtGrantedAuthoritiesConverter(
                ReactiveJwtGrantedAuthoritiesConverterAdapter(
                    JwtGrantedAuthoritiesConverter().apply {
                        setAuthoritiesClaimName("roles")
                        setAuthorityPrefix("ROLE_")
                    }
                )
            )
        }
}

// Access current user in reactive chain
@GetMapping("/my-orders")
fun getMyOrders(auth: Mono<Authentication>): Flux<OrderDto> =
    auth
        .flatMapMany { authentication ->
            orderService.findByUserId(authentication.name)
        }
        .map { it.toDto() }
```

---

## สรุป Part 80

```
WebFlux vs Spring MVC:

MVC:         Blocking, Servlet-based, 1 thread/request
WebFlux:     Non-blocking, Reactive, event-loop

Use WebFlux when:
  ✓ High concurrency with I/O-bound work
  ✓ Streaming (SSE, WebSocket)
  ✓ All dependencies support reactive (R2DBC, Reactive Redis)
  ✗ CPU-bound work (no benefit, added complexity)
  ✗ Blocking dependencies (JDBC, Hibernate)

Reactor Types:
  Mono<T>  = 0..1 item
  Flux<T>  = 0..N items

Key Operations:
  map()       = sync transform
  flatMap()   = async transform (concurrent)
  concatMap() = async transform (preserve order)
  filter()    = conditional
  zip()       = combine two publishers
  merge()     = interleave two publishers
  switchIfEmpty() = fallback if empty

Error Handling:
  onErrorReturn(default)     = fallback value
  onErrorResume { ... }      = fallback publisher
  retryWhen(Retry.backoff()) = retry with backoff

NEVER block in reactive chain:
  block()     = only in tests or main()
  blockFirst()
  
  Use subscribeOn(Schedulers.boundedElastic()) 
  for blocking code wrapped in Mono.fromCallable
```

➡️ [Part 81: Kotlin DSL & Builder Patterns Advanced](./Part-81-KotlinDSL.md)
