# Part 25: Advanced Java - Microservices & Modern Java
## ขั้นตอนที่ 1651-1720: Java ระดับ Production

---

## 25.1 Java Records (Java 16+)

```java
import java.util.*;
import java.time.*;

// ====== Records ======
record Point(double x, double y) {
    // Compact constructor (validation)
    Point {
        if (Double.isNaN(x) || Double.isNaN(y)) {
            throw new IllegalArgumentException("Coordinates cannot be NaN");
        }
    }
    
    // Additional methods
    double distanceTo(Point other) {
        double dx = this.x - other.x;
        double dy = this.y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
    
    Point translate(double dx, double dy) {
        return new Point(x + dx, y + dy);
    }
    
    // Static factory
    static Point origin() { return new Point(0, 0); }
}

record PersonRecord(String name, int age, String email) implements Comparable<PersonRecord> {
    // Additional computed property
    boolean isAdult() { return age >= 18; }
    
    @Override
    public int compareTo(PersonRecord other) {
        return this.name.compareTo(other.name);
    }
}

// Nested records
record Address(String street, String city, String country) {}
record Employee(String id, String name, Address address, double salary) {
    String displayName() { return "[%s] %s".formatted(id, name); }
}

public class RecordsDemo {
    
    public static void main(String[] args) {
        
        Point p1 = new Point(3, 4);
        Point p2 = new Point(0, 0);
        
        System.out.println("p1: " + p1);
        System.out.println("Distance: " + p1.distanceTo(p2));
        System.out.println("Translated: " + p1.translate(1, 1));
        System.out.println("Origin: " + Point.origin());
        System.out.println("Equal: " + p1.equals(new Point(3, 4)));
        
        // Records in collections
        List<PersonRecord> people = List.of(
            new PersonRecord("Charlie", 25, "charlie@example.com"),
            new PersonRecord("Alice", 30, "alice@example.com"),
            new PersonRecord("Bob", 17, "bob@example.com")
        );
        
        people.stream()
              .filter(PersonRecord::isAdult)
              .sorted()
              .forEach(p -> System.out.printf("%-10s (%d)%n", p.name(), p.age()));
        
        // Records with pattern matching
        Object obj = new Point(2, 3);
        if (obj instanceof Point(var x, var y)) {
            System.out.printf("Destructured point: x=%.1f, y=%.1f%n", x, y);
        }
        
        // Nested record
        Employee emp = new Employee("E001", "Alice",
            new Address("123 Main St", "Bangkok", "Thailand"), 85000);
        System.out.println(emp.displayName() + " from " + emp.address().city());
    }
}
```

---

## 25.2 Sealed Classes & Pattern Matching (Java 17+)

```java
// Sealed hierarchy
sealed interface Expr permits Expr.Num, Expr.Add, Expr.Mul, Expr.Neg {
    
    record Num(double value) implements Expr {}
    record Add(Expr left, Expr right) implements Expr {}
    record Mul(Expr left, Expr right) implements Expr {}
    record Neg(Expr expr) implements Expr {}
    
    default double evaluate() {
        return switch (this) {
            case Num(var v) -> v;
            case Add(var l, var r) -> l.evaluate() + r.evaluate();
            case Mul(var l, var r) -> l.evaluate() * r.evaluate();
            case Neg(var e) -> -e.evaluate();
        };
    }
    
    default String prettyPrint() {
        return switch (this) {
            case Num(var v) -> String.valueOf(v);
            case Add(var l, var r) -> "(%s + %s)".formatted(l.prettyPrint(), r.prettyPrint());
            case Mul(var l, var r) -> "(%s * %s)".formatted(l.prettyPrint(), r.prettyPrint());
            case Neg(var e) -> "(-" + e.prettyPrint() + ")";
        };
    }
    
    static Expr num(double v) { return new Num(v); }
    static Expr add(Expr l, Expr r) { return new Add(l, r); }
    static Expr mul(Expr l, Expr r) { return new Mul(l, r); }
    static Expr neg(Expr e) { return new Neg(e); }
}

public class SealedPatternDemo {
    
    public static void main(String[] args) {
        
        // (2 + 3) * -(4)
        Expr expr = Expr.mul(
            Expr.add(Expr.num(2), Expr.num(3)),
            Expr.neg(Expr.num(4))
        );
        
        System.out.println(expr.prettyPrint() + " = " + expr.evaluate());
        
        // Pattern matching with guards (Java 21)
        List<Object> items = List.of(42, "hello", 3.14, -5, "world", 100L);
        
        for (Object item : items) {
            String desc = switch (item) {
                case Integer i when i > 0 -> "positive int: " + i;
                case Integer i           -> "non-positive int: " + i;
                case String s when s.length() > 4 -> "long string: " + s;
                case String s            -> "short string: " + s;
                case Double d            -> "double: " + d;
                case Long l              -> "long: " + l;
                default                  -> "unknown: " + item;
            };
            System.out.println(desc);
        }
    }
}
```

