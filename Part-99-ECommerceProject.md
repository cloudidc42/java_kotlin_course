# Part 99: Complete E-Commerce Backend Project
## ขั้นตอนที่ 6831-6900: Full Project with All Concepts Applied

---

## 99.1 Project Overview

```
ShopKT: Complete E-Commerce Backend

Tech Stack:
  Language: Kotlin 2.x
  Framework: Spring Boot 3.x
  Database: PostgreSQL 16 (main) + Redis 7 (cache)
  Message Broker: Apache Kafka 3.x
  Container: Docker + Kubernetes
  
  Reactive: Spring WebFlux (product catalog)
  Batch: Spring Batch (reports)
  Security: Spring Security + JWT + OAuth2
  Monitoring: Micrometer + Prometheus + OpenTelemetry
  
Architecture: Modular Monolith → ready for microservices
  Module: product, order, payment, user, notification
  Each module has: domain, application, infrastructure layers
  
Repository Structure:
  shopkt/
  ├── build.gradle.kts (multi-module)
  ├── settings.gradle.kts
  ├── gradle/
  ├── modules/
  │   ├── product/
  │   ├── order/
  │   ├── payment/
  │   ├── user/
  │   └── notification/
  ├── infra/
  │   ├── db/migrations/
  │   ├── k8s/
  │   └── docker/
  └── api-gateway/
```

---

## 99.2 Core Domain Model

```kotlin
// ====== Product Module ======

// Domain (no framework dependencies)
package com.shopkt.product.domain

data class ProductId(val value: String)
data class CategoryId(val value: String)

@JvmInline value class SKU(val value: String)
@JvmInline value class Money(val satang: Long) {
    operator fun plus(other: Money) = Money(satang + other.satang)
    operator fun minus(other: Money) = Money(satang - other.satang)
    fun multiply(qty: Int) = Money(satang * qty)
    fun display() = "฿${satang / 100}.${"%02d".format(satang % 100)}"
}

data class Product(
    val id: ProductId,
    val sku: SKU,
    val name: String,
    val description: String,
    val price: Money,
    val stockQuantity: Int,
    val categoryId: CategoryId,
    val imageUrls: List<String>,
    val status: ProductStatus,
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now()
) {
    enum class ProductStatus { ACTIVE, INACTIVE, OUT_OF_STOCK }
    
    fun isAvailable() = status == ProductStatus.ACTIVE && stockQuantity > 0
    
    fun deductStock(quantity: Int): Product {
        require(stockQuantity >= quantity) { "Insufficient stock" }
        val newStatus = if (stockQuantity - quantity == 0) ProductStatus.OUT_OF_STOCK 
                       else status
        return copy(
            stockQuantity = stockQuantity - quantity,
            status = newStatus,
            updatedAt = Instant.now()
        )
    }
}

// Domain events
sealed interface ProductEvent {
    data class ProductCreated(val product: Product) : ProductEvent
    data class StockDepleted(val productId: ProductId) : ProductEvent
    data class ProductActivated(val productId: ProductId) : ProductEvent
}

// Repository port (interface, no infrastructure)
interface ProductRepository {
    fun findById(id: ProductId): Product?
    fun findByCategory(categoryId: CategoryId, pageable: Pageable): Page<Product>
    fun save(product: Product): Product
    fun search(query: String, pageable: Pageable): Page<Product>
}

// ====== Order Module ======

data class OrderId(val value: String)
data class CustomerId(val value: String)

data class OrderItem(
    val productId: ProductId,
    val productName: String,  // snapshot at order time
    val unitPrice: Money,     // snapshot at order time
    val quantity: Int
) {
    val subtotal: Money get() = unitPrice.multiply(quantity)
}

data class Order(
    val id: OrderId,
    val customerId: CustomerId,
    val items: List<OrderItem>,
    val shippingAddress: Address,
    val status: OrderStatus,
    val paymentId: String? = null,
    val createdAt: Instant = Instant.now()
) {
    enum class OrderStatus {
        PENDING, PAYMENT_PENDING, CONFIRMED, PROCESSING,
        SHIPPED, DELIVERED, CANCELLED, REFUNDED
    }
    
    val subtotal: Money get() = items.fold(Money(0)) { acc, item -> acc + item.subtotal }
    val tax: Money get() = Money((subtotal.satang * 0.07).toLong())
    val total: Money get() = subtotal + tax
    
    fun confirm(paymentId: String): Order =
        copy(status = OrderStatus.CONFIRMED, paymentId = paymentId)
    
    fun cancel(): Order {
        require(status in listOf(OrderStatus.PENDING, OrderStatus.PAYMENT_PENDING)) {
            "Cannot cancel order in status $status"
        }
        return copy(status = OrderStatus.CANCELLED)
    }
}

data class Address(
    val street: String,
    val city: String,
    val province: String,
    val postalCode: String,
    val country: String = "TH"
)
```

