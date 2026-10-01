# Part 40: Ktor (Kotlin Server-Side Framework)
## ขั้นตอนที่ 2701-2770: Modern Kotlin Web Framework

---

## 40.1 Ktor Overview

```
Ktor = Kotlin-native async web framework by JetBrains
  - Coroutines-based (not blocking)
  - Lightweight, modular, plugin-based
  - Works on JVM, native, and JS
  - DSL-style routing

vs Spring Boot:
  Spring Boot   = opinionated, batteries-included, mature ecosystem
  Ktor          = lightweight, flexible, Kotlin-first, smaller startup
  
Best for:
  ✓ Microservices that need minimal footprint
  ✓ Kotlin Multiplatform backends
  ✓ Learning Kotlin-native APIs
  ✓ Simple APIs without Spring's complexity
```

---

## 40.2 Project Setup

```kotlin
// build.gradle.kts
plugins {
    application
    kotlin("jvm") version "2.0.0"
    id("io.ktor.plugin") version "2.3.12"
    kotlin("plugin.serialization") version "2.0.0"
}

application {
    mainClass.set("com.example.ApplicationKt")
}

val ktor_version = "2.3.12"
val kotlin_version = "2.0.0"
val logback_version = "1.4.14"

dependencies {
    // Ktor server
    implementation("io.ktor:ktor-server-core-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-netty-jvm:$ktor_version")
    
    // Plugins
    implementation("io.ktor:ktor-server-content-negotiation-jvm:$ktor_version")
    implementation("io.ktor:ktor-serialization-kotlinx-json-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-auth-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-auth-jwt-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-call-logging-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-request-validation-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-status-pages-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-cors-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-rate-limit-jvm:$ktor_version")
    implementation("io.ktor:ktor-server-metrics-micrometer-jvm:$ktor_version")
    
    // Ktor client
    implementation("io.ktor:ktor-client-core-jvm:$ktor_version")
    implementation("io.ktor:ktor-client-cio-jvm:$ktor_version")
    implementation("io.ktor:ktor-client-content-negotiation-jvm:$ktor_version")
    
    // Database
    implementation("org.jetbrains.exposed:exposed-core:0.53.0")
    implementation("org.jetbrains.exposed:exposed-dao:0.53.0")
    implementation("org.jetbrains.exposed:exposed-jdbc:0.53.0")
    implementation("org.jetbrains.exposed:exposed-java-time:0.53.0")
    implementation("com.zaxxer:HikariCP:5.1.0")
    implementation("org.postgresql:postgresql:42.7.3")
    
    // Logging
    implementation("ch.qos.logback:logback-classic:$logback_version")
    
    testImplementation("io.ktor:ktor-server-test-host-jvm:$ktor_version")
    testImplementation("org.jetbrains.kotlin:kotlin-test-junit5:$kotlin_version")
}
```

---

## 40.3 Application Entry Point

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.server.plugins.statuspages.*
import io.ktor.server.plugins.calllogging.*
import io.ktor.server.plugins.cors.routing.*
import io.ktor.serialization.kotlinx.json.*
import io.ktor.http.*
import io.ktor.server.response.*
import kotlinx.serialization.json.Json

fun main() {
    embeddedServer(Netty, port = 8080, host = "0.0.0.0") {
        module()
    }.start(wait = true)
}

fun Application.module() {
    configureSerialization()
    configureMonitoring()
    configureSecurity()
    configureCORS()
    configureStatusPages()
    configureDatabase()
    configureRouting()
}

fun Application.configureSerialization() {
    install(ContentNegotiation) {
        json(Json {
            prettyPrint = true
            isLenient = true
            ignoreUnknownKeys = true
            encodeDefaults = true
        })
    }
}

fun Application.configureMonitoring() {
    install(CallLogging) {
        level = org.slf4j.event.Level.INFO
        filter { call -> call.request.path().startsWith("/api") }
        format { call ->
            "[${call.request.httpMethod.value}] ${call.request.path()} " +
            "→ ${call.response.status()} (${call.processingTimeMillis()}ms)"
        }
    }
}

fun Application.configureCORS() {
    install(CORS) {
        allowMethod(HttpMethod.Options)
        allowMethod(HttpMethod.Get)
        allowMethod(HttpMethod.Post)
        allowMethod(HttpMethod.Put)
        allowMethod(HttpMethod.Delete)
        allowHeader(HttpHeaders.Authorization)
        allowHeader(HttpHeaders.ContentType)
        allowCredentials = true
        anyHost()  // Change to specific hosts in production
    }
}

