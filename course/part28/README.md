# Part 28: Hilt - Dependency Injection
## ขั้นตอนที่ 676-700

---

## ขั้นตอนที่ 676: Dependency Injection คืออะไร?

DI = การส่ง dependencies จากภายนอกแทนที่จะสร้างข้างใน

```kotlin
// ❌ ไม่ใช้ DI - สร้าง dependencies ข้างใน
class UserViewModel {
    private val database = Room.databaseBuilder(...)  // hard-coded!
    private val api = Retrofit.Builder().build()      // hard-coded!
    private val repo = UserRepository(database, api)  // สร้างเอง
    
    // ปัญหา: test ยาก, เปลี่ยน implementation ยาก
}

// ✅ ใช้ DI - รับ dependencies จากภายนอก
class UserViewModel(
    private val userRepository: UserRepository  // inject จากภายนอก
) : ViewModel() {
    // test ง่าย, เปลี่ยน implementation ได้
}

// Hilt จัดการสร้างและส่ง dependencies ให้อัตโนมัติ
```

---

## ขั้นตอนที่ 677: Hilt Setup

```kotlin
// build.gradle.kts (root project)
// plugins {
//     id("com.google.dagger.hilt.android") version "2.52" apply false
// }

// build.gradle.kts (app module)
// plugins {
//     id("com.google.dagger.hilt.android")
//     id("com.google.devtools.ksp")
// }
//
// dependencies {
//     implementation("com.google.dagger:hilt-android:2.52")
//     ksp("com.google.dagger:hilt-android-compiler:2.52")
//     
//     implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
//     ksp("androidx.hilt:hilt-compiler:1.2.0")
// }

// Application class
@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // Hilt จะ generate code ที่ setup DI container
    }
}

// AndroidManifest.xml
// <application
//     android:name=".MyApplication"
//     ...>
```

---

## ขั้นตอนที่ 678: @Inject Annotation

```kotlin
// ============================================
// Constructor Injection
// ============================================

// Class ที่ Hilt จะ inject ให้
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val apiService: ApiService,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) {
    // Hilt รู้วิธีสร้าง UserRepository เพราะมี @Inject
    
    suspend fun getUsers(): List<User> = withContext(ioDispatcher) {
        userDao.getAllUsers().map { it.toDomain() }
    }
}

// ============================================
// @HiltViewModel - inject ใน ViewModel
// ============================================

@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val analyticsService: AnalyticsService,
    savedStateHandle: SavedStateHandle  // inject โดย Hilt
) : ViewModel() {
    
    val users = userRepository.observeUsers()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}

// ============================================
// @AndroidEntryPoint - inject ใน Activity/Fragment
// ============================================

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    
    // Field injection
    @Inject lateinit var analyticsService: AnalyticsService
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        analyticsService.logEvent("app_opened")
        
        setContent {
            MyApp()
        }
    }
}

@AndroidEntryPoint
class HomeFragment : Fragment() {
    
    @Inject lateinit var userRepository: UserRepository
    
    // ViewModel inject ผ่าน hiltViewModel()
    private val viewModel: UserViewModel by viewModels()
}

// Composable ใช้ hiltViewModel()
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    // ...
}
```

---

## ขั้นตอนที่ 679: @Module และ @Provides

```kotlin
// ใช้ Module เมื่อ Hilt ไม่สามารถ inject ด้วย @Inject ได้
// เช่น: interface, third-party classes, classes ที่ต้องการ configuration

@Module
@InstallIn(SingletonComponent::class)  // อยู่นาน app ทั้ง lifecycle
object DatabaseModule {
    
    @Provides
    @Singleton  // สร้างแค่ครั้งเดียว
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        )
        .addMigrations(MIGRATION_1_2)
        .fallbackToDestructiveMigration()  // dev only!
        .build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao = database.userDao()
    
    @Provides
    fun providePostDao(database: AppDatabase): PostDao = database.postDao()
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor,
        @ApplicationContext context: Context
    ): OkHttpClient {
        val cache = Cache(context.cacheDir, 10 * 1024 * 1024)  // 10 MB
        
        return OkHttpClient.Builder()
            .cache(cache)
            .addInterceptor(authInterceptor)
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG)
                    HttpLoggingInterceptor.Level.BODY
                else
                    HttpLoggingInterceptor.Level.NONE
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(
                GsonConverterFactory.create(
                    GsonBuilder()
                        .setDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'")
                        .create()
                )
            )
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService =
        retrofit.create(ApiService::class.java)
}
```

