# Part 93: Java 21+ Modern Features
## ขั้นตอนที่ 6411-6480: Records, Sealed Classes, Pattern Matching, Virtual Threads

---

## 93.1 Java Records (Java 16+)

```java
// Records = immutable data carriers
// Compiler auto-generates: constructor, getters, equals, hashCode, toString

// Before Java 16 (verbose):
public class ProductOld {
    private final String id;
    private final String name;
    private final BigDecimal price;
    
    public ProductOld(String id, String name, BigDecimal price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
    public String getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    @Override public boolean equals(Object o) { /* ... */ }
    @Override public int hashCode() { /* ... */ }
    @Override public String toString() { /* ... */ }
}

// Java 16+: Records (concise)
public record Product(String id, String name, BigDecimal price) {
    // Compact constructor for validation
    public Product {
        Objects.requireNonNull(id, "id cannot be null");
        Objects.requireNonNull(name, "name cannot be null");
        if (price.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("price cannot be negative");
        }
        name = name.trim();  // can normalize in compact constructor
    }
    
    // Can add methods
    public boolean isExpensive() {
        return price.compareTo(new BigDecimal("1000")) > 0;
    }
    
    // Can add static factory methods
    public static Product free(String id, String name) {
        return new Product(id, name, BigDecimal.ZERO);
    }
    
    // Can implement interfaces
    // Can't extend classes (records extend Record implicitly)
    // Can't have instance fields (only the record components)
}

// Usage
var product = new Product("P001", "Laptop", new BigDecimal("25000"));
System.out.println(product.id());        // accessor (not getId!)
System.out.println(product.name());
System.out.println(product.price());
System.out.println(product);             // Product[id=P001, name=Laptop, price=25000]

// Records work great with Jackson
@JsonDeserialize
public record ApiResponse<T>(
    boolean success,
    T data,
    String message
) {}

// Nested records
public record Address(String street, String city, String country) {}
public record Customer(String id, String name, Address address) {}

var customer = new Customer(
    "C001",
    "Alice",
    new Address("123 Main St", "Bangkok", "Thailand")
);
```

---

## 93.2 Sealed Classes (Java 17+)

```java
// Sealed classes: restrict which classes can extend/implement
// Perfect for algebraic data types / discriminated unions

// sealed class + permits = closed hierarchy
public sealed interface Result<T>
    permits Result.Success, Result.Failure {
    
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String message, Throwable cause) implements Result<T> {
        public Failure(String message) {
            this(message, null);
        }
    }
}

// Exhaustive pattern matching possible!
public <T> String describeResult(Result<T> result) {
    return switch (result) {
        case Result.Success<T> s -> "Succeeded with: " + s.value();
        case Result.Failure<T> f -> "Failed: " + f.message();
        // No default needed! Compiler knows all subtypes
    };
}

// Payment system with sealed classes
public sealed interface PaymentMethod
    permits CreditCard, DebitCard, BankTransfer, Crypto {
}

public record CreditCard(String cardNumber, String expiryDate, String cvv)
    implements PaymentMethod {}
    
public record DebitCard(String cardNumber, String bankCode)
    implements PaymentMethod {}
    
public record BankTransfer(String accountNumber, String bankCode)
    implements PaymentMethod {}
    
public record Crypto(String walletAddress, CryptoType type)
    implements PaymentMethod {}

public enum CryptoType { BITCOIN, ETHEREUM, USDT }

// Process payment - exhaustive switch
public ProcessingFee calculateFee(PaymentMethod method) {
    return switch (method) {
        case CreditCard cc -> new ProcessingFee(cc.cardNumber(), 0.025);  // 2.5%
        case DebitCard dc -> new ProcessingFee(dc.cardNumber(), 0.01);   // 1%
        case BankTransfer bt -> new ProcessingFee(bt.accountNumber(), 0); // Free
        case Crypto c when c.type() == CryptoType.USDT ->
            new ProcessingFee(c.walletAddress(), 0.001);  // Low fee for stablecoin
        case Crypto c -> new ProcessingFee(c.walletAddress(), 0.005);    // Crypto fee
    };
}
```

---

## 93.3 Pattern Matching (Java 21+)