---

## 25.3 Text Blocks (Java 15+)

```java
public class TextBlocksDemo {
    
    public static void main(String[] args) {
        
        // JSON
        String json = """
                {
                    "name": "Alice",
                    "age": 30,
                    "roles": ["ADMIN", "USER"],
                    "address": {
                        "city": "Bangkok",
                        "country": "Thailand"
                    }
                }
                """;
        System.out.println(json);
        
        // HTML
        String html = """
                <!DOCTYPE html>
                <html lang="th">
                <head>
                    <meta charset="UTF-8">
                    <title>%s</title>
                </head>
                <body>
                    <h1>%s</h1>
                    <p>%s</p>
                </body>
                </html>
                """.formatted("หน้าแรก", "สวัสดี", "ยินดีต้อนรับ");
        System.out.println(html);
        
        // SQL
        String sql = """
                SELECT
                    u.id,
                    u.name,
                    u.email,
                    COUNT(o.id) as order_count,
                    SUM(o.total) as total_spent
                FROM users u
                LEFT JOIN orders o ON o.user_id = u.id
                WHERE u.active = true
                    AND u.created_at > :since
                GROUP BY u.id, u.name, u.email
                HAVING COUNT(o.id) > 0
                ORDER BY total_spent DESC
                LIMIT :limit
                """;
        System.out.println(sql);
    }
}
```

---

## 25.4 Modern Java APIs

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class ModernJavaAPIs {
    
    public static void main(String[] args) {
        
        // ====== Stream.toList() (Java 16+) ======
        List<String> filtered = Stream.of("a", "bb", "ccc", "dddd")
            .filter(s -> s.length() > 1)
            .toList();  // immutable list, no need for Collectors.toList()
        System.out.println("Filtered: " + filtered);
        
        // ====== Stream.mapMulti() (Java 16+) ======
        List<Integer> expanded = Stream.of(1, 2, 3)
            .<Integer>mapMulti((n, consumer) -> {
                for (int i = 0; i < n; i++) consumer.accept(i);
            })
            .toList();
        System.out.println("MapMulti: " + expanded);  // [0, 0, 1, 0, 1, 2]
        
        // ====== Collectors.teeing() (Java 12+) ======
        // Apply two collectors and merge results
        var stats = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
            .collect(Collectors.teeing(
                Collectors.summingInt(Integer::intValue),    // sum
                Collectors.counting(),                        // count
                (sum, count) -> "sum=" + sum + ", avg=" + (double) sum / count
            ));
        System.out.println("Stats: " + stats);
        
        // ====== String methods (Java 11+) ======
        String text = "  Hello, World!  ";
        System.out.println(text.strip());          // like trim() but unicode-aware
        System.out.println(text.stripLeading());
        System.out.println(text.stripTrailing());
        System.out.println("  ".isBlank());        // true if empty or whitespace
        System.out.println("Hello\nWorld".lines().count()); // 2
        System.out.println("Ha".repeat(3));        // HaHaHa
        
        // ====== String.formatted() (Java 15+) ======
        String greeting = "Hello, %s! You are %d years old.".formatted("Alice", 30);
        System.out.println(greeting);
        
        // ====== Map.copyOf, List.copyOf, Set.copyOf (Java 10+) ======
        Map<String, Integer> original = new HashMap<>();
        original.put("a", 1); original.put("b", 2);
        Map<String, Integer> copy = Map.copyOf(original);  // immutable copy
        
        // ====== Optional enhancements ======
        Optional<String> opt = Optional.of("value");
        opt.ifPresentOrElse(
            v -> System.out.println("Present: " + v),
            () -> System.out.println("Empty")
        );
        
        // or() - chaining optionals
        Optional<String> result = Optional.<String>empty()
            .or(() -> Optional.of("fallback"));
        System.out.println("Or: " + result);
        
        // stream()
        long count = Optional.of("hello").stream().count();
        System.out.println("Optional stream count: " + count);
        
        // ====== instanceof pattern matching (Java 16+) ======
        Object obj = "Hello World";
        if (obj instanceof String s && s.length() > 5) {
            System.out.println("Long string: " + s.toUpperCase());
        }
        
        // ====== switch expression (Java 14+) ======
        int day = LocalDate.now().getDayOfWeek().getValue();
        String type = switch (day) {
            case 6, 7 -> "Weekend";
            case 1 -> "Monday blues";
            case 5 -> "TGIF!";
            default -> "Weekday";
        };
        System.out.println("Today: " + type);
    }
}
```

---

## 25.5 Microservices Pattern

```java
// ====== Circuit Breaker Pattern ======
import java.util.concurrent.atomic.*;
import java.time.*;
import java.util.function.*;

