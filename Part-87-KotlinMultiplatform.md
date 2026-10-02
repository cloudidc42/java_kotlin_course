# Part 87: Kotlin Multiplatform (KMP)
## ขั้นตอนที่ 5991-6060: Share Code Across Android, iOS, Web, Desktop

---

## 87.1 Kotlin Multiplatform Overview

```
KMP = เขียน Kotlin code หนึ่งครั้ง → รันบนหลาย platform

Targets:
  Android    = Kotlin/JVM (native Android)
  iOS        = Kotlin/Native → compiles to native binary
  Web        = Kotlin/JS → compiles to JavaScript
  Desktop    = Kotlin/JVM (Windows, Mac, Linux via Compose Multiplatform)
  Server     = Kotlin/JVM (Spring Boot, Ktor)
  
Code Sharing Architecture:
  ┌────────────────────────────────────────┐
  │         commonMain (shared code)       │
  │   Business Logic, Domain Models        │
  │   Network, Storage (via expect/actual) │
  ├──────────┬────────────┬───────────────┤
  │androidMain│   iosMain  │     jsMain    │
  │Android-   │iOS-specific│Web-specific   │
  │specific   │code        │code           │
  └──────────┴────────────┴───────────────┘

expect/actual:
  expect = ประกาศใน commonMain (ต้องมี implementation)
  actual = implementation เฉพาะ platform

What to share:
  ✓ Domain models, business logic
  ✓ API clients, serialization
  ✓ Validation rules
  ✓ ViewModels (with Compose Multiplatform)
  
What NOT to share:
  ✗ Platform-specific UI (unless Compose Multiplatform)
  ✗ System APIs (camera, GPS) → use expect/actual
```

---

## 87.2 Project Setup

```kotlin
// build.gradle.kts (shared module)
plugins {
    kotlin("multiplatform")
    kotlin("plugin.serialization")
    id("com.android.library")
    id("app.cash.sqldelight")
}

kotlin {
    // Android target
    androidTarget {
        compilations.all {
            kotlinOptions.jvmTarget = "11"
        }
    }
    
    // iOS targets
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "Shared"
            isStatic = true
        }
    }
    
    // Web target (JS)
    js(IR) {
        browser()
        nodejs()
    }
    
    // Desktop JVM
    jvm("desktop")
    
    sourceSets {
        // Shared across ALL platforms
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.0")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")
                implementation("io.ktor:ktor-client-core:2.3.8")
                implementation("io.ktor:ktor-client-content-negotiation:2.3.8")
                implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.8")
                implementation("app.cash.sqldelight:runtime:2.0.1")
            }
        }
        
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.0")
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-okhttp:2.3.8")
                implementation("app.cash.sqldelight:android-driver:2.0.1")
            }
        }
        
        val iosMain by creating {
            dependsOn(commonMain)
            dependencies {
                implementation("io.ktor:ktor-client-darwin:2.3.8")
                implementation("app.cash.sqldelight:native-driver:2.0.1")
            }
        }
        
        val iosX64Main by getting { dependsOn(iosMain) }
        val iosArm64Main by getting { dependsOn(iosMain) }
        val iosSimulatorArm64Main by getting { dependsOn(iosMain) }
        
        val jsMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-js:2.3.8")
            }
        }
    }
}
```

---

## 87.3 Shared Domain Models

