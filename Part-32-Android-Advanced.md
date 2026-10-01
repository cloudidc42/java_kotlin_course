# Part 32: Advanced Android Development
## ขั้นตอนที่ 2141-2210: Android ระดับ Professional

---

## 32.1 Advanced Jetpack Compose

```kotlin
import androidx.compose.animation.*
import androidx.compose.foundation.*
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.*
import androidx.compose.ui.graphics.*
import androidx.compose.ui.unit.*

// ====== Custom Composable with reusable state ======
class PaginationState(
    val items: List<Any>,
    val isLoading: Boolean,
    val hasMore: Boolean,
    val onLoadMore: () -> Unit
)

@Composable
fun rememberPaginationState(
    viewModel: ListViewModel
): PaginationState {
    val uiState by viewModel.uiState.collectAsState()
    return PaginationState(
        items = uiState.items,
        isLoading = uiState.isLoading,
        hasMore = uiState.hasMore,
        onLoadMore = viewModel::loadMore
    )
}

// ====== LazyColumn with endless scroll ======
@Composable
fun InfiniteList(
    items: List<Product>,
    isLoading: Boolean,
    hasMore: Boolean,
    onLoadMore: () -> Unit,
    onItemClick: (Product) -> Unit
) {
    val listState = rememberLazyListState()
    
    // Detect when near end of list
    LaunchedEffect(listState) {
        snapshotFlow {
            val layoutInfo = listState.layoutInfo
            val totalItems = layoutInfo.totalItemsCount
            val lastVisible = layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            lastVisible >= totalItems - 5  // load more when 5 items from end
        }.collect { nearEnd ->
            if (nearEnd && hasMore && !isLoading) {
                onLoadMore()
            }
        }
    }
    
    LazyColumn(
        state = listState,
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(items, key = { it.id }) { product ->
            ProductCard(product, onClick = { onItemClick(product) })
        }
        
        if (isLoading) {
            item {
                Box(modifier = Modifier.fillMaxWidth().padding(16.dp),
                    contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
        }
        
        if (!hasMore && items.isNotEmpty()) {
            item {
                Text(
                    "No more items",
                    modifier = Modifier.fillMaxWidth().padding(16.dp),
                    style = MaterialTheme.typography.bodySmall,
                    color = Color.Gray,
                    textAlign = androidx.compose.ui.text.style.TextAlign.Center
                )
            }
        }
    }
}

// ====== Animated transitions ======
@Composable
fun AnimatedCounter(count: Int) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            if (targetState > initialState) {
                (slideInVertically { -it } + fadeIn()) togetherWith
                (slideOutVertically { it } + fadeOut())
            } else {
                (slideInVertically { it } + fadeIn()) togetherWith
                (slideOutVertically { -it } + fadeOut())
            }
        }
    ) { value ->
        Text(
            text = value.toString(),
            style = MaterialTheme.typography.headlineLarge
        )
    }
}

// ====== Custom Modifier ======
fun Modifier.shimmer(): Modifier = this.then(
    Modifier.drawWithContent {
        drawContent()
        // simplified shimmer - production would use actual animated gradient
    }
)

@Composable
fun ShimmerPlaceholder(modifier: Modifier = Modifier) {
    val shimmerColors = listOf(
        Color.LightGray.copy(alpha = 0.9f),
        Color.LightGray.copy(alpha = 0.2f),
        Color.LightGray.copy(alpha = 0.9f),
    )
    
    val transition = rememberInfiniteTransition(label = "shimmer")
    val offset by transition.animateFloat(
        initialValue = 0f, targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = androidx.compose.animation.core.tween(1000),
            repeatMode = androidx.compose.animation.core.RepeatMode.Restart
        ),
        label = "shimmer offset"
    )
    
    Box(
        modifier = modifier
            .background(Color.LightGray.copy(alpha = 0.3f))
            .then(Modifier.shimmer())
    )
}
```

---

## 32.2 Navigation with Compose

