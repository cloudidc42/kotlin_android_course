# Part 60: Advanced Architecture Patterns
## ขั้นตอนที่ 976-1000

---

## ขั้นตอนที่ 976: Repository Pattern ขั้นสูง

```kotlin
// ============================================
// Offline-First Repository
// ============================================

class ProductRepository @Inject constructor(
    private val apiService: ApiService,
    private val productDao: ProductDao,
    private val networkMonitor: NetworkMonitor
) {
    
    // Single source of truth = Room DB
    fun observeProducts(): Flow<List<Product>> {
        return productDao.observeAll()
            .map { entities -> entities.map { it.toDomain() } }
            .onStart {
                // Refresh ถ้ามีเน็ต
                if (networkMonitor.isOnline) {
                    refreshProducts()
                }
            }
    }
    
    suspend fun refreshProducts() {
        val response = apiService.getProducts()
        productDao.upsertAll(response.map { it.toEntity() })
    }
    
    fun observeProduct(id: Long): Flow<Product?> = flow {
        // ลอง local ก่อน
        val local = productDao.findById(id)
        if (local != null) {
            emit(local.toDomain())
        }
        
        // Fetch remote
        try {
            val remote = apiService.getProduct(id)
            productDao.upsert(remote.toEntity())
            emit(remote.toDomain())
        } catch (e: Exception) {
            if (local == null) throw e
        }
    }
}

// ============================================
// Network Monitor
// ============================================

class NetworkMonitor @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    val networkState: Flow<Boolean> = callbackFlow {
        val manager = context.getSystemService(ConnectivityManager::class.java)
        
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
        }
        
        manager.registerDefaultNetworkCallback(callback)
        
        // Emit initial state
        val isConnected = manager.activeNetwork != null
        trySend(isConnected)
        
        awaitClose { manager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()
    
    val isOnline: Boolean
        get() {
            val manager = context.getSystemService(ConnectivityManager::class.java)
            return manager.activeNetwork != null
        }
}
```

---

## ขั้นตอนที่ 977: Use Case Composition

```kotlin
// ============================================
// Use Cases แบบ Composable
// ============================================

// Base use case
abstract class UseCase<in Params, out Result> {
    abstract suspend fun execute(params: Params): Result
    
    suspend operator fun invoke(params: Params): Result = execute(params)
}

abstract class FlowUseCase<in Params, out Result> {
    abstract fun execute(params: Params): Flow<Result>
    
    operator fun invoke(params: Params): Flow<Result> = execute(params)
        .flowOn(Dispatchers.Default)
}

// Concrete use cases
class GetUserProfileUseCase @Inject constructor(
    private val userRepository: UserRepository
) : UseCase<Long, UserProfile>() {
    override suspend fun execute(params: Long): UserProfile {
        return userRepository.getUser(params)
    }
}

class UpdateUserProfileUseCase @Inject constructor(
    private val userRepository: UserRepository,
    private val validator: UserValidator
) : UseCase<UpdateUserParams, Result<UserProfile>>() {
    override suspend fun execute(params: UpdateUserParams): Result<UserProfile> {
        val errors = validator.validate(params)
        if (errors.isNotEmpty()) {
            return Result.failure(ValidationException(errors))
        }
        return runCatching { userRepository.updateUser(params) }
    }
}

class ObserveOrdersUseCase @Inject constructor(
    private val orderRepository: OrderRepository
) : FlowUseCase<Long, List<Order>>() {
    override fun execute(params: Long): Flow<List<Order>> {
        return orderRepository.observeByUserId(params)
    }
}

// Composing use cases ใน ViewModel
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val getUserProfile: GetUserProfileUseCase,
    private val updateUserProfile: UpdateUserProfileUseCase,
    private val observeOrders: ObserveOrdersUseCase,
    private val getCurrentUserId: GetCurrentUserIdUseCase
) : ViewModel() {
    
    private val userId = getCurrentUserId()
    
    val profile: StateFlow<ProfileState> = flow {
        emit(ProfileState.Loading)
        val user = getUserProfile(userId)
        emit(ProfileState.Success(user))
    }.catch { e ->
        emit(ProfileState.Error(e.message ?: "Unknown error"))
    }.stateIn(viewModelScope, SharingStarted.Lazily, ProfileState.Idle)
    
    val orders: StateFlow<List<Order>> = observeOrders(userId)
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())
    
    fun updateProfile(params: UpdateUserParams) {
        viewModelScope.launch {
            when (val result = updateUserProfile(params)) {
                is Result.Success -> { /* refresh */ }
                is Result.Failure -> { /* show error */ }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 978: Error Handling Architecture

```kotlin
// ============================================
// Domain Errors
// ============================================

