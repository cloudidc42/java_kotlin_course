# Part 31: Kotlin Advanced - DSL & Type-Safe Builders
## ขั้นตอนที่ 2071-2140: Kotlin ระดับ Expert

---

## 31.1 Kotlin DSL (Domain-Specific Language)

```kotlin
// DSL ช่วยสร้าง readable API ที่มีลักษณะเป็น configuration
// ตัวอย่าง: HTML DSL

// ====== HTML DSL ======
@DslMarker
annotation class HtmlDsl

@HtmlDsl
class TagBuilder(val name: String) {
    private val attributes = mutableMapOf<String, String>()
    private val children = mutableListOf<Any>()
    
    fun attr(name: String, value: String) { attributes[name] = value }
    fun text(content: String) { children.add(content) }
    
    fun tag(name: String, block: TagBuilder.() -> Unit = {}): TagBuilder {
        val child = TagBuilder(name).apply(block)
        children.add(child)
        return child
    }
    
    fun build(indent: Int = 0): String {
        val spaces = "  ".repeat(indent)
        val attrs = if (attributes.isEmpty()) ""
                    else " " + attributes.entries.joinToString(" ") { 
                        """${it.key}="${it.value}"""" 
                    }
        return if (children.isEmpty()) {
            "$spaces<$name$attrs/>"
        } else {
            val childrenStr = children.joinToString("\n") { child ->
                when (child) {
                    is String -> "$spaces  $child"
                    is TagBuilder -> child.build(indent + 1)
                    else -> ""
                }
            }
            "$spaces<$name$attrs>\n$childrenStr\n$spaces</$name>"
        }
    }
}

fun html(block: TagBuilder.() -> Unit): String =
    TagBuilder("html").apply(block).build()

fun TagBuilder.head(block: TagBuilder.() -> Unit) = tag("head", block)
fun TagBuilder.body(block: TagBuilder.() -> Unit) = tag("body", block)
fun TagBuilder.title(block: TagBuilder.() -> Unit) = tag("title", block)
fun TagBuilder.h1(block: TagBuilder.() -> Unit) = tag("h1", block)
fun TagBuilder.p(block: TagBuilder.() -> Unit) = tag("p", block)
fun TagBuilder.div(block: TagBuilder.() -> Unit) = tag("div", block)
fun TagBuilder.a(href: String, block: TagBuilder.() -> Unit) = tag("a") {
    attr("href", href)
    block()
}

fun main() {
    val page = html {
        head {
            title { text("My Page") }
        }
        body {
            h1 { text("Hello, Kotlin DSL!") }
            div {
                attr("class", "container")
                p { text("This is a paragraph.") }
                a("https://kotlinlang.org") {
                    text("Visit Kotlin")
                }
            }
        }
    }
    println(page)
}
```

---

## 31.2 Type-Safe Builder Pattern

```kotlin
// Gradle-style DSL

data class Dependency(val group: String, val name: String, val version: String) {
    override fun toString() = "$group:$name:$version"
}

class DependencyScope(val configuration: String) {
    private val _deps = mutableListOf<Dependency>()
    val deps: List<Dependency> get() = _deps.toList()
    
    operator fun String.invoke(name: String, version: String) {
        _deps.add(Dependency(this, name, version))
    }
    
    fun add(notation: String) {
        val parts = notation.split(":")
        require(parts.size == 3) { "Format: group:name:version" }
        _deps.add(Dependency(parts[0], parts[1], parts[2]))
    }
}

class ProjectConfig {
    var group: String = ""
    var name: String = ""
    var version: String = "1.0.0"
    var jvmTarget: String = "21"
    
    private val _dependencies = mutableMapOf<String, DependencyScope>()
    
    val dependencies: Map<String, DependencyScope> get() = _dependencies
    
    fun dependencies(block: DependencyScope.() -> Unit) {
        val scope = DependencyScope("implementation")
        scope.block()
        _dependencies["implementation"] = scope
    }
    
    fun testDependencies(block: DependencyScope.() -> Unit) {
        val scope = DependencyScope("test")
        scope.block()
        _dependencies["test"] = scope
    }
    
    fun printConfig() {
        println("Project: $group:$name:$version (JVM $jvmTarget)")
        dependencies.forEach { (config, scope) ->
            println("  $config:")
            scope.deps.forEach { println("    $it") }
        }
    }
}

fun project(block: ProjectConfig.() -> Unit): ProjectConfig =
    ProjectConfig().apply(block)

fun main() {
    val config = project {
        group = "com.example"
        name = "my-app"
        version = "2.0.0"
        jvmTarget = "21"
        
        dependencies {
            "org.springframework.boot"("spring-boot-starter-web", "3.2.0")
            "org.springframework.boot"("spring-boot-starter-data-jpa", "3.2.0")
            add("com.fasterxml.jackson.module:jackson-module-kotlin:2.16.0")
        }
        
        testDependencies {
            "org.junit.jupiter"("junit-jupiter", "5.10.0")
            "io.mockk"("mockk", "1.13.8")
        }
    }
    
    config.printConfig()
}
```