```java
// Pattern matching for instanceof (Java 16+)
// Old way:
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.toUpperCase());
}

// New way:
if (obj instanceof String s) {
    System.out.println(s.toUpperCase());  // s is already String
}

// Even shorter with && guard:
if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
}

// ====== Switch Pattern Matching (Java 21) ======
public String formatValue(Object value) {
    return switch (value) {
        case null -> "null";
        case Integer i -> "int: " + i;
        case Long l -> "long: " + l;
        case Double d -> String.format("double: %.2f", d);
        case String s when s.isEmpty() -> "empty string";
        case String s -> "string: " + s;
        case int[] arr -> "int[" + arr.length + "]";
        case List<?> list when list.isEmpty() -> "empty list";
        case List<?> list -> "list with " + list.size() + " elements";
        case Product p when p.price().compareTo(BigDecimal.ZERO) == 0 -> 
            "free product: " + p.name();
        case Product p -> "product: " + p.name() + " @ " + p.price();
        default -> "unknown: " + value.getClass().getSimpleName();
    };
}

// ====== Deconstruction Patterns (Java 21+) ======
public void processCustomer(Object obj) {
    if (obj instanceof Customer(var id, var name, var address)) {
        // Deconstruct Customer record
        System.out.println("Customer " + name + " in " + address.city());
    }
    
    if (obj instanceof Customer(var id, var name,
                                Address(var street, var city, var country))) {
        // Nested deconstruction
        System.out.println(name + " lives at " + street + ", " + city);
    }
}

// ====== Guarded Patterns ======
record Point(int x, int y) {}

String classifyPoint(Point p) {
    return switch (p) {
        case Point(int x, int y) when x == 0 && y == 0 -> "Origin";
        case Point(int x, int y) when x == 0 -> "On Y-axis";
        case Point(int x, int y) when y == 0 -> "On X-axis";
        case Point(int x, int y) when x > 0 && y > 0 -> "Quadrant I";
        case Point(int x, int y) when x < 0 && y > 0 -> "Quadrant II";
        case Point(int x, int y) when x < 0 && y < 0 -> "Quadrant III";
        case Point(int x, int y) -> "Quadrant IV";
    };
}
```

---

## 93.4 Virtual Threads Deep Dive (Java 21)

```java
// Virtual Threads = Project Loom
// Lightweight threads managed by JVM, not OS
// Millions of concurrent threads possible!

// ====== Problem with Platform Threads ======
// Each thread = ~1MB memory
// 10,000 threads = 10GB RAM!
// Thread pool: expensive, blocking I/O wastes threads

// ====== Virtual Threads ======
// Many virtual threads → few OS threads (carrier threads)
// Virtual thread blocks → carrier thread freed immediately
// JVM schedules virtual threads on carrier threads

// Create virtual thread
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("Running in virtual thread: " + Thread.currentThread());
    // Thread[VirtualThread#1,5,main]
});

// Virtual thread per task executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // Creates a new virtual thread for EACH task!
    // No pool sizing needed
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> {
            Thread.sleep(1000);  // This doesn't waste OS thread
            return i;
        })
    );
}  // auto-closes, waits for all tasks

// ====== Virtual Threads + Spring Boot ======
// application.properties:
// spring.threads.virtual.enabled=true
// → All Spring MVC request handling uses virtual threads automatically!

// Manual configuration:
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return protocolHandler ->
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}

// ====== Structured Concurrency (Java 21 Preview) ======
// Handle multiple concurrent tasks with proper lifecycle management
import java.util.concurrent.StructuredTaskScope;

public OrderSummary fetchOrderSummary(String orderId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        // Launch both tasks concurrently
        var orderFuture = scope.fork(() -> orderService.findById(orderId));
        var itemsFuture = scope.fork(() -> itemService.findByOrderId(orderId));
        
        scope.join()           // wait for both
             .throwIfFailed(); // propagate any exception
        
        // Both completed successfully
        return new OrderSummary(
            orderFuture.get(),
            itemsFuture.get()
        );
    }
}

// ShutdownOnSuccess: take first result, cancel others
public <T> T tryMultipleStrategies(List<Callable<T>> strategies) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<T>()) {
        strategies.forEach(scope::fork);
        scope.join();
        return scope.result();  // first successful result
    }
}

// ====== Scoped Values (Java 21 Preview) ======
// Thread-local replacement for virtual threads
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
static final ScopedValue<String> TRANSACTION_ID = ScopedValue.newInstance();

public void processRequest(User user, Runnable handler) {
    String txId = UUID.randomUUID().toString();
    
    ScopedValue.where(CURRENT_USER, user)
               .where(TRANSACTION_ID, txId)
               .run(handler);  // handler can read CURRENT_USER and TRANSACTION_ID
}

// Read in any method in the call stack:
void deepInCallStack() {
    User user = CURRENT_USER.get();      // No need to pass as parameter
    String txId = TRANSACTION_ID.get();
    log.info("Processing for {} in tx {}", user.name(), txId);
}
```

---

## 93.5 Text Blocks & String Features (Java 15+)

