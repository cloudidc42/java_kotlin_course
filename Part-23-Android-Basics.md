# Part 23: Android Development Basics
## ขั้นตอนที่ 1501-1570: สร้าง Android App ด้วย Kotlin

---

## 23.1 Android Architecture

```
Android App Architecture:
┌─────────────────────────────────────────┐
│               UI Layer                  │
│  Activity / Fragment / Compose UI       │
├─────────────────────────────────────────┤
│            ViewModel Layer              │
│  ViewModels + LiveData/StateFlow        │
├─────────────────────────────────────────┤
│           Repository Layer              │
│  Room DB + Retrofit + DataStore         │
├─────────────────────────────────────────┤
│           Data Sources                  │
│  Local DB │ Remote API │ SharedPrefs    │
└─────────────────────────────────────────┘

MVVM Pattern:
  View ← observes → ViewModel ← uses → Repository
  View → sends events → ViewModel → updates state → View
```

---

## 23.2 Project Structure

```
MyAndroidApp/
├── app/
│   ├── src/main/
│   │   ├── java/com/example/myapp/
│   │   │   ├── MainActivity.kt
│   │   │   ├── data/
│   │   │   │   ├── local/
│   │   │   │   │   ├── AppDatabase.kt
│   │   │   │   │   └── UserDao.kt
│   │   │   │   ├── remote/
│   │   │   │   │   ├── ApiService.kt
│   │   │   │   │   └── RetrofitClient.kt
│   │   │   │   └── repository/
│   │   │   │       └── UserRepository.kt
│   │   │   ├── domain/
│   │   │   │   └── model/
│   │   │   │       └── User.kt
│   │   │   └── ui/
│   │   │       ├── home/
│   │   │       │   ├── HomeFragment.kt
│   │   │       │   └── HomeViewModel.kt
│   │   │       └── detail/
│   │   │           ├── DetailFragment.kt
│   │   │           └── DetailViewModel.kt
│   │   ├── res/
│   │   │   ├── layout/
│   │   │   │   ├── activity_main.xml
│   │   │   │   └── fragment_home.xml
│   │   │   ├── values/
│   │   │   │   ├── strings.xml
│   │   │   │   ├── colors.xml
│   │   │   │   └── themes.xml
│   │   │   └── drawable/
│   │   └── AndroidManifest.xml
│   └── build.gradle.kts
├── build.gradle.kts
└── settings.gradle.kts
```

---

## 23.3 build.gradle.kts

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.kapt)
    alias(libs.plugins.hilt.android)
}

android {
    namespace = "com.example.myapp"
    compileSdk = 34
    
    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }
    
    buildFeatures {
        viewBinding = true
        compose = true
    }
    
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.4"
    }
    
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    
    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    // Core
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.appcompat)
    implementation(libs.material)
    implementation(libs.androidx.lifecycle.viewmodel.ktx)
    implementation(libs.androidx.lifecycle.livedata.ktx)
    implementation(libs.androidx.navigation.fragment.ktx)
    
    // Compose
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.ui)
    implementation(libs.androidx.compose.material3)
    implementation(libs.androidx.activity.compose)
    
    // Room Database
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    kapt(libs.androidx.room.compiler)
    
    // Retrofit (HTTP client)
    implementation(libs.retrofit)
    implementation(libs.retrofit.gson)
    implementation(libs.okhttp.logging)
    
    // Hilt (Dependency Injection)
    implementation(libs.hilt.android)
    kapt(libs.hilt.compiler)
    
    // Coroutines
    implementation(libs.kotlinx.coroutines.android)
    
    // Image loading
    implementation(libs.coil)
    
    // Testing
    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.test.ext)
    androidTestImplementation(libs.androidx.espresso.core)
}
```

---

## 23.4 Room Database

```kotlin
// Entity
@Entity(tableName = "notes")
data class Note(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val title: String,
    val content: String,
    val createdAt: Long = System.currentTimeMillis(),
    val isPinned: Boolean = false,
    val category: String = "General"
)

// DAO
@Dao
interface NoteDao {
    @Query("SELECT * FROM notes ORDER BY isPinned DESC, createdAt DESC")
    fun getAllNotes(): Flow<List<Note>>
    
    @Query("SELECT * FROM notes WHERE id = :id")
    suspend fun getNoteById(id: Long): Note?
    
    @Query("SELECT * FROM notes WHERE category = :category ORDER BY createdAt DESC")
    fun getNotesByCategory(category: String): Flow<List<Note>>
    
    @Query("""
        SELECT * FROM notes 
        WHERE title LIKE '%' || :query || '%' 
           OR content LIKE '%' || :query || '%'
        ORDER BY createdAt DESC
    """)
    fun searchNotes(query: String): Flow<List<Note>>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertNote(note: Note): Long
    
    @Update
    suspend fun updateNote(note: Note)
    
    @Delete
    suspend fun deleteNote(note: Note)
    
    @Query("DELETE FROM notes WHERE id = :id")
    suspend fun deleteNoteById(id: Long)
    