```kotlin
// commonMain/kotlin/com/example/shared/domain/Product.kt

import kotlinx.serialization.Serializable
import kotlinx.serialization.SerialName

@Serializable
data class Product(
    val id: String,
    val name: String,
    val description: String,
    val price: Double,
    
    @SerialName("image_url")
    val imageUrl: String,
    
    val category: String,
    val rating: Float,
    
    @SerialName("review_count")
    val reviewCount: Int,
    
    @SerialName("in_stock")
    val inStock: Boolean
)

@Serializable
data class Cart(
    val items: List<CartItem> = emptyList()
) {
    val total: Double get() = items.sumOf { it.subtotal }
    val itemCount: Int get() = items.sumOf { it.quantity }
    
    fun addItem(product: Product, quantity: Int = 1): Cart {
        val existingItem = items.find { it.productId == product.id }
        val updatedItems = if (existingItem != null) {
            items.map { 
                if (it.productId == product.id) it.copy(quantity = it.quantity + quantity) 
                else it 
            }
        } else {
            items + CartItem(product.id, product.name, product.price, quantity, product.imageUrl)
        }
        return copy(items = updatedItems)
    }
    
    fun removeItem(productId: String) = copy(items = items.filter { it.productId != productId })
    
    fun updateQuantity(productId: String, quantity: Int): Cart {
        if (quantity <= 0) return removeItem(productId)
        return copy(items = items.map { 
            if (it.productId == productId) it.copy(quantity = quantity) else it 
        })
    }
}

@Serializable
data class CartItem(
    val productId: String,
    val productName: String,
    val unitPrice: Double,
    val quantity: Int,
    val imageUrl: String
) {
    val subtotal: Double get() = unitPrice * quantity
}
```

---

## 87.4 Shared API Client (Ktor)

```kotlin
// commonMain/kotlin/com/example/shared/data/api/ProductApi.kt

import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.json.Json

class ProductApiClient(baseUrl: String) {
    
    private val client = HttpClient {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                isLenient = true
            })
        }
        install(io.ktor.client.plugins.HttpTimeout) {
            requestTimeoutMillis = 30_000
            connectTimeoutMillis = 10_000
        }
    }
    
    private val baseUrl = baseUrl.trimEnd('/')
    
    suspend fun getProducts(
        category: String? = null,
        search: String? = null,
        page: Int = 0,
        size: Int = 20
    ): Result<List<Product>> = runCatching {
        client.get("$baseUrl/api/v1/products") {
            parameter("page", page)
            parameter("size", size)
            category?.let { parameter("category", it) }
            search?.let { parameter("search", it) }
        }.body<List<Product>>()
    }
    
    suspend fun getProduct(id: String): Result<Product> = runCatching {
        client.get("$baseUrl/api/v1/products/$id").body()
    }
    
    suspend fun searchProducts(query: String): Result<List<Product>> = runCatching {
        client.get("$baseUrl/api/v1/products") {
            parameter("search", query)
        }.body<List<Product>>()
    }
}
```

---

## 87.5 expect/actual for Platform-specific Code

```kotlin
// ====== commonMain: declare expect ======

// Platform-specific database driver
expect class DatabaseDriver(databasePath: String) {
    fun close()
}

// Platform-specific preferences/settings storage
expect class Preferences(name: String) {
    fun getString(key: String, default: String?): String?
    fun putString(key: String, value: String)
    fun remove(key: String)
    fun clear()
}

// Platform logger
expect fun platformLog(tag: String, message: String)

// Coroutine dispatcher for UI thread
expect val MainDispatcher: kotlinx.coroutines.CoroutineDispatcher

// ====== androidMain: actual implementation ======
actual class DatabaseDriver actual constructor(databasePath: String) {
    val driver: SqlDriver = AndroidSqliteDriver(
        AppDatabase.Schema,
        applicationContext,
        databasePath
    )
    actual fun close() = driver.close()
}

actual class Preferences actual constructor(name: String) {
    private val prefs = applicationContext.getSharedPreferences(name, Context.MODE_PRIVATE)
    
    actual fun getString(key: String, default: String?) = prefs.getString(key, default)
    actual fun putString(key: String, value: String) = prefs.edit().putString(key, value).apply()
    actual fun remove(key: String) = prefs.edit().remove(key).apply()
    actual fun clear() = prefs.edit().clear().apply()
}

actual fun platformLog(tag: String, message: String) = android.util.Log.d(tag, message)
actual val MainDispatcher = Dispatchers.Main

// ====== iosMain: actual implementation ======
actual class DatabaseDriver actual constructor(databasePath: String) {
    val driver: SqlDriver = NativeSqliteDriver(AppDatabase.Schema, databasePath)
    actual fun close() = driver.close()
}

actual class Preferences actual constructor(name: String) {
    private val userDefaults = NSUserDefaults.standardUserDefaults
    private val prefix = "$name."
    
    actual fun getString(key: String, default: String?) = 
        userDefaults.stringForKey(prefix + key) ?: default
    actual fun putString(key: String, value: String) = 
        userDefaults.setObject(value, prefix + key)
    actual fun remove(key: String) = userDefaults.removeObjectForKey(prefix + key)
    actual fun clear() = userDefaults.removePersistentDomainForName(prefix)
}

actual fun platformLog(tag: String, message: String) = println("[$tag] $message")
actual val MainDispatcher = Dispatchers.Main

// ====== jsMain: actual implementation ======
actual class DatabaseDriver actual constructor(databasePath: String) {
    // IndexedDB or in-memory for web
    val driver: SqlDriver = WebWorkerDriver(...)
    actual fun close() = driver.close()
}

actual class Preferences actual constructor(name: String) {
    private val prefix = "$name."
    
    actual fun getString(key: String, default: String?) = 
        kotlinx.browser.localStorage.getItem(prefix + key) ?: default
    actual fun putString(key: String, value: String) = 
        kotlinx.browser.localStorage.setItem(prefix + key, value)
    actual fun remove(key: String) = 
        kotlinx.browser.localStorage.removeItem(prefix + key)
    actual fun clear() {
        kotlinx.browser.localStorage.keys()
            .filter { it.startsWith(prefix) }
            .forEach { kotlinx.browser.localStorage.removeItem(it) }
    }
}

actual fun platformLog(tag: String, message: String) = console.log("[$tag] $message")
actual val MainDispatcher = Dispatchers.Main
```

