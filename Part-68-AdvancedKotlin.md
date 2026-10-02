# Part 68: Advanced Kotlin Patterns
## ขั้นตอนที่ 4661-4730: Sealed Classes, Value Classes, DSL Builder, Context Receivers

---

## 68.1 Sealed Classes & Interfaces

```kotlin
// Sealed class = รู้ล่วงหน้าทุก subtype (closed hierarchy)
// คอมไพเลอร์บังคับให้ handle ทุก case ใน when expression

sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// Usage: when บังคับให้ครบทุก branch
fun <T> handle(result: Result<T>): String = when (result) {
    is Result.Success -> "Got data: ${result.data}"
    is Result.Error   -> "Error: ${result.exception.message}"
    Result.Loading    -> "Loading..."
    // ไม่ต้องมี else - คอมไพเลอร์รู้ว่าครบแล้ว
}

// Sealed interface (Kotlin 1.5+): subclasses ใน file เดียวกัน
sealed interface NetworkResult<out T> {
    data class Success<T>(val data: T, val code: Int = 200) : NetworkResult<T>
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>
    data class Exception(val cause: Throwable) : NetworkResult<Nothing>
}

// Powerful when pattern matching
fun <T> NetworkResult<T>.toMessage(): String = when (this) {
    is NetworkResult.Success  -> "OK: $data"
    is NetworkResult.Error    -> "HTTP $code: $message"
    is NetworkResult.Exception -> "Exception: ${cause.message}"
}

// Exhaustive when as expression
suspend fun fetchUser(id: String): String {
    return when (val result = callApi<User>(id)) {
        is NetworkResult.Success  -> result.data.name
        is NetworkResult.Error    -> throw RuntimeException("${result.code}: ${result.message}")
        is NetworkResult.Exception -> throw result.cause
    }
}
```

---

## 68.2 Value Classes (Inline Classes)

```kotlin
// Value class: zero-overhead type safety wrapper
// JVM representation = unwrapped primitive/type (no boxing overhead)

@JvmInline
value class UserId(val value: String) {
    init {
        require(value.isNotBlank()) { "UserId cannot be blank" }
    }
}

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email: $value" }
    }
}

@JvmInline
value class Money(val cents: Long) {
    val baht: Double get() = cents / 100.0
    
    operator fun plus(other: Money) = Money(cents + other.cents)
    operator fun minus(other: Money) = Money(cents - other.cents)
    operator fun times(factor: Double) = Money((cents * factor).toLong())
    
    override fun toString() = "฿%.2f".format(baht)
    
    companion object {
        fun of(baht: Double) = Money((baht * 100).toLong())
        fun of(baht: Int) = Money(baht * 100L)
        val ZERO = Money(0)
    }
}

// ปลอดภัยกว่า primitive types:
fun createOrder(userId: UserId, total: Money): Order {
    // ไม่สามารถส่ง Email แทน UserId ได้โดยบังเอิญ
    return Order(userId.value, total.cents)
}

// Test
val price = Money.of(150.00)
val tax = price * 0.07
val total = price + tax
println("Total: $total")  // Total: ฿160.50

// JVM bytecode: Money เป็น Long (ไม่ boxing object)
```

---

## 68.3 DSL Builder Pattern

```kotlin
// DSL (Domain-Specific Language): สร้าง API ที่อ่านง่ายเหมือนภาษาธรรมชาติ

// Email DSL
data class Email(
    val from: String,
    val to: List<String>,
    val subject: String,
    val body: String,
    val attachments: List<Attachment> = emptyList()
)

data class Attachment(val name: String, val content: ByteArray)

class EmailBuilder {
    var from: String = ""
    var subject: String = ""
    private val to = mutableListOf<String>()
    private val attachments = mutableListOf<Attachment>()
    private var body: String = ""
    
    fun to(email: String) { to.add(email) }
    fun to(vararg emails: String) { to.addAll(emails) }
    
    fun body(block: BodyBuilder.() -> Unit) {
        body = BodyBuilder().apply(block).build()
    }
    
    fun attach(name: String, content: ByteArray) {
        attachments.add(Attachment(name, content))
    }
    
    fun build() = Email(from, to, subject, body, attachments)
    
    class BodyBuilder {
        private val lines = mutableListOf<String>()
        
        fun text(content: String) { lines.add(content) }
        fun br() { lines.add("") }
        fun build() = lines.joinToString("\n")
    }
}

fun email(block: EmailBuilder.() -> Unit): Email =
    EmailBuilder().apply(block).build()

// Usage - reads like natural language:
val msg = email {
    from = "noreply@example.com"
    to("customer@example.com", "support@example.com")
    subject = "ยืนยันคำสั่งซื้อ #12345"
    body {
        text("สวัสดีคุณลูกค้า")
        br()
        text("ขอบคุณสำหรับการสั่งซื้อ")
        text("ยอดรวม: ฿1,500.00")
    }
    attach("receipt.pdf", loadPdf())
}

// HTTP Client DSL
class HttpRequestBuilder {
    var url: String = ""
    var method: String = "GET"
    val headers = mutableMapOf<String, String>()
    var body: String? = null
    
    fun header(name: String, value: String) { headers[name] = value }
    fun bearer(token: String) { header("Authorization", "Bearer $token") }
    fun json(content: String) {
        body = content
        header("Content-Type", "application/json")
    }
}

fun httpGet(url: String, block: HttpRequestBuilder.() -> Unit = {}): HttpRequestBuilder =
    HttpRequestBuilder().apply { this.url = url; this.method = "GET" }.apply(block)

fun httpPost(url: String, block: HttpRequestBuilder.() -> Unit): HttpRequestBuilder =
    HttpRequestBuilder().apply { this.url = url; this.method = "POST" }.apply(block)

// Usage:
val request = httpPost("https://api.example.com/orders") {
    bearer("my-token-here")
    json("""{"productId": "P001", "quantity": 2}""")
}
```

