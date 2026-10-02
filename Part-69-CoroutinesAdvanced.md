# Part 69: Kotlin Coroutines Advanced Patterns
## ขั้นตอนที่ 4731-4800: Channels, Select, Actor Pattern, Flow Operators

---

## 69.1 Channels

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

// Channel = concurrent queue สำหรับ communication ระหว่าง coroutines
// เหมือน Go channels

fun main() = runBlocking {
    
    // Basic Channel
    val channel = Channel<Int>()
    
    // Producer coroutine
    launch {
        for (i in 1..5) {
            channel.send(i)          // suspend ถ้า channel เต็ม
            println("Sent: $i")
        }
        channel.close()              // บอกว่าส่งหมดแล้ว
    }
    
    // Consumer coroutine
    launch {
        for (value in channel) {     // รับจนกว่า channel จะปิด
            println("Received: $value")
            delay(100)
        }
    }
}

// Channel types
fun channelTypes() = runBlocking {
    
    // RENDEZVOUS (default): ไม่มี buffer, producer รอจนมี consumer
    val rendezvous = Channel<String>()
    
    // BUFFERED: buffer N items, producer suspend เมื่อเต็ม
    val buffered = Channel<String>(capacity = 10)
    
    // UNLIMITED: ไม่มีขีดจำกัด (ระวัง OOM)
    val unlimited = Channel<String>(Channel.UNLIMITED)
    
    // CONFLATED: ถ้า consumer ช้า → เก็บแค่ค่าล่าสุด (drop old values)
    val conflated = Channel<String>(Channel.CONFLATED)
}

// Producer pattern using produce
fun numbers(from: Int, to: Int): ReceiveChannel<Int> = GlobalScope.produce {
    for (i in from..to) {
        send(i)
        delay(100)
    }
}

// Fan-out: multiple consumers from one channel
fun fanOut() = runBlocking {
    val numbers = produce { for (i in 1..20) send(i) }
    
    // 3 consumers sharing the same channel
    val jobs = List(3) { workerId ->
        launch {
            for (num in numbers) {
                println("Worker $workerId processing $num")
                delay(200)
            }
        }
    }
    
    jobs.forEach { it.join() }
}
```

---

## 69.2 Flow Advanced Operators

```kotlin
import kotlinx.coroutines.flow.*

// Flow = async stream of values (cold - starts only when collected)

fun stockPriceFlow(): Flow<Double> = flow {
    while (true) {
        emit(getLatestStockPrice())  // produce value
        delay(1000)                  // every second
    }
}

fun flowOperatorsDemo() = runBlocking {
    
    // transform: one value → multiple values (or none)
    (1..10).asFlow()
        .transform { value ->
            if (value % 2 == 0) {
                emit(value)
                emit(value * 10)
            }
        }
        .collect { println(it) }
    
    // flatMapConcat: one value → Flow, concat sequentially
    (1..3).asFlow()
        .flatMapConcat { id -> loadUserDetails(id) }
        .collect { println(it) }
    
    // flatMapMerge: concurrent (up to concurrency limit)
    (1..10).asFlow()
        .flatMapMerge(concurrency = 3) { id -> loadUserDetails(id) }
        .collect { println(it) }
    
    // combine: merge latest values from 2 flows
    val temperatures = flow { emit(25.0); delay(500); emit(26.5) }
    val humidity = flow { emit(65); delay(300); emit(70) }
    
    temperatures.combine(humidity) { temp, hum ->
        "Temp: $temp°C, Humidity: $hum%"
    }.collect { println(it) }
    
    // zip: pair values one-to-one
    val names = flowOf("Alice", "Bob", "Charlie")
    val scores = flowOf(90, 85, 92)
    
    names.zip(scores) { name, score -> "$name: $score" }
        .collect { println(it) }
    
    // buffer: allow producer to run ahead of collector
    stockPriceFlow()
        .buffer(capacity = 10)     // producer can emit 10 ahead
        .collect { price -> processPrice(price) }
    
    // conflate: drop missed values when collector is slow
    stockPriceFlow()
        .conflate()                // keep only latest when busy
        .collect { price -> slowProcessPrice(price) }
    
    // debounce: only emit after quiet period (search bar)
    searchQueryFlow()
        .debounce(300)             // wait 300ms after last input
        .distinctUntilChanged()    // ignore duplicate queries
        .filter { it.length >= 2 } // at least 2 characters
        .flatMapLatest { query ->  // cancel previous search if new query arrives
            searchProducts(query)
        }
        .collect { results -> displayResults(results) }
}

