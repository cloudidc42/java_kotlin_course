# Part 79: Android Development with Kotlin (MVI + Clean Architecture)
## ขั้นตอนที่ 5431-5500: Modern Android Architecture, Jetpack Compose, Room, Hilt

---

## 79.1 Modern Android Architecture

```
Clean Architecture for Android:

  ┌─────────────────────────────────────┐
  │         Presentation Layer          │
  │  Composable UI → ViewModel → State  │
  ├─────────────────────────────────────┤
  │           Domain Layer              │
  │  UseCases, Entities, Repository IF  │
  ├─────────────────────────────────────┤
  │            Data Layer               │
  │  Remote (Retrofit), Local (Room)    │
  └─────────────────────────────────────┘

MVI Pattern:
  Model    = State (immutable data class)
  View     = Composable (render from state)
  Intent   = UserAction/Event (user actions)
  
  Flow:
  User Action → ViewModel.handleAction() → update State → Recompose UI

Dependency Direction:
  Presentation → Domain ← Data
  (Domain knows nothing about Android framework)
```

---

## 79.2 Gradle Setup (Kotlin DSL)

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.hilt.android)
    alias(libs.plugins.ksp)
}

android {
    namespace = "com.example.shop"
    compileSdk = 35
    
    defaultConfig {
        applicationId = "com.example.shop"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"
    }
    
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_21
        targetCompatibility = JavaVersion.VERSION_21
    }
    
    kotlinOptions {
        jvmTarget = "21"
    }
    
    buildFeatures {
        compose = true
        buildConfig = true
    }
}

dependencies {
    // Compose BOM
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)
    implementation(libs.compose.material3)
    implementation(libs.compose.ui.tooling.preview)
    debugImplementation(libs.compose.ui.tooling)
    
    // Navigation
    implementation(libs.navigation.compose)
    
    // ViewModel
    implementation(libs.lifecycle.viewmodel.compose)
    implementation(libs.lifecycle.runtime.compose)
    
    // Hilt
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.hilt.navigation.compose)
    
    // Room
    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)
    
    // Retrofit + Kotlinx Serialization
    implementation(libs.retrofit.core)
    implementation(libs.retrofit.kotlinx.serialization)
    implementation(libs.kotlinx.serialization.json)
    implementation(libs.okhttp.logging)
    
    // Coroutines
    implementation(libs.kotlinx.coroutines.android)
    
    // Coil (image loading)
    implementation(libs.coil.compose)
}
```

---

## 79.3 Domain Layer

```kotlin
// domain/model/Product.kt
data class Product(
    val id: String,
    val name: String,
    val description: String,
    val price: Double,
    val imageUrl: String,
    val category: String,
    val rating: Float,
    val reviewCount: Int,
    val inStock: Boolean
)

data class CartItem(
    val product: Product,
    val quantity: Int
) {
    val subtotal: Double get() = product.price * quantity
}

// domain/repository/ProductRepository.kt
interface ProductRepository {
    suspend fun getProducts(category: String? = null): Result<List<Product>>
    suspend fun getProduct(id: String): Result<Product>
    fun searchProducts(query: String): Flow<List<Product>>
}

interface CartRepository {
    fun getCart(): Flow<List<CartItem>>
    suspend fun addToCart(product: Product, quantity: Int)
    suspend fun updateQuantity(productId: String, quantity: Int)
    suspend fun removeFromCart(productId: String)
    suspend fun clearCart()
}

// domain/usecase/AddToCartUseCase.kt
class AddToCartUseCase(
    private val cartRepository: CartRepository,
    private val productRepository: ProductRepository
) {
    suspend operator fun invoke(productId: String, quantity: Int): Result<Unit> {
        val product = productRepository.getProduct(productId).getOrElse {
            return Result.failure(it)
        }
        
        if (!product.inStock) {
            return Result.failure(Exception("Product is out of stock"))
        }
        
        if (quantity <= 0) {
            return Result.failure(Exception("Quantity must be positive"))
        }
        
        cartRepository.addToCart(product, quantity)
        return Result.success(Unit)
    }
}

class GetCartSummaryUseCase(
    private val cartRepository: CartRepository
) {
    operator fun invoke(): Flow<CartSummary> = cartRepository.getCart().map { items ->
        CartSummary(
            items = items,
            total = items.sumOf { it.subtotal },
            itemCount = items.sumOf { it.quantity }
        )
    }
}