---

## 31.3 Operator Overloading

```kotlin
data class Vector2D(val x: Double, val y: Double) {
    
    operator fun plus(other: Vector2D) = Vector2D(x + other.x, y + other.y)
    operator fun minus(other: Vector2D) = Vector2D(x - other.x, y - other.y)
    operator fun times(scalar: Double) = Vector2D(x * scalar, y * scalar)
    operator fun div(scalar: Double) = Vector2D(x / scalar, y / scalar)
    operator fun unaryMinus() = Vector2D(-x, -y)
    operator fun get(index: Int) = when (index) { 0 -> x; 1 -> y; else -> throw IndexOutOfBoundsException() }
    
    fun magnitude() = Math.sqrt(x * x + y * y)
    fun normalize() = this / magnitude()
    infix fun dot(other: Vector2D) = x * other.x + y * other.y
    infix fun cross(other: Vector2D) = x * other.y - y * other.x
    
    override fun toString() = "(%.2f, %.2f)".format(x, y)
}

// Comparison operators
data class Version(val major: Int, val minor: Int, val patch: Int) : Comparable<Version> {
    
    override fun compareTo(other: Version): Int {
        return compareValuesBy(this, other,
            Version::major, Version::minor, Version::patch
        )
    }
    
    override fun toString() = "$major.$minor.$patch"
    
    operator fun rangeTo(other: Version) = VersionRange(this, other)
}

class VersionRange(val start: Version, val end: Version) {
    operator fun contains(version: Version) = version >= start && version <= end
    operator fun iterator(): Iterator<Version> = object : Iterator<Version> {
        var current = start
        override fun hasNext() = current <= end
        override fun next(): Version {
            val result = current
            current = Version(current.major, current.minor, current.patch + 1)
            if (current.patch > 9) {
                current = Version(current.major, current.minor + 1, 0)
            }
            return result
        }
    }
}

fun main() {
    val v1 = Vector2D(3.0, 4.0)
    val v2 = Vector2D(1.0, 2.0)
    
    println("v1 + v2 = ${v1 + v2}")
    println("v1 - v2 = ${v1 - v2}")
    println("v1 * 2 = ${v1 * 2.0}")
    println("v1 dot v2 = ${v1 dot v2}")
    println("v1 cross v2 = ${v1 cross v2}")
    println("|v1| = ${v1.magnitude()}")
    println("v1[0] = ${v1[0]}")
    
    val v1_0 = Version(1, 0, 0)
    val v1_5 = Version(1, 5, 0)
    val v2_0 = Version(2, 0, 0)
    
    println("\nVersion comparison:")
    println("v1.0 < v2.0: ${v1_0 < v2_0}")
    println("v1.5 in v1.0..v2.0: ${v1_5 in v1_0..v2_0}")
}
```

---

## 31.4 Delegation Pattern