fun loadUserDetails(id: Int): Flow<String> = flow { emit("User $id details") }
fun searchProducts(q: String): Flow<List<String>> = flow { emit(listOf("Product for $q")) }
fun searchQueryFlow(): Flow<String> = flow {}
fun getLatestStockPrice(): Double = Math.random() * 100
suspend fun processPrice(price: Double) { }
suspend fun slowProcessPrice(price: Double) { delay(500) }
suspend fun displayResults(results: List<String>) { }
```

---

## 69.3 StateFlow & SharedFlow

```kotlin
import kotlinx.coroutines.flow.*

// StateFlow: hot flow with current state (like LiveData)
// SharedFlow: hot flow for events (broadcast)

class OrderViewModel {
    
    // StateFlow: always has a value, replays latest to new collectors
    private val _orderState = MutableStateFlow<OrderState>(OrderState.Idle)
    val orderState: StateFlow<OrderState> = _orderState.asStateFlow()
    
    // SharedFlow: one-time events (navigation, snackbar)
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()
    
    fun placeOrder(request: OrderRequest) {
        CoroutineScope(Dispatchers.IO).launch {
            _orderState.value = OrderState.Loading
            
            try {
                val order = orderRepository.create(request)
                _orderState.value = OrderState.Success(order)
                _events.emit(UiEvent.ShowToast("Order placed! #${order.id}"))
                _events.emit(UiEvent.Navigate("order-confirmation/${order.id}"))
            } catch (e: Exception) {
                _orderState.value = OrderState.Error(e.message ?: "Unknown error")
                _events.emit(UiEvent.ShowError(e.message ?: "Failed to place order"))
            }
        }
    }
}

sealed class OrderState {
    object Idle : OrderState()
    object Loading : OrderState()
    data class Success(val order: Order) : OrderState()
    data class Error(val message: String) : OrderState()
}

sealed class UiEvent {
    data class ShowToast(val message: String) : UiEvent()
    data class ShowError(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
}

// Collect in Composable (Android)
@Composable
fun OrderScreen(viewModel: OrderViewModel) {
    val orderState by viewModel.orderState.collectAsStateWithLifecycle()
    
    // Collect one-time events
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowToast -> showToast(event.message)
                is UiEvent.Navigate  -> navController.navigate(event.route)
                is UiEvent.ShowError -> showErrorDialog(event.message)
            }
        }
    }
    
    when (orderState) {
        is OrderState.Loading -> CircularProgressIndicator()
        is OrderState.Success -> SuccessContent((orderState as OrderState.Success).order)
        is OrderState.Error   -> ErrorContent((orderState as OrderState.Error).message)
        OrderState.Idle       -> IdleContent()
    }
}
```

---

## 69.4 Select Expression

```kotlin
import kotlinx.coroutines.selects.*

// select: wait for first available coroutine result (like Haskell's STM)

suspend fun fetchFastest(userId: String): String = coroutineScope {
    
    val cache = async { fetchFromCache(userId) }
    val db = async { fetchFromDatabase(userId) }
    val api = async { fetchFromApi(userId) }
    
    // Return whichever completes first
    select {
        cache.onAwait { result ->
            db.cancel(); api.cancel()
            "CACHE: $result"
        }
        db.onAwait { result ->
            cache.cancel(); api.cancel()
            "DB: $result"
        }
        api.onAwait { result ->
            cache.cancel(); db.cancel()
            "API: $result"
        }
    }
}

// select with channels: process whichever channel has data
suspend fun mergeChannels(ch1: ReceiveChannel<Int>, ch2: ReceiveChannel<Int>) = coroutineScope {
    val output = Channel<Int>()
    
    launch {
        while (true) {
            select<Unit> {
                ch1.onReceive { value -> output.send(value) }
                ch2.onReceive { value -> output.send(value) }
            }
        }
    }
    
    output
}