---

## ขั้นตอนที่ 680: @Binds - bind Interface to Implementation

```kotlin
// @Binds ใช้เมื่อมี interface และ implementation

// Interface
interface UserRepository {
    suspend fun getUsers(): List<User>
    fun observeUsers(): Flow<List<User>>
}

// Implementation
class UserRepositoryImpl @Inject constructor(
    private val userDao: UserDao,
    private val apiService: ApiService
) : UserRepository {
    override suspend fun getUsers(): List<User> = userDao.getAllUsers().map { it.toDomain() }
    override fun observeUsers(): Flow<List<User>> = userDao.observeAllUsers().map { it.map { e -> e.toDomain() } }
}

// Module ใช้ @Binds
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository  // Hilt จะใช้ UserRepositoryImpl เมื่อต้องการ UserRepository
    
    @Binds
    @Singleton
    abstract fun bindPostRepository(
        impl: PostRepositoryImpl
    ): PostRepository
}

// @Binds vs @Provides:
// @Binds: abstract function, เร็วกว่า (compile-time)
// @Provides: concrete function, ยืดหยุ่นกว่า (สำหรับ external classes)
```

---

## ขั้นตอนที่ 681: Scopes

```kotlin
// Scopes กำหนดวงจรชีวิตของ dependency

// SingletonComponent - อยู่ตลอด app lifecycle
@Singleton
class AppConfig @Inject constructor() {
    val apiKey: String = BuildConfig.API_KEY
}

// ActivityRetainedComponent - อยู่ตลอด Activity lifecycle (รอด rotation)
@ActivityRetainedScoped
class UserSession @Inject constructor() {
    var currentUser: User? = null
    var isLoggedIn: Boolean = false
}

// ActivityComponent - อยู่ตาม Activity lifecycle
@ActivityScoped
class NavigationManager @Inject constructor(
    private val activity: Activity
) {
    fun navigate(screen: Screen) { /* ... */ }
}

// ViewModelComponent - อยู่ตาม ViewModel lifecycle
@ViewModelScoped
class UserPagingSource @Inject constructor(
    private val apiService: ApiService
) : PagingSource<Int, User>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        // ...
        return LoadResult.Page(emptyList(), null, null)
    }
    override fun getRefreshKey(state: PagingState<Int, User>): Int? = null
}

// FragmentComponent - อยู่ตาม Fragment lifecycle
@FragmentScoped
class FragmentHelper @Inject constructor(
    private val fragment: Fragment
) {
    // Fragment-specific functionality
}

// ============================================
// Component hierarchy
// ============================================
//
// SingletonComponent
//   └── ActivityRetainedComponent
//         └── ViewModelComponent
//         └── ActivityComponent
//               └── FragmentComponent
//                     └── ViewComponent
//
// Component ที่อยู่สูงกว่า inject ให้ Component ที่อยู่ต่ำกว่าได้
// แต่ไม่สามารถย้อนกลับ
```

---

## ขั้นตอนที่ 682: Qualifiers

```kotlin
// ============================================
// Custom Qualifiers
// ============================================

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class MainDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

// Module
@Module
@InstallIn(SingletonComponent::class)
object DispatchersModule {
    
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
    
    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
    
    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}

// ใช้ Qualifier
class MyRepository @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
    @DefaultDispatcher private val defaultDispatcher: CoroutineDispatcher
) {
    suspend fun fetchData() = withContext(ioDispatcher) {
        // IO work
    }
    
    suspend fun processData(data: List<Any>) = withContext(defaultDispatcher) {
        // CPU work
    }
}

// ============================================
// @Named (ง่ายกว่า แต่ไม่ type-safe)
// ============================================

@Module
@InstallIn(SingletonComponent::class)
object NamedModule {
    
    @Provides
    @Named("production")
    fun provideProductionUrl(): String = "https://api.production.com"
    
    @Provides
    @Named("staging")
    fun provideStagingUrl(): String = "https://api.staging.com"
}

// ใช้ @Named
class ApiClient @Inject constructor(
    @Named("production") private val productionUrl: String
) {
    // ...
}
```