enum CircuitState { CLOSED, OPEN, HALF_OPEN }

class CircuitBreaker<T> {
    private volatile CircuitState state = CircuitState.CLOSED;
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicInteger successCount = new AtomicInteger(0);
    private volatile Instant lastFailureTime;
    
    private final int failureThreshold;
    private final int successThreshold;
    private final Duration resetTimeout;
    private final String name;
    
    public CircuitBreaker(String name, int failureThreshold, int successThreshold,
                          Duration resetTimeout) {
        this.name = name;
        this.failureThreshold = failureThreshold;
        this.successThreshold = successThreshold;
        this.resetTimeout = resetTimeout;
    }
    
    public T execute(Supplier<T> operation, Supplier<T> fallback) {
        return switch (state) {
            case OPEN -> {
                if (shouldAttemptReset()) {
                    state = CircuitState.HALF_OPEN;
                    System.out.println("[" + name + "] HALF-OPEN: attempting reset");
                    yield executeInHalfOpen(operation, fallback);
                }
                System.out.println("[" + name + "] OPEN: using fallback");
                yield fallback.get();
            }
            case HALF_OPEN -> executeInHalfOpen(operation, fallback);
            case CLOSED -> executeInClosed(operation, fallback);
        };
    }
    
    private T executeInClosed(Supplier<T> operation, Supplier<T> fallback) {
        try {
            T result = operation.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure(e);
            return fallback.get();
        }
    }
    
    private T executeInHalfOpen(Supplier<T> operation, Supplier<T> fallback) {
        try {
            T result = operation.get();
            successCount.incrementAndGet();
            if (successCount.get() >= successThreshold) {
                state = CircuitState.CLOSED;
                failureCount.set(0);
                successCount.set(0);
                System.out.println("[" + name + "] CLOSED: service recovered");
            }
            return result;
        } catch (Exception e) {
            state = CircuitState.OPEN;
            lastFailureTime = Instant.now();
            System.out.println("[" + name + "] OPEN: reset failed");
            return fallback.get();
        }
    }
    
    private void onSuccess() { failureCount.set(0); }
    
    private void onFailure(Exception e) {
        int failures = failureCount.incrementAndGet();
        lastFailureTime = Instant.now();
        System.out.println("[" + name + "] Failure " + failures + ": " + e.getMessage());
        
        if (failures >= failureThreshold) {
            state = CircuitState.OPEN;
            System.out.println("[" + name + "] OPEN: failure threshold reached");
        }
    }
    
    private boolean shouldAttemptReset() {
        return lastFailureTime != null &&
               Duration.between(lastFailureTime, Instant.now()).compareTo(resetTimeout) > 0;
    }
    
    public CircuitState getState() { return state; }
    
