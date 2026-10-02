# Part 98: Code Review Best Practices & Anti-patterns
## ขั้นตอนที่ 6761-6830: Kotlin Idioms, Performance Pitfalls, Security Review Checklist

---

## 98.1 Kotlin Code Review Checklist

```kotlin
// ====== Prefer Kotlin Idioms ======

// ✗ Anti-pattern: Java-style null check
fun processUser(user: User?) {
    if (user != null) {
        val name = user.getName()
        if (name != null) {
            println(name.toUpperCase())
        }
    }
}

// ✓ Kotlin way
fun processUser(user: User?) {
    user?.name?.let { println(it.uppercase()) }
}

// ====================================

// ✗ Anti-pattern: mutable everywhere
fun getActiveUsers(users: MutableList<User>): MutableList<User> {
    val result = mutableListOf<User>()
    for (user in users) {
        if (user.isActive) result.add(user)
    }
    return result
}

// ✓ Immutable + functional
fun getActiveUsers(users: List<User>): List<User> =
    users.filter { it.isActive }

// ====================================

// ✗ Anti-pattern: excessive !!
fun processOrder(id: String): Order {
    val order = orderRepository.findById(id).orElse(null)!!  // crash if null
    val customer = customerRepository.findById(order.customerId).orElse(null)!!
    return order.copy(customerName = customer.name)
}

// ✓ Explicit error handling
fun processOrder(id: String): Order {
    val order = orderRepository.findById(id)
        .orElseThrow { OrderNotFoundException(id) }
    val customer = customerRepository.findById(order.customerId)
        .orElseThrow { CustomerNotFoundException(order.customerId) }
    return order.copy(customerName = customer.name)
}

// ====================================

// ✗ Anti-pattern: when not exhaustive
fun describe(status: OrderStatus): String {
    return when (status) {
        OrderStatus.PENDING -> "Pending"
        OrderStatus.ACTIVE -> "Active"
        else -> "Unknown"  // misses new statuses silently!
    }
}

// ✓ Exhaustive when (sealed class or remove else)
fun describe(status: OrderStatus): String = when (status) {
    OrderStatus.PENDING -> "Pending"
    OrderStatus.ACTIVE -> "Active"
    OrderStatus.COMPLETED -> "Completed"
    OrderStatus.CANCELLED -> "Cancelled"
    // Compiler error if new status added → forces developer to handle it
}
```

---

## 98.2 Performance Anti-patterns

```kotlin
// ====== String Concatenation in Loops ======

// ✗ O(n²) - creates new String each iteration
fun buildReport(items: List<Item>): String {
    var report = ""
    for (item in items) {
        report += "- ${item.name}: ${item.price}\n"  // BAD
    }
    return report
}

// ✓ StringBuilder or joinToString
fun buildReport(items: List<Item>): String =
    items.joinToString("\n") { "- ${it.name}: ${it.price}" }

// For complex cases:
fun buildReport(items: List<Item>): String = buildString {
    appendLine("Order Report:")
    items.forEach { item -> appendLine("- ${item.name}: ${item.price}") }
    appendLine("Total: ${items.sumOf { it.price }}")
}

// ====== N+1 Query Problem ======

// ✗ N+1: 1 query for orders + N queries for customers
fun getOrderSummaries(): List<OrderSummary> {
    val orders = orderRepository.findAll()  // 1 query
    return orders.map { order ->
        val customer = customerRepository.findById(order.customerId)  // N queries!
            .orElseThrow()
        OrderSummary(order, customer)
    }
}

// ✓ JOIN in single query
@Query("""
    SELECT new com.example.OrderSummary(o, c)
    FROM Order o JOIN FETCH o.customer c
    WHERE o.status = 'ACTIVE'
""")
fun findOrderSummaries(): List<OrderSummary>

// OR: batch load
fun getOrderSummaries(): List<OrderSummary> {
    val orders = orderRepository.findAll()
    val customerIds = orders.map { it.customerId }.toSet()
    val customers = customerRepository.findAllById(customerIds)
        .associateBy { it.id }  // 1 query, then lookup by map
    
    return orders.map { order ->
        OrderSummary(order, customers[order.customerId]!!)
    }
}

// ====== Unnecessary Object Creation ======

// ✗ Creates many temporary objects
fun findExpensiveItems(items: List<Item>): List<String> {
    return items
        .filter { it.price > 1000 }
        .map { it.name }
        .toList()  // already returns List, toList() is redundant
}

// ✓ Correct (toList is unnecessary here)
fun findExpensiveItems(items: List<Item>): List<String> =
    items.filter { it.price > 1000 }.map { it.name }

// For large collections: use asSequence (lazy evaluation)
fun findExpensiveItems(items: List<Item>): List<String> =
    items.asSequence()
         .filter { it.price > 1000 }
         .map { it.name }
         .toList()  // only materialize at the end

// ====== Blocking in Coroutines ======

// ✗ Blocking call in coroutine
suspend fun fetchData(): Data {
    delay(100)  // ok - suspending
    val result = someBlockingService.fetch()  // BLOCKS the thread! Bad!
    return result
}

// ✓ Use withContext(Dispatchers.IO) for blocking code
suspend fun fetchData(): Data = withContext(Dispatchers.IO) {
    someBlockingService.fetch()  // runs in IO thread pool
}
```

