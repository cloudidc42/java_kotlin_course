# Part 22: Kotlin Coroutines
## ขั้นตอนที่ 1431-1500: Async Programming ด้วย Coroutines

---

## 22.1 Coroutines คืออะไร?

```
Thread vs Coroutine:

Thread:
  - OS-managed
  - Heavy (~1MB stack)
  - Context switch expensive
  - จำกัดจำนวน (ปกติ 1000s)

Coroutine:
  - Kotlin runtime-managed
  - Lightweight (~1KB)
  - Suspend and resume at checkpoints
  - สร้างได้ millions
  
"Thread ที่เบากว่ามาก และ resume ได้"
```

---

## 22.2 Coroutine Basics

```kotlin
import kotlinx.coroutines.*

// Build.gradle.kts:
// implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")

suspend fun fetchData(id: Int): String {
    delay(100)  // suspend function: doesn't block thread
    return "Data for $id"
}

suspend fun processData(data: String): String {
    delay(50)
    return data.uppercase()
}

fun main() = runBlocking {  // creates coroutine scope, blocks current thread
    
    // ====== launch: fire-and-forget ======
    println("Start")
    
    val job = launch {
        println("Coroutine started")
        delay(500)
        println("Coroutine ended")
    }
    
    println("After launch (non-blocking)")
    job.join()  // wait for job to complete
    println("After join")
    
    // ====== async: returns Deferred<T> ======
    println("\n=== async/await ===")
    
    val deferred = async {
        fetchData(1)
    }
    
    // Do other work while waiting...
    println("Doing other work...")
    
    val result = deferred.await()  // get result (suspends)
    println("Result: $result")
    
    // ====== Parallel execution ======
    println("\n=== Parallel ===")
    
    val start = System.currentTimeMillis()
    
    // Sequential (slow)
    val seq1 = fetchData(1)
    val seq2 = fetchData(2)
    val seq3 = fetchData(3)
    println("Sequential: ${System.currentTimeMillis() - start}ms")
    
    // Parallel (fast)
    val parStart = System.currentTimeMillis()
    val d1 = async { fetchData(1) }
    val d2 = async { fetchData(2) }
    val d3 = async { fetchData(3) }
    val (r1, r2, r3) = Triple(d1.await(), d2.await(), d3.await())
    println("Parallel: ${System.currentTimeMillis() - parStart}ms")
    println("Results: $r1, $r2, $r3")
    
    // awaitAll
    val parStart2 = System.currentTimeMillis()
    val deferreds = (1..10).map { async { fetchData(it) } }
    val results = deferreds.awaitAll()
    println("10 parallel fetches: ${System.currentTimeMillis() - parStart2}ms")
    println("Count: ${results.size}")
}
```

---

## 22.3 Coroutine Scope & Context

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    
    // ====== CoroutineScope ======
    val scope = CoroutineScope(Dispatchers.Default)
    
    val job = scope.launch {
        repeat(5) {
            println("Working... $it")
            delay(100)
        }
    }
    
    job.join()
    scope.cancel()  // cancel all children
    
    // ====== Dispatchers ======
    launch(Dispatchers.Main) {
        // UI operations (Android)
        // Not available in desktop
    }
    
    launch(Dispatchers.IO) {
        // I/O operations: file, network, database
        println("IO: ${Thread.currentThread().name}")
        // simulate I/O
        delay(100)
    }
    
    launch(Dispatchers.Default) {
        // CPU intensive: sorting, calculations
        println("Default: ${Thread.currentThread().name}")
        val sum = (1L..1_000_000L).sum()
        println("Sum: $sum")
    }
    
    launch(Dispatchers.Unconfined) {
        // not confined to any thread
        println("Unconfined 1: ${Thread.currentThread().name}")
        delay(100)
        println("Unconfined 2: ${Thread.currentThread().name}")  // may change!
    }
    
    // Custom dispatcher
    val singleThread = newSingleThreadContext("MyThread")
    launch(singleThread) {
        println("Custom: ${Thread.currentThread().name}")
    }
    singleThread.close()
    
    delay(500)
    
    // ====== withContext: switch context ======
    suspend fun loadAndProcess(): String {
        val data = withContext(Dispatchers.IO) {
            // Runs on IO thread
            "loaded data"
        }
        return withContext(Dispatchers.Default) {
            // Runs on Default thread
            data.uppercase()
        }
    }
    
    println("Processed: ${loadAndProcess()}")
    
    // ====== CoroutineContext ======
    launch(CoroutineName("MyCoroutine") + Dispatchers.Default) {
        println("Context name: ${coroutineContext[CoroutineName]?.name}")
    }
    
    delay(100)
}
```

---

## 22.4 Cancellation & Timeouts

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    
    // ====== Cancellation ======
    println("=== Cancellation ===")
    
    val job = launch {
        try {
            repeat(100) {
                println("Working $it")
                delay(100)
            }
        } catch (e: CancellationException) {
            println("Cancelled!")
        } finally {
            println("Cleanup (always runs)")
        }
    }
    
    delay(350)
    println("Cancelling...")
    job.cancel()
    job.join()
    println("Done")
    
    // ====== isActive check ======
    println("\n=== isActive check ===")
    val cpuJob = launch(Dispatchers.Default) {
        var sum = 0L
        for (i in 1..1_000_000_000L) {
            if (!isActive) {
                println("Detected cancellation at $i")
                break
            }
            sum += i
        }
        println("Sum: $sum")
    }
    delay(100)
    cpuJob.cancel()
    cpuJob.join()
    
    // ====== withTimeout ======
    println("\n=== withTimeout ===")
    try {
        withTimeout(200) {
            println("Starting long operation...")
            delay(1000)  // will be cancelled
            println("This won't print")
        }
    } catch (e: TimeoutCancellationException) {
        println("Operation timed out!")
    }
    
    // ====== withTimeoutOrNull ======
    println("\n=== withTimeoutOrNull ===")
    val result = withTimeoutOrNull(200) {
        delay(100)  // fast enough
        "Success!"
    }
    println("Result: $result")
    
    val nullResult = withTimeoutOrNull(100) {
        delay(200)  // too slow
        "Success!"
    }
    println("Null result: $nullResult")
}
```

