# Part 91: Advanced Microservices Design Patterns
## ขั้นตอนที่ 6271-6340: BFF, API Gateway, Service Discovery, Bulkhead

---

## 91.1 Advanced Microservices Patterns

```
Patterns covered:
  BFF (Backend for Frontend)
  API Gateway advanced patterns
  Service Discovery (Consul)
  Bulkhead Pattern
  Sidecar Pattern
  Ambassador Pattern
  Anti-Corruption Layer
  Strangler Fig Pattern
```

---

## 91.2 BFF (Backend for Frontend)

```kotlin
// BFF = dedicated API layer per frontend type
// Mobile BFF, Web BFF, Partner API BFF
// Each optimized for its client's needs

// Problem: generic microservices return too much / too little data
//          mobile needs different fields than web

// ====== Mobile BFF ======
@RestController
@RequestMapping("/mobile/v1")
class MobileBff(
    private val productService: ProductService,
    private val orderService: OrderService,
    private val userService: UserService
) {
    
    // Mobile home page: aggregate data in one call
    // (Mobile can't afford many round trips on slow networks)
    @GetMapping("/home")
    suspend fun getHomeScreen(
        @AuthenticationPrincipal userId: String
    ): MobileHomeResponse {
        // Parallel calls to multiple services
        val (featuredProducts, userOrders, profile) = coroutineScope {
            val products = async { productService.getFeatured(limit = 10) }
            val orders = async { orderService.getRecentOrders(userId, limit = 3) }
            val user = async { userService.getProfile(userId) }
            Triple(products.await(), orders.await(), user.await())
        }
        
        // Shape for mobile (minimal fields, optimized payload)
        return MobileHomeResponse(
            user = MobileUserSummary(
                name = profile.firstName,
                avatarUrl = profile.avatarUrl
            ),
            featuredProducts = featuredProducts.map { product ->
                MobileProductCard(
                    id = product.id,
                    name = product.name,
                    price = product.formattedPrice,
                    imageUrl = product.thumbnailUrl,  // thumbnail, not full image
                    inStock = product.inStock
                )
            },
            recentOrders = userOrders.map { order ->
                MobileOrderSummary(
                    id = order.id,
                    status = order.status,
                    total = order.formattedTotal,
                    itemCount = order.items.size
                )
            }
        )
    }
    
    // Mobile-specific: smaller payloads, fewer fields
    @GetMapping("/products/{id}")
    fun getProduct(@PathVariable id: String): MobileProductDetail {
        val product = productService.findById(id)
        return MobileProductDetail(
            id = product.id,
            name = product.name,
            description = product.shortDescription,  // truncated for mobile
            price = product.formattedPrice,
            images = product.images.take(5).map { it.thumbnailUrl },  // max 5 images
            rating = product.rating,
            reviewCount = product.reviewCount,
            inStock = product.inStock
        )
    }
}

// ====== Web BFF ======
@RestController
@RequestMapping("/web/v1")
class WebBff(
    private val productService: ProductService,
    private val reviewService: ReviewService
) {
    
    // Web can handle more data
    @GetMapping("/products/{id}")
    fun getProduct(@PathVariable id: String): WebProductDetail {
        val product = productService.findById(id)
        val reviews = reviewService.getTopReviews(id, limit = 20)
        val relatedProducts = productService.getRelated(id, limit = 8)
        
        // Full detail for web
        return WebProductDetail(
            product = product,
            reviews = reviews,
            relatedProducts = relatedProducts,
            seoMetadata = SeoMetadata(
                title = "${product.name} - Shop",
                description = product.description.take(160),
                ogImage = product.images.firstOrNull()?.url
            )
        )
    }
}

data class MobileHomeResponse(
    val user: MobileUserSummary,
    val featuredProducts: List<MobileProductCard>,
    val recentOrders: List<MobileOrderSummary>
)
```

---

## 91.3 API Gateway Advanced Patterns

