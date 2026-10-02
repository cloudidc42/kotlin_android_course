# Part 64: Advanced Coroutines & Concurrency
## ขั้นตอนที่ 1076-1100

---

## ขั้นตอนที่ 1076: Coroutine Internals

```kotlin
// ============================================
// Continuation - หัวใจของ Coroutine
// ============================================

/*
 * Coroutine ทำงานโดย CPS (Continuation-Passing Style)
 * Compiler แปลง suspend function เป็น state machine
 *
 * ตัวอย่าง:
 */

// Code ที่เราเขียน:
suspend fun fetchUserAndOrders(userId: Long): Pair<User, List<Order>> {
    val user = fetchUser(userId)
    val orders = fetchOrders(userId)
    return Pair(user, orders)
}

// Compiler แปลงเป็น (simplified):
fun fetchUserAndOrders(userId: Long, continuation: Continuation<Pair<User, List<Order>>>) {
    // State machine
    when (continuation.label) {
        0 -> {
            continuation.label = 1
            fetchUser(userId, continuation)  // suspend point
        }
        1 -> {
            val user = continuation.result as User
            continuation.label = 2
            fetchOrders(userId, continuation)  // suspend point
        }
        2 -> {
            val orders = continuation.result as List<Order>
            continuation.resume(Pair(continuation.user, orders))
        }
    }
}
```

---

## ขั้นตอนที่ 1077: Structured Concurrency Patterns

```kotlin
// ============================================
// supervisorScope vs coroutineScope
// ============================================

// coroutineScope: ถ้า child ล้มเหลว → cancel sibling + propagate
suspend fun fetchAllRequired(): AllData {
    return coroutineScope {
        val users = async { fetchUsers() }
        val products = async { fetchProducts() }
        
        // ถ้า fetchUsers() throw → fetchProducts() ถูก cancel ด้วย
        AllData(users.await(), products.await())
    }
}

// supervisorScope: child ล้มเหลว → ไม่กระทบ sibling
suspend fun fetchAllOptional(): PartialData {
    return supervisorScope {
        val mainData = async { fetchMainData() }
        val optionalData = async { fetchOptionalData() }
        val recommendations = async { fetchRecommendations() }
        
        PartialData(
            main = mainData.await(),  // required
            optional = runCatching { optionalData.await() }.getOrNull(),  // optional
            recommendations = runCatching { recommendations.await() }.getOrDefault(emptyList())
        )
    }
}

// ============================================
// CoroutineExceptionHandler
// ============================================

val handler = CoroutineExceptionHandler { context, exception ->
    println("Caught: ${exception.message}")
    // log to crash reporting
    FirebaseCrashlytics.getInstance().recordException(exception)
}

val scope = CoroutineScope(
    Dispatchers.Default + 
    SupervisorJob() + 
    handler
)

// ============================================
// Cancellation Best Practices
// ============================================

class FileProcessor {
    
    suspend fun processLargeFile(file: File): ProcessedData {
        val lines = file.readLines()
        
        return buildList {
            for ((index, line) in lines.withIndex()) {
                // ตรวจสอบ cancellation ทุก 1000 rows
                if (index % 1000 == 0) {
                    ensureActive()  // throw CancellationException ถ้า cancelled
                    yield()  // ให้ coroutine อื่น run ได้
                }
                
                val processed = processLine(line)
                add(processed)
            }
        }
    }
    
    // ❌ NonCancellable work
    suspend fun badCleanup() {
        // ถ้า coroutine ถูก cancel, withContext(NonCancellable) จะ run ต่อไป
        withContext(NonCancellable) {
            // cleanup code ที่ต้องรัน even ถ้า cancelled
            database.saveState(state)
        }
    }
}
```

---

## ขั้นตอนที่ 1078: Flow Advanced Operators