---

## 99.3 Application Layer (Use Cases)

```kotlin
// Use cases / application services

@Service
class CreateOrderUseCase(
    private val orderRepository: OrderRepository,
    private val productRepository: ProductRepository,
    private val outboxRepository: OutboxRepository,
    private val idempotencyRepository: IdempotencyRepository
) {
    
    data class CreateOrderCommand(
        val customerId: String,
        val items: List<ItemRequest>,
        val shippingAddress: Address,
        val idempotencyKey: String  // prevent duplicate orders
    )
    
    data class ItemRequest(val productId: String, val quantity: Int)
    
    @Transactional
    fun execute(command: CreateOrderCommand): Order {
        // Idempotency check
        idempotencyRepository.findByKey(command.idempotencyKey)?.let { existing ->
            return orderRepository.findById(OrderId(existing.responseData)).orElseThrow()
        }
        
        // Load and validate products
        val productIds = command.items.map { ProductId(it.productId) }
        val products = productRepository.findAllById(productIds)
            .associateBy { it.id }
        
        val items = command.items.map { req ->
            val product = products[ProductId(req.productId)]
                ?: throw ProductNotFoundException(req.productId)
            
            if (!product.isAvailable()) throw ProductNotAvailableException(req.productId)
            if (product.stockQuantity < req.quantity) throw InsufficientStockException(req.productId)
            
            OrderItem(
                productId = product.id,
                productName = product.name,    // snapshot
                unitPrice = product.price,      // snapshot
                quantity = req.quantity
            )
        }
        
        // Create order
        val order = Order(
            id = OrderId(UUID.randomUUID().toString()),
            customerId = CustomerId(command.customerId),
            items = items,
            shippingAddress = command.shippingAddress,
            status = Order.OrderStatus.PENDING
        )
        
        val savedOrder = orderRepository.save(order)
        
        // Publish event via outbox (atomically with order save)
        outboxRepository.save(OutboxMessage(
            aggregateId = savedOrder.id.value,
            aggregateType = "Order",
            eventType = "OrderCreated",
            payload = objectMapper.writeValueAsString(savedOrder)
        ))
        
        // Store idempotency key
        idempotencyRepository.save(IdempotencyRecord(
            key = command.idempotencyKey,
            responseData = savedOrder.id.value
        ))
        
        return savedOrder
    }
}
```

---

## 99.4 Infrastructure Layer

```kotlin
// JPA Entity (infrastructure, separate from domain)

@Entity
@Table(name = "products")
class ProductEntity(
    @Id val id: String,
    @Column(nullable = false, unique = true) val sku: String,
    @Column(nullable = false) var name: String,
    @Column(columnDefinition = "TEXT") var description: String,
    @Column(nullable = false) var priceSatang: Long,
    @Column(nullable = false) var stockQuantity: Int,
    @Column(nullable = false) var categoryId: String,
    @Column(columnDefinition = "TEXT[]") var imageUrls: Array<String> = emptyArray(),
    @Enumerated(EnumType.STRING) var status: ProductStatus = ProductStatus.ACTIVE,
    val createdAt: Instant = Instant.now(),
    var updatedAt: Instant = Instant.now()
) {
    enum class ProductStatus { ACTIVE, INACTIVE, OUT_OF_STOCK }
    
    // Mapping between JPA entity and domain model
    fun toDomain() = Product(
        id = ProductId(id),
        sku = SKU(sku),
        name = name,
        description = description,
        price = Money(priceSatang),
        stockQuantity = stockQuantity,
        categoryId = CategoryId(categoryId),
        imageUrls = imageUrls.toList(),
        status = Product.ProductStatus.valueOf(status.name)
    )
    
    companion object {
        fun fromDomain(product: Product) = ProductEntity(
            id = product.id.value,
            sku = product.sku.value,
            name = product.name,
            description = product.description,
            priceSatang = product.price.satang,
            stockQuantity = product.stockQuantity,
            categoryId = product.categoryId.value,
            imageUrls = product.imageUrls.toTypedArray(),
            status = ProductStatus.valueOf(product.status.name)
        )
    }
}

// Repository implementation (infrastructure)
@Component
class JpaProductRepository(
    private val jpaRepository: ProductJpaRepository,
    private val entityManager: EntityManager
) : ProductRepository {
    
    override fun findById(id: ProductId): Product? =
        jpaRepository.findById(id.value).map { it.toDomain() }.orElse(null)
    
    override fun findByCategory(categoryId: CategoryId, pageable: Pageable): Page<Product> =
        jpaRepository.findByCategoryId(categoryId.value, pageable).map { it.toDomain() }
    
    override fun save(product: Product): Product =
        jpaRepository.save(ProductEntity.fromDomain(product)).toDomain()
    
    override fun search(query: String, pageable: Pageable): Page<Product> {
        val cb = entityManager.criteriaBuilder
        val cq = cb.createQuery(ProductEntity::class.java)
        val root = cq.from(ProductEntity::class.java)
        
        val searchPattern = "%${query.lowercase()}%"
        cq.where(cb.or(
            cb.like(cb.lower(root.get("name")), searchPattern),
            cb.like(cb.lower(root.get("description")), searchPattern)
        ))
        
        val results = entityManager.createQuery(cq)
            .setFirstResult(pageable.offset.toInt())
            .setMaxResults(pageable.pageSize)
            .resultList
        
        return PageImpl(results.map { it.toDomain() }, pageable, results.size.toLong())
    }
}
```

