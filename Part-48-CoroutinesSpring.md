# Part 48: Coroutines in Spring Boot
## ขั้นตอนที่ 3261-3330: Kotlin Coroutines with Spring

---

## 48.1 Coroutines in Spring

```kotlin
// Spring WebFlux + Coroutines = best of both worlds
// Write sequential-looking code, run non-blocking

// Reactive (hard to read):
fun getUser(id: Long): Mono<UserDTO> =
    userRepository.findById(id)
        .flatMap { user ->
            orderRepository.findByUserId(user.id)
                .collectList()
                .map { orders -> UserDTO(user, orders) }
        }

// Coroutines (easy to read):
suspend fun getUser(id: Long): UserDTO {
    val user = userRepository.findById(id) ?: throw NotFoundException("User: $id")
    val orders = orderRepository.findByUserId(user.id)
    return UserDTO(user, orders)
}
```

---

## 48.2 Coroutine-based Controllers

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1")
class UserController(
    private val userService: UserService,
    private val orderService: OrderService
) {
    
    // suspend fun = non-blocking (Spring handles the coroutine scope)
    @GetMapping("/users/{id}")
    suspend fun getUser(@PathVariable id: Long): UserDTO {
        return userService.findById(id)
            ?: throw ResponseStatusException(org.springframework.http.HttpStatus.NOT_FOUND)
    }
    
    // Flow<T> = streaming (like Flux but Kotlin-native)
    @GetMapping("/users", produces = ["application/json"])
    fun listUsers(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): Flow<UserDTO> {
        return userService.findAll(page, size)
    }
    
    // Streaming with SSE
    @GetMapping("/users/stream", produces = ["text/event-stream"])
    fun streamUsers(): Flow<UserDTO> {
        return flow {
            userService.findAll(0, 1000).collect { emit(it) }
        }
    }
    
    // Parallel calls
    @GetMapping("/dashboard")
    suspend fun getDashboard(
        @org.springframework.security.core.annotation.AuthenticationPrincipal
        principal: org.springframework.security.core.userdetails.UserDetails
    ): DashboardDTO {
        return coroutineScope {
            val userDeferred = async { userService.findByUsername(principal.username) }
            val ordersDeferred = async { orderService.getRecentOrders(principal.username) }
            val statsDeferred = async { userService.getStats(principal.username) }
            
            // All three run concurrently!
            DashboardDTO(
                user = userDeferred.await(),
                recentOrders = ordersDeferred.await(),
                stats = statsDeferred.await()
            )
        }
    }
    
    @PostMapping("/orders")
    suspend fun createOrder(
        @RequestBody request: CreateOrderRequest,
        @org.springframework.security.core.annotation.AuthenticationPrincipal
        principal: org.springframework.security.core.userdetails.UserDetails
    ): org.springframework.http.ResponseEntity<OrderDTO> {
        val order = orderService.create(principal.username, request)
        return org.springframework.http.ResponseEntity
            .status(201)
            .body(order)
    }
}

data class DashboardDTO(
    val user: UserDTO?,
    val recentOrders: List<OrderDTO>,
    val stats: UserStats
)
```

---

## 48.3 Coroutine-based Repository (R2DBC)

```kotlin
import kotlinx.coroutines.flow.Flow
import org.springframework.data.repository.kotlin.CoroutineCrudRepository
import org.springframework.data.r2dbc.repository.Query

// CoroutineCrudRepository = R2DBC repository with coroutine support
interface UserRepository : CoroutineCrudRepository<UserEntity, Long> {
    
    suspend fun findByEmail(email: String): UserEntity?
    
    fun findByStatus(status: String): Flow<UserEntity>
    
    @Query("SELECT * FROM users WHERE role = :role ORDER BY created_at DESC LIMIT :limit")
    fun findByRoleLatest(role: String, limit: Int): Flow<UserEntity>
    
    suspend fun countByStatus(status: String): Long
    
    suspend fun existsByEmail(email: String): Boolean
}

