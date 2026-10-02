# Part 81: Kotlin DSL & Advanced Builder Patterns
## ขั้นตอนที่ 5571-5640: Type-safe DSLs, Operator Overloading, Context Receivers

---

## 81.1 DSL คืออะไร

```
DSL = Domain-Specific Language
ภาษาที่ออกแบบมาสำหรับ domain เฉพาะ

Kotlin Internal DSL:
  สร้างใน Kotlin ปกติ แต่ทำให้อ่านเหมือนภาษาที่กำหนดเอง
  ใช้: lambda with receiver, extension functions, operator overloading

ตัวอย่าง DSL ที่เห็นทุกวัน:
  // Gradle Kotlin DSL
  dependencies {
      implementation("org.springframework.boot:spring-boot-starter")
      testImplementation("org.junit.jupiter:junit-jupiter")
  }
  
  // Compose UI
  Column {
      Text("Hello")
      Button(onClick = {}) { Text("Click") }
  }
  
  // Ktor routing
  routing {
      get("/users") { call.respond(users) }
      post("/users") { ... }
  }
```

---

## 81.2 Lambda with Receiver

```kotlin
// Extension function with lambda receiver
// ฟังก์ชันที่ block เป็น T.() -> Unit (this = T ใน block)

// Basic example
fun buildString(block: StringBuilder.() -> Unit): String {
    val sb = StringBuilder()
    sb.block()
    return sb.toString()
}

val result = buildString {
    append("Hello")    // this = StringBuilder
    append(", ")
    append("World")
    appendLine("!")
}

// HTML DSL
class HtmlTag(val name: String) {
    private val children = mutableListOf<HtmlTag>()
    private val attributes = mutableMapOf<String, String>()
    private var text: String? = null
    
    // Operator for attributes
    infix fun String.assign(value: String) = attributes.put(this, value)
    
    fun tag(name: String, block: HtmlTag.() -> Unit): HtmlTag {
        val child = HtmlTag(name)
        child.block()
        children.add(child)
        return child
    }
    
    operator fun String.unaryPlus() { text = this }
    
    fun render(indent: Int = 0): String = buildString {
        val spaces = "  ".repeat(indent)
        val attrStr = if (attributes.isEmpty()) "" 
            else attributes.entries.joinToString(" ") { "${it.key}=\"${it.value}\"" }
                .let { " $it" }
        
        appendLine("$spaces<$name$attrStr>")
        text?.let { appendLine("$spaces  $it") }
        children.forEach { append(it.render(indent + 1)) }
        appendLine("$spaces</$name>")
    }
}

fun html(block: HtmlTag.() -> Unit) = HtmlTag("html").apply(block)

// Usage
val page = html {
    tag("head") {
        tag("title") { +"My Page" }
    }
    tag("body") {
        tag("h1") { 
            "class" assign "heading"
            +"Hello World" 
        }
        tag("p") { +"This is a paragraph" }
    }
}
println(page.render())
```

---

## 81.3 Type-safe SQL DSL

```kotlin
// Simplified Exposed-style SQL DSL

abstract class Table(val tableName: String) {
    val columns = mutableListOf<Column<*>>()
    
    fun varchar(name: String, length: Int): Column<String> =
        Column<String>(name, "VARCHAR($length)").also { columns.add(it) }
    
    fun integer(name: String): Column<Int> =
        Column<Int>(name, "INTEGER").also { columns.add(it) }
    
    fun long(name: String): Column<Long> =
        Column<Long>(name, "BIGINT").also { columns.add(it) }
    
    fun boolean(name: String): Column<Boolean> =
        Column<Boolean>(name, "BOOLEAN").also { columns.add(it) }
}

class Column<T>(val name: String, val type: String) {
    infix fun eq(value: T): Condition = Condition("$name = '$value'")
    infix fun like(pattern: String): Condition = Condition("$name LIKE '$pattern'")
    infix fun gt(value: T): Condition = Condition("$name > '$value'")
    infix fun lt(value: T): Condition = Condition("$name < '$value'")
}

data class Condition(val sql: String) {
    infix fun and(other: Condition) = Condition("($sql AND ${other.sql})")
    infix fun or(other: Condition) = Condition("($sql OR ${other.sql})")
}

// Define tables
object Products : Table("products") {
    val id = varchar("id", 36)
    val name = varchar("name", 255)
    val price = long("price_cents")
    val category = varchar("category", 100)
    val inStock = boolean("in_stock")
}

// Query DSL
class SelectQuery(val table: Table) {
    private var whereClause: Condition? = null
    private var limitValue: Int? = null
    private var orderByColumn: String? = null
    
    fun where(condition: Condition) = apply { whereClause = condition }
    fun limit(n: Int) = apply { limitValue = n }
    fun orderBy(column: Column<*>) = apply { orderByColumn = column.name }
    
    fun toSql(): String = buildString {
        append("SELECT * FROM ${table.tableName}")
        whereClause?.let { append(" WHERE ${it.sql}") }
        orderByColumn?.let { append(" ORDER BY $it") }
        limitValue?.let { append(" LIMIT $it") }
    }
}

fun Table.select() = SelectQuery(this)

// Usage
val query = Products.select()
    .where(
        (Products.category eq "electronics") and
        (Products.inStock eq true) and
        (Products.price lt 50000L)
    )
    .orderBy(Products.price)
    .limit(20)
    .toSql()

// Produces: SELECT * FROM products WHERE ((category = 'electronics' AND in_stock = 'true') AND price_cents < '50000') ORDER BY price_cents LIMIT 20
```

