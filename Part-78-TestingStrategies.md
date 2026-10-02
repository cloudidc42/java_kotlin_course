# Part 78: Testing Strategies (Comprehensive)
## ขั้นตอนที่ 5361-5430: Unit, Integration, Contract, E2E, Performance Tests

---

## 78.1 Testing Pyramid

```
                    /\
                   /  \
                  / E2E \     ← slowest, most expensive, least tests
                 /--------\
                /Integration\  ← medium: test real components together
               /--------------\
              /   Unit Tests    \ ← fastest, cheapest, most tests
             /------------------\

Testing Pyramid Ratio (rough):
  Unit Tests:        70%   (milliseconds)
  Integration Tests: 20%   (seconds)
  E2E Tests:         10%   (minutes)

Test Categories:
  Unit        = single class/function, mock everything else
  Integration = multiple real components (DB, cache, etc.)
  Contract    = API contract between services
  Performance = load, stress, soak testing
  E2E         = full user journey through the system
```

---

## 78.2 Unit Testing with MockK (Kotlin)

```kotlin
import io.mockk.*
import io.mockk.impl.annotations.*
import org.junit.jupiter.api.*
import org.assertj.core.api.*

@ExtendWith(MockKExtension::class)
class OrderServiceTest {
    
    @MockK
    private lateinit var orderRepository: OrderRepository
    
    @MockK
    private lateinit var paymentGateway: PaymentGateway
    
    @MockK
    private lateinit var notificationSender: NotificationSender
    
    @InjectMockKs
    private lateinit var orderService: OrderService
    
    @BeforeEach
    fun setUp() {
        // MockK automatically injects mocks via @InjectMockKs
    }
    
    @Test
    fun `createOrder should save and return new order`() {
        // Given
        val customerId = "customer-1"
        val expectedOrder = Order(
            id = "order-1",
            customerId = customerId,
            items = emptyList(),
            status = OrderStatus.DRAFT,
            createdAt = java.time.Instant.now()
        )
        
        every { orderRepository.save(any()) } returns expectedOrder
        
        // When
        val result = orderService.createOrder(customerId)
        
        // Then
        assertThat(result.customerId).isEqualTo(customerId)
        assertThat(result.status).isEqualTo(OrderStatus.DRAFT)
        assertThat(result.items).isEmpty()
        
        verify(exactly = 1) { orderRepository.save(any()) }
    }
    
    @Test
    fun `submitOrder should charge payment and confirm order`() {
        // Given
        val orderId = "order-1"
        val order = createTestOrder(orderId, OrderStatus.DRAFT,
            items = listOf(createOrderItem("P1", 10000L, 2)))
        
        every { orderRepository.findById(orderId) } returns order
        every { orderRepository.save(any()) } answers { firstArg() }
        every { paymentGateway.charge(any(), any(), any()) } returns 
            PaymentResult.Success("payment-1", "txn-1")
        every { notificationSender.sendOrderConfirmation(any(), any()) } just runs
        
        // When
        val result = orderService.submitOrder(orderId)
        
        // Then
        assertThat(result.status).isEqualTo(OrderStatus.CONFIRMED)
        
        verify { paymentGateway.charge(
            customerId = order.customerId,
            amount = any(),
            reference = orderId
        ) }
        verify { notificationSender.sendOrderConfirmation(any(), any()) }
    }
    
    @Test
    fun `submitOrder should cancel when payment fails`() {
        val orderId = "order-1"
        val order = createTestOrder(orderId, OrderStatus.DRAFT,
            items = listOf(createOrderItem("P1", 10000L, 1)))
        
        every { orderRepository.findById(orderId) } returns order
        every { orderRepository.save(any()) } answers { firstArg() }
        every { paymentGateway.charge(any(), any(), any()) } returns
            PaymentResult.Failure("Insufficient funds", "INSUFFICIENT_FUNDS")
        
        // When
        val result = orderService.submitOrder(orderId)
        
        // Then
        assertThat(result.status).isEqualTo(OrderStatus.CANCELLED)
        verify(exactly = 0) { notificationSender.sendOrderConfirmation(any(), any()) }
    }
    
    @Test
    fun `submitOrder should throw when order not found`() {
        every { orderRepository.findById("nonexistent") } returns null
        
        assertThatThrownBy { orderService.submitOrder("nonexistent") }
            .isInstanceOf(OrderNotFoundException::class.java)
    }
}
```