fun Application.configureStatusPages() {
    install(StatusPages) {
        exception<NotFoundException> { call, cause ->
            call.respond(HttpStatusCode.NotFound, ErrorResponse("NOT_FOUND", cause.message ?: "Resource not found"))
        }
        exception<BadRequestException> { call, cause ->
            call.respond(HttpStatusCode.BadRequest, ErrorResponse("BAD_REQUEST", cause.message ?: "Bad request"))
        }
        exception<AuthorizationException> { call, cause ->
            call.respond(HttpStatusCode.Forbidden, ErrorResponse("FORBIDDEN", cause.message ?: "Forbidden"))
        }
        exception<Throwable> { call, cause ->
            call.application.log.error("Unhandled exception", cause)
            call.respond(HttpStatusCode.InternalServerError, ErrorResponse("INTERNAL_ERROR", "An error occurred"))
        }
    }
}

class NotFoundException(message: String) : RuntimeException(message)
class BadRequestException(message: String) : RuntimeException(message)
class AuthorizationException(message: String) : RuntimeException(message)
```

---

## 40.4 Routing

```kotlin
import io.ktor.server.application.*
import io.ktor.server.auth.*
import io.ktor.server.auth.jwt.*
import io.ktor.server.request.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.http.*

fun Application.configureRouting() {
    val userService = UserService()
    val productService = ProductService()
    
    routing {
        
        // Health check
        get("/health") {
            call.respond(mapOf("status" to "UP", "timestamp" to System.currentTimeMillis()))
        }
        
        // API v1
        route("/api/v1") {
            
            // Public routes
            route("/auth") {
                post("/login") {
                    val req = call.receive<LoginRequest>()
                    val response = userService.login(req)
                    call.respond(HttpStatusCode.OK, response)
                }
                
                post("/register") {
                    val req = call.receive<RegisterRequest>()
                    val user = userService.register(req)
                    call.respond(HttpStatusCode.Created, user)
                }
            }
            
            // Products (public)
            route("/products") {
                get {
                    val page = call.request.queryParameters["page"]?.toInt() ?: 0
                    val size = call.request.queryParameters["size"]?.toInt() ?: 20
                    val search = call.request.queryParameters["search"]
                    
                    val products = productService.findAll(page, size, search)
                    call.respond(products)
                }
                
                get("/{id}") {
                    val id = call.parameters["id"]?.toLongOrNull()
                        ?: throw BadRequestException("Invalid ID")
                    
                    val product = productService.findById(id)
                        ?: throw NotFoundException("Product not found: $id")
                    
                    call.respond(product)
                }
            }
            
            // Protected routes
            authenticate("jwt") {
                
                route("/users") {
                    get {
                        requireRole(call, "ADMIN")
                        val users = userService.findAll()
                        call.respond(users)
                    }
                    
                    get("/me") {
                        val principal = call.principal<JWTPrincipal>()!!
                        val userId = principal.getClaim("userId", Long::class)!!
                        val user = userService.findById(userId)
                            ?: throw NotFoundException("User not found")
                        call.respond(user)
                    }
                    
                    put("/me") {
                        val principal = call.principal<JWTPrincipal>()!!
                        val userId = principal.getClaim("userId", Long::class)!!
                        val req = call.receive<UpdateUserRequest>()
                        val updated = userService.update(userId, req)
                        call.respond(updated)
                    }
                }
                
                route("/orders") {
                    get {
                        val principal = call.principal<JWTPrincipal>()!!
                        val userId = principal.getClaim("userId", Long::class)!!
                        val orders = OrderService().findByUserId(userId)
                        call.respond(orders)
                    }
                    
                    post {
                        val principal = call.principal<JWTPrincipal>()!!
                        val userId = principal.getClaim("userId", Long::class)!!
                        val req = call.receive<CreateOrderRequest>()
                        val order = OrderService().create(userId, req)
                        call.respond(HttpStatusCode.Created, order)
                    }
                    
                    get("/{id}") {
                        val id = call.parameters["id"]?.toLongOrNull()
                            ?: throw BadRequestException("Invalid ID")
                        val order = OrderService().findById(id)
                            ?: throw NotFoundException("Order not found: $id")
                        call.respond(order)
                    }
                }
            }
        }
    }
}

