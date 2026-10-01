# Part 45: Clean Architecture & Domain-Driven Design
## ขั้นตอนที่ 3051-3120: Architecture Principles

---

## 45.1 Clean Architecture

```
Clean Architecture (Robert Martin):

        ┌─────────────────────────────────┐
        │  Frameworks & Drivers           │ ← Web, DB, UI
        │  ┌───────────────────────────┐  │
        │  │  Interface Adapters       │  │ ← Controllers, Repos, Presenters
        │  │  ┌─────────────────────┐  │  │
        │  │  │  Application        │  │  │ ← Use Cases
        │  │  │  ┌───────────────┐  │  │  │
        │  │  │  │  Enterprise   │  │  │  │ ← Entities (Domain)
        │  │  │  │  Business     │  │  │  │
        │  │  │  └───────────────┘  │  │  │
        │  │  └─────────────────────┘  │  │
        │  └───────────────────────────┘  │
        └─────────────────────────────────┘

Dependency Rule: Inner layers know NOTHING about outer layers
  Domain → nothing external
  Use Cases → only Domain
  Adapters → Use Cases + Domain
  Frameworks → Adapters (via interfaces)

Benefits:
  ✓ Testable without DB, UI, or framework
  ✓ Independent of frameworks
  ✓ Independent of database
  ✓ Business rules isolated and protected
```

---

## 45.2 Project Structure

```
src/main/kotlin/com/example/
├── domain/                    ← Enterprise Business Rules
│   ├── model/
│   │   ├── Order.kt           (aggregate root)
│   │   ├── OrderItem.kt       (entity)
│   │   ├── OrderId.kt         (value object)
│   │   ├── Money.kt           (value object)
│   │   └── OrderStatus.kt     (enum)
│   ├── repository/
│   │   └── OrderRepository.kt (interface, no impl)
│   ├── service/
│   │   └── OrderDomainService.kt
│   └── exception/
│       ├── OrderNotFoundException.kt
│       └── InsufficientStockException.kt
│
├── application/               ← Application Business Rules (Use Cases)
│   ├── port/
│   │   ├── input/             (driving ports = what application offers)
│   │   │   ├── CreateOrderUseCase.kt
│   │   │   ├── CancelOrderUseCase.kt
│   │   │   └── GetOrderUseCase.kt
│   │   └── output/            (driven ports = what application needs)
│   │       ├── OrderRepositoryPort.kt
│   │       ├── PaymentPort.kt
│   │       └── NotificationPort.kt
│   └── service/
│       ├── CreateOrderService.kt  (implements input port)
│       └── CancelOrderService.kt
│
├── adapter/                   ← Interface Adapters
│   ├── in/
│   │   ├── web/               (HTTP adapter)
│   │   │   ├── OrderController.kt
│   │   │   └── dto/
│   │   └── messaging/         (Kafka/RabbitMQ adapter)
│   │       └── OrderEventConsumer.kt
│   └── out/
│       ├── persistence/       (DB adapter)
│       │   ├── OrderJpaRepository.kt
│       │   ├── OrderPersistenceAdapter.kt
│       │   └── entity/OrderJpaEntity.kt
│       └── external/          (external service adapter)
│           └── StripePaymentAdapter.kt
│
└── infrastructure/            ← Frameworks & Drivers
    ├── config/
    │   ├── SecurityConfig.kt
    │   └── KafkaConfig.kt
    └── ApplicationMain.kt
```

---

## 45.3 Domain Layer

```kotlin
package com.example.domain.model

import java.time.Instant
import java.util.UUID

// Value Object: immutable, equality by value
@JvmInline
value class OrderId(val value: String) {
    init { require(value.isNotBlank()) { "OrderId cannot be blank" } }
    companion object { fun generate() = OrderId(UUID.randomUUID().toString()) }
}

data class Money(val amount: java.math.BigDecimal, val currency: String = "THB") {
    init { require(amount >= java.math.BigDecimal.ZERO) { "Amount cannot be negative" } }
    
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Currency mismatch: $currency vs ${other.currency}" }
        return copy(amount = amount + other.amount)
    }
    operator fun times(quantity: Int) = copy(amount = amount * java.math.BigDecimal(quantity))
    
    companion object { fun of(amount: Double, currency: String = "THB") = Money(amount.toBigDecimal(), currency) }
}

// Entity: identity by ID
data class OrderItem(
    val id: String = UUID.randomUUID().toString(),
    val productId: String,
    val productName: String,
    val price: Money,
    val quantity: Int
) {
    init { require(quantity > 0) { "Quantity must be positive" } }
    val subtotal: Money get() = price * quantity
}

// Aggregate Root: entry point for the Order aggregate
class Order private constructor(
    val id: OrderId,
    val userId: String,
    private val _items: MutableList<OrderItem> = mutableListOf(),
    var status: OrderStatus = OrderStatus.PENDING,
    val createdAt: Instant = Instant.now()
) {
    val items: List<OrderItem> get() = _items.toList()
    
    val total: Money get() = _items.fold(Money.of(0.0)) { acc, item -> acc + item.subtotal }
    
    // Business rules enforced here
    fun addItem(item: OrderItem) {
        check(status == OrderStatus.PENDING) { "Cannot add items to $status order" }
        _items.add(item)
    }
    
    fun confirm() {
        check(status == OrderStatus.PENDING) { "Cannot confirm $status order" }
        check(_items.isNotEmpty()) { "Cannot confirm empty order" }
        status = OrderStatus.CONFIRMED
    }
    
    fun cancel(reason: String) {
        check(status !in listOf(OrderStatus.DELIVERED, OrderStatus.CANCELLED)) {
            "Cannot cancel $status order"
        }
        status = OrderStatus.CANCELLED
    }
    
    fun ship(trackingNumber: String) {
        check(status == OrderStatus.CONFIRMED) { "Cannot ship $status order" }
        check(trackingNumber.isNotBlank()) { "Tracking number required" }
        status = OrderStatus.SHIPPED
    }
    
    companion object {
        fun create(userId: String): Order {
            require(userId.isNotBlank()) { "UserId required" }
            return Order(id = OrderId.generate(), userId = userId)
        }
        
        // Reconstitute from persistence
        fun restore(
            id: OrderId, userId: String, items: List<OrderItem>,
            status: OrderStatus, createdAt: Instant
        ) = Order(id, userId, items.toMutableList(), status, createdAt)
    }
}

enum class OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
```

