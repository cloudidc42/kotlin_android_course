# Part 14: Coroutines พื้นฐาน
## ขั้นตอนที่ 301-330

---

## ขั้นตอนที่ 301: Coroutines คืออะไร?

```kotlin
// ============================================
// ปัญหาของ Threading แบบเดิม
// ============================================

import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

// ============================================
// Threading แบบเดิม - ปัญหามากมาย
// ============================================

fun oldStyleAsync() {
    // Thread ใช้ทรัพยากรมาก
    // Thread(Runnable {
    //     val data = fetchData()  // blocking!
    //     runOnUiThread { updateUI(data) }  // callback hell
    // }).start()
    
    // Callback Hell
    // fetchUser(userId) { user ->
    //     fetchPosts(user.id) { posts ->
    //         fetchComments(posts.first().id) { comments ->
    //             // ลึกมาก อ่านยาก ดูแลยาก
    //         }
    //     }
    // }
}

// ============================================
// Coroutines - แก้ปัญหาด้วยโค้ดแบบ synchronous
// ============================================

suspend fun fetchUser(id: Int): String {
    delay(100)  // จำลองการ delay โดยไม่ block thread
    return "User#$id"
}

suspend fun fetchPosts(userId: String): List<String> {
    delay(150)
    return listOf("Post1 of $userId", "Post2 of $userId")
}

suspend fun fetchComments(postId: String): List<String> {
    delay(50)
    return listOf("Comment1 on $postId", "Comment2 on $postId")
}

fun main() = runBlocking {
    // ============================================
    // Sequential (ทำทีละอย่าง)
    // ============================================
    
    val seqTime = measureTimeMillis {
        val user = fetchUser(1)
        val posts = fetchPosts(user)
        val comments = fetchComments(posts.first())
        
        println("Sequential result:")
        println("User: $user")
        println("Posts: $posts")
        println("Comments: $comments")
    }
    println("Sequential time: ${seqTime}ms\n")  // ~300ms
    
    // ============================================
    // Coroutines แบบ Sequential แต่อ่านง่าย!
    // ============================================
    
    println("Coroutines ดูเหมือน synchronous แต่ non-blocking:")
    val user = fetchUser(1)
    val posts = fetchPosts(user)
    println("user=$user, posts=$posts")
}
```

---

## ขั้นตอนที่ 302: Coroutine Builders

```kotlin
import kotlinx.coroutines.*

fun main() {
    // ============================================
    // 1. runBlocking - ใช้ใน main() หรือ test
    // Block thread จนกว่า coroutine จะเสร็จ
    // ============================================
    
    println("1. runBlocking:")
    runBlocking {
        println("   ใน runBlocking")
        delay(100)
        println("   หลัง delay 100ms")
    }
    println("   หลัง runBlocking")
    
    // ============================================
    // 2. launch - Fire and forget
    // ไม่ return ค่า, return Job
    // ============================================
    
    println("\n2. launch:")
    runBlocking {
        val job = launch {
            delay(200)
            println("   launch เสร็จแล้ว")
        }
        println("   launch ถูกเรียก (non-blocking)")
        println("   กำลังทำงานอื่น...")
        job.join()  // รอให้ job เสร็จ
        println("   หลัง join()")
    }
    
    // ============================================
    // 3. async - Deferred result
    // Return Deferred<T>, ต้องเรียก .await() เพื่อรับค่า
    // ============================================
    
    println("\n3. async/await:")
    runBlocking {
        val deferred = async {
            delay(200)
            "ผลลัพธ์จาก async"
        }
        println("   async ถูกเรียก")
        val result = deferred.await()  // รอและรับค่า
        println("   result = $result")
    }
    
    // ============================================
    // Parallel execution ด้วย async
    // ============================================
    
    println("\n4. Parallel with async:")
    runBlocking {
        val time = measureTimeMillis {
            // Sequential: 300ms
            // val a = async { delay(100); "A" }.await()
            // val b = async { delay(200); "B" }.await()
            
            // Parallel: 200ms (ทำพร้อมกัน!)
            val deferredA = async { delay(100); "A" }
            val deferredB = async { delay(200); "B" }
            
            val a = deferredA.await()
            val b = deferredB.await()
            println("   a=$a, b=$b")
        }
        println("   เวลา: ${time}ms (parallel = max(100,200) = ~200ms)")
    }
    
    // ============================================
    // coroutineScope - ไม่ block แต่รอ children
    // ============================================
    
    println("\n5. coroutineScope:")
    runBlocking {
        coroutineScope {
            launch { delay(100); println("   child 1") }
            launch { delay(50);  println("   child 2") }
            println("   parent กำลังรอ children...")
        }
        println("   ทุก children เสร็จแล้ว")
    }
}
```