fun requireRole(call: ApplicationCall, vararg roles: String) {
    val principal = call.principal<JWTPrincipal>()!!
    val userRole = principal.getClaim("role", String::class)
    if (userRole !in roles) {
        throw AuthorizationException("Required role: ${roles.joinToString()}")
    }
}
```

---

## 40.5 JWT Authentication

```kotlin
import com.auth0.jwt.JWT
import com.auth0.jwt.algorithms.Algorithm
import io.ktor.server.application.*
import io.ktor.server.auth.*
import io.ktor.server.auth.jwt.*
import io.ktor.http.*
import io.ktor.server.response.*
import java.util.Date

object JwtConfig {
    private val secret = System.getenv("JWT_SECRET") ?: "dev-secret-key-change-in-production"
    private val algorithm = Algorithm.HMAC256(secret)
    const val ISSUER = "ktor-app"
    const val AUDIENCE = "ktor-app-users"
    val TOKEN_EXPIRY_MS = 24 * 60 * 60 * 1000L  // 24 hours
    
    fun makeToken(userId: Long, email: String, role: String): String = JWT.create()
        .withIssuer(ISSUER)
        .withAudience(AUDIENCE)
        .withClaim("userId", userId)
        .withClaim("email", email)
        .withClaim("role", role)
        .withExpiresAt(Date(System.currentTimeMillis() + TOKEN_EXPIRY_MS))
        .sign(algorithm)
    
    val verifier = JWT.require(algorithm)
        .withIssuer(ISSUER)
        .withAudience(AUDIENCE)
        .build()
}

fun Application.configureSecurity() {
    install(Authentication) {
        jwt("jwt") {
            realm = "ktor-app"
            verifier(JwtConfig.verifier)
            
            validate { credential ->
                val userId = credential.payload.getClaim("userId").asLong()
                val role = credential.payload.getClaim("role").asString()
                
                if (userId != null && role != null) {
                    JWTPrincipal(credential.payload)
                } else {
                    null  // reject
                }
            }
            
            challenge { _, _ ->
                call.respond(
                    HttpStatusCode.Unauthorized,
                    ErrorResponse("UNAUTHORIZED", "Invalid or expired token")
                )
            }
        }
    }
}
```

---

## 40.6 Database with Exposed ORM

```kotlin
import org.jetbrains.exposed.dao.*
import org.jetbrains.exposed.dao.id.*
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction
import org.jetbrains.exposed.sql.javatime.*
import java.time.Instant

// ====== Table definitions ======
object UsersTable : LongIdTable("users") {
    val username  = varchar("username", 50).uniqueIndex()
    val email     = varchar("email", 255).uniqueIndex()
    val password  = varchar("password", 255)
    val role      = varchar("role", 20).default("USER")
    val status    = varchar("status", 20).default("ACTIVE")
    val createdAt = timestamp("created_at").defaultExpression(CurrentTimestamp)
}

object ProductsTable : LongIdTable("products") {
    val name        = varchar("name", 255)
    val description = text("description").nullable()
    val price       = decimal("price", 10, 2)
    val stock       = integer("stock").default(0)
    val categoryId  = reference("category_id", CategoriesTable)
    val createdAt   = timestamp("created_at").defaultExpression(CurrentTimestamp)
}

object CategoriesTable : LongIdTable("categories") {
    val name     = varchar("name", 100)
    val slug     = varchar("slug", 100).uniqueIndex()
    val parentId = reference("parent_id", CategoriesTable).nullable()
}

// ====== DAO entities ======
class UserEntity(id: EntityID<Long>) : LongEntity(id) {
    companion object : LongEntityClass<UserEntity>(UsersTable)
    
    var username  by UsersTable.username
    var email     by UsersTable.email
    var password  by UsersTable.password
    var role      by UsersTable.role
    var status    by UsersTable.status
    var createdAt by UsersTable.createdAt
    
    fun toDto() = UserDTO(id.value, username, email, role)
}

class ProductEntity(id: EntityID<Long>) : LongEntity(id) {
    companion object : LongEntityClass<ProductEntity>(ProductsTable)
    