    @Query("SELECT COUNT(*) FROM notes")
    suspend fun getNoteCount(): Int
    
    @Query("SELECT DISTINCT category FROM notes ORDER BY category")
    fun getAllCategories(): Flow<List<String>>
}

// Database
@Database(entities = [Note::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    
    abstract fun noteDao(): NoteDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "notes_database"
                )
                .addMigrations(MIGRATION_1_2)
                .fallbackToDestructiveMigration()
                .build()
                .also { INSTANCE = it }
            }
        }
        
        val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL("ALTER TABLE notes ADD COLUMN tags TEXT DEFAULT ''")
            }
        }
    }
}
```

---

## 23.5 Repository & ViewModel

```kotlin
// Repository
class NoteRepository(private val noteDao: NoteDao) {
    
    val allNotes: Flow<List<Note>> = noteDao.getAllNotes()
    
    fun getNotesByCategory(category: String) = noteDao.getNotesByCategory(category)
    
    fun searchNotes(query: String) = noteDao.searchNotes(query)
    
    fun getAllCategories() = noteDao.getAllCategories()
    
    suspend fun getNoteById(id: Long) = noteDao.getNoteById(id)
    
    suspend fun insertNote(note: Note) = noteDao.insertNote(note)
    
    suspend fun updateNote(note: Note) = noteDao.updateNote(note)
    
    suspend fun deleteNote(note: Note) = noteDao.deleteNote(note)
    
    suspend fun deleteNoteById(id: Long) = noteDao.deleteNoteById(id)
}

// ViewModel
@HiltViewModel
class NoteViewModel @Inject constructor(
    private val repository: NoteRepository
) : ViewModel() {
    
    // UI State
    data class UiState(
        val notes: List<Note> = emptyList(),
        val categories: List<String> = emptyList(),
        val searchQuery: String = "",
        val selectedCategory: String? = null,
        val isLoading: Boolean = false,
        val error: String? = null
    )
    
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadNotes()
        loadCategories()
    }
    
    private fun loadNotes() {
        viewModelScope.launch {
            repository.allNotes.collect { notes ->
                _uiState.update { it.copy(notes = notes) }
            }
        }
    }
    
    private fun loadCategories() {
        viewModelScope.launch {
            repository.getAllCategories().collect { categories ->
                _uiState.update { it.copy(categories = categories) }
            }
        }
    }
    
    fun search(query: String) {
        _uiState.update { it.copy(searchQuery = query) }
        
        viewModelScope.launch {
            if (query.isBlank()) {
                repository.allNotes.collect { notes ->
                    _uiState.update { it.copy(notes = notes) }
                }
            } else {
                repository.searchNotes(query).collect { notes ->
                    _uiState.update { it.copy(notes = notes) }
                }
            }
        }
    }
    
    fun filterByCategory(category: String?) {
        _uiState.update { it.copy(selectedCategory = category) }
        
        viewModelScope.launch {
            val flow = if (category == null) repository.allNotes
                       else repository.getNotesByCategory(category)
            flow.collect { notes ->
                _uiState.update { it.copy(notes = notes) }
            }
        }
    }
    
    fun saveNote(title: String, content: String, category: String = "General") {
        viewModelScope.launch {
            try {
                val note = Note(title = title, content = content, category = category)
                repository.insertNote(note)
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message) }
            }
        }
    }
    
    fun deleteNote(note: Note) {
        viewModelScope.launch {
            repository.deleteNote(note)
        }
    }
    
    fun togglePin(note: Note) {
        viewModelScope.launch {
            repository.updateNote(note.copy(isPinned = !note.isPinned))
        }
    }
}
```

---

## 23.6 Jetpack Compose UI

```kotlin
// NoteApp with Compose
@Composable
fun NoteApp(viewModel: NoteViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    MaterialTheme {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text("Notes") },
                    actions = {
                        IconButton(onClick = { /* filter */ }) {
                            Icon(Icons.Default.FilterList, contentDescription = "Filter")
                        }
                    }
                )
            },
            floatingActionButton = {
                FloatingActionButton(onClick = { /* add note */ }) {
                    Icon(Icons.Default.Add, contentDescription = "Add")
                }
            }
        ) { paddingValues ->
            Column(modifier = Modifier.padding(paddingValues)) {
                SearchBar(
                    query = uiState.searchQuery,
                    onQueryChange = viewModel::search
                )
                NoteList(
                    notes = uiState.notes,
                    onDelete = viewModel::deleteNote,
                    onPin = viewModel::togglePin
                )
            }
        }
    }
}

@Composable
fun SearchBar(query: String, onQueryChange: (String) -> Unit) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 8.dp),
        placeholder = { Text("ค้นหา...") },
        leadingIcon = { Icon(Icons.Default.Search, contentDescription = null) },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = { onQueryChange("") }) {
                    Icon(Icons.Default.Clear, contentDescription = "Clear")
                }
            }
        },
        singleLine = true
    )
}