```java
// Text blocks: multi-line strings without escaping
String json = """
        {
            "name": "Alice",
            "age": 30,
            "email": "alice@example.com"
        }
        """;

String html = """
        <html>
            <body>
                <h1>Hello, World!</h1>
            </body>
        </html>
        """;

String sql = """
        SELECT u.name, o.total
        FROM users u
        JOIN orders o ON u.id = o.user_id
        WHERE u.active = true
          AND o.created_at > '2024-01-01'
        ORDER BY o.total DESC
        """;

// Incidental indentation removed automatically

// ====== String Templates (Java 21 Preview) ======
// Type-safe string interpolation
import static java.lang.StringTemplate.STR;

String name = "Alice";
int age = 30;
String greeting = STR."Hello, \{name}! You are \{age} years old.";
// → "Hello, Alice! You are 30 years old."

// With expressions
String summary = STR."Total: \{order.getItems().stream()
    .mapToDouble(Item::getPrice)
    .sum()} THB";

// SQL template (prevents injection!)
String userId = getUserInput();
PreparedStatement ps = DB.query(STR."SELECT * FROM users WHERE id = \{userId}");
// SQL template processor safely parameterizes the value

// ====== SequencedCollections (Java 21) ======
// New interface: getFirst(), getLast(), addFirst(), addLast(), reversed()

List<String> list = new ArrayList<>(List.of("a", "b", "c"));
String first = list.getFirst();  // "a"
String last = list.getLast();    // "c"
list.addFirst("z");              // ["z", "a", "b", "c"]

LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("one", 1);
map.put("two", 2);
String firstKey = map.firstEntry().getKey();  // "one"
var reversed = map.reversed();               // reversed view
```

---

## 93.6 Functional Enhancements

```java
// ====== Stream Gatherers (Java 22+) ======
// Custom intermediate stream operations
import java.util.stream.Gatherer;
import java.util.stream.Gatherers;

// Built-in gatherers:
List<List<Integer>> windows = IntStream.range(0, 10)
    .boxed()
    .gather(Gatherers.windowSliding(3))  // sliding window of 3
    // [[0,1,2], [1,2,3], [2,3,4], ...]
    .toList();

List<List<Integer>> fixed = IntStream.range(0, 10)
    .boxed()
    .gather(Gatherers.windowFixed(3))   // fixed windows of 3
    // [[0,1,2], [3,4,5], [6,7,8], [9]]
    .toList();

// Take while distinct
List<Integer> distinct = Stream.of(1, 2, 2, 3, 1, 4)
    .gather(Gatherers.mapConcurrent(3, n -> n * 2))  // concurrent mapping
    .toList();

// ====== Unnamed Variables (Java 21+) ======
// Use _ for unused variables
try {
    riskyOperation();
} catch (IOException _) {  // don't need the exception
    log.warn("IO error occurred");
}

for (var _ : list) {  // counting without using element
    count++;
}

// ====== instanceof with pattern guards ======
Object obj = getObject();
if (obj instanceof Number n && n.doubleValue() > 100.0) {
    System.out.println("Large number: " + n);
}

// ====== Switch Expressions (Java 14+) ======
// switch as an expression (returns value)
int days = switch (month) {
    case JANUARY, MARCH, MAY, JULY, AUGUST, OCTOBER, DECEMBER -> 31;
    case APRIL, JUNE, SEPTEMBER, NOVEMBER -> 30;
    case FEBRUARY -> year % 4 == 0 ? 29 : 28;
};

String label = switch (status) {
    case PENDING -> "Pending approval";
    case ACTIVE -> "Active";
    case SUSPENDED -> {
        log.warn("Account suspended");
        yield "Account suspended";  // yield for multi-statement blocks
    }
    case DELETED -> "Deleted";
};
```

---

## สรุป Part 93

```
Java 21+ Modern Features:

Records (Java 16):
  Immutable data carriers
  Auto: constructor, accessors, equals, hashCode, toString
  Compact constructor: validation/normalization
  Can add methods, implement interfaces
  Perfect for DTOs, value objects

Sealed Classes (Java 17):
  Restricts class hierarchy
  Enables exhaustive switch (no default needed!)
  Great for: Result<T>, payment methods, event types
  sealed permits → non-sealed | sealed | final

Pattern Matching:
  instanceof: if (obj instanceof String s) {}
  Switch patterns: case Type t when condition ->
  Deconstruction: Customer(var id, var name, var addr)
  Null handling in switch: case null -> ...
  Guard patterns: case when expression

Virtual Threads (Java 21):
  Lightweight threads managed by JVM
  Millions concurrent without memory issues
  Blocking I/O doesn't waste OS thread
  Spring Boot: spring.threads.virtual.enabled=true
  Structured concurrency: StructuredTaskScope
  Scoped values: replace ThreadLocal

Text Blocks (Java 15):
  """ ... """ = multi-line strings
  Automatic indentation handling
  Great for SQL, JSON, HTML templates

Switch Expressions (Java 14):
  Returns value directly
  yield for multi-statement blocks
  Arrow form: case X -> value

Migration:
  Records → replace data classes
  Sealed + Pattern → replace visitor pattern
  Virtual threads → replace reactive programming
  Text blocks → replace StringBuilder for multiline
```

➡️ [Part 94: Advanced Caching Strategies](./Part-94-Caching.md)
