# Part 57: Kotlin Multiplatform (KMP)
## ขั้นตอนที่ 3891-3960: Share Code Across Android, iOS, Desktop

---

## 57.1 KMP Project Structure

```
Kotlin Multiplatform Project Layout:

shared/
├── src/
│   ├── commonMain/          ← shared code (all platforms)
│   │   └── kotlin/
│   │       ├── domain/
│   │       │   ├── models/
│   │       │   └── repositories/
│   │       ├── data/
│   │       │   ├── remote/  (Ktor HTTP client)
│   │       │   └── local/   (SQLDelight)
│   │       └── presentation/
│   │           └── viewmodel/
│   │
│   ├── androidMain/         ← Android-specific implementations
│   │   └── kotlin/
│   │       └── AndroidPlatform.kt
│   │
│   ├── iosMain/             ← iOS-specific implementations
│   │   └── kotlin/
│   │       └── IOSPlatform.kt
│   │
│   └── desktopMain/         ← Desktop (JVM) implementations
│       └── kotlin/
│           └── DesktopPlatform.kt
│
androidApp/                  ← Android application
iosApp/                      ← iOS application (Xcode)
desktopApp/                  ← Compose Desktop application
```

---

## 57.2 Shared Domain & Data Layer

```kotlin
// commonMain/kotlin/domain/models/Product.kt
data class Product(
    val id: String,
    val name: String,
    val price: Double,
    val category: String,
    val imageUrl: String
)

data class CartItem(
    val product: Product,
    val quantity: Int
) {
    val subtotal: Double get() = product.price * quantity
}

// commonMain/kotlin/domain/repositories/ProductRepository.kt
interface ProductRepository {
    suspend fun getProducts(category: String? = null): List<Product>
    suspend fun getProduct(id: String): Product?
    suspend fun searchProducts(query: String): List<Product>
}

// commonMain/kotlin/data/remote/ProductApiService.kt
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.Serializable

@Serializable
data class ProductResponse(
    val id: String,
    val name: String,
    val price: Double,
    val category: String,
    val imageUrl: String
)

class ProductApiService(private val client: HttpClient) {
    
    suspend fun getProducts(category: String?): List<ProductResponse> {
        return client.get("https://api.example.com/products") {
            category?.let { parameter("category", it) }
        }.body()
    }
    
    suspend fun getProduct(id: String): ProductResponse {
        return client.get("https://api.example.com/products/$id").body()
    }
}

// Shared HTTP client factory
fun createHttpClient(): HttpClient = HttpClient {
    install(ContentNegotiation) {
        json(kotlinx.serialization.json.Json {
            ignoreUnknownKeys = true
            isLenient = true
        })
    }
    install(io.ktor.client.plugins.HttpTimeout) {
        requestTimeoutMillis = 10_000
        connectTimeoutMillis = 5_000
    }
    install(io.ktor.client.plugins.logging.Logging) {
        level = io.ktor.client.plugins.logging.LogLevel.INFO
    }
}

// Repository implementation (shared)
class ProductRepositoryImpl(
    private val apiService: ProductApiService
) : ProductRepository {
    
    override suspend fun getProducts(category: String?) =
        apiService.getProducts(category).map { it.toDomain() }
    
    override suspend fun getProduct(id: String) =
        apiService.getProduct(id).toDomain()
    
    override suspend fun searchProducts(query: String) =
        apiService.getProducts(null)
            .filter { it.name.contains(query, ignoreCase = true) }
            .map { it.toDomain() }
    
    private fun ProductResponse.toDomain() = Product(id, name, price, category, imageUrl)
}
```

---

## 57.3 Shared ViewModel (KMP)

