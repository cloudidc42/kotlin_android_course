# Part 89: Reactive Programming Advanced
## ขั้นตอนที่ 1701-1725

---

## ขั้นตอนที่ 1701: Flow Operators Deep Dive

```kotlin
// ============================================
// Advanced Flow Operators
// ============================================

// zip - รวม 2 flows ทีละ pair
val names = flowOf("Alice", "Bob", "Charlie")
val scores = flowOf(90, 85, 92)

names.zip(scores) { name, score ->
    "$name: $score"
}.collect { println(it) }
// Alice: 90
// Bob: 85
// Charlie: 92

// combine - รวม latest values จากทั้ง 2 flows
val temperature = MutableStateFlow(25.0)
val humidity = MutableStateFlow(60.0)

temperature.combine(humidity) { temp, hum ->
    "Temp: ${temp}°C, Humidity: ${hum}%"
}.collect { println(it) }

// merge - รวม emissions จากหลาย flows
val stream1 = flowOf(1, 2, 3)
val stream2 = flowOf(10, 20, 30)

merge(stream1, stream2).collect { println(it) }
// Order not guaranteed

// flatMapMerge - concurrent flatMap
(1..5).asFlow()
    .flatMapMerge(concurrency = 3) { id ->
        flow {
            delay(Random.nextLong(100, 500))
            emit("Result for $id")
        }
    }
    .collect { println(it) }

// scan - accumulate
(1..5).asFlow()
    .scan(0) { acc, value -> acc + value }
    .collect { println(it) }
// 0, 1, 3, 6, 10, 15

// runningFold - alias for scan with initial value
(1..5).asFlow()
    .runningFold(listOf<Int>()) { acc, value -> acc + value }
    .collect { println(it) }

// buffer - run upstream without waiting for downstream
flow {
    repeat(100) { i ->
        emit(i)  // fast producer
    }
}
.buffer(capacity = 64)
.collect { item ->
    delay(100)  // slow consumer - won't block producer
    println(item)
}
```

---

## ขั้นตอนที่ 1702: StateFlow vs SharedFlow

```kotlin
// ============================================
// StateFlow - สำหรับ UI State
// ============================================

// Characteristics:
// - เก็บ current value (1 element buffer)
// - ต้องมี initial value
// - replays latest value ให้ new collectors
// - hot flow (ทำงานอยู่ตลอด)
// - .value accessible

class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
    
    fun increment() { _count.value++ }
    fun decrement() { _count.update { it - 1 } }
    
    // StateFlow ไม่ replay ถ้าค่าเท่ากัน (distinctUntilChanged implicit)
}

// ============================================
// SharedFlow - สำหรับ Events / Signals
// ============================================

// Characteristics:
// - configurable replay (default: 0)
// - ไม่มี initial value
// - hot flow
// - ใช้สำหรับ one-time events

class UserViewModel : ViewModel() {
    
    // Event flow: replay=0, no buffering (fire-and-forget)
    private val _events = MutableSharedFlow<UserEvent>()
    val events: SharedFlow<UserEvent> = _events.asSharedFlow()
    
    // Message flow: replay=1 (last message persists)
    private val _toast = MutableSharedFlow<String>(replay = 1, extraBufferCapacity = 10)
    val toast = _toast.asSharedFlow()
    
    fun showToast(message: String) {
        viewModelScope.launch {
            _toast.emit(message)
        }
    }
    
    fun navigateToProfile() {
        viewModelScope.launch {
            _events.emit(UserEvent.NavigateToProfile)
        }
    }
}

sealed class UserEvent {
    object NavigateToProfile : UserEvent()
    data class ShowDialog(val message: String) : UserEvent()
}

// Collecting events in Compose
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val context = LocalContext.current
    
    // Collect events (one-time)
    LaunchedEffect(viewModel) {
        viewModel.events.collect { event ->
            when (event) {
                is UserEvent.NavigateToProfile -> { /* navigate */ }
                is UserEvent.ShowDialog -> { /* show dialog */ }
            }
        }
    }
    
    // Collect toast (latest persists)
    val snackbarHostState = remember { SnackbarHostState() }
    
    LaunchedEffect(viewModel) {
        viewModel.toast.collect { message ->
            snackbarHostState.showSnackbar(message)
        }
    }
}
```

---

## ขั้นตอนที่ 1703: Flow in Repository Pattern

