# Part 74: Hexagonal Architecture (Ports & Adapters)
## ขั้นตอนที่ 5081-5150: Clean Architecture, Domain Isolation, Testability

---

## 74.1 Hexagonal Architecture คืออะไร

```
Hexagonal Architecture (Alistair Cockburn, 2005)
ชื่ออื่น: Ports & Adapters, Clean Architecture

Core Idea:
  Domain Logic ต้องไม่รู้จัก/ขึ้นกับ infrastructure

  ┌──────────────────────────────────────────────┐
  │                  ADAPTERS                    │
  │  ┌──────────────────────────────────────┐   │
  │  │             PORTS                    │   │
  │  │  ┌────────────────────────────┐     │   │
  │  │  │    DOMAIN (Application)    │     │   │
  │  │  │  - Entities                │     │   │
  │  │  │  - Use Cases               │     │   │
  │  │  │  - Domain Services         │     │   │
  │  │  └────────────────────────────┘     │   │
  │  └──────────────────────────────────────┘   │
  └──────────────────────────────────────────────┘

  Inbound Adapters:  REST, gRPC, Message Queue → Domain
  Outbound Adapters: Domain → DB, Cache, Email, External API
  
  Ports = Interfaces (abstractions)
  Adapters = Implementations (infrastructure)

Benefits:
  ✓ Test domain without DB/HTTP
  ✓ Replace infrastructure without changing business logic
  ✓ Clear separation of concerns
```

---

## 74.2 Project Structure

```
src/
├── domain/                          ← ไม่ขึ้นกับอะไรทั้งนั้น
│   ├── model/
│   │   ├── Order.kt
│   │   ├── OrderItem.kt
│   │   └── Money.kt
│   ├── port/
│   │   ├── in/
│   │   │   ├── CreateOrderUseCase.kt    ← inbound port
│   │   │   └── GetOrderUseCase.kt
│   │   └── out/
│   │       ├── OrderRepository.kt       ← outbound port
│   │       ├── PaymentGateway.kt
│   │       └── NotificationSender.kt
│   └── service/
│       └── OrderService.kt             ← implements use cases
│
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   └── OrderController.kt      ← REST adapter (inbound)
│   │   └── messaging/
│   │       └── OrderEventListener.kt   ← Kafka adapter (inbound)
│   └── out/
│       ├── persistence/
│       │   └── JpaOrderRepository.kt   ← JPA adapter (outbound)
│       ├── payment/
│       │   └── StripePaymentAdapter.kt ← Stripe adapter (outbound)
│       └── notification/
│           └── EmailNotificationAdapter.kt
│
└── config/
    └── ApplicationConfig.kt            ← wire everything together
```

---

## 74.3 Domain Model (Pure Kotlin)

```kotlin
// domain/model/Order.kt
// ไม่ import Spring, JPA, หรือ library ใดๆ

data class Order(
    val id: OrderId,
    val customerId: CustomerId,
    val items: List<OrderItem>,
    val status: OrderStatus,
    val createdAt: java.time.Instant
) {
    // Domain business rules live here
    
    val total: Money get() = items.fold(Money.ZERO) { acc, item -> acc + item.subtotal }
    
    fun addItem(product: Product, quantity: Int): Order {
        require(quantity > 0) { "Quantity must be positive" }
        require(status == OrderStatus.DRAFT) { "Can only add items to DRAFT orders" }
        
        val existingItem = items.find { it.productId == product.id }
        val updatedItems = if (existingItem != null) {
            items.map { if (it.productId == product.id) it.copy(quantity = it.quantity + quantity) else it }
        } else {
            items + OrderItem(product.id, product.name, product.price, quantity)
        }
        
        return copy(items = updatedItems)
    }
    
    fun submit(): Order {
        require(status == OrderStatus.DRAFT) { "Can only submit DRAFT orders" }
        require(items.isNotEmpty()) { "Cannot submit empty order" }
        require(total.cents > 0) { "Order total must be > 0" }
        
        return copy(status = OrderStatus.SUBMITTED)
    }
    
    fun confirm(): Order {
        require(status == OrderStatus.SUBMITTED) { "Can only confirm SUBMITTED orders" }
        return copy(status = OrderStatus.CONFIRMED)
    }
    
    fun cancel(reason: String): Order {
        require(status in listOf(OrderStatus.DRAFT, OrderStatus.SUBMITTED)) {
            "Cannot cancel order in status: $status"
        }
        return copy(status = OrderStatus.CANCELLED)
    }
}

data class OrderItem(
    val productId: ProductId,
    val productName: String,
    val unitPrice: Money,
    val quantity: Int
) {
    val subtotal: Money get() = unitPrice * quantity
}

enum class OrderStatus { DRAFT, SUBMITTED, CONFIRMED, CANCELLED, COMPLETED }

// Value objects
@JvmInline value class OrderId(val value: String)
@JvmInline value class CustomerId(val value: String)
@JvmInline value class ProductId(val value: String)
```