---

## ขั้นตอนที่ 303: Coroutine Context และ Dispatchers

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // ============================================
    // Dispatchers - Thread Pool สำหรับ Coroutines
    // ============================================
    
    // Dispatchers.Default - CPU-intensive tasks
    // Optimized for computation (CPU count threads)
    launch(Dispatchers.Default) {
        println("Default: ${Thread.currentThread().name}")
        // เหมาะกับ: sorting, parsing, calculations
        val result = (1..1_000_000).sum()
        println("Sum: $result")
    }
    
    // Dispatchers.IO - I/O operations
    // Large pool for blocking I/O (64+ threads by default)
    launch(Dispatchers.IO) {
        println("IO: ${Thread.currentThread().name}")
        // เหมาะกับ: file I/O, network, database
        delay(100)  // จำลอง I/O
        println("IO done")
    }
    
    // Dispatchers.Main - UI thread (Android)
    // ใน Pure Kotlin ไม่มี Main dispatcher
    // launch(Dispatchers.Main) { ... }
    
    // Dispatchers.Unconfined - ไม่จำกัด thread
    launch(Dispatchers.Unconfined) {
        println("Unconfined start: ${Thread.currentThread().name}")
        delay(100)
        // หลัง suspend point อาจเปลี่ยน thread!
        println("Unconfined resume: ${Thread.currentThread().name}")
    }
    
    delay(500)
    
    // ============================================
    // withContext - เปลี่ยน dispatcher
    // ============================================
    
    println("\nwithContext:")
    
    suspend fun processData(): String {
        return withContext(Dispatchers.Default) {
            println("Processing on: ${Thread.currentThread().name}")
            "processed data"
        }
    }
    
    suspend fun saveData(data: String) {
        withContext(Dispatchers.IO) {
            println("Saving on: ${Thread.currentThread().name}")
            delay(50)  // simulate save
        }
    }
    
    val data = processData()
    saveData(data)
    println("Main thread: ${Thread.currentThread().name}")
    
    // ============================================
    // CoroutineContext
    // ============================================
    
    println("\nCoroutineContext:")
    
    launch {
        println("Job: ${coroutineContext[Job]}")
        println("Dispatcher: ${coroutineContext[ContinuationInterceptor]}")
    }
    
    delay(100)
}
```

---

## ขั้นตอนที่ 304: Job และ Lifecycle

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // ============================================
    // Job States
    // Active -> Completing -> Completed
    //                     -> Cancelling -> Cancelled
    //          -> Cancelling -> Cancelled
    // ============================================
    
    val job = launch {
        println("Job started, isActive: ${coroutineContext[Job]?.isActive}")
        delay(500)
        println("Job completing")
    }
    
    println("After launch:")
    println("  isActive: ${job.isActive}")
    println("  isCompleted: ${job.isCompleted}")
    println("  isCancelled: ${job.isCancelled}")
    
    delay(100)
    
    // ============================================
    // Cancellation
    // ============================================
    
    println("\nCancellation:")
    
    val cancellableJob = launch {
        try {
            repeat(10) { i ->
                println("  Working... $i")
                delay(200)
            }
        } catch (e: CancellationException) {
            println("  Job cancelled!")
            // cleanup here
        } finally {
            println("  Cleanup in finally")
        }
    }
    
    delay(500)
    cancellableJob.cancel("Manual cancellation")
    
    println("After cancel:")
    println("  isActive: ${cancellableJob.isActive}")
    println("  isCancelled: ${cancellableJob.isCancelled}")
    
    // ============================================
    // cancelAndJoin
    // ============================================
    
    println("\ncancelAndJoin:")
    val job2 = launch {
        delay(1000)
        println("  This won't print")
    }
    
    delay(100)
    job2.cancelAndJoin()  // cancel + join in one call
    println("  Job2 cancelled and joined")
    
    // ============================================
    // Cooperative Cancellation
    // ============================================
    
    println("\nCooperative Cancellation:")
    
    val job3 = launch(Dispatchers.Default) {
        var i = 0
        while (isActive) {  // ตรวจสอบ isActive เอง
            i++
            // ไม่มี suspend function - ต้องตรวจ isActive
        }
        println("  Counted to $i")
    }
    
    delay(200)
    job3.cancelAndJoin()
    println("  Computation cancelled")
    
    // ============================================
    // Timeout
    // ============================================
    
    println("\nTimeout:")
    
    try {
        withTimeout(300) {
            println("  Starting long operation")
            delay(1000)  // จะ timeout ก่อน
            println("  This won't print")
        }
    } catch (e: TimeoutCancellationException) {
        println("  Timeout!")
    }
    
    // withTimeoutOrNull - ไม่ throw exception
    val result = withTimeoutOrNull(300) {
        delay(1000)
        "ผลลัพธ์"
    }
    println("  withTimeoutOrNull: $result")  // null (timeout)
    
    val result2 = withTimeoutOrNull(1000) {
        delay(200)
        "ผลลัพธ์"
    }
    println("  withTimeoutOrNull: $result2")  // ผลลัพธ์
}
```

