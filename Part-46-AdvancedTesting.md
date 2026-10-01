# Part 46: Advanced Testing Strategies
## ขั้นตอนที่ 3121-3190: Test Architecture & Patterns

---

## 46.1 Testing Pyramid

```
            /\
           /  \    E2E Tests
          /    \   (Slow, Fragile, Expensive)
         /──────\
        /        \ Integration Tests
       /          \ (Medium)
      /────────────\
     /              \ Unit Tests
    /                \ (Fast, Stable, Cheap)
   /──────────────────\

Rule of thumb:
  70% Unit tests
  20% Integration tests
  10% E2E tests

Testing types:
  Unit         = single class, all dependencies mocked
  Integration  = multiple classes, real DB or real HTTP
  Contract     = API consumer/provider contracts (Pact)
  Performance  = load, stress, endurance (Gatling, JMeter)
  E2E          = full system (Playwright, Selenium)
```

---

## 46.2 Advanced JUnit 5

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.*;
import org.junit.jupiter.params.*;
import org.junit.jupiter.params.provider.*;
import java.util.stream.Stream;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)  // Share state across tests
class OrderServiceAdvancedTest {
    
    private OrderService orderService;
    
    @BeforeAll
    void setupAll() {
        // Runs once (PER_CLASS allows non-static)
        orderService = new OrderService(new InMemoryOrderRepository());
    }
    
    @Test
    @Order(1)
    @DisplayName("Create order with multiple items")
    @Tag("smoke")
    void createOrderWithItems() {
        // ...
    }
    
    // ====== Parameterized tests ======
    @ParameterizedTest
    @ValueSource(doubles = {0, -1, -100, Double.NEGATIVE_INFINITY})
    void invalidPrices_shouldThrow(double price) {
        assertThrows(IllegalArgumentException.class, () -> Money.of(price));
    }
    
    @ParameterizedTest
    @CsvSource({
        "product-1, Laptop,  75000.0, 1, 75000.0",
        "product-2, Mouse,   1500.0,  3, 4500.0",
        "product-3, Keyboard,3500.0,  2, 7000.0"
    })
    void orderItem_subtotal(String id, String name, double price, int qty, double expected) {
        var item = new OrderItem(id, name, Money.of(price), qty);
        assertEquals(Money.of(expected), item.getSubtotal());
    }
    
    @ParameterizedTest
    @MethodSource("orderStatusProvider")
    void cannotAddItemsWhenNotPending(OrderStatus status) {
        Order order = Order.create("user-1");
        order.setStatusForTest(status);  // test helper
        assertThrows(IllegalStateException.class, () ->
            order.addItem(new OrderItem("p1", "Product", Money.of(100), 1)));
    }
    
    static Stream<OrderStatus> orderStatusProvider() {
        return Stream.of(
            OrderStatus.CONFIRMED,
            OrderStatus.SHIPPED,
            OrderStatus.DELIVERED,
            OrderStatus.CANCELLED
        );
    }
    
    @ParameterizedTest
    @EnumSource(value = OrderStatus.class, 
                names = {"SHIPPED", "DELIVERED"},
                mode = EnumSource.Mode.INCLUDE)
    void cannotCancelShippedOrDelivered(OrderStatus status) {
        Order order = Order.create("user-1");
        order.setStatusForTest(status);
        assertThrows(IllegalStateException.class, () -> order.cancel("reason"));
    }
    
    // ====== Dynamic tests ======
    @TestFactory
    Stream<DynamicTest> dynamicOrderTests() {
        record TestCase(String name, double total, boolean shouldBeHighValue) {}
        
        return Stream.of(
            new TestCase("zero total", 0, false),
            new TestCase("low total", 999, false),
            new TestCase("threshold", 10000, true),
            new TestCase("high total", 50000, true)
        ).map(tc -> DynamicTest.dynamicTest(
            tc.name(),
            () -> assertEquals(tc.shouldBeHighValue(), isHighValue(tc.total()))
        ));
    }
    
    boolean isHighValue(double total) { return total >= 10000; }
    
    // ====== Nested tests ======
    @Nested
    @DisplayName("When order is PENDING")
    class PendingOrderTests {
        
        Order order;
        
        @BeforeEach
        void setup() {
            order = Order.create("user-1");
        }
        
        @Test
        void canAddItems() {
            assertDoesNotThrow(() ->
                order.addItem(new OrderItem("p1", "Product", Money.of(100), 1)));
        }
        
        @Test
        void canConfirm() {
            order.addItem(new OrderItem("p1", "Product", Money.of(100), 1));
            assertDoesNotThrow(order::confirm);
        }
        