---

## 74.4 Ports (Interfaces)

```kotlin
// domain/port/in/CreateOrderUseCase.kt

// Inbound Port: แสดง what the application can DO
interface CreateOrderUseCase {
    fun createOrder(command: CreateOrderCommand): Order
    fun addItemToOrder(command: AddItemCommand): Order
    fun submitOrder(orderId: OrderId): Order
}

data class CreateOrderCommand(
    val customerId: CustomerId
)

data class AddItemCommand(
    val orderId: OrderId,
    val productId: ProductId,
    val quantity: Int
)

// domain/port/out/OrderRepository.kt

// Outbound Port: แสดง what the domain NEEDS from infrastructure
interface OrderRepository {
    fun save(order: Order): Order
    fun findById(id: OrderId): Order?
    fun findByCustomerId(customerId: CustomerId): List<Order>
    fun exists(id: OrderId): Boolean
}

// domain/port/out/PaymentGateway.kt
interface PaymentGateway {
    fun charge(customerId: CustomerId, amount: Money, reference: String): PaymentResult
    fun refund(paymentId: String, amount: Money): RefundResult
}

sealed class PaymentResult {
    data class Success(val paymentId: String, val transactionId: String) : PaymentResult()
    data class Failure(val reason: String, val code: String) : PaymentResult()
}

// domain/port/out/NotificationSender.kt
interface NotificationSender {
    fun sendOrderConfirmation(order: Order, customerEmail: String)
    fun sendOrderCancellation(order: Order, reason: String, customerEmail: String)
}
```

---

## 74.5 Application Service (Use Case Implementation)

```kotlin
// domain/service/OrderService.kt

@Service
class OrderService(
    private val orderRepository: OrderRepository,  // outbound port
    private val paymentGateway: PaymentGateway,
    private val notificationSender: NotificationSender,
    private val customerRepository: CustomerRepository
) : CreateOrderUseCase, GetOrderUseCase {
    
    override fun createOrder(command: CreateOrderCommand): Order {
        val order = Order(
            id = OrderId(java.util.UUID.randomUUID().toString()),
            customerId = command.customerId,
            items = emptyList(),
            status = OrderStatus.DRAFT,
            createdAt = java.time.Instant.now()
        )
        return orderRepository.save(order)
    }
    
    override fun addItemToOrder(command: AddItemCommand): Order {
        val order = orderRepository.findById(command.orderId)
            ?: throw OrderNotFoundException(command.orderId)
        
        val product = productRepository.findById(command.productId)
            ?: throw ProductNotFoundException(command.productId)
        
        val updatedOrder = order.addItem(product, command.quantity)
        return orderRepository.save(updatedOrder)
    }
    
    override fun submitOrder(orderId: OrderId): Order {
        val order = orderRepository.findById(orderId)
            ?: throw OrderNotFoundException(orderId)
        
        val customer = customerRepository.findById(order.customerId)
            ?: throw CustomerNotFoundException(order.customerId)
        
        // Domain logic: submit (validates business rules internally)
        val submittedOrder = order.submit()
        
        // Payment
        val paymentResult = paymentGateway.charge(
            order.customerId,
            order.total,
            order.id.value
        )
        
        val finalOrder = when (paymentResult) {
            is PaymentResult.Success -> submittedOrder.confirm()
            is PaymentResult.Failure -> submittedOrder.cancel("Payment failed: ${paymentResult.reason}")
        }
        
        val savedOrder = orderRepository.save(finalOrder)
        
        // Notification (non-critical, don't fail if this fails)
        runCatching {
            notificationSender.sendOrderConfirmation(savedOrder, customer.email)
        }
        
        return savedOrder
    }
}
```

