# Part 35: GraphQL API with Spring Boot
## ขั้นตอนที่ 2351-2420: Modern API Design

---

## 35.1 GraphQL Overview

```
REST vs GraphQL:

REST:
  GET /users/1                → User (all fields)
  GET /users/1/orders         → Orders
  GET /orders/42/items        → Items
  3 roundtrips, over-fetching

GraphQL:
  POST /graphql
  query {
    user(id: 1) {
      name email
      orders {
        total status
        items { product { name } quantity }
      }
    }
  }
  1 roundtrip, exactly what you need

Advantages:
  ✓ No over-fetching (request only needed fields)
  ✓ No under-fetching (get related data in one query)
  ✓ Strongly typed schema (self-documenting)
  ✓ Real-time with subscriptions
  
Disadvantages:
  ✗ Caching is harder (all POST to same endpoint)
  ✗ N+1 problem needs DataLoader
  ✗ Complexity for simple APIs
```

---

## 35.2 Spring Boot GraphQL Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.graphql</groupId>
    <artifactId>spring-graphql-test</artifactId>
    <scope>test</scope>
</dependency>
```

```yaml
# application.yml
spring:
  graphql:
    graphiql:
      enabled: true   # UI at /graphiql
    schema:
      locations: classpath:graphql/   # .graphqls files here
    websocket:
      path: /graphql
```

---

## 35.3 GraphQL Schema

```graphql
# src/main/resources/graphql/schema.graphqls

scalar DateTime

# ====== Types ======
type User {
    id: ID!
    name: String!
    email: String!
    role: Role!
    createdAt: DateTime!
    orders: [Order!]!
    orderCount: Int!
}

type Order {
    id: ID!
    user: User!
    items: [OrderItem!]!
    status: OrderStatus!
    total: Float!
    createdAt: DateTime!
}

type OrderItem {
    id: ID!
    product: Product!
    quantity: Int!
    price: Float!
    subtotal: Float!
}

type Product {
    id: ID!
    name: String!
    description: String
    price: Float!
    stock: Int!
    category: Category!
}

type Category {
    id: ID!
    name: String!
    products: [Product!]!
    productCount: Int!
}

type Page {
    page: Int!
    pageSize: Int!
    total: Int!
    totalPages: Int!
}

type UserPage {
    users: [User!]!
    pagination: Page!
}

type ProductPage {
    products: [Product!]!
    pagination: Page!
}

# ====== Enums ======
enum Role { USER ADMIN MODERATOR }
enum OrderStatus { PENDING PROCESSING SHIPPED DELIVERED CANCELLED }

# ====== Queries ======
type Query {
    # Users
    user(id: ID!): User
    users(page: Int = 0, size: Int = 20, search: String): UserPage!
    me: User
    
    # Products
    product(id: ID!): Product
    products(
        page: Int = 0,
        size: Int = 20,
        category: ID,
        search: String,
        minPrice: Float,
        maxPrice: Float
    ): ProductPage!
    
    # Orders
    order(id: ID!): Order
    myOrders: [Order!]!
    
    # Categories
    categories: [Category!]!
}

# ====== Mutations ======
type Mutation {
    # Users
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
    
    # Products
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    
    # Orders
    createOrder(items: [OrderItemInput!]!): Order!
    updateOrderStatus(id: ID!, status: OrderStatus!): Order!
    cancelOrder(id: ID!): Order!
}

# ====== Subscriptions ======
type Subscription {
    orderStatusChanged(orderId: ID!): Order!
    newOrder: Order!
}

# ====== Inputs ======
input CreateUserInput {
    name: String!
    email: String!
    password: String!
    role: Role = USER
}

input UpdateUserInput {
    name: String
    email: String
}

input CreateProductInput {
    name: String!
    description: String
    price: Float!
    stock: Int!
    categoryId: ID!
}

input UpdateProductInput {
    name: String
    description: String
    price: Float
    stock: Int
}

input OrderItemInput {
    productId: ID!
    quantity: Int!
}
```

---

## 35.4 Resolvers

```java
import org.springframework.graphql.data.method.annotation.*;
import org.springframework.stereotype.*;
import reactor.core.publisher.*;

// ====== Query Resolver ======
@Controller
public class UserQueryResolver {
    
    private final UserService userService;
    
    public UserQueryResolver(UserService userService) {
        this.userService = userService;
    }
    