---

## ขั้นตอนที่ 683: Testing กับ Hilt

```kotlin
// ============================================
// Unit Test - ไม่ต้องใช้ Hilt
// ============================================

class UserViewModelTest {
    
    private val fakeRepository = FakeUserRepository()
    private val viewModel = UserViewModel(fakeRepository)
    
    @Test
    fun `test load users`() = runTest {
        // ไม่ต้องการ Hilt ใน unit test
        viewModel.loadUsers()
        assertEquals(expectedUsers, viewModel.users.value)
    }
}

// ============================================
// Integration Test ด้วย Hilt
// ============================================

@HiltAndroidTest
class UserScreenTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()
    
    @Inject
    lateinit var userRepository: UserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun testUserListDisplayed() {
        composeRule.onNodeWithText("Alice").assertIsDisplayed()
    }
}

// ============================================
// Replace Module ใน Test
// ============================================

@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class]
)
@Module
abstract class FakeRepositoryModule {
    
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        fake: FakeUserRepository
    ): UserRepository
}

class FakeUserRepository @Inject constructor() : UserRepository {
    val users = mutableListOf<User>()
    
    override suspend fun getUsers() = users.toList()
    override fun observeUsers() = MutableStateFlow(users.toList()).asStateFlow()
}
```

---

## ขั้นตอนที่ 684: Assisted Injection

```kotlin
// Assisted Injection - เมื่อ dependency บางตัวต้องส่ง runtime

// ต้องเพิ่ม dependency:
// implementation("com.google.dagger:hilt-android:2.52")
// ksp("com.google.dagger:hilt-android-compiler:2.52")

@AssistedFactory
interface DetailViewModelFactory {
    fun create(postId: Long): DetailViewModel
}

class DetailViewModel @AssistedInject constructor(
    @Assisted val postId: Long,  // runtime value
    private val repository: PostRepository,  // normal injection
    private val analyticsService: AnalyticsService
) : ViewModel() {
    
    val post = repository.observePost(postId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}

// ใช้ใน Composable
@Composable
fun DetailScreen(postId: Long) {
    val factory: DetailViewModelFactory = hiltViewModel()
    val viewModel = factory.create(postId)
    
    val post by viewModel.post.collectAsStateWithLifecycle()
    // ...
}
```

---

## แบบฝึกหัด Part 28

```kotlin
// แบบฝึกหัด: Setup DI สำหรับ Weather App

// Interfaces
interface WeatherApi {
    suspend fun getCurrentWeather(city: String): WeatherData
    suspend fun getForecast(city: String, days: Int): List<ForecastDay>
}

interface WeatherRepository {
    suspend fun getWeather(city: String): Result<WeatherData>
    suspend fun getForecast(city: String): Result<List<ForecastDay>>
}

// TODO: สร้าง Module สำหรับ
// 1. NetworkModule - OkHttp, Retrofit, WeatherApi
// 2. RepositoryModule - bind WeatherRepository
// 3. DispatcherModule - IO, Main, Default dispatchers

// TODO: Implement WeatherRepositoryImpl ที่ inject ผ่าน Hilt

// TODO: สร้าง WeatherViewModel ที่ใช้ @HiltViewModel

// TODO: เพิ่ม @HiltAndroidApp ใน Application class
// TODO: เพิ่ม @AndroidEntryPoint ใน Activity
```

---

*Part 28 จบแล้ว | ก่อนหน้า: [Part 27](../part27/README.md) | ถัดไป: [Part 29](../part29/README.md)*
