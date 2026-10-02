# Part 76: GraphQL API with Spring
## ขั้นตอนที่ 5221-5290: Schema Definition, Resolvers, Subscriptions

---

## 76.1 GraphQL vs REST

```
REST:
  GET /users/1          → returns ALL user fields
  GET /users/1/orders   → separate request
  GET /orders/1/items   → separate request
  Total: 3 requests, over-fetching on each

GraphQL:
  Single query, client requests EXACTLY what it needs:
  
  query {
    user(id: "1") {
      name
      email
      orders(last: 5) {
        id
        total
        items {
          productName
          quantity
        }
      }
    }
  }
  Total: 1 request, no over-fetching

Benefits:
  ✓ Client specifies fields → no over/under-fetching
  ✓ Single endpoint /graphql
  ✓ Strong typing via schema
  ✓ Real-time via subscriptions
  ✓ Self-documenting (introspection)

Drawbacks:
  ✗ Complex backend (N+1 problem with DataLoader)
  ✗ Caching harder (not GET requests)
  ✗ Overkill for simple CRUD APIs
```

---

## 76.2 Schema Definition

```graphql
# src/main/resources/graphql/schema.graphqls

scalar DateTime
scalar BigDecimal

type Query {
    user(id: ID!): User
    users(page: Int = 0, size: Int = 20): UserPage!
    order(id: ID!): Order
    myOrders: [Order!]!
    products(category: String, search: String): [Product!]!
    product(id: ID!): Product
}

type Mutation {
    createOrder: Order!
    addItemToOrder(orderId: ID!, productId: ID!, quantity: Int!): Order!
    submitOrder(orderId: ID!): Order!
    cancelOrder(orderId: ID!, reason: String!): Order!
}

type Subscription {
    orderStatusUpdated(orderId: ID!): OrderStatusUpdate!
    newOrderForAdmin: Order!
}

type User {
    id: ID!
    name: String!
    email: String!
    createdAt: DateTime!
    orders(status: OrderStatus): [Order!]!
}

type Order {
    id: ID!
    customerId: ID!
    customer: User!         # resolved separately
    status: OrderStatus!
    items: [OrderItem!]!
    total: BigDecimal!
    createdAt: DateTime!
    updatedAt: DateTime!
}

type OrderItem {
    id: ID!
    product: Product!
    quantity: Int!
    unitPrice: BigDecimal!
    subtotal: BigDecimal!
}

type Product {
    id: ID!
    name: String!
    price: BigDecimal!
    category: String!
    stock: Int!
    inStock: Boolean!
}

type UserPage {
    content: [User!]!
    totalElements: Int!
    totalPages: Int!
    number: Int!
}

type OrderStatusUpdate {
    orderId: ID!
    oldStatus: OrderStatus!
    newStatus: OrderStatus!
    timestamp: DateTime!
}

enum OrderStatus {
    DRAFT
    SUBMITTED
    CONFIRMED
    CANCELLED
    COMPLETED
}

input CreateOrderInput {
    customerId: ID!
}

input AddItemInput {
    orderId: ID!
    productId: ID!
    quantity: Int!
}
```

---

## 76.3 Spring for GraphQL Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

```yaml
# application.yaml
spring:
  graphql:
    graphiql:
      enabled: true    # enable GraphiQL playground at /graphiql
    websocket:
      path: /graphql-ws
    schema:
      printer:
        enabled: true
```

---

## 76.4 Resolvers (Controllers)

```java
import org.springframework.graphql.data.method.annotation.*;
import org.springframework.stereotype.*;

@Controller
class UserGraphQLController {
    
    private final UserService userService;
    
    UserGraphQLController(UserService userService) {
        this.userService = userService;
    }
    
    @QueryMapping
    User user(@Argument String id) {
        return userService.findById(id);
    }
    
    @QueryMapping
    UserPage users(@Argument int page, @Argument int size) {
        return userService.findAll(PageRequest.of(page, size));
    }
    
    // Nested resolver: User → orders
    // Only called when client requests user.orders
    @SchemaMapping(typeName = "User", field = "orders")
    List<Order> ordersForUser(User user, @Argument String status) {
        return orderService.findByUserId(user.getId(), status);
    }
}

@Controller
class OrderGraphQLController {
    
    private final OrderService orderService;
    private final BatchLoaderRegistry batchLoaderRegistry;
    
    OrderGraphQLController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    @QueryMapping
    Order order(@Argument String id) {
        return orderService.findById(id);
    }
    
    @QueryMapping
    List<Order> myOrders(@AuthenticationPrincipal org.springframework.security.oauth2.jwt.Jwt jwt) {
        return orderService.findByCustomerId(jwt.getSubject());
    }
    
    @MutationMapping
    Order createOrder(@AuthenticationPrincipal org.springframework.security.oauth2.jwt.Jwt jwt) {
        return orderService.createOrder(jwt.getSubject());
    }
    
    @MutationMapping
    Order addItemToOrder(@Argument String orderId,
                         @Argument String productId,
                         @Argument int quantity) {
        return orderService.addItem(orderId, productId, quantity);
    }
    
    @MutationMapping
    Order submitOrder(@Argument String orderId) {
        return orderService.submit(orderId);
    }
    
    // Resolver for Order.customer field (uses DataLoader to avoid N+1)
    @SchemaMapping(typeName = "Order", field = "customer")
    @BatchMapping
    java.util.concurrent.CompletableFuture<User> customer(Order order,
            org.dataloader.DataLoader<String, User> userLoader) {
        return userLoader.load(order.getCustomerId());
    }
}
```