    public static void main(String[] args) throws InterruptedException {
        
        CircuitBreaker<String> cb = new CircuitBreaker<>(
            "PaymentService", 3, 2, Duration.ofSeconds(2)
        );
        
        int[] callCount = {0};
        
        Supplier<String> operation = () -> {
            callCount[0]++;
            if (callCount[0] <= 5 || callCount[0] > 8) {
                throw new RuntimeException("Service unavailable");
            }
            return "Payment processed";
        };
        
        Supplier<String> fallback = () -> "Payment queued (fallback)";
        
        for (int i = 1; i <= 12; i++) {
            String result = cb.execute(operation, fallback);
            System.out.printf("Call %2d: %-30s [%s]%n", i, result, cb.getState());
            Thread.sleep(200);
        }
    }
}
```

---

## 25.6 Full Program: REST API with Rate Limiting & Caching

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.time.*;

// In-memory LRU Cache
class LRUCache<K, V> {
    private final int capacity;
    private final long ttlMs;
    private final LinkedHashMap<K, CacheEntry<V>> map;
    
    record CacheEntry<V>(V value, long expiresAt) {
        boolean isExpired() { return System.currentTimeMillis() > expiresAt; }
    }
    
    public LRUCache(int capacity, Duration ttl) {
        this.capacity = capacity;
        this.ttlMs = ttl.toMillis();
        this.map = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, CacheEntry<V>> eldest) {
                return size() > capacity;
            }
        };
    }
    
    public synchronized Optional<V> get(K key) {
        var entry = map.get(key);
        if (entry == null || entry.isExpired()) {
            map.remove(key);
            return Optional.empty();
        }
        return Optional.of(entry.value());
    }
    
    public synchronized void put(K key, V value) {
        map.put(key, new CacheEntry<>(value, System.currentTimeMillis() + ttlMs));
    }
    
    public synchronized int size() { return map.size(); }
}

// Rate Limiter (Token Bucket)
class RateLimiter {
    private final int maxTokens;
    private final double refillRate;  // tokens per second
    private double tokens;
    private long lastRefill;
    
    public RateLimiter(int maxTokens, int requestsPerSecond) {
        this.maxTokens = maxTokens;
        this.refillRate = requestsPerSecond;
        this.tokens = maxTokens;
        this.lastRefill = System.nanoTime();
    }
    
    public synchronized boolean tryAcquire() {
        refill();
        if (tokens >= 1) {
            tokens--;
            return true;
        }
        return false;
    }
    
    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefill) / 1_000_000_000.0;
        tokens = Math.min(maxTokens, tokens + elapsed * refillRate);
        lastRefill = now;
    }
}

// Simulated product service
class ProductService {
    private static final Map<Integer, Map<String, Object>> PRODUCTS = Map.of(
        1, Map.of("id", 1, "name", "Laptop", "price", 75000),
        2, Map.of("id", 2, "name", "Phone", "price", 35000),
        3, Map.of("id", 3, "name", "Tablet", "price", 28000)
    );
    
    private static final LRUCache<Integer, Map<String, Object>> cache =
        new LRUCache<>(100, Duration.ofMinutes(5));
    
    private static final AtomicLong cacheHits = new AtomicLong();
    private static final AtomicLong cacheMisses = new AtomicLong();
    
    static Map<String, Object> getProduct(int id) {
        return cache.get(id).map(p -> {
            cacheHits.incrementAndGet();
            return p;
        }).orElseGet(() -> {
            cacheMisses.incrementAndGet();
            // Simulate DB fetch
            var product = PRODUCTS.get(id);
            if (product != null) cache.put(id, product);
            return product;
        });
    }
    
    static Map<String, Long> getCacheStats() {
        return Map.of("hits", cacheHits.get(), "misses", cacheMisses.get(),
                      "size", (long) cache.size());
    }
}

public class ProductAPI {
    
    private static final RateLimiter rateLimiter = new RateLimiter(10, 5);
    
    static Map<String, Object> handleRequest(String clientId, int productId) {
        if (!rateLimiter.tryAcquire()) {
            return Map.of("error", "Rate limit exceeded", "retryAfter", 1);
        }
        
        var product = ProductService.getProduct(productId);
        if (product == null) {
            return Map.of("error", "Product not found", "id", productId);
        }
        
        return Map.of("success", true, "data", product, "client", clientId);
    }
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Product API Simulation ===\n");
        
        // Simulate multiple requests
        for (int i = 1; i <= 15; i++) {
            String client = "client" + (i % 3 + 1);
            int productId = (i % 4) + 1;  // 1-4 (4 will 404)
            
            var response = handleRequest(client, productId);
            System.out.printf("Request %2d [%s] product=%d: %s%n",
                i, client, productId,
                response.containsKey("error") ? "ERROR: " + response.get("error") :
                "OK: " + ((Map<?,?>)response.get("data")).get("name"));
            
            Thread.sleep(50);
        }
        
        System.out.println("\nCache stats: " + ProductService.getCacheStats());
    }
}
```

---

## สรุป Part 25

| Modern Java Feature | Java Version |
|--------------------|-------------|
| Records | 16 (preview 14) |
| Sealed Classes | 17 (preview 15) |
| Pattern Matching switch | 21 |
| Text Blocks | 15 |
| `instanceof` pattern | 16 |
| Switch expressions | 14 |
| `Stream.toList()` | 16 |
| `String.isBlank()` | 11 |
| `var` | 10 |
| Optional.or() | 9 |

➡️ [Part 26: Testing with JUnit 5](./Part-26-Testing-JUnit5.md)