---

## 68.4 Scope Functions Mastery

```kotlin
// 5 scope functions: let, run, with, apply, also

data class User(var name: String, var email: String, var age: Int = 0)
data class Product(var id: String = "", var price: Double = 0.0, var inStock: Boolean = false)

fun scopeFunctionsDemo() {
    
    // let: transform + null safety
    val nameLength = "  Hello World  "
        .trim()
        .let { it.length }  // it = receiver
    
    // Null safety with let
    val user: User? = getUser()
    user?.let {
        sendWelcomeEmail(it.email)
        updateLastLogin(it.name)
    }
    
    // run: execute block, return result (object context)
    val userStr = User("Alice", "alice@example.com").run {
        "$name <$email>"  // this = receiver
    }
    
    // with: same as run but takes object as parameter (not extension)
    val html = with(StringBuilder()) {
        append("<html>")
        append("<body>Hello</body>")
        append("</html>")
        toString()
    }
    
    // apply: configure object, return same object
    val product = Product().apply {
        id = "P001"
        price = 199.99
        inStock = true
    }
    
    // also: side effects, return same object
    val savedUser = User("Bob", "bob@example.com")
        .also { println("Creating user: $it") }
        .also { userRepository.save(it) }
        .also { auditLog.record("USER_CREATED", it.name) }
    
    // Chain them
    val processedOrder = createOrder()
        .apply { status = "PROCESSING" }
        .also { orderRepository.save(it) }
        .also { notificationService.sendConfirmation(it) }
}

// Practical: builder pattern using apply
fun createHttpClient(): okhttp3.OkHttpClient =
    okhttp3.OkHttpClient.Builder()
        .connectTimeout(10, java.util.concurrent.TimeUnit.SECONDS)
        .readTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
        .addInterceptor { chain ->
            chain.proceed(
                chain.request().newBuilder()
                    .addHeader("User-Agent", "MyApp/1.0")
                    .build()
            )
        }
        .build()
```

---

## 68.5 Kotlin Contracts & Smart Casts

```kotlin
import kotlin.contracts.*

// Contract: บอกคอมไพเลอร์เกี่ยวกับ guarantees ของ function

@OptIn(ExperimentalContracts::class)
fun require(value: Boolean, message: () -> String) {
    contract {
        returns() implies value  // หลังจาก return ได้ → value == true
    }
    if (!value) throw IllegalArgumentException(message())
}

@OptIn(ExperimentalContracts::class)
fun String?.isNotNullOrBlank(): Boolean {
    contract {
        returns(true) implies (this@isNotNullOrBlank != null)
    }
    return this != null && isNotBlank()
}

// Smart cast ทำงานได้หลัง contract
fun processName(name: String?) {
    if (name.isNotNullOrBlank()) {
        // name is now String (not String?) due to contract
        println(name.uppercase())  // ไม่ต้องใช้ name!!.uppercase()
    }
}

// Kotlin coroutine contracts
@OptIn(ExperimentalContracts::class)
inline fun <T> withLogging(label: String, block: () -> T): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    println("START: $label")
    val result = block()
    println("END: $label")
    return result
}

// val ถูก smart cast เพราะ callsInPlace guarantee
fun example() {
    val result: String
    withLogging("computation") {
        result = "hello"  // ok เพราะ block รันแน่ๆ 1 ครั้ง
    }
    println(result)  // ใช้ได้โดยไม่ต้อง lateinit
}
```

---

## สรุป Part 68

```kotlin
// Advanced Kotlin Summary

// Sealed classes: exhaustive when, no else needed
sealed class State { data class Active(val id: Int) : State(); object Inactive : State() }
val msg = when (val s = getState()) {
    is State.Active -> "Active: ${s.id}"
    State.Inactive  -> "Inactive"
}

// Value classes: type safety with zero overhead
@JvmInline value class OrderId(val value: String)
@JvmInline value class Money(val cents: Long)
// ไม่สามารถส่ง Money ตรงที่ต้องการ OrderId (compile error)

// DSL: apply + extension function + lambda with receiver
fun buildEmail(block: EmailBuilder.() -> Unit) = EmailBuilder().apply(block).build()
val email = buildEmail {
    from = "a@b.com"
    to("c@d.com")
    subject = "Test"
}

// Scope functions:
// let   = transform, null safety, it
// run   = transform, object context, this
// with  = non-extension run
// apply = configure, return self, this
// also  = side effects, return self, it
```

➡️ [Part 69: Kotlin Coroutines Advanced Patterns](./Part-69-CoroutinesAdvanced.md)