```kotlin
// commonMain/kotlin/presentation/viewmodel/ProductViewModel.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class ProductViewModel(
    private val productRepository: ProductRepository,
    private val coroutineScope: CoroutineScope
) {
    
    private val _uiState = MutableStateFlow<ProductUiState>(ProductUiState.Loading)
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()
    
    private val _searchQuery = MutableStateFlow("")
    
    init {
        // React to search query changes
        coroutineScope.launch {
            _searchQuery
                .debounce(300)
                .distinctUntilChanged()
                .collectLatest { query ->
                    loadProducts(query)
                }
        }
    }
    
    fun loadProducts(query: String = "") {
        coroutineScope.launch {
            _uiState.value = ProductUiState.Loading
            try {
                val products = if (query.isBlank()) {
                    productRepository.getProducts()
                } else {
                    productRepository.searchProducts(query)
                }
                _uiState.value = ProductUiState.Success(products)
            } catch (e: Exception) {
                _uiState.value = ProductUiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun onSearchQueryChanged(query: String) {
        _searchQuery.value = query
    }
}

sealed class ProductUiState {
    object Loading : ProductUiState()
    data class Success(val products: List<Product>) : ProductUiState()
    data class Error(val message: String) : ProductUiState()
}
```

---

## 57.4 Platform-Specific Code (expect/actual)

```kotlin
// commonMain: declare expected API
expect class Platform {
    val name: String
    val version: String
}

expect fun currentTimeMillis(): Long
expect fun generateUUID(): String
expect fun log(message: String)

// commonMain: shared code using expect
fun greet(): String = "Hello from ${Platform().name} ${Platform().version}"

// -----------------------------------------------
// androidMain: actual implementation
actual class Platform {
    actual val name: String = "Android"
    actual val version: String = android.os.Build.VERSION.RELEASE
}

actual fun currentTimeMillis(): Long = System.currentTimeMillis()
actual fun generateUUID(): String = java.util.UUID.randomUUID().toString()
actual fun log(message: String) { android.util.Log.d("KMP", message) }

// -----------------------------------------------
// iosMain: actual implementation
actual class Platform {
    actual val name: String = "iOS"
    actual val version: String = platform.UIKit.UIDevice.currentDevice.systemVersion
}

actual fun currentTimeMillis(): Long = 
    (platform.Foundation.NSDate.date().timeIntervalSince1970 * 1000).toLong()

actual fun generateUUID(): String =
    platform.Foundation.NSUUID().UUIDString

actual fun log(message: String) { println(message) }

// -----------------------------------------------
// desktopMain: actual implementation
actual class Platform {
    actual val name: String = "Desktop"
    actual val version: String = System.getProperty("java.version")
}

actual fun currentTimeMillis(): Long = System.currentTimeMillis()
actual fun generateUUID(): String = java.util.UUID.randomUUID().toString()
actual fun log(message: String) { println("[Desktop] $message") }
```

---

## 57.5 SQLDelight for Shared Database

```sql
-- commonMain/sqldelight/com/example/Product.sq
CREATE TABLE products (
    id TEXT NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    price REAL NOT NULL,
    category TEXT NOT NULL,
    imageUrl TEXT NOT NULL,
    cachedAt INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);

-- Named queries
selectAll:
SELECT * FROM products ORDER BY name;

selectByCategory:
SELECT * FROM products WHERE category = ? ORDER BY name;

insertProduct:
INSERT OR REPLACE INTO products VALUES (?, ?, ?, ?, ?, ?);

deleteAll:
DELETE FROM products;

deleteExpired:
DELETE FROM products WHERE cachedAt < ?;
```