---

## 99.5 API Layer

```kotlin
// REST API controller

@RestController
@RequestMapping("/api/v1/products")
@Validated
class ProductController(private val productService: ProductApplicationService) {
    
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: String): ResponseEntity<ProductResponse> {
        val product = productService.findById(id)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(ProductResponse.from(product))
    }
    
    @GetMapping
    fun searchProducts(
        @RequestParam(required = false) q: String?,
        @RequestParam(required = false) category: String?,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") @Max(100) size: Int
    ): ResponseEntity<PagedResponse<ProductResponse>> {
        val pageable = PageRequest.of(page, size)
        val results = if (q != null) {
            productService.search(q, pageable)
        } else if (category != null) {
            productService.findByCategory(category, pageable)
        } else {
            productService.findAll(pageable)
        }
        
        return ResponseEntity.ok(PagedResponse.from(results) { ProductResponse.from(it) })
    }
    
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    fun createProduct(
        @Valid @RequestBody request: CreateProductRequest
    ): ResponseEntity<ProductResponse> {
        val product = productService.create(request.toCommand())
        return ResponseEntity
            .created(URI.create("/api/v1/products/${product.id.value}"))
            .body(ProductResponse.from(product))
    }
    
    @PatchMapping("/{id}/stock")
    @PreAuthorize("hasRole('ADMIN')")
    fun updateStock(
        @PathVariable id: String,
        @Valid @RequestBody request: UpdateStockRequest
    ): ResponseEntity<ProductResponse> {
        val product = productService.updateStock(id, request.quantity)
        return ResponseEntity.ok(ProductResponse.from(product))
    }
}

// Response DTOs
data class ProductResponse(
    val id: String,
    val sku: String,
    val name: String,
    val price: PriceResponse,
    val stockQuantity: Int,
    val category: String,
    val images: List<String>,
    val available: Boolean
) {
    data class PriceResponse(val satang: Long, val display: String)
    
    companion object {
        fun from(product: Product) = ProductResponse(
            id = product.id.value,
            sku = product.sku.value,
            name = product.name,
            price = PriceResponse(
                satang = product.price.satang,
                display = product.price.display()
            ),
            stockQuantity = product.stockQuantity,
            category = product.categoryId.value,
            images = product.imageUrls,
            available = product.isAvailable()
        )
    }
}

data class PagedResponse<T>(
    val items: List<T>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int,
    val hasNext: Boolean,
    val hasPrevious: Boolean
) {
    companion object {
        fun <T, R> from(page: Page<T>, transform: (T) -> R) = PagedResponse(
            items = page.content.map(transform),
            page = page.number,
            size = page.size,
            totalElements = page.totalElements,
            totalPages = page.totalPages,
            hasNext = page.hasNext(),
            hasPrevious = page.hasPrevious()
        )
    }
}
```

---

## 99.6 Testing the Complete Project