```kotlin
// Spring Cloud Gateway configuration

@Configuration
class GatewayConfig {
    
    @Bean
    fun routeLocator(builder: RouteLocatorBuilder): RouteLocator =
        builder.routes()
            // ====== Route with filters ======
            .route("order-service") { r ->
                r.path("/api/v*/orders/**")
                    .filters { f ->
                        f
                            // Rate limiting per user
                            .requestRateLimiter { config ->
                                config.setRateLimiter(redisRateLimiter())
                                config.setKeyResolver { exchange ->
                                    exchange.request.headers.getFirst("Authorization")
                                        ?.removePrefix("Bearer ")
                                        ?.let { token -> Mono.just(extractUserId(token)) }
                                        ?: Mono.just(exchange.request.remoteAddress?.hostString ?: "anonymous")
                                }
                            }
                            // Circuit breaker
                            .circuitBreaker { config ->
                                config.setName("order-service-cb")
                                config.setFallbackUri("forward:/fallback/orders")
                            }
                            // Retry on 503
                            .retry { config ->
                                config.setRetries(2)
                                config.setStatuses(HttpStatus.SERVICE_UNAVAILABLE)
                                config.setBackoff(Duration.ofMillis(100), Duration.ofSeconds(2), 2, false)
                            }
                            // Add request headers
                            .addRequestHeader("X-Gateway-Version", "1.0")
                            // Remove sensitive response headers
                            .removeResponseHeader("Server")
                            .removeResponseHeader("X-Powered-By")
                    }
                    .uri("lb://order-service")  // load balanced
            }
            
            // ====== API Versioning with path rewrite ======
            .route("product-service-v1") { r ->
                r.path("/api/v1/products/**")
                    .filters { f ->
                        // Rewrite /api/v1/products → /products
                        f.rewritePath("/api/v1/(?<segment>.*)", "/\${segment}")
                    }
                    .uri("lb://product-service")
            }
            
            // ====== Canary deployment ======
            .route("product-service-canary") { r ->
                r.path("/api/v2/products/**")
                    .and()
                    .weight("product-service-v2-group", 10)  // 10% traffic to canary
                    .uri("lb://product-service-v2")
            }
            .route("product-service-stable") { r ->
                r.path("/api/v2/products/**")
                    .and()
                    .weight("product-service-v2-group", 90)  // 90% to stable
                    .uri("lb://product-service")
            }
            
            .build()
    
    @Bean
    fun redisRateLimiter(): RedisRateLimiter =
        RedisRateLimiter(10, 20, 1)  // 10 requests/sec, burst 20
    
    // Fallback controller
    @RestController
    class FallbackController {
        @GetMapping("/fallback/orders")
        fun ordersFallback(): ResponseEntity<Map<String, String>> =
            ResponseEntity.status(503).body(mapOf(
                "message" to "Order service is temporarily unavailable",
                "code" to "SERVICE_UNAVAILABLE"
            ))
    }
}
```

---

## 91.4 Bulkhead Pattern

```kotlin
// Bulkhead: isolate resources for different operations
// Prevents one slow operation from consuming all threads

// ====== Thread Pool Bulkhead (Resilience4j) ======
@Configuration
class BulkheadConfig {
    
    @Bean
    fun threadPoolBulkheadRegistry(): ThreadPoolBulkheadRegistry {
        val config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(5)
            .queueCapacity(20)
            .keepAliveTime(Duration.ofMillis(500))
            .build()
        
        return ThreadPoolBulkheadRegistry.of(mapOf(
            "payment" to ThreadPoolBulkheadConfig.custom()
                .maxThreadPoolSize(5)     // limit payment threads
                .coreThreadPoolSize(3)
                .queueCapacity(10)
                .build(),
            "product-search" to config,
            "recommendations" to ThreadPoolBulkheadConfig.custom()
                .maxThreadPoolSize(3)     // recommendations get fewer resources
                .coreThreadPoolSize(2)
                .queueCapacity(5)
                .build()
        ))
    }
}

@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val recommendationEngine: RecommendationEngine,
    threadPoolBulkheadRegistry: ThreadPoolBulkheadRegistry
) {
    private val searchBulkhead = threadPoolBulkheadRegistry.bulkhead("product-search")
    private val recommendBulkhead = threadPoolBulkheadRegistry.bulkhead("recommendations")
    
    fun searchProducts(query: String): List<Product> {
        // Runs in isolated thread pool
        return ThreadPoolBulkhead.executeSupplier(searchBulkhead) {
            productRepository.search(query)
        }.toCompletableFuture().get(5, TimeUnit.SECONDS)
    }
    
    fun getRecommendations(userId: String): List<Product> {
        // If recommendation pool is full, fall back immediately
        return try {
            ThreadPoolBulkhead.executeSupplier(recommendBulkhead) {
                recommendationEngine.getForUser(userId)
            }.toCompletableFuture().get(2, TimeUnit.SECONDS)
        } catch (e: BulkheadFullException) {
            // Fallback: return popular products
            productRepository.findTopRated(10)
        } catch (e: TimeoutException) {
            productRepository.findTopRated(10)
        }
    }
}

// ====== Semaphore Bulkhead ======
// Limit concurrent access without separate thread pool
@Service
class ExternalApiService(bulkheadRegistry: BulkheadRegistry) {
    
    private val googleMapsApi = bulkheadRegistry.bulkhead("google-maps-api",
        BulkheadConfig.custom()
            .maxConcurrentCalls(20)     // max 20 concurrent calls
            .maxWaitDuration(Duration.ofMillis(100))  // wait 100ms max
            .build()
    )
    
    fun getDistanceMatrix(origins: List<String>, destinations: List<String>): DistanceMatrix {
        return Bulkhead.executeSupplier(googleMapsApi) {
            // This limits how many concurrent Google Maps API calls we make
            googleMapsClient.getDistanceMatrix(origins, destinations)
        }
    }
}
```

---

## 91.5 Strangler Fig Pattern