```kotlin
import androidx.navigation.compose.*
import androidx.navigation.*

// Define routes as sealed class
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Profile : Screen("profile/{userId}") {
        fun createRoute(userId: Long) = "profile/$userId"
    }
    object Settings : Screen("settings")
    object ProductDetail : Screen("product/{productId}?category={category}") {
        fun createRoute(productId: Long, category: String? = null) =
            "product/$productId" + (category?.let { "?category=$it" } ?: "")
    }
}

@Composable
fun AppNavigation(startDestination: String = Screen.Home.route) {
    val navController = rememberNavController()
    
    NavHost(navController = navController, startDestination = startDestination) {
        
        composable(Screen.Home.route) {
            HomeScreen(
                onNavigateToProfile = { userId ->
                    navController.navigate(Screen.Profile.createRoute(userId))
                },
                onNavigateToSettings = {
                    navController.navigate(Screen.Settings.route)
                }
            )
        }
        
        composable(
            route = Screen.Profile.route,
            arguments = listOf(navArgument("userId") { type = NavType.LongType })
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getLong("userId") ?: return@composable
            ProfileScreen(userId = userId, onBack = { navController.popBackStack() })
        }
        
        composable(Screen.Settings.route) {
            SettingsScreen(onBack = { navController.popBackStack() })
        }
        
        composable(
            route = Screen.ProductDetail.route,
            arguments = listOf(
                navArgument("productId") { type = NavType.LongType },
                navArgument("category") {
                    type = NavType.StringType
                    nullable = true
                    defaultValue = null
                }
            )
        ) { backStackEntry ->
            val productId = backStackEntry.arguments?.getLong("productId") ?: return@composable
            val category = backStackEntry.arguments?.getString("category")
            ProductDetailScreen(productId, category, onBack = { navController.popBackStack() })
        }
    }
}

// Bottom Navigation
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val tabs = listOf(
        Triple("Home", Screen.Home.route, Icons.Default.Home),
        Triple("Profile", Screen.Profile.createRoute(0), Icons.Default.Person),
        Triple("Settings", Screen.Settings.route, Icons.Default.Settings)
    )
    
    Scaffold(
        bottomBar = {
            NavigationBar {
                val navBackStack by navController.currentBackStackEntryAsState()
                val current = navBackStack?.destination?.route
                
                tabs.forEach { (label, route, icon) ->
                    NavigationBarItem(
                        selected = current == route,
                        onClick = {
                            navController.navigate(route) {
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        label = { Text(label) },
                        icon = { Icon(icon, contentDescription = label) }
                    )
                }
            }
        }
    ) { padding ->
        AppNavigation()
    }
}
```

---

## 32.3 Advanced Room Database

```kotlin
import androidx.room.*
import kotlinx.coroutines.flow.*

// ====== Entities ======
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val email: String,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)

@Entity(
    tableName = "orders",
    foreignKeys = [ForeignKey(
        entity = UserEntity::class,
        parentColumns = ["id"],
        childColumns = ["user_id"],
        onDelete = ForeignKey.CASCADE
    )],
    indices = [Index("user_id")]
)
data class OrderEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "user_id") val userId: Long,
    val total: Double,
    val status: String,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)

// ====== Relation ======
data class UserWithOrders(
    @Embedded val user: UserEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "user_id"
    )
    val orders: List<OrderEntity>
)

// ====== DAOs ======
@Dao
interface UserDao {
    
    @Query("SELECT * FROM users ORDER BY created_at DESC")
    fun getAllUsers(): Flow<List<UserEntity>>
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserById(id: Long): UserEntity?
    
    @Query("SELECT * FROM users WHERE email = :email")
    suspend fun getUserByEmail(email: String): UserEntity?
    
    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithOrders(userId: Long): UserWithOrders?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: UserEntity): Long
    
    @Update
    suspend fun updateUser(user: UserEntity)
    
    @Delete
    suspend fun deleteUser(user: UserEntity)
    
    @Query("DELETE FROM users WHERE id = :id")
    suspend fun deleteUserById(id: Long)
    
    @Query("SELECT COUNT(*) FROM users")
    suspend fun getUserCount(): Int
    
    @Query("SELECT * FROM users WHERE name LIKE '%' || :query || '%' OR email LIKE '%' || :query || '%'")
    fun searchUsers(query: String): Flow<List<UserEntity>>
}

// ====== Database with Migrations ======
@Database(
    entities = [UserEntity::class, OrderEntity::class],
    version = 3,
    exportSchema = true
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun orderDao(): OrderDao
    
    companion object {
        val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE users ADD COLUMN profile_image TEXT")
            }
        }
        
        val MIGRATION_2_3 = object : Migration(2, 3) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("CREATE INDEX index_orders_user_id ON orders(user_id)")
            }
        }
        
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getInstance(context: android.content.Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
                    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                    .addCallback(object : Callback() {
                        override fun onCreate(db: SupportSQLiteDatabase) {
                            // Seed initial data
                        }
                    })
                    .build()
                    .also { INSTANCE = it }
            }
        }
    }
}

class Converters {
    @TypeConverter
    fun fromList(list: List<String>): String = list.joinToString(",")
    
    @TypeConverter
    fun toList(value: String): List<String> = 
        if (value.isEmpty()) emptyList() else value.split(",")
}
```

