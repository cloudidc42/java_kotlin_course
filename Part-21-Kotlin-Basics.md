# Part 21: Kotlin - พื้นฐานภาษา Kotlin
## ขั้นตอนที่ 1361-1430: เริ่มต้นกับ Kotlin

---

## 21.1 ทำไมต้อง Kotlin?

```
Java vs Kotlin:

Java (verbose):
    private String name;
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

Kotlin (concise):
    var name: String = ""

Java (verbose):
    if (str != null) {
        System.out.println(str.length());
    }

Kotlin (safe):
    println(str?.length)

Java (verbose):
    Person person = new Person("Alice", 30);

Kotlin (clean):
    val person = Person("Alice", 30)
```

**ข้อดีของ Kotlin:**
- Null safety built-in
- Extension functions
- Data classes
- Coroutines
- Concise syntax
- 100% interoperable กับ Java
- Official Android language

---

## 21.2 Variables, Types, Functions

```kotlin
// ====== Variables ======

fun variablesDemo() {
    // val = immutable (final), var = mutable
    val name: String = "Alice"    // explicit type
    val age = 25                   // inferred type
    var score = 0                  // mutable
    score = 100
    
    // Type inference
    val text = "Hello"             // String
    val number = 42                // Int
    val price = 99.99              // Double
    val isActive = true            // Boolean
    val letter = 'A'               // Char
    
    // String templates
    println("Name: $name, Age: $age")
    println("Score: ${score * 2}")
    println("Is active: ${if (isActive) "yes" else "no"}")
    
    // Multiline string
    val poem = """
        |Roses are red,
        |Violets are blue,
        |Kotlin is awesome,
        |And so are you!
    """.trimMargin()
    println(poem)
    
    // Type system
    val x: Int = 10
    val y: Long = x.toLong()
    val z: Double = x.toDouble()
    val s: String = x.toString()
    
    // Nullable types
    var nullableName: String? = null
    nullableName = "Bob"
    println(nullableName?.length)       // safe call: null if null
    println(nullableName!!.length)      // non-null assertion: throws if null
    println(nullableName?.length ?: 0)  // Elvis operator: default if null
}

// ====== Functions ======

// Basic function
fun greet(name: String): String {
    return "Hello, $name!"
}

// Single expression function
fun square(x: Int) = x * x

// Default parameters
fun createUser(name: String, age: Int = 18, role: String = "USER") =
    "User($name, $age, $role)"

// Named arguments
fun formatDate(year: Int, month: Int, day: Int) = "$year-$month-$day"

// Varargs
fun sum(vararg numbers: Int) = numbers.sum()

// Extension function (add method to existing class)
fun String.shout() = this.uppercase() + "!!!"
fun Int.isPrime(): Boolean {
    if (this < 2) return false
    if (this == 2) return true
    if (this % 2 == 0) return false
    for (i in 3..Math.sqrt(toDouble()).toInt() step 2) {
        if (this % i == 0) return false
    }
    return true
}

// Infix function
infix fun Int.pow(exponent: Int): Int = Math.pow(toDouble(), exponent.toDouble()).toInt()

fun functionsDemo() {
    println(greet("Alice"))
    println(square(7))
    
    println(createUser("Bob"))
    println(createUser("Charlie", role = "ADMIN"))
    println(createUser(name = "Dave", age = 25, role = "MOD"))
    
    println(formatDate(day = 15, month = 6, year = 2024))
    
    println(sum(1, 2, 3, 4, 5))
    
    println("hello".shout())
    println(7.isPrime())
    println(4.isPrime())
    
    println(2 pow 10)  // infix
}

fun main() {
    variablesDemo()
    println("---")
    functionsDemo()
}
```

---

## 21.3 Control Flow