---

## 81.4 Operator Overloading

```kotlin
// Money arithmetic
@JvmInline
value class Money(val cents: Long) {
    
    operator fun plus(other: Money) = Money(cents + other.cents)
    operator fun minus(other: Money) = Money(cents - other.cents)
    operator fun times(factor: Int) = Money(cents * factor)
    operator fun times(factor: Double) = Money((cents * factor).toLong())
    operator fun div(factor: Int) = Money(cents / factor)
    operator fun compareTo(other: Money) = cents.compareTo(other.cents)
    operator fun unaryMinus() = Money(-cents)
    
    override fun toString() = "฿${cents / 100}.${(cents % 100).toString().padStart(2, '0')}"
    
    companion object {
        val ZERO = Money(0)
        fun of(amount: Double) = Money((amount * 100).toLong())
        fun ofCents(cents: Long) = Money(cents)
    }
}

// Usage
val price = Money.of(99.99)
val tax = price * 0.07
val total = price + tax
val discount = Money.of(10.0)
val finalPrice = total - discount

// Matrix DSL
data class Matrix(val rows: Int, val cols: Int, val data: DoubleArray = DoubleArray(rows * cols)) {
    
    operator fun get(r: Int, c: Int) = data[r * cols + c]
    operator fun set(r: Int, c: Int, value: Double) { data[r * cols + c] = value }
    
    operator fun plus(other: Matrix): Matrix {
        require(rows == other.rows && cols == other.cols)
        return Matrix(rows, cols, DoubleArray(rows * cols) { i -> data[i] + other.data[i] })
    }
    
    operator fun times(scalar: Double) = Matrix(rows, cols, DoubleArray(rows * cols) { i -> data[i] * scalar })
    
    operator fun times(other: Matrix): Matrix {
        require(cols == other.rows)
        return Matrix(rows, other.cols).also { result ->
            for (i in 0 until rows)
                for (j in 0 until other.cols)
                    for (k in 0 until cols)
                        result[i, j] += this[i, k] * other[k, j]
        }
    }
}

// DSL for creating matrices
fun matrix(rows: Int, cols: Int, block: Matrix.() -> Unit) = Matrix(rows, cols).apply(block)

val m = matrix(2, 2) {
    set(0, 0, 1.0); set(0, 1, 2.0)
    set(1, 0, 3.0); set(1, 1, 4.0)
}
```

---

## 81.5 Configuration DSL (Spring-style)

```kotlin
// Kotlin-style Spring Bean Configuration DSL

// Define DSL classes
class ApplicationDsl {
    val beans = mutableListOf<BeanDefinition<*>>()
    
    inline fun <reified T : Any> bean(noinline factory: () -> T): BeanDefinition<T> =
        BeanDefinition(T::class, factory).also { beans.add(it) }
}

data class BeanDefinition<T : Any>(
    val type: kotlin.reflect.KClass<T>,
    val factory: () -> T,
    var name: String? = null,
    var singleton: Boolean = true
) {
    infix fun named(name: String) = apply { this.name = name }
    fun prototype() = apply { singleton = false }
}

fun application(block: ApplicationDsl.() -> Unit): ApplicationDsl =
    ApplicationDsl().apply(block)

// Usage
val app = application {
    bean { DataSource().apply { url = "jdbc:postgresql://..." } }
    bean { UserRepository(beans.filterIsInstance<DataSource>().first().factory()) }
    bean<UserService> {
        UserServiceImpl(
            repository = beans.last { it.type == UserRepository::class }.factory() as UserRepository
        )
    } named "primaryUserService"
}

// HTTP Client DSL
class HttpClient {
    var baseUrl: String = ""
    var timeout: Int = 30
    val headers = mutableMapOf<String, String>()
    var retryCount: Int = 3
    
    fun header(name: String, value: String) { headers[name] = value }
    fun auth(token: String) = header("Authorization", "Bearer $token")
    fun contentType(type: String) = header("Content-Type", type)
}

fun httpClient(block: HttpClient.() -> Unit) = HttpClient().apply(block)

val client = httpClient {
    baseUrl = "https://api.example.com"
    timeout = 60
    retryCount = 5
    auth("my-api-token")
    contentType("application/json")
    header("X-Client-Version", "1.0.0")
}
```

---

## 81.6 Kotlin Context Receivers (Kotlin 1.6.20+)