// Service using coroutine repository
@org.springframework.stereotype.Service
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: org.springframework.security.crypto.password.PasswordEncoder
) {
    
    suspend fun findById(id: Long): UserDTO? =
        userRepository.findById(id)?.toDTO()
    
    suspend fun findByEmail(email: String): UserDTO? =
        userRepository.findByEmail(email)?.toDTO()
    
    fun findAll(page: Int, size: Int): Flow<UserDTO> =
        userRepository.findAll()
            .drop(page * size)
            .take(size)
            .map { it.toDTO() }
    
    @org.springframework.transaction.annotation.Transactional
    suspend fun create(request: CreateUserRequest): UserDTO {
        if (userRepository.existsByEmail(request.email)) {
            throw IllegalArgumentException("Email already registered: ${request.email}")
        }
        
        val entity = UserEntity(
            name = request.name,
            email = request.email,
            password = passwordEncoder.encode(request.password),
            role = "USER"
        )
        
        return userRepository.save(entity).toDTO()
    }
    
    private fun UserEntity.toDTO() = UserDTO(id!!, name, email, role)
}
```

---

## 48.4 Structured Concurrency

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// All launched coroutines are children of the scope
// If one fails, all siblings are cancelled
suspend fun processOrders(orderIds: List<Long>): List<OrderResult> = coroutineScope {
    
    orderIds
        .map { id ->
            async {
                try {
                    val order = fetchOrder(id)
                    OrderResult.Success(id, order)
                } catch (e: Exception) {
                    OrderResult.Failure(id, e.message ?: "Unknown error")
                }
            }
        }
        .awaitAll()
}

// Parallel with rate limiting
suspend fun processWithLimit(items: List<String>, concurrency: Int = 10): List<String> =
    items.chunked(concurrency).flatMap { chunk ->
        coroutineScope {
            chunk.map { item ->
                async { processItem(item) }
            }.awaitAll()
        }
    }

// Timeout and cancellation
suspend fun fetchWithTimeout(orderId: Long): OrderDTO? =
    withTimeoutOrNull(5_000) {  // 5 second timeout
        fetchOrder(orderId)
    }

// Flow operators
fun userActivityStream(userId: Long): Flow<ActivityEvent> = flow {
    repeat(Int.MAX_VALUE) {
        delay(1000)
        emit(ActivityEvent(userId, System.currentTimeMillis()))
    }
}
    .buffer(capacity = 64)                  // buffered channel
    .conflate()                              // drop old if consumer is slow
    .distinctUntilChanged()                  // skip duplicates
    .debounce(500)                          // wait 500ms after last emission
    .catch { e -> emit(ActivityEvent.error(e)) }  // handle errors in flow

// Combining flows
suspend fun getUserActivity(userId: Long): Flow<DashboardUpdate> {
    val profileUpdates = userProfileUpdates(userId)
    val orderUpdates = orderStatusUpdates(userId)
    val notifications = notificationStream(userId)
    
    return merge(profileUpdates, orderUpdates, notifications)
        .onEach { update -> logActivity(update) }
}

// Channel-based producer/consumer
fun <T> batched(source: Flow<T>, batchSize: Int = 100, timeoutMs: Long = 500): Flow<List<T>> =
    source.chunked(batchSize)

// StateFlow: state holder (like LiveData)
@org.springframework.stereotype.Service
class OrderStatusService {
    
    private val _orderStatus = MutableStateFlow<Map<String, String>>(emptyMap())
    val orderStatus: StateFlow<Map<String, String>> = _orderStatus.asStateFlow()
    
    fun updateStatus(orderId: String, status: String) {
        _orderStatus.update { current ->
            current + (orderId to status)
        }
    }
    
    // SharedFlow: events (not state)
    private val _events = MutableSharedFlow<OrderEvent>(replay = 0, extraBufferCapacity = 64)
    val events: SharedFlow<OrderEvent> = _events.asSharedFlow()
    
    suspend fun publishEvent(event: OrderEvent) {
        _events.emit(event)
    }
}

data class ActivityEvent(val userId: Long, val timestamp: Long) {
    companion object {
        fun error(e: Throwable) = ActivityEvent(-1, -1)
    }
}

suspend fun fetchOrder(id: Long): OrderDTO = TODO()
suspend fun processItem(item: String): String = item.uppercase()
fun userProfileUpdates(userId: Long): Flow<DashboardUpdate> = emptyFlow()
fun orderStatusUpdates(userId: Long): Flow<DashboardUpdate> = emptyFlow()
fun notificationStream(userId: Long): Flow<DashboardUpdate> = emptyFlow()
suspend fun logActivity(update: DashboardUpdate) {}

sealed class OrderResult {
    data class Success(val id: Long, val order: OrderDTO) : OrderResult()
    data class Failure(val id: Long, val error: String) : OrderResult()
}
sealed class DashboardUpdate
class OrderEvent
```