```kotlin
fun controlFlowDemo() {
    
    // ====== if as expression ======
    val age = 20
    val status = if (age >= 18) "Adult" else "Minor"
    println(status)
    
    // Multi-line if expression
    val grade = 85
    val letterGrade = if (grade >= 90) "A"
                      else if (grade >= 80) "B"
                      else if (grade >= 70) "C"
                      else if (grade >= 60) "D"
                      else "F"
    println("Grade: $letterGrade")
    
    // ====== when (Kotlin's switch) ======
    val day = 3
    val dayName = when (day) {
        1 -> "Monday"
        2 -> "Tuesday"
        3 -> "Wednesday"
        4, 5 -> "Thu/Fri"
        6, 7 -> "Weekend"
        else -> "Invalid"
    }
    println("Day: $dayName")
    
    // when with ranges
    val score = 75
    val result = when (score) {
        in 90..100 -> "Excellent"
        in 80..89  -> "Good"
        in 70..79  -> "Average"
        in 60..69  -> "Below average"
        else       -> "Fail"
    }
    println("Result: $result")
    
    // when without argument (replaces if-else chain)
    val x = 15
    when {
        x < 0        -> println("Negative")
        x == 0       -> println("Zero")
        x in 1..10   -> println("Small")
        x in 11..100 -> println("Medium")
        else         -> println("Large")
    }
    
    // when with type checking
    val obj: Any = "Hello"
    val description = when (obj) {
        is String -> "String of length ${obj.length}"
        is Int    -> "Integer: ${obj * 2}"
        is List<*>-> "List of ${obj.size} elements"
        null      -> "null"
        else      -> "Unknown type"
    }
    println(description)
    
    // ====== Loops ======
    // for loop with range
    print("1 to 5: ")
    for (i in 1..5) print("$i ")
    println()
    
    print("1 to 10 step 2: ")
    for (i in 1..10 step 2) print("$i ")
    println()
    
    print("5 down to 1: ")
    for (i in 5 downTo 1) print("$i ")
    println()
    
    // for with index
    val fruits = listOf("Apple", "Banana", "Cherry")
    for ((index, fruit) in fruits.withIndex()) {
        println("  $index: $fruit")
    }
    
    // while and do-while
    var n = 1
    while (n <= 5) {
        print("$n ")
        n++
    }
    println()
    
    var m = 5
    do {
        print("$m ")
        m--
    } while (m > 0)
    println()
    
    // repeat
    repeat(3) { i -> println("Repeat $i") }
    
    // forEach
    (1..5).forEach { print("${it * it} ") }
    println()
}

fun main() = controlFlowDemo()
```

---

## 21.4 Classes & Objects