```kotlin
// ============================================
// Flow Operators ที่มีประโยชน์
// ============================================

// flatMapLatest - cancel previous แล้ว start ใหม่
class SearchViewModel @Inject constructor(
    private val searchRepository: SearchRepository
) : ViewModel() {
    
    private val _query = MutableStateFlow("")
    
    val results: StateFlow<List<SearchResult>> = _query
        .debounce(300)  // รอ 300ms หลัง user หยุดพิมพ์
        .filter { it.length >= 2 }  // อย่างน้อย 2 ตัวอักษร
        .distinctUntilChanged()  // ไม่ search ถ้า query เหมือนเดิม
        .flatMapLatest { query ->  // cancel previous search
            searchRepository.search(query)
                .catch { emit(emptyList()) }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
    
    fun onQueryChange(query: String) {
        _query.value = query
    }
}

// ============================================
// combine - merge หลาย flows
// ============================================

class DashboardViewModel @Inject constructor(
    private val userRepo: UserRepository,
    private val orderRepo: OrderRepository,
    private val notifRepo: NotificationRepository
) : ViewModel() {
    
    data class DashboardState(
        val user: User? = null,
        val recentOrders: List<Order> = emptyList(),
        val unreadCount: Int = 0
    )
    
    val state: StateFlow<DashboardState> = combine(
        userRepo.observeCurrentUser(),
        orderRepo.observeRecentOrders(limit = 5),
        notifRepo.observeUnreadCount()
    ) { user, orders, unreadCount ->
        DashboardState(user, orders, unreadCount)
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), DashboardState())
}

// ============================================
// Flow Retry & Error handling
// ============================================

fun fetchWithRetry(id: Long): Flow<Data> = flow {
    emit(apiService.getData(id))
}.retry(retries = 3) { cause ->
    cause is IOException  // retry เฉพาะ network errors
}.retryWhen { cause, attempt ->
    if (cause is IOException && attempt < 3) {
        delay(2000L * (attempt + 1))  // exponential backoff
        true
    } else {
        false
    }
}

// ============================================
// SharedFlow vs StateFlow
// ============================================

class EventBus {
    // SharedFlow - สำหรับ events (ไม่มี initial value, ไม่เก็บ state)
    private val _events = MutableSharedFlow<AppEvent>()
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()
    
    suspend fun emit(event: AppEvent) = _events.emit(event)
}

class AppViewModel @Inject constructor(
    private val eventBus: EventBus
) : ViewModel() {
    
    // StateFlow - สำหรับ state (มี initial value, เก็บ state ล่าสุด)
    private val _state = MutableStateFlow(AppState.initial())
    val state: StateFlow<AppState> = _state.asStateFlow()
    
    init {
        viewModelScope.launch {
            eventBus.events.collect { event ->
                when (event) {
                    is AppEvent.UserLoggedIn -> _state.update { it.copy(isLoggedIn = true) }
                    is AppEvent.UserLoggedOut -> _state.update { it.copy(isLoggedIn = false) }
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1079: Coroutines ใน Production

```kotlin
// ============================================
// Dispatcher Best Practices
// ============================================

class OrderProcessor @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
    @DefaultDispatcher private val defaultDispatcher: CoroutineDispatcher,
    @MainDispatcher private val mainDispatcher: CoroutineDispatcher
) {
    suspend fun processOrder(order: Order): ProcessedOrder {
        // Network/DB operations → IO
        val enriched = withContext(ioDispatcher) {
            fetchOrderDetails(order.id)
        }
        
        // CPU-intensive computation → Default
        val calculated = withContext(defaultDispatcher) {
            calculatePricing(enriched)
        }
        
        return calculated
    }
}

// ============================================
// Testing Coroutines
// ============================================

class SearchViewModelTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    private val fakeSearchRepository = FakeSearchRepository()
    private lateinit var viewModel: SearchViewModel
    
    @Before
    fun setup() {
        viewModel = SearchViewModel(fakeSearchRepository)
    }
    
    @Test
    fun `search results update after debounce`() = runTest {
        fakeSearchRepository.setResults("kotlin", listOf(
            SearchResult(1, "Kotlin Programming"),
            SearchResult(2, "Kotlin Coroutines")
        ))
        
        viewModel.onQueryChange("kotlin")
        
        // Fast-forward time past debounce
        advanceTimeBy(500)
        
        val results = viewModel.results.value
        assertEquals(2, results.size)
    }
}

// MainDispatcherRule
class MainDispatcherRule(
    private val testDispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    
    override fun finished(description: Description) {
        Dispatchers.resetMain()
        testDispatcher.cleanupTestCoroutines()
    }
}
```

---

## แบบฝึกหัด Part 64

```kotlin
// แบบฝึกหัด: Implement Download Manager ด้วย Coroutines

data class DownloadTask(
    val id: String,
    val url: String,
    val fileName: String
)

sealed class DownloadState {
    object Idle : DownloadState()
    data class Downloading(val progress: Float) : DownloadState()
    data class Success(val filePath: String) : DownloadState()
    data class Error(val message: String) : DownloadState()
}

class DownloadManager(
    private val httpClient: HttpClient,
    private val fileDir: File
) {
    
    // TODO: implement:
    // 1. downloadFile(task) - download พร้อม progress updates
    //    - ใช้ Flow<DownloadState> เพื่อ emit progress
    //    - support cancellation
    //    - retry ถ้า fail (3 ครั้ง)
    
    // 2. downloadMultiple(tasks) - download หลายไฟล์พร้อมกัน
    //    - จำกัดจำนวน concurrent downloads (ใช้ Semaphore)
    //    - collect progress ของแต่ละไฟล์
    
    fun downloadFile(task: DownloadTask): Flow<DownloadState> = flow {
        emit(DownloadState.Downloading(0f))
        // TODO: implement actual download
        TODO()
    }
    
    suspend fun downloadMultiple(
        tasks: List<DownloadTask>,
        maxConcurrent: Int = 3
    ): Map<String, DownloadState> {
        val semaphore = Semaphore(maxConcurrent)
        // TODO: implement parallel downloads with limit
        TODO()
    }
}
```

---

*Part 64 จบแล้ว | ก่อนหน้า: [Part 63](../part63/README.md) | ถัดไป: [Part 65](../part65/README.md)*