---

## 48.5 Testing Coroutines

```kotlin
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.Test
import kotlin.test.*

class UserServiceCoroutineTest {
    
    private val userRepository = mockk<UserRepository>()
    private val service = UserService(userRepository, mockPasswordEncoder())
    
    @Test
    fun `findById returns user when exists`() = runTest {
        val entity = UserEntity(1L, "Alice", "alice@test.com", "USER")
        coEvery { userRepository.findById(1L) } returns entity
        
        val result = service.findById(1L)
        
        assertNotNull(result)
        assertEquals("Alice", result.name)
        coVerify { userRepository.findById(1L) }
    }
    
    @Test
    fun `create throws when email exists`() = runTest {
        coEvery { userRepository.existsByEmail("alice@test.com") } returns true
        
        assertFailsWith<IllegalArgumentException> {
            service.create(CreateUserRequest("Alice", "alice@test.com", "password"))
        }
        
        coVerify(exactly = 0) { userRepository.save(any()) }
    }
    
    @Test
    fun `concurrent requests complete correctly`() = runTest {
        val userIds = (1L..10L).toList()
        coEvery { userRepository.findById(any()) } answers {
            delay(100)  // simulate DB delay
            UserEntity(firstArg(), "User ${firstArg()}", "user@test.com", "USER")
        }
        
        // Run all concurrently
        val results = coroutineScope {
            userIds.map { id -> async { service.findById(id) } }.awaitAll()
        }
        
        assertEquals(10, results.size)
        assertTrue(results.all { it != null })
    }
    
    @Test
    fun `flow emits correct items`() = runTest {
        val entities = (1..5).map { UserEntity(it.toLong(), "User $it", "u$it@test.com", "USER") }
        every { userRepository.findAll() } returns entities.asFlow()
        
        val results = service.findAll(0, 5).toList()
        
        assertEquals(5, results.size)
        assertEquals("User 1", results.first().name)
    }
    
    // Test with virtual time
    @Test
    fun `timeout cancels slow operation`() = runTest {
        coEvery { userRepository.findById(1L) } coAnswers {
            delay(10_000)  // simulate very slow query
            UserEntity(1L, "Alice", "alice@test.com", "USER")
        }
        
        val result = withTimeoutOrNull(1_000) {
            service.findById(1L)
        }
        
        assertNull(result)  // timed out
    }
    
    private fun mockPasswordEncoder(): org.springframework.security.crypto.password.PasswordEncoder =
        mockk { every { encode(any()) } answers { "hashed:${firstArg<String>()}" } }
}

// Use io.mockk:mockk for coroutine mocking
fun <T> mockk(block: T.() -> Unit = {}): T = io.mockk.mockk(block = block)
fun coEvery(block: suspend () -> Unit) = io.mockk.coEvery { block() }
fun coVerify(exactly: Int = 1, block: suspend () -> Unit) = io.mockk.coVerify(exactly = exactly) { block() }
fun every(block: () -> Unit) = io.mockk.every { block() }
```

---

## สรุป Part 48

```kotlin
Coroutines + Spring Key Points:

// Suspend functions = non-blocking, sequential code
suspend fun doWork(): Result { ... }

// Flow<T> = cold stream (Kotlin equivalent of Flux<T>)
fun streamData(): Flow<Data> = flow { emit(...) }

// Parallel with coroutineScope + async
val result = coroutineScope {
    val a = async { fetchA() }
    val b = async { fetchB() }
    combine(a.await(), b.await())
}

// Spring auto-converts:
//   suspend fun → Mono<T>
//   Flow<T>     → Flux<T>

// CoroutineCrudRepository = reactive JPA for Kotlin

Testing:
  runTest { ... }   = test coroutines with virtual time
  coEvery { ... }   = mock suspend functions (MockK)
```

➡️ [Part 49: Message-Driven Architecture](./Part-49-MessageDriven.md)