```kotlin
// commonMain: SQLDelight database usage
import com.example.db.AppDatabase

class LocalProductDataSource(private val db: AppDatabase) {
    
    fun getAll(): List<Product> =
        db.productQueries.selectAll().executeAsList()
            .map { it.toDomain() }
    
    fun getByCategory(category: String): List<Product> =
        db.productQueries.selectByCategory(category).executeAsList()
            .map { it.toDomain() }
    
    fun insertAll(products: List<Product>) {
        db.productQueries.transaction {
            products.forEach { p ->
                db.productQueries.insertProduct(
                    id = p.id,
                    name = p.name,
                    price = p.price,
                    category = p.category,
                    imageUrl = p.imageUrl,
                    cachedAt = currentTimeMillis() / 1000
                )
            }
        }
    }
    
    private fun com.example.db.Products.toDomain() =
        Product(id, name, price, category, imageUrl)
}

// Cache-first repository
class CachingProductRepository(
    private val local: LocalProductDataSource,
    private val remote: ProductApiService
) : ProductRepository {
    
    override suspend fun getProducts(category: String?): List<Product> {
        val cached = if (category != null) local.getByCategory(category)
                     else local.getAll()
        
        if (cached.isNotEmpty()) return cached
        
        val fresh = remote.getProducts(category).map {
            Product(it.id, it.name, it.price, it.category, it.imageUrl)
        }
        local.insertAll(fresh)
        return fresh
    }
    
    override suspend fun getProduct(id: String): Product? =
        remote.getProduct(id).let {
            Product(it.id, it.name, it.price, it.category, it.imageUrl)
        }
    
    override suspend fun searchProducts(query: String): List<Product> =
        remote.getProducts(null)
            .filter { it.name.contains(query, ignoreCase = true) }
            .map { Product(it.id, it.name, it.price, it.category, it.imageUrl) }
}
```

---

## 57.6 Compose Multiplatform UI

```kotlin
// commonMain: Compose Multiplatform (shared UI)
import androidx.compose.runtime.*
import androidx.compose.material3.*
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.*

@Composable
fun ProductListScreen(viewModel: ProductViewModel) {
    val uiState by viewModel.uiState.collectAsState()
    var searchQuery by remember { mutableStateOf("") }
    
    Column {
        // Search bar
        OutlinedTextField(
            value = searchQuery,
            onValueChange = { query ->
                searchQuery = query
                viewModel.onSearchQueryChanged(query)
            },
            label = { Text("Search products") },
            modifier = Modifier.fillMaxWidth().padding(16.dp)
        )
        
        // State-based content
        when (val state = uiState) {
            is ProductUiState.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = androidx.compose.ui.Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is ProductUiState.Success -> {
                LazyColumn {
                    items(state.products) { product ->
                        ProductCard(product)
                    }
                }
            }
            is ProductUiState.Error -> {
                Column(Modifier.fillMaxSize(), 
                       verticalArrangement = Arrangement.Center,
                       horizontalAlignment = androidx.compose.ui.Alignment.CenterHorizontally) {
                    Text("Error: ${state.message}", color = MaterialTheme.colorScheme.error)
                    Button(onClick = { viewModel.loadProducts() }) {
                        Text("Retry")
                    }
                }
            }
        }
    }
}

@Composable
fun ProductCard(product: Product) {
    Card(
        modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp, vertical = 8.dp)
    ) {
        Row(Modifier.padding(16.dp)) {
            Column(Modifier.weight(1f)) {
                Text(product.name, style = MaterialTheme.typography.titleMedium)
                Text(product.category, style = MaterialTheme.typography.bodySmall,
                     color = MaterialTheme.colorScheme.outline)
            }
            Text(
                "฿${String.format("%.2f", product.price)}",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

---

## สรุป Part 57

```
Kotlin Multiplatform:

Share Code Between Platforms:
  Android + iOS + Desktop + Web (JS/WASM)
  
  commonMain = platform-independent code
    ✓ Domain models (data classes)
    ✓ Repository interfaces
    ✓ Business logic
    ✓ ViewModels (with StateFlow)
    ✓ Network (Ktor HTTP client)
    ✓ Database (SQLDelight)

  Platform-specific:
    androidMain = Android APIs
    iosMain     = iOS APIs
    desktopMain = JVM APIs

  expect/actual = platform-specific implementations
    Platform class, UUID, logging, date/time

Libraries (KMP-compatible):
  Ktor         = HTTP client
  SQLDelight   = type-safe SQL for all platforms
  kotlinx.serialization = JSON serialization
  kotlinx.coroutines    = async programming
  Compose Multiplatform = shared UI

Benefits:
  ✓ Write logic once, deploy everywhere
  ✓ Consistent business rules across platforms
  ✓ Platform-native UI (or shared Compose UI)
```

➡️ [Part 58: Android Jetpack Compose](./Part-58-JetpackCompose.md)