---

## 32.4 Hilt Dependency Injection

```kotlin
import dagger.hilt.android.lifecycle.*
import dagger.hilt.android.*
import dagger.*
import javax.inject.*

// ====== Modules ======
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: android.content.Context): AppDatabase {
        return AppDatabase.getInstance(context)
    }
    
    @Provides
    @Singleton
    fun provideUserDao(db: AppDatabase): UserDao = db.userDao()
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): okhttp3.OkHttpClient {
        return okhttp3.OkHttpClient.Builder()
            .addInterceptor(okhttp3.logging.HttpLoggingInterceptor().apply {
                level = okhttp3.logging.HttpLoggingInterceptor.Level.BODY
            })
            .connectTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
            .readTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(client: okhttp3.OkHttpClient): retrofit2.Retrofit {
        return retrofit2.Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(
                retrofit2.converter.gson.GsonConverterFactory.create()
            )
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: retrofit2.Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}

// ====== Repository with injection ======
@Singleton
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val apiService: ApiService
) {
    
    fun getUsers(): Flow<List<UserEntity>> = userDao.getAllUsers()
    
    suspend fun syncUsers() {
        val remoteUsers = apiService.getUsers()
        remoteUsers.forEach { userDao.insertUser(it.toEntity()) }
    }
    
    suspend fun getUserById(id: Long): UserEntity? = userDao.getUserById(id)
}

// ====== ViewModel with injection ======
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : androidx.lifecycle.ViewModel() {
    
    sealed class UiState {
        object Loading : UiState()
        data class Success(val users: List<UserEntity>) : UiState()
        data class Error(val message: String) : UiState()
    }
    
    private val _uiState = kotlinx.coroutines.flow.MutableStateFlow<UiState>(UiState.Loading)
    val uiState: kotlinx.coroutines.flow.StateFlow<UiState> = _uiState
    
    init {
        loadUsers()
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            try {
                userRepository.getUsers().collect { users ->
                    _uiState.value = UiState.Success(users)
                }
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun syncUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                userRepository.syncUsers()
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Sync failed")
            }
        }
    }
}

// ====== Activity/Fragment with Hilt ======
@HiltAndroidApp
class MyApplication : android.app.Application()

@dagger.hilt.android.AndroidEntryPoint
class MainActivity : androidx.activity.ComponentActivity() {
    
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: android.os.Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val uiState by viewModel.uiState.collectAsState()
            
            MaterialTheme {
                when (val state = uiState) {
                    is UserViewModel.UiState.Loading -> CircularProgressIndicator()
                    is UserViewModel.UiState.Success -> UserList(state.users)
                    is UserViewModel.UiState.Error -> ErrorMessage(state.message)
                }
            }
        }
    }
}
```