```kotlin
import kotlin.properties.Delegates
import kotlin.reflect.KProperty

// ====== Property Delegation ======

// Lazy - computed on first access
class HeavyComputation {
    val result: String by lazy {
        println("Computing...")
        Thread.sleep(100)  // simulated expensive operation
        "computed value"
    }
}

// Observable - notified on change
class FormField {
    var value: String by Delegates.observable("") { prop, old, new ->
        println("${prop.name} changed: '$old' → '$new'")
    }
    
    var validated: Boolean by Delegates.vetoable(false) { _, _, new ->
        println("Validating: $new")
        new  // allow all changes
    }
}

// notNull - throw if accessed before set
class Service {
    var config: Map<String, String> by Delegates.notNull()
}

// Custom delegate
class UpperCaseDelegate {
    private var value: String = ""
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String = value.uppercase()
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        this.value = value.trim()
    }
}

class UserProfile {
    var username: String by UpperCaseDelegate()
    var displayName: String by UpperCaseDelegate()
}

// Map delegation
class ServerConfig(val map: Map<String, Any>) {
    val host: String by map
    val port: Int by map
    val debug: Boolean by map
}

// Class delegation
interface Printer {
    fun print(text: String)
    fun printLine(text: String)
}

class ConsolePrinter : Printer {
    override fun print(text: String) = kotlin.io.print(text)
    override fun printLine(text: String) = println(text)
}

class LoggingPrinter(delegate: Printer) : Printer by delegate {
    override fun printLine(text: String) {
        println("[LOG ${java.time.LocalTime.now()}] $text")
    }
}

fun main() {
    val hc = HeavyComputation()
    println("Before access")
    println(hc.result)  // computed here
    println(hc.result)  // returned from cache
    
    val field = FormField()
    field.value = "hello"
    field.value = "world"
    
    val profile = UserProfile()
    profile.username = "  alice  "
    profile.displayName = "alice smith"
    println("Username: ${profile.username}")
    println("Display: ${profile.displayName}")
    
    val config = ServerConfig(mapOf("host" to "localhost", "port" to 8080, "debug" to true))
    println("${config.host}:${config.port} debug=${config.debug}")
    
    val printer = LoggingPrinter(ConsolePrinter())
    printer.print("Hello ")    // uses ConsolePrinter
    printer.printLine("World") // uses LoggingPrinter override
}
```

---

## 31.5 Kotlin Multiplatform (KMP) Basics

```kotlin
// commonMain/kotlin/com/example/Calculator.kt
// Shared between JVM, Android, iOS, JS

expect fun currentTimeMillis(): Long  // platform-specific

class Calculator {
    
    fun add(a: Double, b: Double) = a + b
    fun subtract(a: Double, b: Double) = a - b
    fun multiply(a: Double, b: Double) = a * b
    fun divide(a: Double, b: Double): Double {
        require(b != 0.0) { "Cannot divide by zero" }
        return a / b
    }
    
    fun calculate(expression: String): Double {
        val parts = expression.trim().split("\\s+".toRegex())
        require(parts.size == 3) { "Format: <num> <op> <num>" }
        val a = parts[0].toDouble()
        val op = parts[1]
        val b = parts[2].toDouble()
        return when (op) {
            "+" -> add(a, b)
            "-" -> subtract(a, b)
            "*" -> multiply(a, b)
            "/" -> divide(a, b)
            else -> throw IllegalArgumentException("Unknown operator: $op")
        }
    }
    
    fun timeOperation(block: () -> Double): Pair<Double, Long> {
        val start = currentTimeMillis()
        val result = block()
        return result to (currentTimeMillis() - start)
    }
}

// jvmMain/kotlin/com/example/Platform.kt
actual fun currentTimeMillis(): Long = System.currentTimeMillis()

// jsMain/kotlin/com/example/Platform.kt
actual fun currentTimeMillis(): Long = js("Date.now()").unsafeCast<Long>()

// iosMain/kotlin/com/example/Platform.kt
actual fun currentTimeMillis(): Long = 
    (platform.Foundation.NSDate.timeIntervalSinceReferenceDate * 1000).toLong()
```

```kotlin
// build.gradle.kts for KMP
plugins {
    kotlin("multiplatform") version "1.9.22"
}

kotlin {
    jvm()
    js(IR) { browser() }
    iosArm64()
    iosSimulatorArm64()
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
                implementation("io.ktor:ktor-client-core:2.3.7")
            }
        }
        val jvmMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-cio:2.3.7")
            }
        }
        val jsMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-js:2.3.7")
            }
        }
        val iosMain by creating {
            dependsOn(commonMain)
            dependencies {
                implementation("io.ktor:ktor-client-darwin:2.3.7")
            }
        }
        val iosArm64Main by getting { dependsOn(iosMain) }
        val iosSimulatorArm64Main by getting { dependsOn(iosMain) }
        
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
    }
}
```

---

## 31.6 Advanced Coroutines

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// ====== Custom CoroutineScope ======
class AppComponent : CoroutineScope by CoroutineScope(SupervisorJob() + Dispatchers.Default) {
    
    fun start() {
        launch { task1() }
        launch { task2() }
    }
    
    fun shutdown() {
        cancel()  // cancel all children
    }
    
    private suspend fun task1() { delay(1000); println("Task 1 done") }
    private suspend fun task2() { delay(500); println("Task 2 done") }
}

