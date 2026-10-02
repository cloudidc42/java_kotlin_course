# Part 58: Android Jetpack Compose
## ขั้นตอนที่ 3961-4030: Modern Android UI Development

---

## 58.1 Compose Fundamentals

```kotlin
import androidx.compose.runtime.*
import androidx.compose.material3.*
import androidx.compose.foundation.layout.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

// Compose is declarative: describe what UI should look like, not how to update it
// Recomposition: when state changes, only affected composables re-run

@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}

// State management
@Composable
fun Counter() {
    // remember: survives recomposition
    var count by remember { mutableIntStateOf(0) }
    
    Column(
        horizontalAlignment = androidx.compose.ui.Alignment.CenterHorizontally,
        modifier = Modifier.padding(16.dp)
    ) {
        Text(
            text = "Count: $count",
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(Modifier.height(8.dp))
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) { Text("-") }
            Button(onClick = { count++ }) { Text("+") }
        }
    }
}

// State hoisting: lift state up for reuse/testing
@Composable
fun CounterScreen() {
    var count by remember { mutableIntStateOf(0) }
    CounterContent(
        count = count,
        onIncrement = { count++ },
        onDecrement = { count-- }
    )
}

@Composable
fun CounterContent(
    count: Int,
    onIncrement: () -> Unit,
    onDecrement: () -> Unit
) {
    Column {
        Text("Count: $count")
        Row {
            Button(onClick = onDecrement) { Text("-") }
            Button(onClick = onIncrement) { Text("+") }
        }
    }
}
```

---

## 58.2 Layouts & Modifiers

```kotlin
import androidx.compose.foundation.*
import androidx.compose.foundation.layout.*
import androidx.compose.ui.*
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.*

@Composable
fun LayoutExamples() {
    // Column: vertical layout
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Item 1")
        Text("Item 2")
    }
    
    // Row: horizontal layout
    Row(
        modifier = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text("Left")
        Text("Right")
    }
    
    // Box: overlay layout
    Box(
        modifier = Modifier
            .size(100.dp)
            .background(Color.Blue)
    ) {
        Text("Center", modifier = Modifier.align(Alignment.Center), color = Color.White)
        Text("TopEnd", modifier = Modifier.align(Alignment.TopEnd), color = Color.White)
    }
}

// Custom product card
@Composable
fun ProductCard(
    product: Product,
    onAddToCart: (Product) -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(4.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Column(modifier = Modifier.weight(1f)) {
                    Text(
                        text = product.name,
                        style = MaterialTheme.typography.titleMedium
                    )
                    Text(
                        text = product.category,
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.outline
                    )
                }
                Text(
                    text = "฿${String.format("%.0f", product.price)}",
                    style = MaterialTheme.typography.titleLarge,
                    color = MaterialTheme.colorScheme.primary
                )
            }
            
            Spacer(Modifier.height(12.dp))
            
            Button(
                onClick = { onAddToCart(product) },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Add to Cart")
            }
        }
    }
}
```

---

## 58.3 LazyList & State

```kotlin
import androidx.compose.foundation.lazy.*

// LazyColumn = RecyclerView equivalent (only renders visible items)
@Composable
fun ProductListScreen(
    products: List<Product>,
    onProductClick: (Product) -> Unit,
    onAddToCart: (Product) -> Unit
) {
    val listState = rememberLazyListState()
    val coroutineScope = rememberCoroutineScope()
    
    Box {
        LazyColumn(
            state = listState,
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            item {
                Text(
                    "Products (${products.size})",
                    style = MaterialTheme.typography.headlineSmall
                )
                Spacer(Modifier.height(8.dp))
            }
            
            items(
                items = products,
                key = { it.id }  // stable key = better performance
            ) { product ->
                ProductCard(
                    product = product,
                    onAddToCart = onAddToCart,
                    modifier = Modifier.animateItemPlacement()  // animate list changes
                )
            }
            
            item {
                Spacer(Modifier.height(80.dp))  // space for FAB
            }
        }
        
        // Scroll to top FAB (show when not at top)
        val showScrollToTop by remember {
            derivedStateOf { listState.firstVisibleItemIndex > 0 }
        }
        
        if (showScrollToTop) {
            FloatingActionButton(
                onClick = {
                    coroutineScope.launch { listState.animateScrollToItem(0) }
                },
                modifier = Modifier
                    .align(Alignment.BottomEnd)
                    .padding(16.dp)
            ) {
                Text("↑")
            }
        }
    }
}

// LazyGrid
@Composable
fun ProductGrid(products: List<Product>) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 150.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(products) { product ->
            ProductCard(product = product, onAddToCart = {})
        }
    }
}
```

---

## 58.4 ViewModel + Compose