data class CartSummary(
    val items: List<CartItem>,
    val total: Double,
    val itemCount: Int
)
```

---

## 79.4 MVI ViewModel

```kotlin
// presentation/product/ProductListViewModel.kt

data class ProductListState(
    val products: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = "",
    val selectedCategory: String? = null,
    val cartItemCount: Int = 0
)

sealed interface ProductListEvent {
    data class SearchQueryChanged(val query: String) : ProductListEvent
    data class CategorySelected(val category: String?) : ProductListEvent
    data class AddToCart(val productId: String) : ProductListEvent
    data object Refresh : ProductListEvent
}

sealed interface ProductListEffect {
    data class ShowSnackbar(val message: String) : ProductListEffect
    data class NavigateToProduct(val productId: String) : ProductListEffect
}

@HiltViewModel
class ProductListViewModel @Inject constructor(
    private val productRepository: ProductRepository,
    private val addToCartUseCase: AddToCartUseCase,
    private val getCartSummaryUseCase: GetCartSummaryUseCase
) : ViewModel() {
    
    private val _state = MutableStateFlow(ProductListState())
    val state: StateFlow<ProductListState> = _state.asStateFlow()
    
    private val _effects = MutableSharedFlow<ProductListEffect>()
    val effects: SharedFlow<ProductListEffect> = _effects.asSharedFlow()
    
    private val searchQuery = MutableStateFlow("")
    
    init {
        // Cart count
        viewModelScope.launch {
            getCartSummaryUseCase()
                .collect { summary ->
                    _state.update { it.copy(cartItemCount = summary.itemCount) }
                }
        }
        
        // Search with debounce
        viewModelScope.launch {
            searchQuery
                .debounce(300)
                .distinctUntilChanged()
                .collect { query ->
                    loadProducts(query, _state.value.selectedCategory)
                }
        }
        
        loadProducts()
    }
    
    fun handleEvent(event: ProductListEvent) {
        when (event) {
            is ProductListEvent.SearchQueryChanged -> {
                _state.update { it.copy(searchQuery = event.query) }
                searchQuery.value = event.query
            }
            is ProductListEvent.CategorySelected -> {
                _state.update { it.copy(selectedCategory = event.category) }
                loadProducts(_state.value.searchQuery, event.category)
            }
            is ProductListEvent.AddToCart -> addToCart(event.productId)
            is ProductListEvent.Refresh -> loadProducts(
                _state.value.searchQuery, _state.value.selectedCategory
            )
        }
    }
    
    private fun loadProducts(query: String = "", category: String? = null) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            
            productRepository.getProducts(category)
                .onSuccess { products ->
                    val filtered = if (query.isBlank()) products
                    else products.filter { it.name.contains(query, ignoreCase = true) }
                    
                    _state.update { it.copy(products = filtered, isLoading = false) }
                }
                .onFailure { error ->
                    _state.update { it.copy(
                        isLoading = false,
                        error = error.message ?: "Failed to load products"
                    ) }
                }
        }
    }
    
    private fun addToCart(productId: String) {
        viewModelScope.launch {
            addToCartUseCase(productId, 1)
                .onSuccess {
                    _effects.emit(ProductListEffect.ShowSnackbar("Added to cart!"))
                }
                .onFailure { error ->
                    _effects.emit(ProductListEffect.ShowSnackbar(error.message ?: "Failed to add"))
                }
        }
    }
}
```

---

## 79.5 Jetpack Compose UI

```kotlin
// presentation/product/ProductListScreen.kt

@Composable
fun ProductListScreen(
    onNavigateToProduct: (String) -> Unit,
    onNavigateToCart: () -> Unit,
    viewModel: ProductListViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    
    // Collect one-shot effects
    LaunchedEffect(Unit) {
        viewModel.effects.collect { effect ->
            when (effect) {
                is ProductListEffect.ShowSnackbar -> 
                    snackbarHostState.showSnackbar(effect.message)
                is ProductListEffect.NavigateToProduct ->
                    onNavigateToProduct(effect.productId)
            }
        }
    }
    
    Scaffold(
        topBar = {
            ProductListTopBar(
                cartItemCount = state.cartItemCount,
                onCartClick = onNavigateToCart
            )
        },
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
        ) {
            // Search bar
            SearchBar(
                query = state.searchQuery,
                onQueryChange = { viewModel.handleEvent(ProductListEvent.SearchQueryChanged(it)) },
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp)
            )
            
            // Category chips
            CategoryFilter(
                selectedCategory = state.selectedCategory,
                onCategorySelected = { viewModel.handleEvent(ProductListEvent.CategorySelected(it)) }
            )
            
            // Content
            when {
                state.isLoading -> LoadingIndicator()
                state.error != null -> ErrorState(
                    message = state.error!!,
                    onRetry = { viewModel.handleEvent(ProductListEvent.Refresh) }
                )
                state.products.isEmpty() -> EmptyState()
                else -> ProductGrid(
                    products = state.products,
                    onProductClick = onNavigateToProduct,
                    onAddToCart = { viewModel.handleEvent(ProductListEvent.AddToCart(it)) }
                )
            }
        }
    }
}