---

## 78.3 Integration Testing with Testcontainers

```kotlin
import org.testcontainers.containers.*
import org.testcontainers.junit.jupiter.*
import org.springframework.boot.test.*
import org.springframework.beans.factory.annotation.*

@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryIntegrationTest {
    
    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:15-alpine").apply {
            withDatabaseName("testdb")
            withUsername("test")
            withPassword("test")
            withInitScript("db/init.sql")
        }
        
        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
    
    @Autowired
    private lateinit var orderRepository: OrderRepository
    
    @Autowired
    private lateinit var jdbcTemplate: JdbcTemplate
    
    @BeforeEach
    fun setUp() {
        jdbcTemplate.execute("DELETE FROM orders")
        jdbcTemplate.execute("DELETE FROM order_items")
    }
    
    @Test
    fun `should save and retrieve order`() {
        val order = Order(
            id = "order-1",
            customerId = "customer-1",
            items = listOf(OrderItem("P1", "Product 1", Money.of(100.0), 2)),
            status = OrderStatus.DRAFT,
            createdAt = java.time.Instant.now()
        )
        
        orderRepository.save(order)
        
        val retrieved = orderRepository.findById("order-1")
        
        assertThat(retrieved).isNotNull
        assertThat(retrieved!!.customerId).isEqualTo("customer-1")
        assertThat(retrieved.items).hasSize(1)
        assertThat(retrieved.total).isEqualTo(Money.of(200.0))
    }
    
    @Test
    fun `should find orders by customer with pagination`() {
        // Create 15 orders for same customer
        repeat(15) { i ->
            orderRepository.save(createOrder("order-$i", "customer-1"))
        }
        
        val page = orderRepository.findByCustomerId("customer-1", PageRequest.of(0, 10))
        
        assertThat(page.content).hasSize(10)
        assertThat(page.totalElements).isEqualTo(15)
        assertThat(page.totalPages).isEqualTo(2)
    }
}
```

---

## 78.4 Contract Testing with Spring Cloud Contract

```kotlin
// Producer side: define contracts
// src/test/resources/contracts/order-service/shouldReturnOrder.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return order by id"
    
    request {
        method GET()
        url "/api/orders/order-1" 
        headers {
            contentType(applicationJson())
            header("Authorization", "Bearer test-token")
        }
    }
    
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            id: "order-1",
            customerId: "customer-1",
            status: "DRAFT",
            total: 200.0
        ])
    }
}

// Producer test base class
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
abstract class ContractTestBase {
    
    @Autowired
    protected lateinit var mockMvc: MockMvc
    
    @MockBean
    private lateinit var orderService: OrderService
    
    @BeforeEach
    fun setUp() {
        val order = Order("order-1", "customer-1", emptyList(), 
                         OrderStatus.DRAFT, java.time.Instant.now())
        every { orderService.findById("order-1") } returns order
    }
}

// Consumer side: use generated stubs
@SpringBootTest
@AutoConfigureStubRunner(
    ids = ["com.example:order-service:+:stubs:8080"],
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderClientContractTest {
    
    @Autowired
    private lateinit var orderClient: OrderClient
    
    @Test
    fun `order client should match contract`() {
        val order = orderClient.getOrder("order-1")
        
        assertThat(order.id).isEqualTo("order-1")
        assertThat(order.status).isEqualTo("DRAFT")
    }
}
```

---

## 78.5 Performance Testing with Gatling