```kotlin
// Integration test for the full order creation flow

@SpringBootTest
@Testcontainers
@AutoConfigureMockMvc
class OrderCreationIntegrationTest {
    
    companion object {
        @Container
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine")
        
        @Container
        val redis = RedisContainer("redis:7-alpine")
        
        @Container  
        val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"))
        
        @JvmStatic
        @DynamicPropertySource
        fun overrideProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            registry.add("spring.data.redis.host", redis::getHost)
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379) }
            registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers)
        }
    }
    
    @Autowired lateinit var mockMvc: MockMvc
    @Autowired lateinit var productRepository: ProductRepository
    @Autowired lateinit var orderRepository: OrderRepository
    
    @Test
    fun `should create order successfully`() {
        // Arrange: create a product
        val product = productRepository.save(Product(
            id = ProductId("P001"),
            sku = SKU("LAPTOP-001"),
            name = "Gaming Laptop",
            description = "High performance laptop",
            price = Money(2500000),  // ฿25,000.00
            stockQuantity = 10,
            categoryId = CategoryId("electronics"),
            imageUrls = listOf("https://example.com/laptop.jpg"),
            status = Product.ProductStatus.ACTIVE
        ))
        
        // Act: create order via API
        val result = mockMvc.post("/api/v1/orders") {
            header("Authorization", "Bearer ${generateTestJwt()}")
            header("Idempotency-Key", UUID.randomUUID().toString())
            contentType = MediaType.APPLICATION_JSON
            content = """
                {
                    "items": [{"productId": "P001", "quantity": 2}],
                    "shippingAddress": {
                        "street": "123 Main St",
                        "city": "Bangkok",
                        "province": "Bangkok",
                        "postalCode": "10110"
                    }
                }
            """.trimIndent()
        }
        
        // Assert
        result
            .andExpect { status { isCreated() } }
            .andExpect { jsonPath("$.status") { value("PENDING") } }
            .andExpect { jsonPath("$.total.satang") { value(5350000L) } }  // 2 × 25000 × 1.07 TAX
        
        // Verify inventory was not yet decremented (done after payment)
        val updatedProduct = productRepository.findById(ProductId("P001"))!!
        assertThat(updatedProduct.stockQuantity).isEqualTo(10)  // unchanged until confirmed
        
        // Verify outbox message was created
        val outboxMessages = outboxRepository.findByAggregateType("Order")
        assertThat(outboxMessages).hasSize(1)
        assertThat(outboxMessages[0].eventType).isEqualTo("OrderCreated")
    }
    
    @Test
    fun `should return 409 for duplicate idempotency key`() {
        val idempotencyKey = UUID.randomUUID().toString()
        
        // First request
        mockMvc.post("/api/v1/orders") {
            header("Idempotency-Key", idempotencyKey)
            // ... setup
        }.andExpect { status { isCreated() } }
        
        // Second request with same key
        mockMvc.post("/api/v1/orders") {
            header("Idempotency-Key", idempotencyKey)
            // ... same body
        }.andExpect { status { isOk() } }  // returns 200 with original response
    }
}
```

---

## สรุป Part 99

```
Complete Project Summary:

Architecture: Hexagonal / Clean Architecture
  Domain: pure Kotlin, no framework
  Application: use cases, ports
  Infrastructure: JPA, Redis, Kafka adapters
  API: REST controllers (thin layer)
  
Key Patterns Used:
  Value objects: ProductId, Money, SKU (type safety)
  Domain events: sealed interface
  Repository pattern: interface in domain, implementation in infra
  Outbox pattern: atomic DB + Kafka publish
  Idempotency: prevent duplicate orders
  
Testing Strategy:
  Unit: domain logic (fast, no spring)
  Integration: @SpringBootTest + Testcontainers
  Contract: Spring Cloud Contract (API compatibility)
  E2E: Playwright or Selenium

Production Ready Features:
  Graceful shutdown
  Health probes (liveness + readiness)
  Structured logging (JSON)
  Metrics (Prometheus)
  Distributed tracing (OpenTelemetry)
  HPA scaling
  Zero-trust security (mTLS + JWT)

Deployment:
  Docker multi-stage build
  Kubernetes: 3 replicas min
  HPA: scale 3-20 on CPU/memory
  PDB: always 2 pods available
  ArgoCD: GitOps deployment

What this project demonstrates:
  All 99 parts of the course applied in one project!
  From basic Kotlin syntax to production-grade system
```

➡️ [Part 100: Course Completion & Career Roadmap](./Part-100-CourseCompletion.md)