```kotlin
// Context receivers: multiple receivers in a single function
// (Still experimental as of Kotlin 2.0)

// Traditional approach: pass dependencies as parameters
fun processOrder(
    order: Order,
    logger: Logger,
    metrics: MetricsCollector,
    db: Database
): Result<Order> = ...

// With context receivers: inject via context
context(Logger, MetricsCollector, Database)
fun processOrder(order: Order): Result<Order> {
    log("Processing order ${order.id}")   // uses Logger
    increment("orders.processed")         // uses MetricsCollector
    save(order)                           // uses Database
    return Result.success(order)
}

// Call with context
with(logger) {
    with(metrics) {
        with(database) {
            processOrder(order)
        }
    }
}

// Better: use a combined context
data class AppContext(
    val logger: Logger,
    val metrics: MetricsCollector,
    val database: Database
) : Logger by logger, MetricsCollector by metrics, Database by database

val ctx = AppContext(MyLogger, PrometheusMetrics, PostgresDatabase)
with(ctx) {
    processOrder(order)
}

// Practical example: test vs production
interface TransactionContext {
    fun begin()
    fun commit()
    fun rollback()
}

context(TransactionContext)
suspend fun transferMoney(from: Account, to: Account, amount: Money) {
    begin()
    try {
        from.debit(amount)
        to.credit(amount)
        commit()
    } catch (e: Exception) {
        rollback()
        throw e
    }
}
```

---

## 81.7 Real-world: Kotlin Migrations DSL

```kotlin
// Custom database migration DSL

class MigrationContext(val jdbcTemplate: JdbcTemplate) {
    
    fun createTable(name: String, block: CreateTableBuilder.() -> Unit) {
        val builder = CreateTableBuilder(name).apply(block)
        jdbcTemplate.execute(builder.build())
    }
    
    fun alterTable(name: String, block: AlterTableBuilder.() -> Unit) {
        val builder = AlterTableBuilder(name).apply(block)
        builder.statements.forEach { jdbcTemplate.execute(it) }
    }
    
    fun execute(sql: String) = jdbcTemplate.execute(sql)
    
    fun insertData(table: String, vararg rows: Map<String, Any?>) {
        rows.forEach { row ->
            val cols = row.keys.joinToString(", ")
            val values = row.values.joinToString(", ") { if (it == null) "NULL" else "'$it'" }
            jdbcTemplate.execute("INSERT INTO $table ($cols) VALUES ($values)")
        }
    }
}

class CreateTableBuilder(private val name: String) {
    private val columns = mutableListOf<String>()
    private val constraints = mutableListOf<String>()
    
    fun column(name: String, type: String, vararg options: String) {
        columns.add("$name $type ${options.joinToString(" ")}")
    }
    fun primaryKey(vararg cols: String) { constraints.add("PRIMARY KEY (${cols.joinToString()})") }
    fun unique(vararg cols: String) { constraints.add("UNIQUE (${cols.joinToString()})") }
    fun index(name: String, vararg cols: String) { /* separate CREATE INDEX */ }
    
    fun build(): String {
        val all = (columns + constraints).joinToString(",\n  ")
        return "CREATE TABLE IF NOT EXISTS $name (\n  $all\n)"
    }
}

// Flyway @Bean migration callback
@Component
class V2_Migration(private val jdbcTemplate: JdbcTemplate) {
    
    fun migrate() {
        val ctx = MigrationContext(jdbcTemplate)
        
        ctx.createTable("product_reviews") {
            column("id", "UUID", "DEFAULT gen_random_uuid()", "NOT NULL")
            column("product_id", "UUID", "NOT NULL")
            column("user_id", "UUID", "NOT NULL")
            column("rating", "SMALLINT", "NOT NULL")
            column("comment", "TEXT")
            column("created_at", "TIMESTAMPTZ", "DEFAULT NOW()", "NOT NULL")
            primaryKey("id")
        }
        
        ctx.insertData("product_reviews",
            mapOf("product_id" to "uuid1", "user_id" to "uuid2", "rating" to 5),
            mapOf("product_id" to "uuid1", "user_id" to "uuid3", "rating" to 4)
        )
    }
}
```

---

## สรุป Part 81

```
Kotlin DSL Techniques:

1. Lambda with Receiver:
   fun dsl(block: Builder.() -> Unit) = Builder().apply(block)
   ใน block: this = Builder → ใช้ methods ได้เลย

2. Extension Functions:
   infix fun String.assign(value: String)
   operator fun Money.plus(other: Money)

3. Operator Overloading:
   +, -, *, /, %, unaryMinus, compareTo
   get/set (indexing: obj[i])
   invoke: obj() calls operator fun invoke()

4. Infix Functions:
   infix fun Column<T>.eq(value: T): Condition
   Usage: Products.name eq "iPhone"

5. @DslMarker:
   Prevent receiver leaking between nested DSLs
   @DslMarker annotation class HtmlTagMarker

@DslMarker Example:
  @HtmlTagMarker
  class HtmlTag { ... }
  
  html {
      body {
          // Can't access html's methods here
          // Without @DslMarker, both html and body receivers 
          // would be accessible (confusing)
      }
  }

Key Patterns:
  Builder pattern → type-safe configuration
  DSL = readability + compile-time safety
  Avoid if DSL is harder to understand than direct code
```

➡️ [Part 82: CI/CD Pipeline with GitHub Actions](./Part-82-CICD.md)