---

## 98.3 Security Anti-patterns

```kotlin
// ====== SQL Injection ======

// ✗ NEVER do string interpolation in SQL
@Repository
class UserRepository(private val jdbcTemplate: JdbcTemplate) {
    
    // VULNERABLE: SQL injection
    fun findByUsername(username: String): User? {
        val sql = "SELECT * FROM users WHERE username = '$username'"
        // username = "admin' OR '1'='1" → returns all users!
        return jdbcTemplate.queryForObject(sql, userRowMapper)
    }
    
    // ✓ SAFE: parameterized query
    fun findByUsername(username: String): User? =
        jdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE username = ?",
            userRowMapper,
            username  // safely parameterized
        )
}

// ====== Insecure Logging ======

// ✗ Log sensitive data
log.info("User logged in: email=${user.email}, password=${user.password}")
log.debug("Payment processed: card=${card.number}, cvv=${card.cvv}")

// ✓ Log only safe identifiers
log.info("User logged in: userId=${user.id}")
log.debug("Payment processed: orderId=${order.id}, last4=${card.last4}")

// ✓ Mask in toString/serialization
data class CreditCard(
    val number: String,
    val cvv: String
) {
    override fun toString() = "CreditCard(number=****${number.takeLast(4)}, cvv=***)"
}

// ====== Hardcoded Secrets ======

// ✗ Hardcoded credentials
val apiKey = "sk-live-abc123secret"
val dbPassword = "password123"

// ✓ From environment / Vault
val apiKey: String = System.getenv("STRIPE_API_KEY")
    ?: throw IllegalStateException("STRIPE_API_KEY environment variable required")

// ====== Path Traversal ======

// ✗ Path traversal vulnerability
@GetMapping("/files/{filename}")
fun downloadFile(@PathVariable filename: String): ResponseEntity<Resource> {
    val file = File("/uploads/$filename")  // ../../etc/passwd
    return ResponseEntity.ok(FileSystemResource(file))
}

// ✓ Normalize and validate path
@GetMapping("/files/{filename}")
fun downloadFile(@PathVariable filename: String): ResponseEntity<Resource> {
    val uploadDir = Path.of("/uploads").toAbsolutePath().normalize()
    val filePath = uploadDir.resolve(filename).normalize()
    
    // Ensure file is within upload directory
    if (!filePath.startsWith(uploadDir)) {
        throw AccessDeniedException("Invalid file path")
    }
    
    if (!Files.exists(filePath)) throw NoSuchFileException(filename)
    
    return ResponseEntity.ok(FileSystemResource(filePath.toFile()))
}

// ====== IDOR (Insecure Direct Object Reference) ======

// ✗ No authorization check
@GetMapping("/orders/{orderId}")
fun getOrder(@PathVariable orderId: String): Order {
    return orderRepository.findById(orderId).orElseThrow()
    // Any user can see ANY order by guessing ID!
}

// ✓ Verify ownership
@GetMapping("/orders/{orderId}")
fun getOrder(
    @PathVariable orderId: String,
    @AuthenticationPrincipal principal: JwtAuthenticationToken
): Order {
    val order = orderRepository.findById(orderId).orElseThrow()
    
    if (order.customerId != principal.name) {  // check ownership
        throw AccessDeniedException("Order does not belong to this user")
    }
    
    return order
}
```

---

## 98.4 Code Smell Detection