---

## 87.6 Shared ViewModel

```kotlin
// commonMain: shared ViewModel using Coroutines
// Works with Compose Multiplatform

abstract class BaseViewModel {
    protected val viewModelScope = CoroutineScope(
        SupervisorJob() + MainDispatcher
    )
    
    open fun onCleared() {
        viewModelScope.cancel()
    }
}

class ProductListViewModel(
    private val productApi: ProductApiClient,
    private val preferences: Preferences
) : BaseViewModel() {
    
    private val _state = MutableStateFlow(ProductListState())
    val state: StateFlow<ProductListState> = _state.asStateFlow()
    
    init {
        loadProducts()
    }
    
    fun loadProducts(category: String? = null) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            
            productApi.getProducts(category = category)
                .onSuccess { products ->
                    _state.update { it.copy(products = products, isLoading = false) }
                }
                .onFailure { error ->
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
        }
    }
    
    fun addToFavorites(productId: String) {
        val favorites = getFavorites().toMutableSet()
        favorites.add(productId)
        preferences.putString("favorites", favorites.joinToString(","))
    }
    
    private fun getFavorites(): Set<String> =
        preferences.getString("favorites", null)
            ?.split(",")?.toSet() ?: emptySet()
}

data class ProductListState(
    val products: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

// Android: use in Compose
// iOS: use in SwiftUI via StateFlow observer
// Web: use in Compose/HTML
```

---

## สรุป Part 87

```
Kotlin Multiplatform:

Code Sharing:
  commonMain  = shared everywhere
  androidMain = Android-specific
  iosMain     = iOS-specific
  jsMain      = Web-specific

expect/actual Pattern:
  expect class = interface/contract in commonMain
  actual class = platform implementation
  
  Use for: database driver, preferences, logging,
           datetime, crypto, platform features

Ktor for HTTP (works everywhere):
  HTTP client that works on all platforms
  Replace OkHttp (Android only) with Ktor

SQLDelight for Database:
  SQL first: write .sq files → generates Kotlin
  Works: Android (SQLite), iOS (SQLite), JVM (JDBC)

Compose Multiplatform:
  Share UI code across Android + Desktop + Web (beta)
  Still use platform-specific for iOS

KMP Maturity:
  ✓ Stable: domain/business logic sharing
  ✓ Stable: Ktor network, SQLDelight storage
  ✓ Stable: Android + iOS code sharing
  ⚡ Beta: Compose Multiplatform for iOS
  ⚡ Alpha: Kotlin/Wasm for Web

Practical Advice:
  Start with sharing models + API client only
  Add business logic gradually
  Keep platform UI separate until comfortable
```

➡️ [Part 88: Java Decompilation & APK Analysis](./Part-88-APKAnalysis.md)