---

## 32.5 WorkManager (Background Jobs)

```kotlin
import androidx.work.*
import kotlinx.coroutines.*

// ====== Worker ======
class SyncWorker(
    context: android.content.Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val userId = inputData.getLong("user_id", -1)
            
            // Set progress
            setProgress(workDataOf("status" to "Syncing..."))
            
            // Do actual work
            syncData(userId)
            
            // Return output
            val output = workDataOf("synced_count" to 42)
            Result.success(output)
            
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()  // retry up to 3 times
            } else {
                Result.failure(workDataOf("error" to e.message))
            }
        }
    }
    
    private suspend fun syncData(userId: Long) {
        delay(2000)  // simulate network call
        setForeground(createForegroundInfo())
    }
    
    private fun createForegroundInfo(): ForegroundInfo {
        val notification = androidx.core.app.NotificationCompat.Builder(applicationContext, "sync")
            .setSmallIcon(android.R.drawable.ic_menu_upload)
            .setContentTitle("Syncing data")
            .setProgress(100, 50, false)
            .build()
        return ForegroundInfo(1001, notification)
    }
}

// ====== Schedule work ======
class WorkScheduler(private val context: android.content.Context) {
    
    private val workManager = WorkManager.getInstance(context)
    
    fun schedulePeriodicSync() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
        
        val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
            repeatInterval = 15,
            repeatIntervalTimeUnit = java.util.concurrent.TimeUnit.MINUTES
        )
            .setConstraints(constraints)
            .setBackoffCriteria(
                BackoffPolicy.EXPONENTIAL,
                10, java.util.concurrent.TimeUnit.SECONDS
            )
            .setInputData(workDataOf("type" to "periodic"))
            .build()
        
        workManager.enqueueUniquePeriodicWork(
            "periodic_sync",
            ExistingPeriodicWorkPolicy.KEEP,
            syncRequest
        )
    }
    
    fun scheduleImmediateSync(userId: Long) {
        val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
            .setInputData(workDataOf("user_id" to userId))
            .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
            .build()
        
        workManager.enqueue(syncRequest)
        
        // Observe result
        workManager.getWorkInfoByIdLiveData(syncRequest.id).observeForever { info ->
            when (info?.state) {
                WorkInfo.State.SUCCEEDED -> println("Sync succeeded!")
                WorkInfo.State.FAILED -> println("Sync failed: ${info.outputData.getString("error")}")
                WorkInfo.State.RUNNING -> {
                    val progress = info.progress.getString("status")
                    println("Progress: $progress")
                }
                else -> {}
            }
        }
    }
    
    // Chain workers
    fun scheduleChainedWork() {
        val downloadWork = OneTimeWorkRequestBuilder<DownloadWorker>().build()
        val processWork = OneTimeWorkRequestBuilder<ProcessWorker>().build()
        val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()
        
        workManager
            .beginWith(downloadWork)
            .then(processWork)
            .then(uploadWork)
            .enqueue()
    }
}
```

---

## สรุป Part 32

| Feature | Android API |
|---------|-----------|
| Lists | `LazyColumn`, `LazyRow` |
| Navigation | `NavHost`, `NavController` |
| Animation | `AnimatedContent`, `animateFloatAsState` |
| Background | `WorkManager` (PeriodicWork, OneTimeWork) |
| DI | Hilt (`@HiltViewModel`, `@Inject`, `@Module`) |
| Database | Room (entities, relations, migrations) |
| State | `StateFlow`, `collectAsState()` |

**Production Checklist:**
- ✅ ProGuard/R8 obfuscation
- ✅ Baseline Profiles (startup optimization)
- ✅ App Bundle (`.aab`) ไม่ใช่ APK
- ✅ Network Security Config (certificate pinning)
- ✅ SafetyNet/Play Integrity API
- ✅ Crash reporting (Firebase Crashlytics)

➡️ [Part 33: Performance Optimization](./Part-33-Performance.md)