---

## 45.4 Application Layer (Use Cases)

```kotlin
package com.example.application

// ====== Input Ports (what the application offers) ======
interface CreateOrderUseCase {
    data class Command(
        val userId: String,
        val items: List<OrderItemCommand>
    )
    data class OrderItemCommand(val productId: String, val quantity: Int)
    
    fun execute(command: Command): OrderDTO
}

interface CancelOrderUseCase {
    data class Command(val orderId: String, val userId: String, val reason: String)
    fun execute(command: Command)
}

// ====== Output Ports (what the application needs) ======
interface OrderRepositoryPort {
    fun save(order: com.example.domain.model.Order): com.example.domain.model.Order
    fun findById(id: com.example.domain.model.OrderId): com.example.domain.model.Order?
    fun findByUserId(userId: String): List<com.example.domain.model.Order>
}

interface ProductCatalogPort {
    data class ProductInfo(val id: String, val name: String, val price: Double, val stock: Int)
    fun getProduct(productId: String): ProductInfo?
}

interface PaymentPort {
    data class PaymentResult(val transactionId: String, val status: String)
    fun charge(orderId: String, amount: Double): PaymentResult
}

interface NotificationPort {
    fun orderConfirmed(userId: String, orderId: String)
    fun orderShipped(userId: String, orderId: String, trackingNumber: String)
}

// ====== Use Case Implementation ======
@org.springframework.stereotype.Service
@org.springframework.transaction.annotation.Transactional
class CreateOrderService(
    private val orderRepository: OrderRepositoryPort,
    private val productCatalog: ProductCatalogPort,
    private val paymentService: PaymentPort,
    private val notificationService: NotificationPort
) : CreateOrderUseCase {
    
    override fun execute(command: CreateOrderUseCase.Command): OrderDTO {
        // 1. Create order aggregate
        val order = com.example.domain.model.Order.create(command.userId)
        
        // 2. Add items (validates stock and gets prices)
        command.items.forEach { itemCmd ->
            val product = productCatalog.getProduct(itemCmd.productId)
                ?: throw RuntimeException("Product not found: ${itemCmd.productId}")
            
            if (product.stock < itemCmd.quantity) {
                throw RuntimeException("Insufficient stock for ${product.name}")
            }
            
            order.addItem(
                com.example.domain.model.OrderItem(
                    productId = product.id,
                    productName = product.name,
                    price = com.example.domain.model.Money.of(product.price),
                    quantity = itemCmd.quantity
                )
            )
        }
        
        // 3. Confirm and save
        order.confirm()
        val saved = orderRepository.save(order)
        
        // 4. Process payment
        val payment = paymentService.charge(saved.id.value, saved.total.amount.toDouble())
        
        // 5. Notify
        notificationService.orderConfirmed(command.userId, saved.id.value)
        
        return saved.toDTO()
    }
}

@org.springframework.stereotype.Service
@org.springframework.transaction.annotation.Transactional
class CancelOrderService(
    private val orderRepository: OrderRepositoryPort,
    private val notificationService: NotificationPort
) : CancelOrderUseCase {
    
    override fun execute(command: CancelOrderUseCase.Command) {
        val order = orderRepository.findById(
            com.example.domain.model.OrderId(command.orderId)
        ) ?: throw RuntimeException("Order not found: ${command.orderId}")
        
        // Authorization: only the order owner can cancel
        require(order.userId == command.userId) { "Not authorized to cancel this order" }
        
        order.cancel(command.reason)
        orderRepository.save(order)
    }
}

data class OrderDTO(
    val id: String, val userId: String, val status: String,
    val total: Double, val itemCount: Int
)

fun com.example.domain.model.Order.toDTO() = OrderDTO(
    id = id.value, userId = userId, status = status.name,
    total = total.amount.toDouble(), itemCount = items.size
)
```