sealed class DomainError : Exception() {
    // Network errors
    sealed class NetworkError : DomainError() {
        object NoInternetConnection : NetworkError()
        data class ServerError(val code: Int, val message: String) : NetworkError()
        object Timeout : NetworkError()
        object Unknown : NetworkError()
    }
    
    // Auth errors
    sealed class AuthError : DomainError() {
        object Unauthorized : AuthError()
        object SessionExpired : AuthError()
        object Forbidden : AuthError()
    }
    
    // Business errors
    sealed class BusinessError : DomainError() {
        data class ValidationError(val fields: Map<String, String>) : BusinessError()
        data class NotFound(val resource: String) : BusinessError()
        data class Conflict(val message: String) : BusinessError()
    }
    
    // Data errors
    sealed class DataError : DomainError() {
        object DatabaseError : DataError()
        data class ParseError(val message: String) : DataError()
    }
}

// Error Mapper
class ApiErrorMapper {
    fun map(throwable: Throwable): DomainError {
        return when (throwable) {
            is UnknownHostException, is ConnectException -> 
                DomainError.NetworkError.NoInternetConnection
            
            is SocketTimeoutException -> 
                DomainError.NetworkError.Timeout
            
            is HttpException -> when (throwable.code()) {
                401 -> DomainError.AuthError.Unauthorized
                403 -> DomainError.AuthError.Forbidden
                404 -> DomainError.BusinessError.NotFound("resource")
                409 -> DomainError.BusinessError.Conflict(throwable.message())
                in 500..599 -> DomainError.NetworkError.ServerError(
                    throwable.code(), throwable.message()
                )
                else -> DomainError.NetworkError.Unknown
            }
            
            else -> DomainError.NetworkError.Unknown
        }
    }
}

// UI Error Messages
@Composable
fun DomainError.toUserMessage(): String = when (this) {
    is DomainError.NetworkError.NoInternetConnection -> 
        stringResource(R.string.error_no_internet)
    is DomainError.NetworkError.Timeout -> 
        stringResource(R.string.error_timeout)
    is DomainError.NetworkError.ServerError -> 
        stringResource(R.string.error_server, code)
    is DomainError.AuthError.SessionExpired -> 
        stringResource(R.string.error_session_expired)
    is DomainError.BusinessError.ValidationError -> 
        fields.values.firstOrNull() ?: stringResource(R.string.error_validation)
    else -> stringResource(R.string.error_unknown)
}
```

---

## ขั้นตอนที่ 979: Dependency Injection Advanced

```kotlin
// ============================================
// Multi-binding กับ Hilt
// ============================================

// Scenario: หลาย Analytics providers
interface AnalyticsTracker {
    fun track(event: AnalyticsEvent)
}

class FirebaseAnalyticsTracker @Inject constructor(
    private val analytics: FirebaseAnalytics
) : AnalyticsTracker {
    override fun track(event: AnalyticsEvent) {
        analytics.logEvent(event.name, event.params.toBundle())
    }
}

class MixpanelTracker @Inject constructor(
    private val mixpanel: MixpanelAPI
) : AnalyticsTracker {
    override fun track(event: AnalyticsEvent) {
        mixpanel.track(event.name, JSONObject(event.params))
    }
}

// Hilt Module กับ @IntoSet
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {
    
    @Binds
    @IntoSet
    abstract fun bindFirebaseTracker(impl: FirebaseAnalyticsTracker): AnalyticsTracker
    
    @Binds
    @IntoSet
    abstract fun bindMixpanelTracker(impl: MixpanelTracker): AnalyticsTracker
}

// Composite tracker - เรียกทุก trackers
class CompositeAnalyticsTracker @Inject constructor(
    private val trackers: Set<@JvmSuppressWildcards AnalyticsTracker>
) : AnalyticsTracker {
    override fun track(event: AnalyticsEvent) {
        trackers.forEach { it.track(event) }
    }
}

// ============================================
// Qualified Dependencies
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

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    
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