@Composable
fun ProductGrid(
    products: List<Product>,
    onProductClick: (String) -> Unit,
    onAddToCart: (String) -> Unit
) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 160.dp),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(products, key = { it.id }) { product ->
            ProductCard(
                product = product,
                onClick = { onProductClick(product.id) },
                onAddToCart = { onAddToCart(product.id) }
            )
        }
    }
}

@Composable
fun ProductCard(
    product: Product,
    onClick: () -> Unit,
    onAddToCart: () -> Unit
) {
    Card(
        onClick = onClick,
        modifier = Modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Column {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = product.name,
                contentScale = ContentScale.Crop,
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(1f)
            )
            
            Column(modifier = Modifier.padding(12.dp)) {
                Text(
                    text = product.name,
                    style = MaterialTheme.typography.titleSmall,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
                Spacer(modifier = Modifier.height(4.dp))
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Text(
                        text = "฿${product.price.toInt()}",
                        style = MaterialTheme.typography.titleMedium,
                        color = MaterialTheme.colorScheme.primary
                    )
                    IconButton(
                        onClick = onAddToCart,
                        enabled = product.inStock
                    ) {
                        Icon(Icons.Default.AddShoppingCart, "Add to cart")
                    }
                }
                if (!product.inStock) {
                    Text(
                        text = "Out of stock",
                        style = MaterialTheme.typography.labelSmall,
                        color = MaterialTheme.colorScheme.error
                    )
                }
            }
        }
    }
}
```

---

## 79.6 Room Database (Local Cache)

```kotlin
// data/local/AppDatabase.kt

@Entity(tableName = "products")
data class ProductEntity(
    @PrimaryKey val id: String,
    val name: String,
    val description: String,
    val price: Double,
    val imageUrl: String,
    val category: String,
    val rating: Float,
    val reviewCount: Int,
    val inStock: Boolean,
    val cachedAt: Long = System.currentTimeMillis()
)

@Entity(tableName = "cart_items")
data class CartItemEntity(
    @PrimaryKey val productId: String,
    val productName: String,
    val productPrice: Double,
    val productImageUrl: String,
    val quantity: Int
)

@Dao
interface ProductDao {
    @Query("SELECT * FROM products WHERE (:category IS NULL OR category = :category)")
    fun getProducts(category: String?): Flow<List<ProductEntity>>
    
    @Query("SELECT * FROM products WHERE id = :id")
    suspend fun getProduct(id: String): ProductEntity?
    
    @Upsert
    suspend fun upsertProducts(products: List<ProductEntity>)
    
    @Query("DELETE FROM products WHERE cachedAt < :expiry")
    suspend fun deleteExpired(expiry: Long)
}

@Dao
interface CartDao {
    @Query("SELECT * FROM cart_items")
    fun getCartItems(): Flow<List<CartItemEntity>>
    
    @Upsert
    suspend fun upsert(item: CartItemEntity)
    
    @Query("UPDATE cart_items SET quantity = :quantity WHERE productId = :productId")
    suspend fun updateQuantity(productId: String, quantity: Int)
    
    @Query("DELETE FROM cart_items WHERE productId = :productId")
    suspend fun delete(productId: String)
    
    @Query("DELETE FROM cart_items")
    suspend fun clearAll()
}

