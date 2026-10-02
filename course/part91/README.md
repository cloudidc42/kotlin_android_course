# Part 91: Advanced Hilt Dependency Injection
## ขั้นตอนที่ 1751-1775

---

## ขั้นตอนที่ 1751: Hilt Multi-binding

```kotlin
// ============================================
// @IntoSet - ลงทะเบียน multiple implementations ใน Set
// ============================================

// Use case: Plugin system, event listeners, validators

interface Validator<T> {
    fun validate(value: T): ValidationResult
}

// ลงทะเบียน validators หลายตัว
@Module
@InstallIn(SingletonComponent::class)
object ValidatorModule {
    
    @Provides
    @IntoSet
    fun provideEmailValidator(): Validator<String> = EmailValidator()
    
    @Provides
    @IntoSet
    fun providePhoneValidator(): Validator<String> = PhoneValidator()
    
    @Provides
    @IntoSet
    fun provideUrlValidator(): Validator<String> = UrlValidator()
}

// ใช้ทุก validators
class CompositeValidator @Inject constructor(
    private val validators: Set<@JvmSuppressWildcards Validator<String>>
) {
    fun validateAll(value: String): List<ValidationResult> {
        return validators.map { it.validate(value) }
    }
    
    fun isValid(value: String): Boolean {
        return validators.all { it.validate(value) is ValidationResult.Valid }
    }
}

// ============================================
// @IntoMap - ลงทะเบียน implementations ใน Map
// ============================================

// Use case: Factory pattern, strategy pattern

interface PaymentProcessor {
    suspend fun process(amount: Double): PaymentResult
}

@MapKey
annotation class PaymentMethodKey(val value: PaymentMethod)

enum class PaymentMethod { CREDIT_CARD, PROMPTPAY, TRUEMONEY, LINEPAY }

@Module
@InstallIn(SingletonComponent::class)
object PaymentModule {
    
    @Provides
    @IntoMap
    @PaymentMethodKey(PaymentMethod.CREDIT_CARD)
    fun provideCreditCardProcessor(stripe: StripeClient): PaymentProcessor =
        CreditCardProcessor(stripe)
    
    @Provides
    @IntoMap
    @PaymentMethodKey(PaymentMethod.PROMPTPAY)
    fun providePromptPayProcessor(): PaymentProcessor = PromptPayProcessor()
    
    @Provides
    @IntoMap
    @PaymentMethodKey(PaymentMethod.TRUEMONEY)
    fun provideTrueMoneyProcessor(): PaymentProcessor = TrueMoneyProcessor()
}

class PaymentGateway @Inject constructor(
    private val processors: Map<PaymentMethod, @JvmSuppressWildcards PaymentProcessor>
) {
    suspend fun process(method: PaymentMethod, amount: Double): PaymentResult {
        val processor = processors[method]
            ?: throw IllegalArgumentException("Unsupported payment method: $method")
        return processor.process(amount)
    }
}
```

---

## ขั้นตอนที่ 1752: Hilt Entry Points

```kotlin
// ============================================
// EntryPoint - inject dependencies ใน non-Hilt classes
// ============================================

// ใช้เมื่อ: WorkManager, ContentProvider, BroadcastReceiver ที่ไม่ใช้ @AndroidEntryPoint

// 1. Define entry point interface
@EntryPoint
@InstallIn(SingletonComponent::class)
interface WorkerEntryPoint {
    fun userRepository(): UserRepository
    fun notificationManager(): AppNotificationManager
    fun analyticsTracker(): AnalyticsTracker
}

// 2. ใช้ใน Worker
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    // Get entry point
    private val entryPoint = EntryPointAccessors.fromApplication(
        context,
        WorkerEntryPoint::class.java
    )
    
    private val userRepository = entryPoint.userRepository()
    private val notificationManager = entryPoint.notificationManager()
    
    override suspend fun doWork(): Result {
        return try {
            userRepository.syncWithServer()
            notificationManager.showSyncComplete()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

// ============================================
// EntryPoint ใน ContentProvider
// ============================================

@EntryPoint
@InstallIn(SingletonComponent::class)
interface AppContentProviderEntryPoint {
    fun fileRepository(): FileRepository
}

class AppContentProvider : ContentProvider() {
    
    private val fileRepository: FileRepository by lazy {
        EntryPointAccessors.fromApplication(
            requireNotNull(context),
            AppContentProviderEntryPoint::class.java
        ).fileRepository()
    }
    
    override fun query(uri: Uri, projection: Array<String>?, selection: String?, 
                      selectionArgs: Array<String>?, sortOrder: String?): Cursor? {
        return fileRepository.query(uri, projection)
    }
    // ...
}
```

---

## ขั้นตอนที่ 1753: Hilt Testing