```kotlin
// ====== Data Classes ======
data class Person(
    val name: String,
    val age: Int,
    val email: String = ""
) {
    fun greet() = "Hi, I'm $name, age $age"
    
    // Additional property (not in constructor)
    val isAdult get() = age >= 18
}

// ====== Regular Class ======
class BankAccount(
    val accountId: String,
    private var balance: Double = 0.0
) {
    private val transactions = mutableListOf<String>()
    
    fun deposit(amount: Double) {
        require(amount > 0) { "Amount must be positive" }
        balance += amount
        transactions.add("+฿$amount")
    }
    
    fun withdraw(amount: Double) {
        require(amount > 0) { "Amount must be positive" }
        require(amount <= balance) { "Insufficient funds" }
        balance -= amount
        transactions.add("-฿$amount")
    }
    
    fun getBalance() = balance
    
    fun printStatement() {
        println("Account: $accountId")
        transactions.forEach { println("  $it") }
        println("Balance: ฿$balance")
    }
}

// ====== Sealed Classes ======
sealed class Shape {
    abstract fun area(): Double
    abstract fun perimeter(): Double
    
    data class Circle(val radius: Double) : Shape() {
        override fun area() = Math.PI * radius * radius
        override fun perimeter() = 2 * Math.PI * radius
    }
    
    data class Rectangle(val width: Double, val height: Double) : Shape() {
        override fun area() = width * height
        override fun perimeter() = 2 * (width + height)
    }
    
    data class Triangle(val a: Double, val b: Double, val c: Double) : Shape() {
        override fun area(): Double {
            val s = perimeter() / 2
            return Math.sqrt(s * (s - a) * (s - b) * (s - c))
        }
        override fun perimeter() = a + b + c
    }
}

fun describeShape(shape: Shape): String = when (shape) {
    is Shape.Circle    -> "Circle r=%.2f, area=%.2f".format(shape.radius, shape.area())
    is Shape.Rectangle -> "Rectangle ${shape.width}×${shape.height}, area=${shape.area()}"
    is Shape.Triangle  -> "Triangle %.2f, area=%.2f".format(shape.perimeter(), shape.area())
}

// ====== Object (Singleton) ======
object AppConfig {
    val version = "1.0.0"
    val debug = true
    var currentUser: String? = null
    
    fun info() = "App v$version, debug=$debug"
}

// ====== Companion Object (static) ======
class Counter(val name: String) {
    private var count = 0
    
    fun increment() = ++count
    fun reset() { count = 0 }
    fun getCount() = count
    
    companion object {
        private var totalCreated = 0
        
        fun create(name: String): Counter {
            totalCreated++
            return Counter(name)
        }
        
        fun getTotalCreated() = totalCreated
    }
}

fun classesDemo() {
    // Data class
    val alice = Person("Alice", 28, "alice@example.com")
    val bob = alice.copy(name = "Bob", age = 30)  // copy with modification
    
    println(alice)         // Auto-generated toString
    println(alice.greet())
    println("Is adult: ${alice.isAdult}")
    println("Equal: ${alice == alice.copy()}")  // structural equality
    
    val (name, age) = alice  // destructuring
    println("Destructured: name=$name, age=$age")
    
    // BankAccount
    println("\n=== BankAccount ===")
    val acc = BankAccount("ACC001", 1000.0)
    acc.deposit(500.0)
    acc.withdraw(200.0)
    acc.printStatement()
    
    // Shapes
    println("\n=== Shapes ===")
    val shapes: List<Shape> = listOf(
        Shape.Circle(5.0),
        Shape.Rectangle(4.0, 6.0),
        Shape.Triangle(3.0, 4.0, 5.0)
    )
    shapes.forEach { println(describeShape(it)) }
    
    // Singleton
    println("\n=== AppConfig ===")
    println(AppConfig.info())
    AppConfig.currentUser = "Alice"
    
    // Companion Object
    println("\n=== Counter ===")
    val c1 = Counter.create("visits")
    val c2 = Counter.create("clicks")
    repeat(5) { c1.increment() }
    repeat(3) { c2.increment() }
    println("Visits: ${c1.getCount()}, Clicks: ${c2.getCount()}")
    println("Total counters created: ${Counter.getTotalCreated()}")
}

fun main() = classesDemo()
```

---

## 21.5 Collections & Functional