---

## 74.6 Adapters (Infrastructure)

```kotlin
// adapter/in/web/OrderController.kt
@RestController
@RequestMapping("/api/orders")
class OrderController(
    private val createOrderUseCase: CreateOrderUseCase,
    private val getOrderUseCase: GetOrderUseCase
) {
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createOrder(@RequestBody request: CreateOrderRequest,
                    @AuthenticationPrincipal jwt: Jwt): OrderResponse {
        val customerId = CustomerId(jwt.subject)
        val order = createOrderUseCase.createOrder(CreateOrderCommand(customerId))
        return OrderResponse.from(order)
    }
    
    @PostMapping("/{orderId}/items")
    fun addItem(@PathVariable orderId: String,
                @RequestBody request: AddItemRequest): OrderResponse {
        val order = createOrderUseCase.addItemToOrder(AddItemCommand(
            OrderId(orderId), ProductId(request.productId), request.quantity
        ))
        return OrderResponse.from(order)
    }
    
    @PostMapping("/{orderId}/submit")
    fun submit(@PathVariable orderId: String): OrderResponse {
        val order = createOrderUseCase.submitOrder(OrderId(orderId))
        return OrderResponse.from(order)
    }
}

// adapter/out/persistence/JpaOrderRepository.kt
@Repository
class JpaOrderRepository(
    private val jpaRepo: SpringDataOrderRepository
) : OrderRepository {
    
    override fun save(order: Order): Order =
        jpaRepo.save(order.toEntity()).toDomain()
    
    override fun findById(id: OrderId): Order? =
        jpaRepo.findById(id.value).map { it.toDomain() }.orElse(null)
    
    override fun findByCustomerId(customerId: CustomerId): List<Order> =
        jpaRepo.findByCustomerId(customerId.value).map { it.toDomain() }
    
    override fun exists(id: OrderId): Boolean = jpaRepo.existsById(id.value)
}

// adapter/out/payment/StripePaymentAdapter.kt
@Component
class StripePaymentAdapter(
    private val stripeClient: StripeClient
) : PaymentGateway {
    
    override fun charge(customerId: CustomerId, amount: Money, reference: String): PaymentResult {
        return try {
            val charge = stripeClient.charges().create(
                mapOf("amount" to amount.cents, "currency" to "thb",
                      "customer" to customerId.value, "idempotency_key" to reference)
            )
            PaymentResult.Success(charge.id, charge.balanceTransaction)
        } catch (e: com.stripe.exception.CardException) {
            PaymentResult.Failure(e.message ?: "Card error", e.code ?: "CARD_ERROR")
        }
    }
    
    override fun refund(paymentId: String, amount: Money): RefundResult =
        RefundResult.Success("refund-id")
}
```

---

## สรุป Part 74

```
Hexagonal Architecture Summary:

Layers:
  Domain (core)    = Business rules, no dependencies
  Port             = Interface (what we need / what we offer)
  Adapter (outer)  = Implementation (Spring, JPA, HTTP, Kafka)

Dependency Rule:
  Domain ← Adapter (adapters depend on domain, not reverse)
  Domain should compile without Spring/JPA/HTTP

Inbound Ports (Primary):
  CreateOrderUseCase, GetOrderUseCase
  → Implemented by: OrderService (in domain)
  → Called by: REST controller, Kafka listener

Outbound Ports (Secondary):
  OrderRepository, PaymentGateway, NotificationSender
  → Implemented by: JpaOrderRepository, StripeAdapter, EmailAdapter
  → Used by: OrderService (domain)

Testing Benefits:
  Unit test domain: OrderService + mock ports (no Spring context)
  Integration test adapters: test JPA with Testcontainers
  E2E test: full stack

Mock for unit test:
  val orderRepo = mockk<OrderRepository>()
  val payment = mockk<PaymentGateway>()
  val service = OrderService(orderRepo, payment, ...)
  
  // No Spring context needed = fast tests
```

➡️ [Part 75: Event Sourcing & CQRS](./Part-75-EventSourcing.md)