---

## 45.5 Adapter Layer

```kotlin
package com.example.adapter.in.web

import com.example.application.CreateOrderUseCase
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val createOrder: CreateOrderUseCase,
    private val cancelOrder: com.example.application.CancelOrderUseCase
) {
    
    @PostMapping
    fun createOrder(
        @RequestBody request: CreateOrderRequest,
        @org.springframework.security.core.annotation.AuthenticationPrincipal principal: org.springframework.security.core.userdetails.UserDetails
    ): org.springframework.http.ResponseEntity<*> {
        
        val command = CreateOrderUseCase.Command(
            userId = principal.username,
            items = request.items.map {
                CreateOrderUseCase.OrderItemCommand(it.productId, it.quantity)
            }
        )
        
        val result = createOrder.execute(command)
        return org.springframework.http.ResponseEntity.status(201).body(result)
    }
    
    @DeleteMapping("/{id}")
    fun cancelOrder(
        @PathVariable id: String,
        @RequestBody request: CancelOrderRequest,
        @org.springframework.security.core.annotation.AuthenticationPrincipal principal: org.springframework.security.core.userdetails.UserDetails
    ) {
        cancelOrder.execute(
            com.example.application.CancelOrderUseCase.Command(
                orderId = id,
                userId = principal.username,
                reason = request.reason
            )
        )
    }
}

data class CreateOrderRequest(
    val items: List<OrderItemRequest>
)
data class OrderItemRequest(val productId: String, val quantity: Int)
data class CancelOrderRequest(val reason: String)

// ====== Persistence Adapter ======
package com.example.adapter.out.persistence

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Component

@Entity
@Table(name = "orders")
class OrderJpaEntity(
    @Id val id: String,
    val userId: String,
    @Enumerated(EnumType.STRING) var status: com.example.domain.model.OrderStatus,
    val createdAt: java.time.Instant,
    @OneToMany(cascade = [CascadeType.ALL], fetch = FetchType.EAGER)
    @JoinColumn(name = "order_id")
    var items: MutableList<OrderItemJpaEntity> = mutableListOf()
)

@Entity
@Table(name = "order_items")
class OrderItemJpaEntity(
    @Id val id: String,
    val orderId: String,
    val productId: String,
    val productName: String,
    val priceAmount: java.math.BigDecimal,
    val quantity: Int
)

interface OrderJpaRepository : JpaRepository<OrderJpaEntity, String>

@Component
class OrderPersistenceAdapter(
    private val jpaRepository: OrderJpaRepository
) : com.example.application.OrderRepositoryPort {
    
    override fun save(order: com.example.domain.model.Order): com.example.domain.model.Order {
        val entity = order.toEntity()
        jpaRepository.save(entity)
        return order
    }
    
    override fun findById(id: com.example.domain.model.OrderId): com.example.domain.model.Order? {
        return jpaRepository.findById(id.value).orElse(null)?.toDomain()
    }
    
    override fun findByUserId(userId: String): List<com.example.domain.model.Order> {
        return jpaRepository.findAll()
            .filter { it.userId == userId }
            .map { it.toDomain() }
    }
    
    private fun com.example.domain.model.Order.toEntity() = OrderJpaEntity(
        id = id.value, userId = userId, status = status, createdAt = createdAt,
        items = items.map { item ->
            OrderItemJpaEntity(item.id, id.value, item.productId, item.productName,
                item.price.amount, item.quantity)
        }.toMutableList()
    )
    
    private fun OrderJpaEntity.toDomain() = com.example.domain.model.Order.restore(
        id = com.example.domain.model.OrderId(id),
        userId = userId,
        items = items.map { it ->
            com.example.domain.model.OrderItem(
                id = it.id, productId = it.productId, productName = it.productName,
                price = com.example.domain.model.Money(it.priceAmount),
                quantity = it.quantity
            )
        },
        status = status,
        createdAt = createdAt
    )
}
```

---

## สรุป Part 45

```
Clean Architecture Benefits:
  ✓ Domain logic isolated = easy to test (no DB, no HTTP)
  ✓ Can swap DB without changing business rules
  ✓ Use case = single responsibility
  ✓ Dependency inversion: domain doesn't know JPA/Spring

Hexagonal Architecture (Ports & Adapters):
  Driving adapters   = HTTP, CLI, Message Consumer
  Driven adapters    = DB, External API, Email
  Ports              = interfaces between layers

DDD Concepts:
  Aggregate Root     = consistency boundary
  Value Object       = immutable, equality by value
  Entity             = identity by ID
  Repository         = collection abstraction
  Domain Service     = logic spanning multiple aggregates
  Use Case/Service   = orchestrate domain objects
```

➡️ [Part 46: Advanced Testing Strategies](./Part-46-AdvancedTesting.md)