```kotlin
import androidx.lifecycle.*
import kotlinx.coroutines.flow.*

// ViewModel
@dagger.hilt.android.lifecycle.HiltViewModel
class ProductsViewModel @javax.inject.Inject constructor(
    private val productRepository: ProductRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<ProductsUiState>(ProductsUiState.Loading)
    val uiState: StateFlow<ProductsUiState> = _uiState.asStateFlow()
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    init {
        viewModelScope.launch {
            _searchQuery
                .debounce(300)
                .distinctUntilChanged()
                .collectLatest { query -> loadProducts(query) }
        }
    }
    
    private fun loadProducts(query: String) {
        viewModelScope.launch {
            _uiState.value = ProductsUiState.Loading
            try {
                val products = if (query.isBlank()) productRepository.getProducts()
                               else productRepository.searchProducts(query)
                _uiState.value = ProductsUiState.Success(products)
            } catch (e: Exception) {
                _uiState.value = ProductsUiState.Error(e.message ?: "Error loading products")
            }
        }
    }
    
    fun onSearchQueryChanged(query: String) { _searchQuery.value = query }
    fun retry() { loadProducts(_searchQuery.value) }
    
    fun addToCart(product: Product) {
        viewModelScope.launch {
            // Add to cart logic
        }
    }
}

sealed interface ProductsUiState {
    object Loading : ProductsUiState
    data class Success(val products: List<Product>) : ProductsUiState
    data class Error(val message: String) : ProductsUiState
}

// Screen Composable using ViewModel
@Composable
fun ProductsRoute(
    viewModel: ProductsViewModel = androidx.lifecycle.viewmodel.compose.viewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val searchQuery by viewModel.searchQuery.collectAsStateWithLifecycle()
    
    ProductsScreen(
        uiState = uiState,
        searchQuery = searchQuery,
        onSearchQueryChanged = viewModel::onSearchQueryChanged,
        onAddToCart = viewModel::addToCart,
        onRetry = viewModel::retry
    )
}

@Composable
fun ProductsScreen(
    uiState: ProductsUiState,
    searchQuery: String,
    onSearchQueryChanged: (String) -> Unit,
    onAddToCart: (Product) -> Unit,
    onRetry: () -> Unit
) {
    Column(modifier = Modifier.fillMaxSize()) {
        SearchBar(query = searchQuery, onQueryChanged = onSearchQueryChanged)
        
        when (uiState) {
            is ProductsUiState.Loading -> LoadingContent()
            is ProductsUiState.Success -> ProductListScreen(
                products = uiState.products,
                onProductClick = {},
                onAddToCart = onAddToCart
            )
            is ProductsUiState.Error -> ErrorContent(
                message = uiState.message,
                onRetry = onRetry
            )
        }
    }
}
```

---

## 58.5 Navigation Compose

```kotlin
import androidx.navigation.*
import androidx.navigation.compose.*

// Route definitions
sealed class Screen(val route: String) {
    object ProductList : Screen("products")
    object ProductDetail : Screen("products/{productId}") {
        fun createRoute(productId: String) = "products/$productId"
    }
    object Cart : Screen("cart")
    object Checkout : Screen("checkout")
}

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(navController = navController, startDestination = Screen.ProductList.route) {
        
        composable(Screen.ProductList.route) {
            ProductsRoute(
                onProductClick = { product ->
                    navController.navigate(Screen.ProductDetail.createRoute(product.id))
                },
                onCartClick = {
                    navController.navigate(Screen.Cart.route)
                }
            )
        }
        
        composable(
            route = Screen.ProductDetail.route,
            arguments = listOf(navArgument("productId") { type = NavType.StringType })
        ) { backStackEntry ->
            val productId = backStackEntry.arguments?.getString("productId")!!
            ProductDetailRoute(
                productId = productId,
                onBack = { navController.popBackStack() },
                onAddToCart = {
                    navController.navigate(Screen.Cart.route) {
                        launchSingleTop = true
                    }
                }
            )
        }
        
        composable(Screen.Cart.route) {
            CartRoute(
                onBack = { navController.popBackStack() },
                onCheckout = { navController.navigate(Screen.Checkout.route) }
            )
        }
        
        composable(Screen.Checkout.route) {
            CheckoutRoute(
                onOrderPlaced = {
                    navController.navigate(Screen.ProductList.route) {
                        popUpTo(Screen.ProductList.route) { inclusive = true }
                    }
                }
            )
        }
    }
}
```

---

## สรุป Part 58

```
Jetpack Compose Key Points:

State:
  remember { mutableStateOf(...) }  = local state
  hoist state up for reusability    = stateless composables
  ViewModel + StateFlow              = screen-level state

Layout:
  Column = vertical
  Row    = horizontal
  Box    = overlay
  LazyColumn = RecyclerView (virtualized, good for long lists)
  LazyVerticalGrid = grid layout

Performance:
  key = { item.id } in LazyColumn   = stable identity
  derivedStateOf { ... }            = compute only when deps change
  remember(deps) { ... }            = recalculate when deps change

ViewModel Pattern:
  ViewModel holds StateFlow<UiState>
  Sealed class UiState = Loading | Success | Error
  Composable observes with collectAsStateWithLifecycle()

Navigation:
  NavHost + NavController
  Type-safe routes with sealed class Screen
  Pass data as path params (/products/{id})
```

➡️ [Part 59: Spring Batch Processing](./Part-59-SpringBatch.md)