    var name        by ProductsTable.name
    var description by ProductsTable.description
    var price       by ProductsTable.price
    var stock       by ProductsTable.stock
    var category    by CategoryEntity referencedOn ProductsTable.categoryId
    var createdAt   by ProductsTable.createdAt
    
    fun toDto() = ProductDTO(id.value, name, description, price.toDouble(), stock, category.name)
}

class CategoryEntity(id: EntityID<Long>) : LongEntity(id) {
    companion object : LongEntityClass<CategoryEntity>(CategoriesTable)
    
    var name by CategoriesTable.name
    var slug by CategoriesTable.slug
}

// ====== Database config ======
fun Application.configureDatabase() {
    val config = com.zaxxer.hikari.HikariConfig().apply {
        jdbcUrl = System.getenv("DATABASE_URL") ?: "jdbc:postgresql://localhost:5432/mydb"
        username = System.getenv("DB_USER") ?: "postgres"
        password = System.getenv("DB_PASSWORD") ?: "postgres"
        maximumPoolSize = 20
        isAutoCommit = false
        transactionIsolation = "TRANSACTION_REPEATABLE_READ"
    }
    
    val dataSource = com.zaxxer.hikari.HikariDataSource(config)
    
    Database.connect(dataSource)
    
    transaction {
        addLogger(StdOutSqlLogger)
        SchemaUtils.create(UsersTable, CategoriesTable, ProductsTable)
    }
}

// ====== Repository ======
class UserRepository {
    
    fun findAll(): List<UserDTO> = transaction {
        UserEntity.all().map { it.toDto() }
    }
    
    fun findById(id: Long): UserDTO? = transaction {
        UserEntity.findById(id)?.toDto()
    }
    
    fun findByEmail(email: String): UserEntity? = transaction {
        UserEntity.find { UsersTable.email eq email }.singleOrNull()
    }
    
    fun create(username: String, email: String, hashedPassword: String): UserDTO = transaction {
        UserEntity.new {
            this.username = username
            this.email = email
            this.password = hashedPassword
        }.toDto()
    }
    
    fun update(id: Long, name: String? = null, email: String? = null): UserDTO? = transaction {
        UserEntity.findByIdAndUpdate(id) { user ->
            name?.let { user.username = it }
            email?.let { user.email = it }
        }?.toDto()
    }
}
```

---

## 40.7 Testing Ktor

```kotlin
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.server.testing.*
import kotlin.test.*

class ApplicationTest {
    
    @Test
    fun testHealthCheck() = testApplication {
        application { module() }
        
        val response = client.get("/health")
        
        assertEquals(HttpStatusCode.OK, response.status)
        assertTrue(response.bodyAsText().contains("UP"))
    }
    
    @Test
    fun testLogin_success() = testApplication {
        application { module() }
        
        val response = client.post("/api/v1/auth/login") {
            contentType(ContentType.Application.Json)
            setBody("""{"email":"admin@example.com","password":"password123"}""")
        }
        
        assertEquals(HttpStatusCode.OK, response.status)
        val body = response.bodyAsText()
        assertTrue(body.contains("token"))
    }
    
    @Test
    fun testProtectedRoute_withoutToken() = testApplication {
        application { module() }
        
        val response = client.get("/api/v1/users/me")
        
        assertEquals(HttpStatusCode.Unauthorized, response.status)
    }
    
    @Test
    fun testProtectedRoute_withToken() = testApplication {
        application { module() }
        
        val token = JwtConfig.makeToken(1L, "test@example.com", "USER")
        
        val response = client.get("/api/v1/users/me") {
            bearerAuth(token)
        }
        
        assertEquals(HttpStatusCode.OK, response.status)
    }
}
```

---

## สรุป Part 40

| Ktor Concept | คำอธิบาย |
|-------------|---------|
| `embeddedServer` | Start Ktor server |
| `install()` | Install plugins |
| `routing {}` | Define routes |
| `authenticate {}` | Protect routes |
| `call.receive<T>()` | Parse request body |
| `call.respond()` | Send response |
| `testApplication {}` | Test with in-memory server |

**Ktor Plugin Ecosystem:**
- `ContentNegotiation` = JSON serialization
- `CallLogging` = request logging
- `StatusPages` = exception handling
- `CORS` = cross-origin requests
- `Authentication` = JWT, basic, OAuth
- `RateLimit` = request throttling

➡️ [Part 41: Spring WebFlux & Reactive Programming](./Part-41-WebFlux.md)