---

## 22.5 Flow (Kotlin's equivalent to RxJava/Streams)

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Cold flow: starts when collected
fun numbers(count: Int): Flow<Int> = flow {
    for (i in 1..count) {
        delay(100)
        emit(i)
    }
}

fun fibonacci(): Flow<Long> = flow {
    var a = 0L; var b = 1L
    while (true) {
        emit(a)
        val next = a + b; a = b; b = next
    }
}

fun main() = runBlocking {
    
    // ====== Basic Flow ======
    println("=== Basic Flow ===")
    numbers(5).collect { value ->
        println("Got: $value")
    }
    
    // ====== Flow operators ======
    println("\n=== Flow operators ===")
    numbers(10)
        .filter { it % 2 == 0 }
        .map { it * it }
        .take(3)
        .collect { println("Even square: $it") }
    
    // ====== Fibonacci ======
    println("\n=== Fibonacci ===")
    fibonacci()
        .take(10)
        .collect { print("$it ") }
    println()
    
    // ====== StateFlow (hot flow, like LiveData) ======
    println("\n=== StateFlow ===")
    val stateFlow = MutableStateFlow(0)
    
    val collector = launch {
        stateFlow.collect { value ->
            println("StateFlow: $value")
        }
    }
    
    delay(100)
    stateFlow.value = 1
    delay(100)
    stateFlow.value = 2
    delay(100)
    stateFlow.value = 3
    delay(100)
    
    collector.cancel()
    
    // ====== SharedFlow (hot flow, multicasting) ======
    println("\n=== SharedFlow ===")
    val sharedFlow = MutableSharedFlow<String>()
    
    val sub1 = launch { sharedFlow.collect { println("Sub1: $it") } }
    val sub2 = launch { sharedFlow.collect { println("Sub2: $it") } }
    
    delay(100)
    sharedFlow.emit("Hello")
    delay(100)
    sharedFlow.emit("World")
    delay(100)
    
    sub1.cancel()
    sub2.cancel()
    
    // ====== Flow combination ======
    println("\n=== Combine Flows ===")
    
    val prices = flow {
        listOf(100.0, 200.0, 300.0).forEach { price ->
            delay(100)
            emit(price)
        }
    }
    
    val discounts = flow {
        listOf(0.1, 0.2, 0.15).forEach { discount ->
            delay(150)
            emit(discount)
        }
    }
    
    prices.combine(discounts) { price, discount ->
        price * (1 - discount)
    }.collect { finalPrice ->
        println("Final price: %.2f".format(finalPrice))
    }
    
    // ====== Error handling in Flow ======
    println("\n=== Flow Error Handling ===")
    
    flow {
        emit(1)
        emit(2)
        throw RuntimeException("Flow error!")
        emit(3)
    }
    .catch { e -> println("Caught: ${e.message}") }
    .onCompletion { e ->
        if (e == null) println("Completed successfully")
        else println("Completed with error")
    }
    .collect { println("Value: $it") }
}
```

---

## 22.6 Full Program: Coroutine-based Download Manager

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import java.util.concurrent.atomic.AtomicLong

data class DownloadTask(
    val id: Int,
    val url: String,
    val size: Long  // bytes to simulate
)

data class DownloadResult(
    val task: DownloadTask,
    val success: Boolean,
    val durationMs: Long,
    val error: String? = null
)

class DownloadManager(private val maxConcurrent: Int = 3) {
    
    private val completedBytes = AtomicLong(0)
    private val totalBytes = AtomicLong(0)
    
    // Simulate download with progress
    private suspend fun download(task: DownloadTask): DownloadResult {
        val start = System.currentTimeMillis()
        
        try {
            // Simulate chunks
            val chunkSize = task.size / 10
            for (chunk in 1..10) {
                delay(50)  // simulate network I/O
                completedBytes.addAndGet(chunkSize)
                
                if (task.id % 7 == 0 && chunk == 5) {
                    throw RuntimeException("Network timeout for task ${task.id}")
                }
            }
            
            val duration = System.currentTimeMillis() - start
            println("[✅] Task ${task.id}: ${task.url} (${duration}ms)")
            return DownloadResult(task, true, duration)
            
        } catch (e: Exception) {
            val duration = System.currentTimeMillis() - start
            println("[❌] Task ${task.id}: ${e.message} (${duration}ms)")
            return DownloadResult(task, false, duration, e.message)
        }
    }
    
    fun downloadAll(tasks: List<DownloadTask>): Flow<DownloadResult> = flow {
        totalBytes.set(tasks.sumOf { it.size })
        completedBytes.set(0)
        
        // Process in batches of maxConcurrent
        tasks.chunked(maxConcurrent).forEach { batch ->
            coroutineScope {
                batch.map { task -> async { download(task) } }
                     .awaitAll()
                     .forEach { emit(it) }
            }
            
            val progress = (completedBytes.get().toDouble() / totalBytes.get()) * 100
            println("  Progress: ${progress.toInt()}%")
        }
    }
    
    fun getProgress(): Double {
        val total = totalBytes.get()
        if (total == 0L) return 0.0
        return (completedBytes.get().toDouble() / total) * 100
    }
}

fun main() = runBlocking {
    
    val manager = DownloadManager(maxConcurrent = 3)
    
    val tasks = (1..15).map { id ->
        DownloadTask(id, "https://example.com/file$id.dat", (10 + id * 5).toLong() * 1024)
    }
    
    println("Starting download of ${tasks.size} files...")
    println("Total size: ${tasks.sumOf { it.size } / 1024}KB\n")
    
    val start = System.currentTimeMillis()
    val results = mutableListOf<DownloadResult>()
    
    manager.downloadAll(tasks).collect { result ->
        results.add(result)
    }
    
    val elapsed = System.currentTimeMillis() - start
    
    println("\n=== Download Summary ===")
    println("Total time: ${elapsed}ms")
    println("Successful: ${results.count { it.success }}")
    println("Failed: ${results.count { !it.success }}")
    println("Avg duration: ${results.map { it.durationMs }.average().toLong()}ms")
    
    if (results.any { !it.success }) {
        println("\nFailed downloads:")
        results.filter { !it.success }
               .forEach { println("  - Task ${it.task.id}: ${it.error}") }
    }
    
    val totalMb = tasks.sumOf { it.size }.toDouble() / 1024 / 1024
    val throughput = totalMb / (elapsed / 1000.0)
    println("Throughput: %.2f MB/s (simulated)".format(throughput))
}
```

---

## สรุป Part 22

| Feature | คำอธิบาย |
|---------|---------|
| `suspend fun` | ฟังก์ชันที่ suspend ได้ |
| `launch` | fire-and-forget coroutine |
| `async/await` | coroutine ที่คืนค่า |
| `delay()` | suspend (ไม่ block thread) |
| `Dispatchers.IO` | สำหรับ I/O operations |
| `Dispatchers.Default` | สำหรับ CPU operations |
| `withTimeout` | timeout สำหรับ coroutine |
| `Flow` | async sequence of values |
| `StateFlow` | observable state (hot) |
| `SharedFlow` | event bus (hot) |

➡️ [Part 23: Android Development Basics](./Part-23-Android-Basics.md)