---

## ขั้นตอนที่ 305: Coroutine Exception Handling

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // ============================================
    // Exception Propagation
    // ============================================
    
    println("=== Exception Propagation ===")
    
    // launch - exception propagates to parent
    val job1 = launch {
        try {
            throw RuntimeException("Error in launch")
        } catch (e: RuntimeException) {
            println("Caught in launch: ${e.message}")
        }
    }
    job1.join()
    
    // async - exception stored in Deferred, thrown on await
    val deferred = async {
        throw RuntimeException("Error in async")
        "result"
    }
    
    try {
        deferred.await()
    } catch (e: RuntimeException) {
        println("Caught from async.await(): ${e.message}")
    }
    
    // ============================================
    // CoroutineExceptionHandler
    // ============================================
    
    println("\n=== CoroutineExceptionHandler ===")
    
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Handler caught: ${exception.message}")
    }
    
    val scope = CoroutineScope(Dispatchers.Default + handler)
    
    val job = scope.launch {
        throw RuntimeException("Unhandled exception")
    }
    
    job.join()
    
    // ============================================
    // SupervisorJob - ลูก fail ไม่กระทบพ่อ
    // ============================================
    
    println("\n=== SupervisorJob ===")
    
    // ปกติ - ถ้าลูก fail พ่อ cancel ลูกทุกคน
    // SupervisorJob - ถ้าลูก fail ลูกคนอื่นไม่ได้รับผลกระทบ
    
    val supervisor = SupervisorJob()
    val supervisorScope = CoroutineScope(Dispatchers.Default + supervisor + handler)
    
    val child1 = supervisorScope.launch {
        println("Child1: started")
        delay(100)
        throw RuntimeException("Child1 failed!")
    }
    
    val child2 = supervisorScope.launch {
        println("Child2: started")
        delay(500)
        println("Child2: completed successfully")
    }
    
    joinAll(child1, child2)
    println("Both children done")
    
    // ============================================
    // supervisorScope { }
    // ============================================
    
    println("\n=== supervisorScope block ===")
    
    try {
        supervisorScope {
            val a = launch {
                delay(100)
                throw RuntimeException("A failed")
            }
            
            val b = launch {
                delay(500)
                println("B completed")
            }
            
            a.join()  // throws exception
        }
    } catch (e: RuntimeException) {
        println("supervisorScope caught: ${e.message}")
    }
}
```

---

## ขั้นตอนที่ 306: Structured Concurrency

```kotlin
import kotlinx.coroutines.*