suspend fun fetchFromCache(id: String): String { delay(50); return "user-from-cache" }
suspend fun fetchFromDatabase(id: String): String { delay(100); return "user-from-db" }
suspend fun fetchFromApi(id: String): String { delay(200); return "user-from-api" }
```

---

## 69.5 Coroutine Context & Dispatchers

```kotlin
import kotlinx.coroutines.*

// Dispatcher: กำหนดว่า coroutine รันบน thread pool ไหน
// Dispatchers.Default  = CPU-intensive (= number of CPU cores)
// Dispatchers.IO       = I/O operations (expandable pool, default 64 threads)
// Dispatchers.Main     = UI thread (Android/JavaFX)
// Dispatchers.Unconfined = no specific thread (avoid in production)

fun dispatcherDemo() = runBlocking {
    
    // CPU-intensive: computation, parsing
    val result = withContext(Dispatchers.Default) {
        (1..10_000_000).sum()
    }
    
    // I/O: database, network, file
    val data = withContext(Dispatchers.IO) {
        readFileFromDisk("data.json")
    }
    
    // Custom dispatcher with limited parallelism
    val singleThread = Executors.newSingleThreadExecutor().asCoroutineDispatcher()
    withContext(singleThread) {
        // exclusive access to non-thread-safe resource
    }
}

// CoroutineContext: key-value metadata
fun contextDemo() = runBlocking {
    
    // Job: lifecycle management
    val job = launch {
        repeat(10) { i ->
            delay(100)
            println("Task $i")
        }
    }
    
    delay(350)
    job.cancel()  // cancels at next suspension point
    job.join()
    
    // CoroutineName: for debugging
    launch(CoroutineName("user-loader") + Dispatchers.IO) {
        println("Running in: ${coroutineContext[CoroutineName]?.name}")
        // Loading user from DB...
    }
    
    // Combine contexts
    val customContext = Dispatchers.IO + CoroutineName("api-call") + Job()
    launch(customContext) { /* ... */ }
}

// Exception handling
fun exceptionHandling() = runBlocking {
    
    // CoroutineExceptionHandler: catch unhandled exceptions
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught: ${exception.message}")
    }
    
    val scope = CoroutineScope(Dispatchers.IO + handler)
    
    scope.launch {
        throw RuntimeException("Something went wrong")
    }
    
    delay(100)
    
    // supervisorScope: child failure doesn't cancel siblings
    supervisorScope {
        val job1 = launch { delay(1000); println("Job 1 done") }
        val job2 = launch { throw RuntimeException("Job 2 failed") }
        val job3 = launch { delay(500); println("Job 3 done") }
        
        // job2 fails, but job1 and job3 continue
    }
}
```

---

## สรุป Part 69

```
Coroutines Advanced Summary:

Channels:
  Channel<T>() = concurrent queue (suspend on full/empty)
  produce { } = easy channel producer
  RENDEZVOUS | BUFFERED | UNLIMITED | CONFLATED
  Fan-out: multiple consumers from one channel

Flow Operators:
  transform     = 1 value → N values
  flatMapLatest = cancel previous on new value (search bar)
  debounce      = wait for quiet period
  conflate      = drop old, keep latest (slow consumer)
  combine       = merge 2 flows (latest from each)
  zip           = pair values 1:1

StateFlow vs SharedFlow:
  StateFlow  = always has value, replay=1, use for state
  SharedFlow = no default value, configurable replay, use for events

Dispatchers:
  Default = CPU cores (math, parsing)
  IO      = file/DB/network (64 threads default)
  Main    = UI thread

select { }:
  Wait for first available: Deferred.onAwait or Channel.onReceive
  Cancels others (or handle each manually)

Best Practices:
  ✓ viewModelScope / lifecycleScope (auto-cancel)
  ✓ supervisorScope when children independent
  ✓ withContext(Dispatchers.IO) for blocking calls
  ✗ GlobalScope (leaks)
  ✗ runBlocking in coroutines (deadlock risk)
```

➡️ [Part 70: Chaos Engineering & Resilience Testing](./Part-70-ChaosEngineering.md)