        @Test
        void canCancel() {
            assertDoesNotThrow(() -> order.cancel("Changed mind"));
        }
        
        @Nested
        @DisplayName("After adding items")
        class WithItemsTests {
            
            @BeforeEach
            void addItems() {
                order.addItem(new OrderItem("p1", "Product A", Money.of(100), 2));
                order.addItem(new OrderItem("p2", "Product B", Money.of(50), 1));
            }
            
            @Test
            void totalIsCorrect() {
                assertEquals(Money.of(250), order.getTotal());
            }
            
            @Test
            void itemCountIsCorrect() {
                assertEquals(2, order.getItems().size());
            }
        }
    }
}
```

---

## 46.3 Mockito Advanced

```java
import org.mockito.*;
import org.mockito.junit.jupiter.*;
import java.util.List;

@ExtendWith(MockitoExtension.class)
class OrderServiceMockTest {
    
    @Mock OrderRepository orderRepository;
    @Mock PaymentService paymentService;
    @Mock NotificationService notificationService;
    @Spy List<String> auditLog = new ArrayList<>();  // Spy: real object + verify calls
    @InjectMocks OrderService orderService;
    
    @Captor ArgumentCaptor<Order> orderCaptor;
    
    @Test
    void createOrder_capturesAndValidatesSavedOrder() {
        // Arrange
        when(orderRepository.save(any())).thenAnswer(invocation -> {
            Order o = invocation.getArgument(0);
            o.setId("generated-id");
            return o;
        });
        
        // Act
        orderService.createOrder("user-1", List.of(
            new OrderItem("p1", "Laptop", Money.of(75000), 1)
        ));
        
        // Assert: capture what was saved
        verify(orderRepository).save(orderCaptor.capture());
        Order saved = orderCaptor.getValue();
        assertAll(
            () -> assertEquals("user-1", saved.getUserId()),
            () -> assertEquals(OrderStatus.CONFIRMED, saved.getStatus()),
            () -> assertEquals(1, saved.getItems().size()),
            () -> assertEquals(Money.of(75000), saved.getTotal())
        );
    }
    
    @Test
    void methodCallOrder_verifiedWithInOrder() {
        when(orderRepository.save(any())).thenReturn(new Order("o1", "user-1"));
        when(paymentService.charge(any(), anyDouble())).thenReturn(new PaymentResult("tx-1", "SUCCESS"));
        
        orderService.createOrder("user-1", List.of(
            new OrderItem("p1", "Product", Money.of(100), 1)
        ));
        
        // Verify order of method calls
        InOrder inOrder = inOrder(orderRepository, paymentService, notificationService);
        inOrder.verify(orderRepository).save(any());
        inOrder.verify(paymentService).charge(eq("o1"), eq(100.0));
        inOrder.verify(notificationService).orderConfirmed(eq("user-1"), eq("o1"));
    }
    
    @Test
    void paymentFailure_rollsBackOrder() {
        when(orderRepository.save(any())).thenReturn(new Order("o1", "user-1"));
        when(paymentService.charge(any(), anyDouble()))
            .thenThrow(new PaymentException("Card declined"));
        
        assertThrows(PaymentException.class, () ->
            orderService.createOrder("user-1", List.of(
                new OrderItem("p1", "Product", Money.of(100), 1)
            )));
        
        // Verify order was deleted on failure
        verify(orderRepository).delete("o1");
        verifyNoInteractions(notificationService);
    }
    
    @Test
    void retryOnTransientError() {
        when(paymentService.charge(any(), anyDouble()))
            .thenThrow(new TransientException("Timeout"))
            .thenThrow(new TransientException("Timeout"))
            .thenReturn(new PaymentResult("tx-1", "SUCCESS"));
        
        // Should succeed on 3rd attempt
        assertDoesNotThrow(() -> orderService.createOrderWithRetry("user-1", List.of(
            new OrderItem("p1", "Product", Money.of(100), 1)
        )));
        
        verify(paymentService, times(3)).charge(any(), anyDouble());
    }
    
    @Test
    void partialMockWithSpy() {
        OrderService spy = Mockito.spy(orderService);
        
        // Override specific method but keep others real
        doReturn(true).when(spy).isHighValueOrder(any());
        
        // Can verify method calls on spy
        spy.createOrder("user-1", List.of());
        verify(spy).isHighValueOrder(any());
    }
}
```

---

## 46.4 Contract Testing with Pact

```java
// Consumer side (order-service consuming user-service API)
import au.com.dius.pact.consumer.dsl.*;
import au.com.dius.pact.consumer.junit5.*;

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service", port = "8181")
class UserClientContractTest {
    