```kotlin
// ============================================
// Hilt Testing - Replace dependencies
// ============================================

// TestModule - replace production modules
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [NetworkModule::class]
)
object FakeNetworkModule {
    
    @Provides
    @Singleton
    fun provideProductApi(): ProductApi = FakeProductApi()
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(FakeInterceptor())
        .build()
}

// Test class
@HiltAndroidTest
class ProductListViewModelTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val mainDispatcherRule = MainDispatcherRule()
    
    @Inject
    lateinit var productRepository: ProductRepository
    
    @Inject
    lateinit var getProductsUseCase: GetProductsUseCase
    
    private lateinit var viewModel: ProductListViewModel
    
    @Before
    fun setup() {
        hiltRule.inject()
        viewModel = ProductListViewModel(getProductsUseCase)
    }
    
    @Test
    fun `loads products on init`() = runTest {
        val state = viewModel.uiState.value
        assertIs<ProductListUiState.Success>(state)
    }
}

// ============================================
// @BindValue - bind test value directly
// ============================================

@HiltAndroidTest
class SearchViewModelTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    // Override a specific binding
    @BindValue
    @JvmField
    val searchRepository: SearchRepository = FakeSearchRepository()
    
    @BindValue
    @JvmField
    val timeProvider: TimeProvider = FakeTimeProvider()
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
}
```

---

## ขั้นตอนที่ 1754: Custom Scopes

```kotlin
// ============================================
// Custom Hilt Scope
// ============================================

// Default scopes:
// @Singleton - สร้าง 1 ครั้ง, ใช้ตลอด app
// @ActivityScoped - 1 instance ต่อ Activity
// @ViewModelScoped - 1 instance ต่อ ViewModel

// Custom scope: UserScope (เมื่อ user logged in)
@Scope
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class UserScope

// Define component
@DefineComponent(parent = SingletonComponent::class)
@UserScope
interface UserComponent {
    
    @DefineComponent.Builder
    interface Builder {
        fun user(@BindsInstance user: User): Builder
        fun build(): UserComponent
    }
}

// Install in UserComponent
@Module
@InstallIn(UserComponent::class)
object UserModule {
    
    @Provides
    @UserScope
    fun provideUserPreferences(user: User, dataStore: DataStore<Preferences>): UserPreferences {
        return UserPreferences(user.id, dataStore)
    }
    
    @Provides
    @UserScope
    fun provideUserCart(user: User): Cart {
        return Cart(userId = user.id)
    }
}

// UserComponentManager - manages lifecycle
class UserComponentManager @Inject constructor(
    private val componentBuilder: UserComponent.Builder
) {
    var userComponent: UserComponent? = null
        private set
    
    fun onUserLogin(user: User) {
        userComponent = componentBuilder.user(user).build()
    }
    
    fun onUserLogout() {
        userComponent = null
    }
}

// Access UserComponent
@EntryPoint
@InstallIn(UserComponent::class)
interface UserEntryPoint {
    fun userPreferences(): UserPreferences
    fun userCart(): Cart
}

// Usage
val userComponent = userComponentManager.userComponent
    ?: error("User not logged in")

val entryPoint = EntryPointAccessors.fromComponent(userComponent, UserEntryPoint::class.java)
val preferences = entryPoint.userPreferences()
```

---

## ขั้นตอนที่ 1755: Hilt + WorkManager

```kotlin
// ============================================
// HiltWorker - inject dependencies ใน Worker
// ============================================

@HiltWorker
class DataSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val userRepository: UserRepository,
    private val syncService: SyncService,
    private val analyticsTracker: AnalyticsTracker
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        val userId = inputData.getLong("userId", -1L)
        if (userId == -1L) return Result.failure()
        
        return try {
            analyticsTracker.track("sync_started")
            syncService.syncUser(userId)
            analyticsTracker.track("sync_completed")
            
            Result.success(
                workDataOf("syncedAt" to System.currentTimeMillis())
            )
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                analyticsTracker.track("sync_failed", mapOf("error" to e.message))
                Result.failure()
            }
        }
    }
    
    companion object {
        fun buildRequest(userId: Long): OneTimeWorkRequest {
            return OneTimeWorkRequestBuilder<DataSyncWorker>()
                .setInputData(workDataOf("userId" to userId))
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED)
                        .build()
                )
                .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
                .addTag("sync_$userId")
                .build()
        }
    }
}

// Application setup
@HiltAndroidApp
class MyApplication : Application(), Configuration.Provider {
    
    @Inject
    lateinit var workerFactory: HiltWorkerFactory
    
    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}

// Enqueue work
class SyncRepository @Inject constructor(
    private val workManager: WorkManager
) {
    fun scheduleSyncForUser(userId: Long) {
        workManager.enqueueUniqueWork(
            "sync_user_$userId",
            ExistingWorkPolicy.REPLACE,
            DataSyncWorker.buildRequest(userId)
        )
    }
    
    fun observeSyncState(userId: Long): Flow<WorkInfo?> {
        return workManager
            .getWorkInfosByTagFlow("sync_$userId")
            .map { it.firstOrNull() }
    }
}
```

---

*Part 91 จบแล้ว | ก่อนหน้า: [Part 90](../part90/README.md) | ถัดไป: [Part 92](../part92/README.md)*