// ====== Channel ======
suspend fun producer(channel: kotlinx.coroutines.channels.SendChannel<Int>) {
    for (i in 1..5) {
        delay(100)
        channel.send(i)
        println("Sent: $i")
    }
    channel.close()
}

// ====== Flow operators ======
fun numberFlow() = flow {
    for (i in 1..10) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking {
    
    // Channel producer-consumer
    val channel = kotlinx.coroutines.channels.Channel<Int>()
    launch { producer(channel) }
    for (value in channel) {
        println("Received: $value")
    }
    
    // Advanced Flow
    println("\n=== Flow ZIP ===")
    val f1 = flowOf(1, 2, 3)
    val f2 = flowOf("a", "b", "c")
    
    f1.zip(f2) { num, str -> "$num$str" }
      .collect { println(it) }
    
    println("\n=== Flow FlatMap ===")
    flowOf(1, 2, 3)
        .flatMapMerge { num ->
            flow {
                delay(100)
                emit(num * 10)
                emit(num * 100)
            }
        }
        .collect { println(it) }
    
    // Buffer to improve throughput
    println("\n=== Buffered Flow ===")
    numberFlow()
        .buffer(10)         // buffer up to 10 elements
        .filter { it % 2 == 0 }
        .map { it * it }
        .collect { print("$it ") }
    println()
    
    // conflate - skip intermediate values (only keep latest)
    println("\n=== Conflated Flow ===")
    flow {
        for (i in 1..20) {
            emit(i)
        }
    }
    .conflate()
    .collect { value ->
        delay(100)  // slow consumer
        println("Consumed: $value")
    }
}
```

---

## 31.7 Kotlin Serialization

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*
import kotlinx.serialization.modules.*

@Serializable
data class User(
    val id: Long,
    val name: String,
    val email: String,
    @SerialName("created_at") val createdAt: String,
    val role: Role = Role.USER
) {
    @Serializable
    enum class Role { USER, ADMIN, MODERATOR }
}

@Serializable
data class Page<T>(
    val data: List<T>,
    val page: Int,
    val pageSize: Int,
    val total: Int
)

// Polymorphism
@Serializable
sealed class Shape {
    abstract fun area(): Double
}

@Serializable
@SerialName("circle")
data class Circle(val radius: Double) : Shape() {
    override fun area() = Math.PI * radius * radius
}

@Serializable
@SerialName("rectangle")
data class Rectangle(val width: Double, val height: Double) : Shape() {
    override fun area() = width * height
}

fun main() {
    
    val json = Json {
        prettyPrint = true
        ignoreUnknownKeys = true
        isLenient = true
        encodeDefaults = true
    }
    
    val user = User(1L, "Alice", "alice@example.com", "2024-01-01", User.Role.ADMIN)
    
    // Encode
    val jsonStr = json.encodeToString(user)
    println("Encoded:")
    println(jsonStr)
    
    // Decode
    val decoded = json.decodeFromString<User>("""
        {
            "id": 2,
            "name": "Bob",
            "email": "bob@example.com",
            "created_at": "2024-02-01",
            "unknown_field": "ignored"
        }
    """.trimIndent())
    println("\nDecoded: $decoded")
    
    // Collections
    val page = Page(
        data = listOf(user, decoded),
        page = 1, pageSize = 10, total = 100
    )
    println("\nPage: ${json.encodeToString(page)}")
    
    // Sealed class
    val shapes: List<Shape> = listOf(Circle(5.0), Rectangle(3.0, 4.0))
    val shapesJson = json.encodeToString(shapes)
    println("\nShapes: $shapesJson")
    
    val decoded2 = json.decodeFromString<List<Shape>>(shapesJson)
    decoded2.forEach { println("${it::class.simpleName} area = ${it.area()}") }
}
```

---

## สรุป Part 31

| Feature | ใช้สำหรับ |
|---------|---------|
| DSL | Readable configuration APIs |
| Type-safe builders | Builder pattern ที่ type-safe |
| Operator overloading | Custom operators (+, *, []) |
| Property delegation | lazy, observable, custom logic |
| Class delegation | Composition over inheritance |
| KMP | Share code across platforms |
| Channels | Producer-consumer patterns |
| kotlinx.serialization | Type-safe JSON/other formats |

➡️ [Part 32: Advanced Android Development](./Part-32-Android-Advanced.md)