    @Pact(provider = "user-service", consumer = "order-service")
    RequestResponsePact getUserByIdPact(PactDslWithProvider builder) {
        return builder
            .given("user with id 1 exists")
            .uponReceiving("a request for user 1")
                .path("/api/v1/users/1")
                .method("GET")
                .headers(Map.of("Accept", "application/json"))
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .integerType("id", 1)
                    .stringType("name", "Alice")
                    .stringType("email", "alice@example.com")
                    .stringMatcher("role", "USER|ADMIN", "USER")
                )
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "getUserByIdPact")
    void getUserById_matchesContract(MockServer mockServer) {
        UserClient client = new UserClient("http://localhost:" + mockServer.getPort());
        
        UserDTO user = client.getUser(1L);
        
        assertAll(
            () -> assertEquals(1L, user.id()),
            () -> assertNotNull(user.name()),
            () -> assertNotNull(user.email())
        );
    }
}
```

---

## 46.5 Integration Tests with Testcontainers

```java
import org.testcontainers.containers.*;
import org.testcontainers.junit.jupiter.*;
import org.springframework.boot.test.context.*;
import org.springframework.test.context.*;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OrderIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Autowired TestRestTemplate restTemplate;
    @Autowired OrderRepository orderRepository;
    @Autowired UserRepository userRepository;
    
    @Test
    void createOrder_fullFlow() {
        // Setup
        User user = userRepository.save(new User("Alice", "alice@test.com", "ROLE_USER"));
        String token = loginAndGetToken("alice@test.com", "password");
        
        // Act
        CreateOrderRequest request = new CreateOrderRequest(List.of(
            new OrderItemRequest("PROD-1", 2)
        ));
        
        var response = restTemplate.postForEntity(
            "/api/v1/orders",
            withAuth(request, token),
            OrderDTO.class
        );
        
        // Assert HTTP response
        assertEquals(201, response.getStatusCodeValue());
        assertNotNull(response.getBody().id());
        
        // Assert DB state
        Order savedOrder = orderRepository.findById(response.getBody().id()).orElseThrow();
        assertEquals("CONFIRMED", savedOrder.getStatus().name());
        assertEquals(user.getId(), savedOrder.getUserId());
    }
    
    private String loginAndGetToken(String email, String password) { return ""; }
    private <T> org.springframework.http.HttpEntity<T> withAuth(T body, String token) {
        var headers = new org.springframework.http.HttpHeaders();
        headers.setBearerAuth(token);
        return new org.springframework.http.HttpEntity<>(body, headers);
    }
}
```

---

## 46.6 Mutation Testing with PIT

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.3</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.example.domain.*</param>
        </targetClasses>
        <targetTests>
            <param>com.example.*Test</param>
        </targetTests>
        <mutators>DEFAULTS</mutators>
        <threads>4</threads>
        <timeoutFactor>1.5</timeoutFactor>
        <mutationThreshold>80</mutationThreshold>  <!-- fail if < 80% mutations killed -->
        <outputFormats>HTML,XML</outputFormats>
    </configuration>
</plugin>
```

```
Mutation Testing:
  PIT introduces mutations (bugs) in your code:
    - Change > to >= 
    - Negate conditions
    - Remove method calls
    - Change return values
    
  If your tests catch the mutation → mutation KILLED (good)
  If tests still pass with mutation → mutation SURVIVED (bad)
  
  Mutation Coverage > 80% = tests are meaningful
  High code coverage ≠ high mutation coverage
```

---

## สรุป Part 46

```
Testing Strategy Summary:

Unit Tests:
  - Test single class in isolation
  - Mock all dependencies
  - Fast (<100ms)
  - High coverage (>80%)

Integration Tests:
  - Test multiple components together
  - Use Testcontainers for real DB/cache
  - @SpringBootTest for full context

Contract Tests (Pact):
  - Verify API contracts between services
  - Consumer defines expectations
  - Provider verifies against them

Mutation Tests (PIT):
  - Verify test quality, not just coverage
  - Kill >80% mutations

Test Pyramid:
  70% Unit → 20% Integration → 10% E2E

FIRST Principles:
  Fast      = run quickly (<100ms for unit)
  Isolated  = no shared state
  Repeatable = same result every time
  Self-validating = pass/fail, no manual check
  Timely    = write before or with code
```

➡️ [Part 47: Spring Boot Advanced Features](./Part-47-SpringBootAdvanced.md)