```kotlin
// ============================================
// Reactive Repository
// ============================================

class ReactiveProductRepository @Inject constructor(
    private val productDao: ProductDao,
    private val productApi: ProductApi,
    private val networkMonitor: NetworkMonitor
) : ProductRepository {
    
    // Single source of truth: Room
    // Refresh from network when online
    override fun observeProducts(categoryId: Long): Flow<List<Product>> {
        return productDao.observeByCategory(categoryId)
            .map { entities -> entities.map { it.toDomain() } }
            .onStart {
                // Trigger refresh if online
                if (networkMonitor.isOnline) {
                    refreshCategory(categoryId)
                }
            }
    }
    
    // Offline-first search
    override fun searchProducts(query: Flow<String>): Flow<List<Product>> {
        return query
            .debounce(300)
            .distinctUntilChanged()
            .flatMapLatest { searchQuery ->
                if (searchQuery.length < 2) {
                    flowOf(emptyList())
                } else {
                    productDao.search(searchQuery)
                        .map { entities -> entities.map { it.toDomain() } }
                        .onStart {
                            // Also search network for comprehensive results
                            if (networkMonitor.isOnline) {
                                productApi.search(searchQuery).body()?.forEach {
                                    productDao.upsert(it.toEntity())
                                }
                            }
                        }
                }
            }
    }
    
    // Paginated flow
    fun observePagedProducts(): Flow<PagingData<Product>> {
        return Pager(
            config = PagingConfig(pageSize = 20, enablePlaceholders = false),
            pagingSourceFactory = { ProductPagingSource(productApi, productDao) }
        ).flow
    }
    
    private suspend fun refreshCategory(categoryId: Long) {
        try {
            val products = productApi.getByCategory(categoryId).body() ?: return
            productDao.upsertAll(products.map { it.toEntity() })
        } catch (e: Exception) {
            // Silently fail - local data still shown
        }
    }
}

// ============================================
// Flow Error Handling
// ============================================

fun <T> Flow<T>.handleErrors(
    onError: suspend (Throwable) -> Unit = {}
): Flow<Result<T>> {
    return this
        .map { value -> Result.success(value) }
        .catch { e ->
            onError(e)
            emit(Result.failure(e))
        }
}

fun <T> Flow<T>.retryWithExponentialBackoff(
    maxRetries: Int = 3,
    initialDelay: Long = 1000L,
    maxDelay: Long = 30_000L,
    factor: Double = 2.0
): Flow<T> {
    var currentDelay = initialDelay
    return retry(maxRetries.toLong()) { e ->
        if (e is IOException) {
            delay(currentDelay)
            currentDelay = minOf(currentDelay * factor, maxDelay.toDouble()).toLong()
            true
        } else {
            false
        }
    }
}

// Usage
productRepository.observeProducts(categoryId = 1L)
    .retryWithExponentialBackoff()
    .handleErrors { e -> analytics.trackError("product_load_failed", e) }
    .collect { result ->
        result
            .onSuccess { products -> updateUi(products) }
            .onFailure { error -> showError(error) }
    }
```

---

## ขั้นตอนที่ 1704: Channel-based Patterns

```kotlin
// ============================================
// Channel สำหรับ Worker coordination
// ============================================

class DownloadQueue @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) {
    
    private val taskChannel = Channel<DownloadTask>(capacity = 100)
    private val resultChannel = Channel<DownloadResult>(capacity = Channel.UNLIMITED)
    
    val results: Flow<DownloadResult> = resultChannel.receiveAsFlow()
    
    fun start(scope: CoroutineScope, workerCount: Int = 3) {
        // Multiple workers consume from same channel
        repeat(workerCount) { workerId ->
            scope.launch(ioDispatcher) {
                for (task in taskChannel) {
                    try {
                        val data = download(task.url)
                        resultChannel.send(DownloadResult.Success(task, data))
                    } catch (e: Exception) {
                        resultChannel.send(DownloadResult.Failed(task, e))
                    }
                }
            }
        }
    }
    
    suspend fun enqueue(task: DownloadTask) {
        taskChannel.send(task)
    }
    
    fun stop() {
        taskChannel.close()
        resultChannel.close()
    }
    
    private suspend fun download(url: String): ByteArray {
        return withContext(ioDispatcher) {
            // HTTP download
            ByteArray(0)
        }
    }
}

data class DownloadTask(val id: Long, val url: String, val fileName: String)
sealed class DownloadResult {
    data class Success(val task: DownloadTask, val data: ByteArray) : DownloadResult()
    data class Failed(val task: DownloadTask, val error: Throwable) : DownloadResult()
}
```

---

## แบบฝึกหัด Part 89

```kotlin
// แบบฝึกหัด: Reactive Form Validation

// สร้าง Signup Form ที่:
// 1. Validate ทุก field แบบ reactive
// 2. Show error เฉพาะหลังจาก field ถูก touch แล้ว
// 3. Submit button enable เมื่อ form valid ทั้งหมด
// 4. Username availability check (debounce 500ms)
// 5. Password strength indicator

class SignupFormState {
    
    private val _username = MutableStateFlow("")
    private val _email = MutableStateFlow("")
    private val _password = MutableStateFlow("")
    private val _confirmPassword = MutableStateFlow("")
    
    private val touchedFields = mutableSetOf<String>()
    
    val usernameError: Flow<String?> = _username
        .debounce(500)
        .flatMapLatest { username ->
            // TODO: check availability
            flow { emit(validateUsername(username)) }
        }
    
    val passwordStrength: Flow<PasswordStrength> = _password.map { password ->
        calculateStrength(password)
    }
    
    val isFormValid: Flow<Boolean> = combine(
        _username, _email, _password, _confirmPassword
    ) { username, email, password, confirm ->
        username.length >= 3 &&
        email.contains("@") &&
        password.length >= 8 &&
        password == confirm
    }
    
    fun onUsernameChange(value: String) { _username.value = value; touchedFields.add("username") }
    fun onEmailChange(value: String) { _email.value = value }
    fun onPasswordChange(value: String) { _password.value = value }
    fun onConfirmPasswordChange(value: String) { _confirmPassword.value = value }
    
    private fun validateUsername(username: String): String? = when {
        username.length < 3 -> "อย่างน้อย 3 ตัวอักษร"
        !username.matches(Regex("[a-zA-Z0-9_]+")) -> "ใช้ตัวอักษร ตัวเลข หรือ _ เท่านั้น"
        else -> null
    }
}

enum class PasswordStrength { WEAK, FAIR, STRONG, VERY_STRONG }
```

---

*Part 89 จบแล้ว | ก่อนหน้า: [Part 88](../part88/README.md) | ถัดไป: [Part 90](../part90/README.md)*