```kotlin
// Strangler Fig: gradually migrate from monolith to microservices
// Intercept traffic at the edge → route old/new paths

// Phase 1: Route products to new service, everything else to monolith
@Configuration
class StranglerFigGateway {
    
    @Bean
    fun stranglerRoutes(builder: RouteLocatorBuilder): RouteLocator =
        builder.routes()
            // New microservice handles products
            .route("new-product-service") { r ->
                r.path("/api/products/**", "/api/categories/**")
                    .uri("lb://product-microservice")
            }
            // New microservice handles orders
            .route("new-order-service") { r ->
                r.path("/api/orders/**", "/api/cart/**")
                    .uri("lb://order-microservice")
            }
            // Everything else still goes to monolith
            .route("legacy-monolith") { r ->
                r.path("/**")
                    .uri("http://legacy-app:8080")
            }
            .build()
}

// Anti-Corruption Layer: translate between old and new models
@Component
class LegacyProductAdapter(
    private val legacyHttpClient: RestTemplate
) : ProductRepository {  // implements new domain port
    
    override fun findById(id: ProductId): Product? {
        val legacyProduct = try {
            legacyHttpClient.getForObject(
                "/legacy/api/items/${id.value}",
                LegacyProductDto::class.java
            )
        } catch (e: HttpClientErrorException.NotFound) {
            return null
        }
        
        // Translate legacy model to new domain model
        return legacyProduct?.let { legacy ->
            Product(
                id = ProductId(legacy.itemCode),
                name = legacy.itemName,
                description = legacy.longDescription ?: legacy.shortDescription,
                price = Money(legacy.priceInSatang),  // legacy uses satang (Thai)
                imageUrl = "${LEGACY_CDN_URL}/${legacy.imagePath}",
                category = translateCategory(legacy.categoryCode),
                inStock = legacy.stockQty > 0
            )
        }
    }
    
    private fun translateCategory(legacyCode: String): String = when (legacyCode) {
        "ELEC" -> "electronics"
        "CLTH" -> "clothing"
        "FOOD" -> "food"
        else -> "other"
    }
}
```

---

## 91.6 Service Discovery (Consul)

```yaml
# application.yaml
spring:
  cloud:
    consul:
      host: consul
      port: 8500
      discovery:
        service-name: order-service
        instance-id: ${spring.application.name}:${random.uuid}
        health-check-path: /actuator/health
        health-check-interval: 10s
        prefer-ip-address: true
        tags:
          - version=2.1.0
          - env=production
          - region=ap-southeast-1
```

```kotlin
// Service Discovery client
@Service
class ServiceDiscoveryClient(
    private val discoveryClient: DiscoveryClient,
    private val loadBalancerClient: LoadBalancerClient
) {
    
    // List all instances of a service
    fun getInstances(serviceId: String): List<ServiceInstance> =
        discoveryClient.getInstances(serviceId)
    
    // Load-balanced URL for a service
    fun getServiceUrl(serviceId: String): String? =
        loadBalancerClient.choose(serviceId)?.let { instance ->
            "http://${instance.host}:${instance.port}"
        }
    
    // Health check across all instances
    fun checkServiceHealth(serviceId: String): Map<String, Boolean> {
        return getInstances(serviceId).associate { instance ->
            val id = "${instance.host}:${instance.port}"
            id to (instance.metadata["status"] == "UP")
        }
    }
}

// Using @LoadBalanced RestTemplate
@Configuration
class LoadBalancedConfig {
    
    @Bean
    @LoadBalanced  // Intercepts http://service-name/ → actual IP:port
    fun loadBalancedRestTemplate(): RestTemplate = RestTemplate()
}

@Service
class OrderServiceClient(
    @LoadBalanced private val restTemplate: RestTemplate
) {
    fun getOrder(orderId: String): Order {
        // Spring resolves "order-service" via Consul/Eureka
        return restTemplate.getForObject(
            "http://order-service/api/orders/$orderId",
            Order::class.java
        )!!
    }
}
```

---

## สรุป Part 91

```
Advanced Microservices Patterns:

BFF (Backend for Frontend):
  Separate API layer per client type
  Web BFF: full data, SEO metadata
  Mobile BFF: minimal data, aggregated calls
  Partner BFF: rate-limited, documented API

API Gateway:
  Rate limiting: per user, per IP
  Circuit breaker: fallback on downstream failure
  Canary: weight-based traffic splitting
  Path rewrite: /api/v1/products → /products

Bulkhead:
  Thread pool: isolate thread pools per operation
  Semaphore: limit concurrent access
  Prevents: one slow service consuming all threads
  
  Use: separate pools for critical vs non-critical ops
  payment (critical, 5 threads) vs recommendations (3 threads)

Strangler Fig:
  Gradually move traffic from monolith to services
  API Gateway routes to new service
  Unmigrated paths → legacy
  Anti-Corruption Layer: translate models
  
Service Discovery:
  Register on startup, deregister on shutdown
  Health check every N seconds
  @LoadBalanced: auto-resolve service name → IP
  Consul: DNS + HTTP API for discovery

When to split a monolith:
  ✓ Independent scaling needed
  ✓ Different deployment cycles
  ✓ Team autonomy needed
  ✗ Don't split just because microservices are trendy
  Start with: monolith → modular monolith → selective extraction
```

➡️ [Part 92: Advanced Spring Boot Configuration](./Part-92-SpringConfig.md)