@Database(entities = [ProductEntity::class, CartItemEntity::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao
    abstract fun cartDao(): CartDao
}

// data/repository/ProductRepositoryImpl.kt
class ProductRepositoryImpl @Inject constructor(
    private val api: ProductApi,
    private val productDao: ProductDao
) : ProductRepository {
    
    override suspend fun getProducts(category: String?): Result<List<Product>> {
        return try {
            // Network first, fall back to cache
            val response = api.getProducts(category)
            productDao.upsertProducts(response.map { it.toEntity() })
            Result.success(response.map { it.toDomain() })
        } catch (e: Exception) {
            // Fall back to cache
            val cached = productDao.getProducts(category).first()
            if (cached.isEmpty()) {
                Result.failure(e)
            } else {
                Result.success(cached.map { it.toDomain() })
            }
        }
    }
    
    override fun searchProducts(query: String): Flow<List<Product>> =
        productDao.getProducts(null)
            .map { entities ->
                entities.filter { it.name.contains(query, ignoreCase = true) }
                    .map { it.toDomain() }
            }
}
```

---

## 79.7 Hilt Dependency Injection

```kotlin
// di/AppModule.kt

@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "shop.db")
            .fallbackToDestructiveMigration()
            .build()
    
    @Provides
    fun provideProductDao(db: AppDatabase): ProductDao = db.productDao()
    
    @Provides
    fun provideCartDao(db: AppDatabase): CartDao = db.cartDao()
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                    else HttpLoggingInterceptor.Level.NONE
        })
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(okHttpClient)
        .addConverterFactory(
            KotlinxSerializationConverterFactory.create(Json { ignoreUnknownKeys = true })
        )
        .build()
    
    @Provides
    @Singleton
    fun provideProductApi(retrofit: Retrofit): ProductApi = 
        retrofit.create(ProductApi::class.java)
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    @Singleton
    abstract fun bindProductRepository(impl: ProductRepositoryImpl): ProductRepository
    
    @Binds
    @Singleton
    abstract fun bindCartRepository(impl: CartRepositoryImpl): CartRepository
}
```

---

## 79.8 Navigation

```kotlin
// navigation/AppNavigation.kt

sealed class Screen(val route: String) {
    object ProductList : Screen("products")
    object ProductDetail : Screen("products/{productId}") {
        fun createRoute(productId: String) = "products/$productId"
    }
    object Cart : Screen("cart")
    object Checkout : Screen("checkout")
    object OrderHistory : Screen("orders")
}

@Composable
fun AppNavigation(navController: NavHostController = rememberNavController()) {
    NavHost(navController = navController, startDestination = Screen.ProductList.route) {
        composable(Screen.ProductList.route) {
            ProductListScreen(
                onNavigateToProduct = { navController.navigate(Screen.ProductDetail.createRoute(it)) },
                onNavigateToCart = { navController.navigate(Screen.Cart.route) }
            )
        }
        
        composable(
            route = Screen.ProductDetail.route,
            arguments = listOf(navArgument("productId") { type = NavType.StringType })
        ) { backStackEntry ->
            val productId = backStackEntry.arguments?.getString("productId")!!
            ProductDetailScreen(
                productId = productId,
                onNavigateBack = { navController.popBackStack() },
                onNavigateToCart = { navController.navigate(Screen.Cart.route) }
            )
        }
        
        composable(Screen.Cart.route) {
            CartScreen(
                onNavigateBack = { navController.popBackStack() },
                onNavigateToCheckout = { navController.navigate(Screen.Checkout.route) }
            )
        }
    }
}
```

---

## สรุป Part 79

```
Android MVI + Clean Architecture:

Layers:
  Domain:       Pure Kotlin, no Android
  Data:         Room (local), Retrofit (remote)
  Presentation: ViewModel (MVI), Compose (UI)

MVI Flow:
  UserEvent → ViewModel.handleEvent() → State update → Recompose
  Side effects (navigation, snackbar) via SharedFlow

State Management:
  StateFlow<ScreenState>   = current state (hot, single value)
  SharedFlow<Effect>       = one-shot effects (navigation, toasts)

Dependency Injection (Hilt):
  @HiltAndroidApp on Application
  @HiltViewModel on ViewModel
  @Inject constructor on classes
  @Module + @InstallIn(SingletonComponent) for providers

Local Persistence (Room):
  @Entity    = table definition
  @Dao       = query interface
  @Database  = database holder
  Flow<List> = reactive query (auto-update on change)

API (Retrofit + Kotlinx Serialization):
  interface ProductApi with @GET, @POST, @Path, @Query
  KotlinxSerializationConverterFactory
  @Serializable data classes

Cache Strategy:
  Network first → cache on success
  On failure → return cached data
  Clear expired cache periodically
```

➡️ [Part 80: Reactive Programming with Spring WebFlux](./Part-80-WebFlux.md)