```scala
// src/test/scala/simulations/OrderServiceSimulation.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderServiceSimulation extends Simulation {
    
    val httpProtocol = http
        .baseUrl("http://localhost:8080")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer test-token")
    
    // Scenario 1: Browse and create orders
    val browseAndOrder = scenario("Browse and Create Order")
        .exec(
            http("Get Products")
                .get("/api/products")
                .check(status.is(200))
                .check(jsonPath("$[0].id").saveAs("productId"))
        )
        .pause(1)
        .exec(
            http("Create Order")
                .post("/api/orders")
                .body(StringBody("""{}"""))
                .check(status.is(201))
                .check(jsonPath("$.id").saveAs("orderId"))
        )
        .exec(
            http("Add Item")
                .post("/api/orders/#{orderId}/items")
                .body(StringBody("""{"productId": "#{productId}", "quantity": 2}"""))
                .check(status.is(200))
        )
        .exec(
            http("Submit Order")
                .post("/api/orders/#{orderId}/submit")
                .check(status.is(200))
        )
    
    // Load test: 100 concurrent users, ramp up over 30 seconds
    setUp(
        browseAndOrder.inject(
            rampUsers(100).during(30.seconds),
            constantUsersPerSec(50).during(2.minutes)
        )
    )
    .protocols(httpProtocol)
    .assertions(
        global.responseTime.percentile(99).lt(1000),  // P99 < 1s
        global.successfulRequests.percent.gt(99)       // > 99% success
    )
}
```

---

## 78.6 Test Data Builders

```kotlin
// Test fixtures with builder pattern
class OrderBuilder {
    var id: String = "order-${java.util.UUID.randomUUID()}"
    var customerId: String = "customer-1"
    var status: OrderStatus = OrderStatus.DRAFT
    var items: MutableList<OrderItem> = mutableListOf()
    var createdAt: java.time.Instant = java.time.Instant.now()
    
    fun withItem(productId: String = "P1", priceCents: Long = 10000L, qty: Int = 1) = apply {
        items.add(OrderItem(
            productId = productId,
            productName = "Product $productId",
            unitPrice = Money(priceCents),
            quantity = qty
        ))
    }
    
    fun withStatus(status: OrderStatus) = apply { this.status = status }
    fun withCustomer(customerId: String) = apply { this.customerId = customerId }
    
    fun build() = Order(id, customerId, items.toList(), status, createdAt)
}

fun order(block: OrderBuilder.() -> Unit = {}) = OrderBuilder().apply(block).build()

// Usage in tests
@Test
fun testSomething() {
    val draftOrder = order {
        withCustomer("customer-1")
        withItem("P1", 15000L, 2)
        withItem("P2", 9900L, 1)
        withStatus(OrderStatus.DRAFT)
    }
    
    val submittedOrder = order { withStatus(OrderStatus.SUBMITTED) }
}
```

---

## สรุป Part 78

```
Testing Strategy:

Unit Tests (70%):
  MockK for Kotlin: mockk<Interface>()
  every { method() } returns value
  verify { method() } was called
  Fast: < 50ms each, no Spring context

Integration Tests (20%):
  Testcontainers: real PostgreSQL, Redis, Kafka
  @SpringBootTest(webEnvironment = RANDOM_PORT)
  Test real DB queries, cache behavior, event flow
  Slower: 2-30s each

Contract Tests:
  Spring Cloud Contract: producer defines, consumer verifies
  Prevent breaking changes in API contracts
  Run in CI before deploy

Performance Tests (Gatling):
  Load test: expected traffic
  Stress test: 2-3x expected (find breaking point)
  Soak test: sustained load (find memory leaks)
  
  Assertions:
    P99 < 1000ms
    Error rate < 1%
    Throughput > 100 req/s

Test Data:
  Builder pattern for complex objects
  Factories for common fixtures
  Keep tests independent (no shared state)

Coverage Targets:
  Domain/Service: 90%+ (business logic)
  Repository: 80%+ (integration test)
  Controller: 70%+ (happy path + main errors)
  Don't target 100%: diminishing returns
```

➡️ [Part 79: Kotlin for Android Development](./Part-79-AndroidDevelopment.md)