```kotlin
fun collectionsDemo() {
    
    // ====== Immutable collections ======
    val list = listOf("Java", "Kotlin", "Python", "Go")
    val set = setOf(1, 2, 3, 2, 1)     // {1, 2, 3} - dedup
    val map = mapOf("a" to 1, "b" to 2, "c" to 3)
    
    // ====== Mutable collections ======
    val mutableList = mutableListOf("A", "B", "C")
    mutableList.add("D")
    mutableList.removeAt(0)
    
    val mutableMap = mutableMapOf<String, Int>()
    mutableMap["key1"] = 10
    mutableMap["key2"] = 20
    mutableMap.getOrPut("key3") { 30 }
    
    // ====== Collection operations ======
    val numbers = (1..10).toList()
    
    println("filter: " + numbers.filter { it % 2 == 0 })
    println("map: " + numbers.map { it * it })
    println("flatMap: " + listOf(1,2,3).flatMap { n -> (1..n).toList() })
    println("reduce: " + numbers.reduce { acc, n -> acc + n })
    println("fold: " + numbers.fold(1) { acc, n -> acc * n })
    println("sum: " + numbers.sum())
    println("avg: " + numbers.average())
    println("sorted: " + listOf(5,2,8,1,9).sorted())
    println("sortedBy: " + list.sortedBy { it.length })
    println("groupBy: " + numbers.groupBy { if (it % 2 == 0) "even" else "odd" })
    println("partition: " + numbers.partition { it > 5 })
    println("take: " + numbers.take(3))
    println("drop: " + numbers.drop(7))
    println("first: " + numbers.first { it > 5 })
    println("find: " + numbers.find { it > 5 })
    println("any: " + numbers.any { it > 9 })
    println("all: " + numbers.all { it > 0 })
    println("none: " + numbers.none { it > 10 })
    println("count: " + numbers.count { it % 3 == 0 })
    println("distinct: " + listOf(1,2,2,3,3,3).distinct())
    println("zip: " + listOf(1,2,3).zip(listOf("a","b","c")))
    println("flatten: " + listOf(listOf(1,2), listOf(3,4)).flatten())
    
    // Chaining
    val result = (1..20)
        .filter { it % 2 != 0 }
        .map { it * it }
        .take(5)
        .sum()
    println("Odd squares sum (first 5): $result")
    
    // associate
    val nameToLength = list.associate { it to it.length }
    println("nameToLength: $nameToLength")
    
    // maxByOrNull, minByOrNull
    println("Longest: " + list.maxByOrNull { it.length })
    println("Shortest: " + list.minByOrNull { it.length })
    
    // joinToString
    println("Joined: " + list.joinToString(", ", "[", "]"))
}

fun main() = collectionsDemo()
```

---

## 21.6 Null Safety

```kotlin
fun nullSafetyDemo() {
    
    // Nullable types
    var name: String? = null
    var number: Int? = null
    
    // Safe call ?.
    println(name?.length)        // null
    println(name?.uppercase())   // null
    
    name = "Kotlin"
    println(name?.length)        // 6
    
    // Elvis operator ?:
    val length = name?.length ?: 0
    println("Length: $length")
    
    val value = number ?: -1
    println("Value: $value")
    
    // Non-null assertion !! (throws KotlinNullPointerException)
    name = "Hello"
    println(name!!.length)  // safe here because not null
    
    // let for null-safe block
    val city: String? = "Bangkok"
    city?.let { c ->
        println("City: $c, length: ${c.length}")
    }
    
    // Null can be used in safe calls
    val addresses: List<String?> = listOf("Bangkok", null, "Chiang Mai", null)
    val cities = addresses.filterNotNull()
    println("Non-null cities: $cities")
    
    // Safe cast
    val obj: Any = "Hello"
    val str: String? = obj as? String    // safe cast, returns null if fails
    val num: Int? = obj as? Int          // null, not throws
    println("str: $str, num: $num")
    
    // !! with custom message via require/checkNotNull
    fun processName(name: String?) {
        val n = name ?: throw IllegalArgumentException("Name cannot be null")
        println("Processing: $n")
    }
    
    fun safeName(name: String?) {
        val n = checkNotNull(name) { "Name is required" }
        println("Safe: $n")
    }
    
    processName("Alice")
    try { processName(null) } catch (e: Exception) { println("Caught: ${e.message}") }
    
    // Nullable collections
    val nullableList: List<String>? = null
    val safeList = nullableList ?: emptyList()
    println("List size: ${safeList.size}")
    
    // orEmpty() shortcut
    println("Empty: ${nullableList.orEmpty().size}")
}

fun main() = nullSafetyDemo()
```

---

## สรุป Part 21

| Kotlin Feature | Java Equivalent |
|----------------|----------------|
| `val` | `final` variable |
| `var` | regular variable |
| `data class` | class + equals/hashCode/toString/copy |
| `object` | Singleton |
| `companion object` | static members |
| `sealed class` | limited hierarchy |
| `?` (nullable) | no direct equivalent |
| `?.` (safe call) | if-null check |
| `?:` (Elvis) | ternary with null check |
| `when` | switch expression |
| Extension functions | utility class methods |

➡️ [Part 22: Kotlin - Coroutines](./Part-22-Kotlin-Coroutines.md)
