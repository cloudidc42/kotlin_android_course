# Part 20: Kotlin Multiplatform (KMP)
## ขั้นตอนที่ 476-500

---

## ขั้นตอนที่ 476: KMP คืออะไร?

Kotlin Multiplatform ช่วยให้ share code ระหว่าง platforms ต่างๆ

```
KMP Architecture:

┌─────────────────────────────────────────────┐
│            Shared (commonMain)               │
│  Business Logic, Models, Repository,         │
│  Use Cases, Network (Ktor), DB (SQLDelight) │
└────────────┬──────────────────┬─────────────┘
             │                  │
   ┌──────────▼──────┐  ┌───────▼────────┐
   │  Android (main) │  │   iOS (main)   │
   │  Compose UI     │  │   SwiftUI      │
   │  Android-specific│  │  iOS-specific  │
   └─────────────────┘  └────────────────┘

Share: ~70-80% of code
Platform-specific: ~20-30% (UI, permissions, etc.)
```

---

## ขั้นตอนที่ 477: KMP Project Setup

```kotlin
// build.gradle.kts (shared module)
plugins {
    kotlin("multiplatform")
    kotlin("native.cocoapods")
    id("com.android.library")
    kotlin("plugin.serialization")
    id("app.cash.sqldelight")
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions { jvmTarget = "17" }
        }
    }
    
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    cocoapods {
        summary = "Shared code"
        homepage = "https://github.com/example/myapp"
        version = "1.0"
        ios.deploymentTarget = "16.0"
        framework {
            baseName = "shared"
        }
    }
    
    sourceSets {
        commonMain.dependencies {
            implementation(libs.kotlinx.coroutines.core)
            implementation(libs.ktor.client.core)
            implementation(libs.ktor.client.content.negotiation)
            implementation(libs.ktor.serialization.kotlinx.json)
            implementation(libs.kotlinx.serialization.json)
            implementation(libs.sqldelight.runtime)
            implementation(libs.koin.core)
        }
        
        androidMain.dependencies {
            implementation(libs.ktor.client.android)
            implementation(libs.sqldelight.android.driver)
            implementation(libs.koin.android)
        }
        
        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
            implementation(libs.sqldelight.native.driver)
        }
        
        commonTest.dependencies {
            implementation(libs.kotlin.test)
            implementation(libs.kotlinx.coroutines.test)
        }
    }
}
```

---

## ขั้นตอนที่ 478: expect/actual

```kotlin
// ============================================
// expect/actual - platform-specific code
// ============================================

// commonMain - ประกาศ expect
expect class Platform() {
    val name: String
    val version: String
}

expect fun getPlatformName(): String
expect fun getCurrentTimeMillis(): Long
expect fun formatDate(timestamp: Long, format: String): String

// androidMain - implement actual
actual class Platform actual constructor() {
    actual val name: String = "Android"
    actual val version: String = android.os.Build.VERSION.RELEASE
}

actual fun getPlatformName(): String = "Android ${android.os.Build.VERSION.SDK_INT}"

actual fun getCurrentTimeMillis(): Long = System.currentTimeMillis()

actual fun formatDate(timestamp: Long, format: String): String {
    return java.text.SimpleDateFormat(format, java.util.Locale.getDefault())
        .format(java.util.Date(timestamp))
}

// iosMain - implement actual
actual class Platform actual constructor() {
    actual val name: String = UIDevice.currentDevice.systemName()
    actual val version: String = UIDevice.currentDevice.systemVersion
}

actual fun getPlatformName(): String = "iOS"

actual fun getCurrentTimeMillis(): Long =
    (NSDate.date().timeIntervalSince1970 * 1000).toLong()

actual fun formatDate(timestamp: Long, format: String): String {
    val date = NSDate.dateWithTimeIntervalSince1970(timestamp / 1000.0)
    val formatter = NSDateFormatter()
    formatter.dateFormat = format
    return formatter.stringFromDate(date)
}
```

---

## ขั้นตอนที่ 479: Shared Data Layer กับ Ktor

```kotlin
// ============================================
// Shared Network Layer (commonMain)
// ============================================

// Kotlinx Serialization แทน Gson
@Serializable
data class Post(
    val id: Long,
    @SerialName("user_id") val userId: Long,
    val title: String,
    val body: String
)

@Serializable
data class User(
    val id: Long,
    val name: String,
    val email: String
)

// HTTP Client ที่ทำงานบนทุก platform
class ApiClient {
    private val client = HttpClient {
        install(ContentNegotiation) {
            json(Json {
                prettyPrint = true
                isLenient = true
                ignoreUnknownKeys = true
            })
        }
        install(HttpTimeout) {
            requestTimeoutMillis = 30000
            connectTimeoutMillis = 15000
        }
        install(Logging) {
            level = LogLevel.BODY
        }
    }
    
    private val baseUrl = "https://jsonplaceholder.typicode.com"
    
    suspend fun getPosts(): List<Post> = client.get("$baseUrl/posts").body()
    
    suspend fun getPost(id: Long): Post = client.get("$baseUrl/posts/$id").body()
    
    suspend fun createPost(title: String, body: String, userId: Long): Post {
        return client.post("$baseUrl/posts") {
            contentType(ContentType.Application.Json)
            setBody(mapOf(
                "title" to title,
                "body" to body,
                "userId" to userId
            ))
        }.body()
    }
    
    suspend fun getUsers(): List<User> = client.get("$baseUrl/users").body()
    
    fun close() = client.close()
}
```

---

## ขั้นตอนที่ 480: Shared Database กับ SQLDelight