```kotlin
// ====== Long Method ======
// If function > 30 lines, consider extracting

// ✗ Too much in one method
fun processCheckout(cart: Cart, payment: Payment, user: User): Order {
    // validate cart
    if (cart.items.isEmpty()) throw EmptyCartException()
    for (item in cart.items) {
        if (item.quantity <= 0) throw InvalidQuantityException(item.id)
        if (item.price < 0) throw InvalidPriceException(item.id)
    }
    
    // check inventory
    for (item in cart.items) {
        val stock = inventoryService.getStock(item.productId)
        if (stock < item.quantity) throw InsufficientStockException(item.productId)
    }
    
    // calculate pricing
    val subtotal = cart.items.sumOf { it.price * it.quantity }
    val tax = subtotal * 0.07
    val discount = if (user.isPremium) subtotal * 0.1 else 0.0
    val total = subtotal + tax - discount
    
    // process payment
    val paymentResult = paymentGateway.charge(payment, total)
    if (!paymentResult.success) throw PaymentFailedException(paymentResult.errorCode)
    
    // create order
    val order = orderRepository.save(Order(
        userId = user.id,
        items = cart.items,
        total = total,
        paymentId = paymentResult.id
    ))
    
    // send notifications
    emailService.sendOrderConfirmation(user.email, order)
    smsService.sendSmsNotification(user.phone, "Order ${order.id} confirmed")
    
    // update inventory
    for (item in cart.items) {
        inventoryService.decrementStock(item.productId, item.quantity)
    }
    
    return order
}

// ✓ Extract meaningful steps
fun processCheckout(cart: Cart, payment: Payment, user: User): Order {
    validateCart(cart)
    checkInventoryAvailability(cart.items)
    val pricing = calculatePricing(cart, user)
    val paymentResult = chargePayment(payment, pricing.total)
    val order = createOrder(user, cart, pricing, paymentResult)
    notifyCustomer(user, order)
    decrementInventory(cart.items)
    return order
}

// ====== Feature Envy ======
// Method that uses another class's data more than its own

// ✗ Feature envy
class OrderProcessor {
    fun calculateDiscount(customer: Customer): BigDecimal {
        // This knows too much about Customer internals
        val base = if (customer.loyaltyLevel == "GOLD") 0.15
                   else if (customer.loyaltyLevel == "SILVER") 0.10
                   else if (customer.purchaseCount > 50) 0.05
                   else 0.0
        
        val loyaltyBonus = customer.points / 1000 * 0.01
        return BigDecimal(base + loyaltyBonus)
    }
}

// ✓ Move to where the data is
class Customer {
    fun calculateDiscount(): BigDecimal {
        val base = when (loyaltyLevel) {
            "GOLD" -> 0.15
            "SILVER" -> 0.10
            else -> if (purchaseCount > 50) 0.05 else 0.0
        }
        val loyaltyBonus = points / 1000 * 0.01
        return BigDecimal(base + loyaltyBonus)
    }
}

// ====== Primitive Obsession ======

// ✗ Using primitives for domain concepts
fun transferMoney(fromAccountId: String, toAccountId: String, amount: Double) { }

// ✓ Value objects
@JvmInline
value class AccountId(val value: String)

@JvmInline  
value class Money(val cents: Long) {
    operator fun plus(other: Money) = Money(cents + other.cents)
    fun toDisplay() = "฿${cents / 100}.${cents % 100}"
}

fun transferMoney(from: AccountId, to: AccountId, amount: Money) { }
```

---

## 98.5 Review Checklist Template

```
Code Review Checklist:

Correctness:
  □ Does the code do what it's supposed to do?
  □ Edge cases handled? (null, empty, zero, negative, max values)
  □ Thread safety? (shared mutable state, race conditions)
  □ Error handling? (exceptions caught appropriately)

Performance:
  □ No N+1 queries
  □ Indexes on query fields
  □ No memory leaks (resources closed, listeners removed)
  □ No blocking I/O in async context
  □ String building in loops uses StringBuilder
  □ Large collection uses lazy evaluation (Sequence/Stream)

Security:
  □ No SQL injection (parameterized queries)
  □ No sensitive data in logs
  □ Authorization check on every data access
  □ No path traversal vulnerabilities
  □ Input validation at API boundaries
  □ No hardcoded secrets
  □ Rate limiting on public endpoints

Maintainability:
  □ Single responsibility (functions do one thing)
  □ Meaningful names (no single-letter vars except loop indices)
  □ No magic numbers (use constants)
  □ Exhaustive when/switch (no hidden else for sealed classes)
  □ Tests for new/changed logic
  □ Breaking changes documented

Kotlin Idioms:
  □ Use data class instead of verbose POJO
  □ Use sealed class for restricted hierarchies
  □ Use extension functions to add behavior
  □ Prefer immutable collections
  □ Use stdlib functions (filter, map, groupBy, etc.)
  □ Safe calls (?.) instead of null checks
  □ Elvis operator (?:) for defaults
  □ Use scope functions appropriately (let, run, also, apply, with)
```

---

## สรุป Part 98

```
Code Review Best Practices:

High-Impact Issues (always fix):
  SQL injection: use parameterized queries
  Sensitive data in logs: mask PII, secrets
  No authorization check: IDOR vulnerability
  N+1 queries: JOIN or batch load
  Blocking in coroutines: use withContext(IO)
  Path traversal: normalize and validate paths

Medium Impact:
  Exhaustive when: remove else for sealed classes
  Long methods: extract to named functions
  Feature envy: move logic to where data lives
  Mutable shared state: use synchronized or atomic

Kotlin Code Quality:
  Use ?.let{} instead of if (x != null)
  Prefer val over var
  Use stdlib functions (filter/map/groupBy)
  Value classes for domain primitives
  sealed class + exhaustive when

Performance Review:
  Look for: + in loop, N+1, missing index
  Verify: async/reactive code doesn't block
  Check: large data uses streaming/pagination

Review Mindset:
  Ask "what happens when X is null?"
  Ask "what if this is called concurrently?"
  Ask "what if the user is malicious?"
  Ask "what if this is called with 1M records?"
  Praise good code too, not only critique
  Suggest, don't demand (unless it's a bug)
```

➡️ [Part 99: Complete E-Commerce Project](./Part-99-ECommerceProject.md)