// ใช้งาน Qualified
class DataRepository @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
    @DefaultDispatcher private val defaultDispatcher: CoroutineDispatcher
) {
    suspend fun fetchData(): Data = withContext(ioDispatcher) {
        // network call
    }
    
    suspend fun processData(data: Data): ProcessedData = withContext(defaultDispatcher) {
        // cpu-intensive work
    }
}
```

---

## ขั้นตอนที่ 980: สรุป Professional Android Development

```kotlin
// ============================================
// สรุปสิ่งที่เรียนมาตลอดหลักสูตร
// ============================================

/*
 * Kotlin Fundamentals (Part 01-20):
 * ✅ Variables, types, control flow
 * ✅ Functions, lambdas, higher-order functions
 * ✅ OOP: classes, inheritance, interfaces
 * ✅ Null safety, generics, type system
 * ✅ Coroutines, Flow, StateFlow
 * ✅ Testing: JUnit, MockK, Turbine
 * ✅ File I/O, DataStore
 * ✅ DSL, Builder pattern
 * ✅ Kotlin Multiplatform
 *
 * Android Fundamentals (Part 21-30):
 * ✅ Activity, Fragment, Lifecycle
 * ✅ Jetpack Compose
 * ✅ Navigation Compose
 * ✅ ViewModel, State management
 * ✅ Retrofit, Networking
 * ✅ Room Database
 * ✅ Hilt DI
 * ✅ Testing in Android
 * ✅ WorkManager
 *
 * Architecture (Part 51-53):
 * ✅ Clean Architecture
 * ✅ MVVM, MVI
 * ✅ Modularization
 *
 * Professional Level (Part 54-60):
 * ✅ Performance Optimization
 * ✅ App Security
 * ✅ Compose Animations
 * ✅ CI/CD, Release
 * ✅ Accessibility, Localization
 * ✅ Custom Composables
 * ✅ Advanced Architecture
 *
 * World-class Topics (Part 61+):
 * - Kotlin compiler plugins
 * - Compose internals, Slot API
 * - Advanced coroutines internals
 * - Memory profiling
 * - Custom Gradle plugins
 * - Large-scale app architecture
 * - Machine Learning on-device
 * - Camera X, ML Kit
 */

// Professional Development Checklist:
object ProfessionalChecklist {
    val architecture = listOf(
        "Clean Architecture layers",
        "Single source of truth",
        "Unidirectional data flow",
        "Modular project structure",
        "Convention plugins"
    )
    
    val codeQuality = listOf(
        "Kotlin idioms",
        "Null safety",
        "Coroutines best practices",
        "Unit tests > 80% coverage",
        "UI tests for critical flows"
    )
    
    val performance = listOf(
        "Recomposition analysis",
        "Baseline Profiles",
        "Memory leak detection",
        "Paging for large lists",
        "Background work optimization"
    )
    
    val security = listOf(
        "HTTPS + certificate pinning",
        "Encrypted storage",
        "Biometric auth",
        "ProGuard/R8 rules",
        "No hardcoded secrets"
    )
    
    val ux = listOf(
        "Smooth animations",
        "Offline-first",
        "Error states",
        "Loading states",
        "Accessibility (TalkBack)"
    )
    
    val devops = listOf(
        "CI/CD pipeline",
        "Automated testing",
        "Versioning strategy",
        "Play Store releases",
        "Crash reporting (Firebase)"
    )
}
```

---

## จบหลักสูตร Part 60

```
🎉 ยินดีด้วย! คุณได้เรียนรู้ครบ 1000 ขั้นตอนแล้ว!

จาก "Hello World" สู่ Professional Android Developer:

✅ Kotlin ตั้งแต่พื้นฐานถึงขั้นสูง
✅ Android native app development
✅ Jetpack Compose UI
✅ Clean Architecture
✅ Testing
✅ Performance & Security
✅ CI/CD & Release

ก้าวต่อไปของคุณ:
1. สร้าง app จริงๆ ด้วยสิ่งที่เรียนมา
2. Contribute to open source
3. เรียนรู้ Kotlin Multiplatform
4. ศึกษา Compose internals
5. ร่วมชุมชน: KotlinConf, Android Dev Summit

Resources:
- kotlinlang.org
- developer.android.com
- github.com/android/architecture-samples
- github.com/android/nowinandroid
```

---

*Part 60 จบแล้ว - จบหลักสูตร! | ก่อนหน้า: [Part 59](../part59/README.md)*