    @QueryMapping
    public User user(@Argument Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new RuntimeException("User not found: " + id));
    }
    
    @QueryMapping
    public UserPage users(
            @Argument int page,
            @Argument int size,
            @Argument String search) {
        return userService.findAll(page, size, search);
    }
    
    @QueryMapping
    @org.springframework.security.access.prepost.PreAuthorize("isAuthenticated()")
    public User me(
            @org.springframework.security.core.annotation.AuthenticationPrincipal
            org.springframework.security.core.userdetails.UserDetails principal) {
        return userService.findByUsername(principal.getUsername())
            .orElseThrow();
    }
}

// ====== Field Resolver (N+1 prevention) ======
@Controller
public class UserFieldResolver {
    
    private final OrderService orderService;
    
    public UserFieldResolver(OrderService orderService) {
        this.orderService = orderService;
    }
    
    // Called for each User in results
    @SchemaMapping(typeName = "User")
    public List<Order> orders(User user) {
        return orderService.findByUserId(user.getId());
    }
    
    @SchemaMapping(typeName = "User")
    public int orderCount(User user) {
        return orderService.countByUserId(user.getId());
    }
}

// ====== Mutation Resolver ======
@Controller
public class UserMutationResolver {
    
    private final UserService userService;
    
    public UserMutationResolver(UserService userService) {
        this.userService = userService;
    }
    
    @MutationMapping
    public User createUser(@Argument CreateUserInput input) {
        return userService.createUser(input);
    }
    
    @MutationMapping
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    public User updateUser(@Argument Long id, @Argument UpdateUserInput input) {
        return userService.updateUser(id, input);
    }
    
    @MutationMapping
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    public boolean deleteUser(@Argument Long id) {
        userService.deleteUser(id);
        return true;
    }
}

// ====== Subscription Resolver ======
@Controller
public class OrderSubscriptionResolver {
    
    private final Sinks.Many<Order> orderSink = Sinks.many().multicast().onBackpressureBuffer();
    
    @SubscriptionMapping
    public Flux<Order> orderStatusChanged(@Argument Long orderId) {
        return orderSink.asFlux()
            .filter(order -> order.getId().equals(orderId));
    }
    
    @SubscriptionMapping
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    public Flux<Order> newOrder() {
        return orderSink.asFlux();
    }
    
    // Called when order status changes
    public void publishOrderUpdate(Order order) {
        orderSink.tryEmitNext(order);
    }
}
```

---

## 35.5 DataLoader (N+1 Solution)

```java
import org.dataloader.*;
import org.springframework.graphql.execution.*;

// Without DataLoader: N+1 problem
// 1 query for users + N queries for orders (one per user)

// With DataLoader: batch loading
// 1 query for users + 1 batch query for ALL orders

@Component
public class UserDataLoaderConfig implements BatchLoaderRegistry.RegistrationSpec<Long, User> {
    
    private final UserService userService;
    
    public UserDataLoaderConfig(UserService userService) {
        this.userService = userService;
    }
    
    @Bean
    public BatchLoaderRegistry batchLoaderRegistry() {
        return BatchLoaderRegistry.newRegistry()
            .forTypePair(Long.class, User.class)
            .withOptions(o -> o.setBatchingEnabled(true).setMaxBatchSize(100))
            .registerBatchLoader((userIds, env) -> {
                // Batch load all users in one query
                return userService.findAllByIds(userIds);
            });
    }
}

// Use in resolver
@Controller
public class OrderFieldResolverWithBatching {
    
    @SchemaMapping(typeName = "Order")
    public CompletableFuture<User> user(Order order, DataLoader<Long, User> userLoader) {
        // Instead of querying DB immediately, this is batched
        return userLoader.load(order.getUserId());
    }
}
```

---

## 35.6 GraphQL Error Handling

```java
import graphql.ErrorType;
import graphql.GraphQLError;
import graphql.schema.DataFetchingEnvironment;
import org.springframework.graphql.execution.DataFetcherExceptionResolverAdapter;
import org.springframework.stereotype.Component;

@Component
public class GraphQLExceptionHandler extends DataFetcherExceptionResolverAdapter {
    
    @Override
    protected GraphQLError resolveToSingleError(Throwable ex, DataFetchingEnvironment env) {
        
        if (ex instanceof ResourceNotFoundException e) {
            return GraphQLError.newError()
                .errorType(ErrorType.NOT_FOUND)
                .message(e.getMessage())
                .path(env.getExecutionStepInfo().getPath())
                .build();
        }
        
        if (ex instanceof ValidationException e) {
            return GraphQLError.newError()
                .errorType(ErrorType.BAD_REQUEST)
                .message(e.getMessage())
                .build();
        }
        
        if (ex instanceof org.springframework.security.access.AccessDeniedException e) {
            return GraphQLError.newError()
                .errorType(ErrorType.FORBIDDEN)
                .message("Access denied")
                .build();
        }
        
        // Don't expose internal errors
        return GraphQLError.newError()
            .errorType(ErrorType.INTERNAL_ERROR)
            .message("An internal error occurred")
            .build();
    }
}
```

---

## 35.7 GraphQL Queries (Client Side)

```graphql
# ====== Query Examples ======