// ============================================
// Structured Concurrency
// - Coroutines เป็น hierarchy
// - Parent รอ children ทุกตัว
// - ถ้า parent cancel, children cancel ด้วย
// - ถ้า child fail, parent fail ด้วย (ยกเว้น supervisor)
// ============================================

class UserRepository {
    suspend fun fetchUser(id: Int): String {
        delay(100)
        return "User#$id"
    }
    
    suspend fun fetchUserPosts(userId: String): List<String> {
        delay(150)
        return listOf("Post1 by $userId", "Post2 by $userId")
    }
    
    suspend fun fetchUserProfile(userId: String): Map<String, String> {
        delay(200)
        return mapOf("bio" to "Hello from $userId", "avatar" to "avatar_url")
    }
}

class NewsRepository {
    suspend fun fetchLatestNews(): List<String> {
        delay(300)
        return listOf("News1", "News2", "News3")
    }
}

fun main() = runBlocking {
    val userRepo = UserRepository()
    val newsRepo = NewsRepository()
    
    // ============================================
    // Sequential (ช้า)
    // ============================================
    
    println("Sequential:")
    val time1 = measureTimeMillis {
        val user = userRepo.fetchUser(1)
        val posts = userRepo.fetchUserPosts(user)
        val profile = userRepo.fetchUserProfile(user)
        val news = newsRepo.fetchLatestNews()
        
        println("User: $user")
        println("Posts: $posts")
        println("Profile: $profile")
        println("News: $news")
    }
    println("Time: ${time1}ms (~750ms)\n")
    
    // ============================================
    // Parallel (เร็วกว่า)
    // ============================================
    
    println("Parallel:")
    val time2 = measureTimeMillis {
        val user = userRepo.fetchUser(1)  // ต้องรู้ user ก่อน
        
        // หลังจากรู้ user แล้ว เรียกพร้อมกัน
        coroutineScope {
            val postsDeferred = async { userRepo.fetchUserPosts(user) }
            val profileDeferred = async { userRepo.fetchUserProfile(user) }
            val newsDeferred = async { newsRepo.fetchLatestNews() }
            
            val posts = postsDeferred.await()
            val profile = profileDeferred.await()
            val news = newsDeferred.await()
            
            println("User: $user")
            println("Posts: $posts")
            println("Profile: $profile")
            println("News: $news")
        }
    }
    println("Time: ${time2}ms (~300ms = max of parallel tasks)\n")
    
    // ============================================
    // awaitAll - รอทุก Deferred พร้อมกัน
    // ============================================
    
    println("awaitAll:")
    val time3 = measureTimeMillis {
        val results = coroutineScope {
            val deferreds = (1..5).map { id ->
                async { userRepo.fetchUser(id) }
            }
            deferreds.awaitAll()  // รอทั้งหมดพร้อมกัน
        }
        println("Users: $results")
    }
    println("Time: ${time3}ms (~100ms)")
}
```

---

## ขั้นตอนที่ 307: Coroutines ใน Android

```kotlin
// ============================================
// การใช้ Coroutines ใน Android
// ============================================

// build.gradle.kts dependencies:
// implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.0")
// implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0")

/*
// ViewModel ด้วย viewModelScope
class UserViewModel(private val userRepository: UserRepository) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState<User>>(UiState.Idle)
    val uiState: StateFlow<UiState<User>> = _uiState.asStateFlow()
    
    fun loadUser(userId: Int) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            
            try {
                val user = userRepository.fetchUser(userId)  // suspend function
                _uiState.value = UiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun loadMultipleData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            
            try {
                coroutineScope {
                    val userDeferred = async { userRepository.fetchUser(1) }
                    val postsDeferred = async { postsRepository.fetchPosts() }
                    
                    val user = userDeferred.await()
                    val posts = postsDeferred.await()
                    
                    _uiState.value = UiState.Success(UserWithPosts(user, posts))
                }
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}

// Fragment/Activity
class UserFragment : Fragment() {
    
    private val viewModel: UserViewModel by viewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // Collect state
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UiState.Idle -> {}
                        is UiState.Loading -> showLoading()
                        is UiState.Success -> showUser(state.data)
                        is UiState.Error -> showError(state.message)
                    }
                }
            }
        }
        
        viewModel.loadUser(1)
    }
}
*/