@Composable
fun NoteList(
    notes: List<Note>,
    onDelete: (Note) -> Unit,
    onPin: (Note) -> Unit
) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(notes, key = { it.id }) { note ->
            NoteCard(note = note, onDelete = { onDelete(note) }, onPin = { onPin(note) })
        }
    }
}

@Composable
fun NoteCard(note: Note, onDelete: () -> Unit, onPin: () -> Unit) {
    var expanded by remember { mutableStateOf(false) }
    
    Card(
        modifier = Modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(4.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    if (note.isPinned) {
                        Icon(
                            Icons.Default.PushPin,
                            contentDescription = "Pinned",
                            tint = MaterialTheme.colorScheme.primary,
                            modifier = Modifier.size(16.dp)
                        )
                        Spacer(modifier = Modifier.width(4.dp))
                    }
                    Text(
                        text = note.title,
                        style = MaterialTheme.typography.titleMedium,
                        fontWeight = FontWeight.Bold
                    )
                }
                Row {
                    IconButton(onClick = onPin) {
                        Icon(
                            if (note.isPinned) Icons.Default.PushPin else Icons.Default.PinOff,
                            contentDescription = "Pin"
                        )
                    }
                    IconButton(onClick = onDelete) {
                        Icon(Icons.Default.Delete, contentDescription = "Delete",
                             tint = MaterialTheme.colorScheme.error)
                    }
                }
            }
            
            Spacer(modifier = Modifier.height(4.dp))
            
            Text(
                text = note.content,
                style = MaterialTheme.typography.bodyMedium,
                maxLines = if (expanded) Int.MAX_VALUE else 2,
                overflow = TextOverflow.Ellipsis
            )
            
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Chip(label = note.category)
                TextButton(onClick = { expanded = !expanded }) {
                    Text(if (expanded) "ย่อ" else "อ่านเพิ่ม")
                }
            }
        }
    }
}

@Composable
fun Chip(label: String) {
    Surface(
        shape = MaterialTheme.shapes.small,
        color = MaterialTheme.colorScheme.secondaryContainer,
        modifier = Modifier.padding(top = 4.dp)
    ) {
        Text(
            text = label,
            modifier = Modifier.padding(horizontal = 8.dp, vertical = 4.dp),
            style = MaterialTheme.typography.labelSmall
        )
    }
}
```

---

## 23.7 Retrofit (HTTP Client)

```kotlin
// API Service
interface ApiService {
    @GET("users")
    suspend fun getUsers(): Response<List<UserDto>>
    
    @GET("users/{id}")
    suspend fun getUserById(@Path("id") id: Int): Response<UserDto>
    
    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): Response<UserDto>
    
    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: Int, @Body user: UpdateUserRequest): Response<UserDto>
    
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: Int): Response<Unit>
    
    @GET("posts")
    suspend fun getPosts(@Query("userId") userId: Int? = null,
                         @Query("_limit") limit: Int = 20): Response<List<PostDto>>
}

// DTOs
data class UserDto(val id: Int, val name: String, val email: String, val phone: String)
data class PostDto(val id: Int, val userId: Int, val title: String, val body: String)
data class CreateUserRequest(val name: String, val email: String)
data class UpdateUserRequest(val name: String?, val email: String?)

// Retrofit setup
object RetrofitClient {
    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"
    
    val instance: ApiService by lazy {
        val logging = HttpLoggingInterceptor().apply {
            level = HttpLoggingInterceptor.Level.BODY
        }
        
        val client = OkHttpClient.Builder()
            .addInterceptor(logging)
            .addInterceptor { chain ->
                val request = chain.request().newBuilder()
                    .addHeader("Accept", "application/json")
                    .addHeader("Content-Type", "application/json")
                    .build()
                chain.proceed(request)
            }
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
        
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ApiService::class.java)
    }
}

// Safe API call wrapper
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val code: Int, val message: String) : ApiResult<Nothing>()
    data class Exception(val throwable: Throwable) : ApiResult<Nothing>()
}

suspend fun <T> safeApiCall(call: suspend () -> Response<T>): ApiResult<T> {
    return try {
        val response = call()
        if (response.isSuccessful) {
            val body = response.body()
            if (body != null) ApiResult.Success(body)
            else ApiResult.Error(response.code(), "Empty response body")
        } else {
            ApiResult.Error(response.code(), response.message())
        }
    } catch (e: java.io.IOException) {
        ApiResult.Exception(e)
    } catch (e: java.lang.Exception) {
        ApiResult.Exception(e)
    }
}
```

---

## สรุป Part 23

| Component | Library/Framework |
|-----------|------------------|
| UI | Jetpack Compose |
| Navigation | Navigation Component |
| Database | Room (SQLite) |
| HTTP | Retrofit + OkHttp |
| DI | Hilt |
| Async | Coroutines + Flow |
| State | ViewModel + StateFlow |
| Image | Coil |

➡️ [Part 24: APK Decompile & Smali](./Part-24-APK-Decompile-Smali.md)