# Get user with orders
query GetUserWithOrders($userId: ID!) {
    user(id: $userId) {
        id
        name
        email
        orderCount
        orders {
            id
            status
            total
            createdAt
            items {
                quantity
                price
                product {
                    name
                    category {
                        name
                    }
                }
            }
        }
    }
}

# Variables:
# { "userId": "1" }

# ==============================

# Paginated products search
query SearchProducts(
    $search: String,
    $category: ID,
    $minPrice: Float,
    $maxPrice: Float,
    $page: Int,
    $size: Int
) {
    products(
        search: $search,
        category: $category,
        minPrice: $minPrice,
        maxPrice: $maxPrice,
        page: $page,
        size: $size
    ) {
        products {
            id
            name
            price
            stock
            category { name }
        }
        pagination {
            page
            pageSize
            total
            totalPages
        }
    }
}

# ====== Mutations ======

mutation CreateOrder($items: [OrderItemInput!]!) {
    createOrder(items: $items) {
        id
        status
        total
        items {
            product { name }
            quantity
            price
        }
    }
}

# Variables:
# {
#   "items": [
#     { "productId": "1", "quantity": 2 },
#     { "productId": "3", "quantity": 1 }
#   ]
# }

# ====== Subscription ======

subscription WatchOrderStatus($orderId: ID!) {
    orderStatusChanged(orderId: $orderId) {
        id
        status
        updatedAt
    }
}

# ====== Introspection ======

# Get all available types
query {
    __schema {
        types {
            name
            kind
        }
    }
}

# Get fields of a type
query {
    __type(name: "User") {
        fields {
            name
            type { name kind }
        }
    }
}
```

---

## 35.8 Testing GraphQL

```java
import org.springframework.graphql.test.tester.*;
import org.springframework.boot.test.autoconfigure.graphql.*;

@GraphQlTest(UserQueryResolver.class)
class UserResolverTest {
    
    @Autowired
    private GraphQlTester graphQlTester;
    
    @MockBean
    private UserService userService;
    
    @Test
    void user_returnsCorrectData() {
        User mockUser = new User(1L, "Alice", "alice@example.com", Role.USER);
        when(userService.findById(1L)).thenReturn(Optional.of(mockUser));
        
        graphQlTester
            .document("""
                query {
                    user(id: "1") {
                        id
                        name
                        email
                    }
                }
                """)
            .execute()
            .path("user.name").entity(String.class).isEqualTo("Alice")
            .path("user.email").entity(String.class).isEqualTo("alice@example.com");
    }
    
    @Test
    void user_notFound_returnsError() {
        when(userService.findById(99L)).thenReturn(Optional.empty());
        
        graphQlTester
            .document("""
                query {
                    user(id: "99") { id name }
                }
                """)
            .execute()
            .errors()
            .satisfy(errors -> {
                assertFalse(errors.isEmpty());
                assertTrue(errors.get(0).getMessage().contains("not found"));
            });
    }
    
    @Test
    void createUser_returnsCreatedUser() {
        User created = new User(2L, "Bob", "bob@example.com", Role.USER);
        when(userService.createUser(any())).thenReturn(created);
        
        graphQlTester
            .document("""
                mutation {
                    createUser(input: {
                        name: "Bob",
                        email: "bob@example.com",
                        password: "password123"
                    }) {
                        id name email
                    }
                }
                """)
            .execute()
            .path("createUser.name").entity(String.class).isEqualTo("Bob");
    }
}
```

---

## สรุป Part 35

| Concept | คำอธิบาย |
|---------|---------|
| `@QueryMapping` | Map query field to method |
| `@MutationMapping` | Map mutation to method |
| `@SubscriptionMapping` | Real-time with Flux<T> |
| `@SchemaMapping` | Field resolver (lazy loading) |
| `DataLoader` | Batch loading (N+1 prevention) |
| `DataFetcherExceptionResolverAdapter` | Error handling |
| `@Argument` | Extract GraphQL argument |

**GraphQL Schema First:**
1. เขียน `.graphqls` schema ก่อน
2. Implement resolvers
3. DataLoader สำหรับ relationships

➡️ [Part 36: Event Sourcing & CQRS](./Part-36-EventSourcing.md)