---

## 76.5 DataLoader (N+1 Prevention)

```java
import org.dataloader.*;
import org.springframework.graphql.execution.*;

// Without DataLoader:
// Query for 10 orders → 10 separate DB calls for each order's customer
// (N+1 problem)

// With DataLoader:
// Batch all customer IDs, single DB call for all

@Configuration
class DataLoaderConfig {
    
    private final UserService userService;
    
    DataLoaderConfig(UserService userService) {
        this.userService = userService;
    }
    
    @Bean
    RuntimeWiringConfigurer dataLoaderConfigurer() {
        return wiringBuilder -> {
            // Register user DataLoader
            wiringBuilder.directiveWiring(new DataLoaderRegistrar(batchLoaderRegistry -> {
                batchLoaderRegistry.forTypePair(String.class, User.class)
                    .withName("userLoader")
                    .registerBatchLoader((ids, env) ->
                        // Single DB call for ALL user IDs
                        userService.findByIds(ids)
                    );
            }));
        };
    }
}

// Alternative: Spring for GraphQL @BatchMapping
@Controller
class OrderCustomerBatchController {
    
    private final UserService userService;
    
    OrderCustomerBatchController(UserService userService) {
        this.userService = userService;
    }
    
    // Spring for GraphQL will batch these automatically
    @BatchMapping(typeName = "Order", field = "customer")
    java.util.Map<Order, User> customers(List<Order> orders) {
        // Called ONCE for all orders in the query
        var customerIds = orders.stream()
            .map(Order::getCustomerId)
            .distinct()
            .toList();
        
        var userMap = userService.findByIds(customerIds)
            .stream()
            .collect(java.util.stream.Collectors.toMap(User::getId, u -> u));
        
        return orders.stream()
            .collect(java.util.stream.Collectors.toMap(
                o -> o,
                o -> userMap.get(o.getCustomerId())
            ));
    }
}
```

---

## 76.6 Subscriptions

```java
import reactor.core.publisher.*;

@Controller
class OrderSubscriptionController {
    
    private final Sinks.Many<OrderStatusUpdate> orderStatusSink = 
        Sinks.many().multicast().onBackpressureBuffer();
    
    // Subscription resolver
    @SubscriptionMapping
    Flux<OrderStatusUpdate> orderStatusUpdated(@Argument String orderId) {
        return orderStatusSink.asFlux()
            .filter(update -> update.orderId().equals(orderId));
    }
    
    // Called when order status changes
    public void publishOrderStatusUpdate(String orderId, String oldStatus, String newStatus) {
        orderStatusSink.tryEmitNext(new OrderStatusUpdate(
            orderId, oldStatus, newStatus, java.time.Instant.now()
        ));
    }
    
    record OrderStatusUpdate(
        String orderId,
        String oldStatus,
        String newStatus,
        java.time.Instant timestamp
    ) {}
}

// Client subscription (JavaScript):
/*
const subscription = gql`
  subscription {
    orderStatusUpdated(orderId: "order-123") {
      orderId
      oldStatus
      newStatus
      timestamp
    }
  }
`;

// Using Apollo Client
const { data } = useSubscription(subscription);
*/
```

---

## สรุป Part 76

```
GraphQL Key Concepts:

Schema:
  type Query     = read operations
  type Mutation  = write operations
  type Subscription = real-time push
  scalar, enum, interface, union

Resolvers:
  @QueryMapping      = handles Query.fieldName
  @MutationMapping   = handles Mutation.fieldName
  @SchemaMapping     = handles TypeName.fieldName (nested)
  @BatchMapping      = batch loading (avoid N+1)

N+1 Problem:
  10 orders → 10 separate DB calls for customer
  Fix: @BatchMapping = Spring calls once with all orders
  → single DB query for all customers

DataLoader Pattern:
  Collect all IDs during query execution
  Execute ONE batch query for all IDs
  Return map of ID → Result

Security:
  @PreAuthorize on resolver methods
  @AuthenticationPrincipal to get current user
  
  Depth limiting: prevent overly deep queries
  Query complexity analysis: prevent expensive queries

When to use GraphQL:
  ✓ Multiple clients (mobile, web, partner APIs) with different needs
  ✓ Aggregating data from multiple services
  ✓ Rapidly evolving data requirements
  ✗ Simple CRUD with fixed data shape (use REST)
  ✗ File uploads (use REST multipart)
```

➡️ [Part 77: gRPC & Protocol Buffers](./Part-77-gRPC.md)