```kotlin
// src/commonMain/sqldelight/com/example/db/Post.sq

// CREATE TABLE Post (
//   id INTEGER NOT NULL PRIMARY KEY,
//   userId INTEGER NOT NULL,
//   title TEXT NOT NULL,
//   body TEXT NOT NULL
// );
//
// selectAll:
// SELECT * FROM Post ORDER BY id ASC;
//
// selectById:
// SELECT * FROM Post WHERE id = ?;
//
// insertOrReplace:
// INSERT OR REPLACE INTO Post(id, userId, title, body)
// VALUES (?, ?, ?, ?);
//
// deleteById:
// DELETE FROM Post WHERE id = ?;
//
// deleteAll:
// DELETE FROM Post;
//
// count:
// SELECT COUNT(*) FROM Post;

// Database Driver Factory - expect/actual
expect class DatabaseDriverFactory(context: Any?) {
    fun createDriver(): SqlDriver
}

// androidMain
actual class DatabaseDriverFactory actual constructor(private val context: Context) {
    actual fun createDriver(): SqlDriver {
        return AndroidSqliteDriver(AppDatabase.Schema, context, "app.db")
    }
}

// iosMain
actual class DatabaseDriverFactory actual constructor(context: Any?) {
    actual fun createDriver(): SqlDriver {
        return NativeSqliteDriver(AppDatabase.Schema, "app.db")
    }
}

// Shared Repository
class PostRepository(driverFactory: DatabaseDriverFactory) {
    
    private val database = AppDatabase(driverFactory.createDriver())
    private val postQueries = database.postQueries
    private val apiClient = ApiClient()
    
    fun observePosts(): Flow<List<PostEntity>> = postQueries
        .selectAll()
        .asFlow()
        .mapToList(Dispatchers.Default)
    
    suspend fun refreshPosts() {
        val posts = apiClient.getPosts()
        postQueries.transaction {
            postQueries.deleteAll()
            posts.forEach { post ->
                postQueries.insertOrReplace(
                    id = post.id,
                    userId = post.userId,
                    title = post.title,
                    body = post.body
                )
            }
        }
    }
    
    suspend fun getPost(id: Long): PostEntity? =
        postQueries.selectById(id).executeAsOneOrNull()
}
```

---

## ขั้นตอนที่ 481: Koin สำหรับ KMP

```kotlin
// ============================================
// Koin DI ที่ทำงานบน KMP
// ============================================

// Shared Module
val sharedModule = module {
    single { ApiClient() }
    single { PostRepository(get()) }
    single { UserRepository(get()) }
}

// Android Module
val androidModule = module {
    single { DatabaseDriverFactory(get<Context>()) }
    viewModel { PostListViewModel(get()) }
    viewModel { UserViewModel(get()) }
}

// iOS Module
val iosModule = module {
    single { DatabaseDriverFactory(null) }
}

// Android Application
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApplication)
            modules(sharedModule, androidModule)
        }
    }
}

// iOS (in AppDelegate)
// KoinApplication.start(
//     application: KoinApplication = KoinApplication.init(),
//     appDeclaration: AppDeclaration = {}
// ) -> KoinApplication {
//     return application.apply {
//         modules(SharedModuleKt.sharedModule, IOSModuleKt.iosModule)
//     }
// }

// Shared ViewModel (KMP ViewModel)
class PostListViewModel(
    private val postRepository: PostRepository
) : ViewModel() {
    
    val posts: StateFlow<List<PostEntity>> = postRepository
        .observePosts()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
    
    fun refresh() {
        viewModelScope.launch {
            postRepository.refreshPosts()
        }
    }
}
```

---

## ขั้นตอนที่ 482: Testing ใน KMP

```kotlin
// commonTest - ทดสอบบนทุก platform
class PostRepositoryTest {
    
    private val testDispatcher = StandardTestDispatcher()
    
    @Test
    fun `getPosts returns list from API`() = runTest(testDispatcher) {
        val mockClient = MockApiClient()
        mockClient.setPosts(listOf(
            Post(1, 1, "Test Post", "Content")
        ))
        
        val repository = PostRepository(mockClient, MockDatabase())
        
        repository.refreshPosts()
        
        val posts = repository.observePosts().first()
        assertEquals(1, posts.size)
        assertEquals("Test Post", posts[0].title)
    }
}

// Fake/Mock ที่ใช้ใน common test
class MockApiClient : ApiClientInterface {
    private val posts = mutableListOf<Post>()
    
    fun setPosts(newPosts: List<Post>) { posts.clear(); posts.addAll(newPosts) }
    
    override suspend fun getPosts(): List<Post> = posts.toList()
    override suspend fun getPost(id: Long): Post = posts.first { it.id == id }
}
```

---

## แบบฝึกหัด Part 20

```kotlin
// แบบฝึกหัด: สร้าง Shared Logic สำหรับ Notes App

// commonMain
@Serializable
data class Note(
    val id: String,
    val title: String,
    val content: String,
    val createdAt: Long,
    val updatedAt: Long
)

// TODO: สร้าง:
// 1. NoteApiService ด้วย Ktor (CRUD operations)
// 2. NoteRepository ที่มี local cache
// 3. expect/actual สำหรับ UUID generation
//    - Android: UUID.randomUUID().toString()
//    - iOS: NSUUID().UUIDString

// expect
expect fun generateId(): String

// TODO: implement actual สำหรับแต่ละ platform

// Shared use cases:
class CreateNoteUseCase(private val repository: NoteRepository) {
    suspend operator fun invoke(title: String, content: String): Note {
        val note = Note(
            id = generateId(),  // platform-specific
            title = title,
            content = content,
            createdAt = getCurrentTimeMillis(),
            updatedAt = getCurrentTimeMillis()
        )
        return repository.saveNote(note)
    }
}
```

---

*Part 20 จบแล้ว | ก่อนหน้า: [Part 19](../part19/README.md) | ถัดไป: [Part 21](../part21/README.md)*