// Pure Kotlin simulation
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

data class User(val id: Int, val name: String, val email: String)

sealed class UiState<out T> {
    object Idle : UiState<Nothing>()
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

class UserViewModel {
    private val _uiState = MutableStateFlow<UiState<User>>(UiState.Idle)
    val uiState: StateFlow<UiState<User>> = _uiState.asStateFlow()
    
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    fun loadUser(userId: Int) {
        scope.launch {
            _uiState.value = UiState.Loading
            delay(200)  // simulate network
            
            try {
                val user = User(userId, "User#$userId", "user$userId@example.com")
                _uiState.value = UiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun clear() { scope.cancel() }
}

fun main() = runBlocking {
    val viewModel = UserViewModel()
    
    // Observe state
    val observeJob = launch {
        viewModel.uiState.collect { state ->
            println("State: $state")
        }
    }
    
    viewModel.loadUser(1)
    delay(500)
    
    observeJob.cancel()
    viewModel.clear()
}
```

---

## แบบฝึกหัด Part 14

```kotlin
import kotlinx.coroutines.*

// แบบฝึกหัดที่ 1: Parallel Fetch
// สร้าง suspend function ที่ fetch ข้อมูล 3 อย่างพร้อมกัน
// แล้ว combine ผลลัพธ์

suspend fun fetchWeather(): String {
    delay(300)
    return "อากาศดี 28°C"
}

suspend fun fetchNews(): List<String> {
    delay(400)
    return listOf("ข่าว 1", "ข่าว 2", "ข่าว 3")
}

suspend fun fetchStocks(): Map<String, Double> {
    delay(200)
    return mapOf("AAPL" to 150.0, "GOOGL" to 2800.0)
}

// TODO: สร้าง fetchDashboard() ที่ fetch ทั้งสามพร้อมกัน
suspend fun fetchDashboard(): Triple<String, List<String>, Map<String, Double>> {
    return coroutineScope {
        // TODO: ใช้ async เพื่อ fetch พร้อมกัน
        TODO()
    }
}

// แบบฝึกหัดที่ 2: Retry Logic
suspend fun <T> retry(
    maxAttempts: Int = 3,
    delayMs: Long = 1000,
    block: suspend () -> T
): T {
    // TODO: ลอง block() ซ้ำ maxAttempts ครั้ง ถ้า fail delay แล้วลองใหม่
    TODO()
}

fun main() = runBlocking {
    val (weather, news, stocks) = fetchDashboard()
    println("Weather: $weather")
    println("News: $news")
    println("Stocks: $stocks")
    
    // Test retry
    var attempt = 0
    val result = retry(maxAttempts = 3, delayMs = 100) {
        attempt++
        if (attempt < 3) throw RuntimeException("Attempt $attempt failed")
        "Success on attempt $attempt"
    }
    println(result)
}
```

### เฉลย

```kotlin
suspend fun fetchDashboard(): Triple<String, List<String>, Map<String, Double>> {
    return coroutineScope {
        val weatherDeferred = async { fetchWeather() }
        val newsDeferred = async { fetchNews() }
        val stocksDeferred = async { fetchStocks() }
        
        Triple(weatherDeferred.await(), newsDeferred.await(), stocksDeferred.await())
    }
}

suspend fun <T> retry(maxAttempts: Int = 3, delayMs: Long = 1000, block: suspend () -> T): T {
    repeat(maxAttempts - 1) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            println("Attempt ${attempt + 1} failed: ${e.message}")
            delay(delayMs)
        }
    }
    return block()  // last attempt - let it throw
}
```

---

*Part 14 จบแล้ว | ก่อนหน้า: [Part 13](../part13/README.md) | ถัดไป: [Part 15](../part15/README.md)*
