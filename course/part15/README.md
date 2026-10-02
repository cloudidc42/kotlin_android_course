# Part 15: Advanced Coroutines
## ขั้นตอนที่ 331-360

---

## ขั้นตอนที่ 331: Channels

Channel เป็นวิธีส่ง data ระหว่าง coroutines

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

// Channel พื้นฐาน
suspend fun channelBasic() = coroutineScope {
    val channel = Channel<Int>()
    
    // Producer coroutine
    launch {
        for (x in 1..5) {
            println("Sending $x")
            channel.send(x)
        }
        channel.close()
    }
    
    // Consumer coroutine
    launch {
        for (value in channel) {
            println("Received $value")
        }
        println("Channel closed")
    }
}

// Buffered Channel
suspend fun bufferedChannel() = coroutineScope {
    val channel = Channel<Int>(capacity = 4)  // buffer size
    
    launch {
        for (x in 1..10) {
            channel.send(x)  // ไม่ block จนกว่า buffer เต็ย
            println("Sent $x")
        }
        channel.close()
    }
    
    delay(100)  // wait ให้ producer ส่งก่อน
    
    for (value in channel) {
        println("Got $value")
        delay(50)
    }
}

// Channel Types
// RENDEZVOUS (default 0) - ต้องมี receiver ก่อน sender ถึงจะผ่าน
// BUFFERED (64) - buffer ขนาดเล็ก
// UNLIMITED - buffer ไม่จำกัด (ระวัง OOM)
// CONFLATED - เก็บแค่ค่าล่าสุด

val conflatedChannel = Channel<Int>(Channel.CONFLATED)
```

---

## ขั้นตอนที่ 332: Produce และ Actor

```kotlin
// produce - สร้าง ReceiveChannel
fun CoroutineScope.produceNumbers(max: Int): ReceiveChannel<Int> = produce {
    for (i in 1..max) {
        send(i)
        delay(50)
    }
}

fun CoroutineScope.squareNumbers(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (n in numbers) {
        send(n * n)
    }
}

suspend fun pipelineExample() = coroutineScope {
    val numbers = produceNumbers(5)
    val squares = squareNumbers(numbers)
    
    for (sq in squares) {
        println(sq)  // 1, 4, 9, 16, 25
    }
}

// Fan-out: 1 producer, หลาย consumer
fun CoroutineScope.produceTasks(): ReceiveChannel<String> = produce {
    var i = 0
    while (true) {
        send("Task-${i++}")
        delay(100)
    }
}

suspend fun fanOut() = coroutineScope {
    val tasks = produceTasks()
    
    repeat(3) { id ->
        launch {
            for (task in tasks) {
                println("Worker $id processing $task")
                delay(200)
            }
        }
    }
    
    delay(1000)
    coroutineContext.cancelChildren()
}

// Fan-in: หลาย producer, 1 consumer
fun CoroutineScope.sendString(
    channel: SendChannel<String>,
    s: String,
    time: Long
) = launch {
    while (true) {
        delay(time)
        channel.send(s)
    }
}

suspend fun fanIn() = coroutineScope {
    val channel = Channel<String>()
    sendString(channel, "foo", 200L)
    sendString(channel, "BAR", 500L)
    
    repeat(6) {
        println(channel.receive())
    }
    coroutineContext.cancelChildren()
}
```

---

## ขั้นตอนที่ 333: select Expression

```kotlin
import kotlinx.coroutines.selects.select

// select - รอ coroutine ที่เสร็จก่อน
suspend fun selectExample() = coroutineScope {
    val channel1 = produce { delay(100); send("Channel 1") }
    val channel2 = produce { delay(50); send("Channel 2") }
    
    val result = select<String> {
        channel1.onReceive { it }
        channel2.onReceive { it }
    }
    
    println(result)  // Channel 2 (เร็วกว่า)
    coroutineContext.cancelChildren()
}

// Timeout ด้วย select
suspend fun withSelectTimeout(
    timeout: Long,
    block: suspend () -> String
): String {
    return select {
        async { block() }.onAwait { it }
        onTimeout(timeout) { "Timeout!" }
    }
}

suspend fun main() {
    val result = withSelectTimeout(200) {
        delay(100)
        "Done in time"
    }
    println(result)  // Done in time
    
    val timedOut = withSelectTimeout(50) {
        delay(200)
        "Too slow"
    }
    println(timedOut)  // Timeout!
}
```

---

## ขั้นตอนที่ 334: Mutex และ Semaphore

```kotlin
import kotlinx.coroutines.sync.*

// Mutex - mutual exclusion (lock/unlock)
class SafeCounter {
    private val mutex = Mutex()
    private var count = 0
    
    suspend fun increment() {
        mutex.withLock {
            count++
        }
    }
    
    fun getCount() = count
}

suspend fun mutexExample() {
    val counter = SafeCounter()
    
    coroutineScope {
        repeat(1000) {
            launch { counter.increment() }
        }
    }
    
    println("Count: ${counter.getCount()}")  // Count: 1000
}

// Semaphore - จำกัดจำนวน concurrent access
val semaphore = Semaphore(3)  // max 3 concurrent

suspend fun limitedConcurrency(id: Int) {
    semaphore.withPermit {
        println("Worker $id started")
        delay(200)
        println("Worker $id done")
    }
}

suspend fun semaphoreExample() = coroutineScope {
    repeat(10) { i ->
        launch { limitedConcurrency(i) }
    }
}
```

---

## ขั้นตอนที่ 335: Coroutine Context และ Elements

```kotlin
// CoroutineContext เป็น collection ของ CoroutineContext.Element
// Elements หลัก: Job, CoroutineDispatcher, CoroutineName, CoroutineExceptionHandler

suspend fun contextElements() = coroutineScope {
    // ดู context ปัจจุบัน
    println(coroutineContext)
    
    // CoroutineName
    launch(CoroutineName("my-coroutine")) {
        println("Name: ${coroutineContext[CoroutineName]?.name}")
    }
    
    // combine contexts ด้วย +
    val context = Dispatchers.IO + CoroutineName("io-task") + Job()
    
    launch(context) {
        println("Running in: ${coroutineContext[CoroutineName]?.name}")
        println("Dispatcher: ${coroutineContext[CoroutineDispatcher]}")
    }
}

// Custom CoroutineContext.Element
class RequestId(val value: String) : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<RequestId>
    override val key: CoroutineContext.Key<*> = Key
}

suspend fun processRequest(requestId: String) {
    withContext(RequestId(requestId)) {
        doWork()
    }
}

suspend fun doWork() {
    val requestId = coroutineContext[RequestId]?.value
    println("Processing request: $requestId")
    // log, trace, etc.
}
```

---

## ขั้นตอนที่ 336: Structured Concurrency Pattern

```kotlin
// Pattern ที่ดีสำหรับ Structured Concurrency

class DownloadManager(private val scope: CoroutineScope) {
    
    // Parallel downloads ด้วย structured concurrency
    suspend fun downloadAll(urls: List<String>): List<ByteArray> {
        return coroutineScope {
            urls.map { url ->
                async { download(url) }
            }.awaitAll()
        }
    }
    
    // Sequential with error handling
    suspend fun downloadSequential(urls: List<String>): List<Result<ByteArray>> {
        return urls.map { url ->
            try {
                Result.Success(download(url))
            } catch (e: Exception) {
                Result.Error(e)
            }
        }
    }
    
    // Race - เอาผลจาก coroutine ที่เร็วที่สุด
    suspend fun downloadFastest(urls: List<String>): ByteArray {
        return coroutineScope {
            val deferred = urls.map { url -> async { download(url) } }
            
            // cancel ที่เหลือเมื่อได้ผลแรก
            select {
                deferred.forEach { d ->
                    d.onAwait { result ->
                        deferred.forEach { it.cancel() }
                        result
                    }
                }
            }
        }
    }
    
    private suspend fun download(url: String): ByteArray {
        // simulate download
        delay(100)
        return ByteArray(1024)
    }
}

// CoroutineScope extension ที่ safe
fun CoroutineScope.launchSafely(
    context: CoroutineContext = EmptyCoroutineContext,
    block: suspend CoroutineScope.() -> Unit
): Job = launch(context) {
    try {
        block()
    } catch (e: CancellationException) {
        throw e  // rethrow CancellationException
    } catch (e: Exception) {
        // log error
        println("Error in coroutine: ${e.message}")
    }
}
```

---

## ขั้นตอนที่ 337: Testing Coroutines

```kotlin
import kotlinx.coroutines.test.*

// runTest - ทดสอบ coroutine โดยควบคุม time
class CoroutineTest {
    
    @Test
    fun testDelayedFunction() = runTest {
        // delay ใน runTest จะถูก skip ทันที (virtual time)
        val result = withTimeout(1000) {
            delay(500)  // skip immediately
            "Success"
        }
        assertEquals("Success", result)
    }
    
    @Test
    fun testFlow() = runTest {
        val flow = flow {
            emit(1)
            delay(100)
            emit(2)
            delay(100)
            emit(3)
        }
        
        val collected = flow.toList()
        assertEquals(listOf(1, 2, 3), collected)
    }
    
    // TestCoroutineDispatcher
    @Test
    fun testWithDispatcher() = runTest {
        val testDispatcher = StandardTestDispatcher(testScheduler)
        
        var result = ""
        val job = launch(testDispatcher) {
            delay(1000)
            result = "Done"
        }
        
        assertEquals("", result)    // ยังไม่ผ่าน
        advanceTimeBy(1001)          // เดิน virtual time
        assertEquals("Done", result) // ผ่านแล้ว
    }
    
    // ทดสอบ ViewModel
    @Test
    fun testViewModel() = runTest {
        val repository = FakeUserRepository()
        val viewModel = UserViewModel(repository)
        
        viewModel.loadUsers()
        
        // ต้องรอ coroutine เสร็จ
        advanceUntilIdle()
        
        val users = viewModel.users.value
        assertFalse(users.isEmpty())
    }
}
```

---

## ขั้นตอนที่ 338: Coroutine Anti-patterns

```kotlin
// ❌ Anti-pattern: GlobalScope
fun badExample() {
    GlobalScope.launch {
        // leak! ไม่มีวิธียกเลิก, ไม่ผูกกับ lifecycle
        delay(1000)
        updateUI()  // อาจ crash เพราะ activity ถูก destroy แล้ว
    }
}

// ✅ ถูกต้อง: ใช้ viewModelScope หรือ lifecycleScope
class GoodViewModel : ViewModel() {
    fun goodExample() {
        viewModelScope.launch {
            delay(1000)
            // ถ้า ViewModel ถูก destroy, coroutine จะถูก cancel
        }
    }
}

// ❌ Anti-pattern: runBlocking ใน coroutine
suspend fun badRunBlocking() {
    runBlocking {  // blocks thread! อาจทำให้ deadlock
        delay(1000)
    }
}

// ✅ ถูกต้อง: ใช้ coroutineScope หรือ withContext
suspend fun goodSuspend() {
    coroutineScope {
        delay(1000)
    }
}

// ❌ Anti-pattern: catch Exception แล้วไม่ throw CancellationException
suspend fun badCatch() {
    try {
        delay(1000)
    } catch (e: Exception) {
        // ❌ จะ catch CancellationException ด้วย!
        println("Error: ${e.message}")
    }
}

// ✅ ถูกต้อง
suspend fun goodCatch() {
    try {
        delay(1000)
    } catch (e: CancellationException) {
        throw e  // rethrow CancellationException เสมอ!
    } catch (e: Exception) {
        println("Error: ${e.message}")
    }
}

// ❌ Anti-pattern: ไม่ใช้ withContext ตอนเปลี่ยน dispatcher
class BadRepository {
    suspend fun getData(): String {
        // ไม่ระบุ dispatcher, ทำงานบน caller's dispatcher
        return File("large.txt").readText()  // blocking IO!
    }
}

// ✅ ถูกต้อง
class GoodRepository {
    suspend fun getData(): String = withContext(Dispatchers.IO) {
        File("large.txt").readText()
    }
}
```

---

## ขั้นตอนที่ 339: Advanced Flow Patterns

```kotlin
// Retry with exponential backoff
fun <T> Flow<T>.retryWithBackoff(
    times: Int = 3,
    initialDelay: Long = 1000,
    maxDelay: Long = 30000,
    factor: Double = 2.0
): Flow<T> = retryWhen { cause, attempt ->
    if (attempt < times && cause !is CancellationException) {
        val delayTime = minOf(initialDelay * factor.pow(attempt.toInt()).toLong(), maxDelay)
        println("Retry attempt ${attempt + 1} after ${delayTime}ms")
        delay(delayTime)
        true
    } else false
}

// Safe collect
suspend fun <T> Flow<T>.safeCollect(
    onError: (Throwable) -> Unit = {},
    onComplete: () -> Unit = {},
    collector: suspend (T) -> Unit
) {
    catch { e -> onError(e) }
        .onCompletion { onComplete() }
        .collect(collector)
}

// Throttle
fun <T> Flow<T>.throttleFirst(windowDuration: Long): Flow<T> = flow {
    var lastEmitTime = 0L
    collect { value ->
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastEmitTime >= windowDuration) {
            lastEmitTime = currentTime
            emit(value)
        }
    }
}

// Distinct until changed with custom equality
fun <T, K> Flow<T>.distinctUntilChangedBy(selector: (T) -> K): Flow<T> =
    distinctUntilChanged { old, new -> selector(old) == selector(new) }

// Combine latest from multiple flows
fun <A, B, C, R> combineLatest(
    flow1: Flow<A>,
    flow2: Flow<B>,
    flow3: Flow<C>,
    transform: (A, B, C) -> R
): Flow<R> = combine(flow1, flow2, flow3, transform)

// ใช้งาน
data class UiState(
    val loading: Boolean = false,
    val query: String = "",
    val filter: String = "all"
)

val loading = MutableStateFlow(false)
val query = MutableStateFlow("")
val filter = MutableStateFlow("all")

val uiState = combineLatest(loading, query, filter) { l, q, f ->
    UiState(loading = l, query = q, filter = f)
}
```

---

## ขั้นตอนที่ 340: Coroutines ใน Production

```kotlin
// ViewModel pattern ที่สมบูรณ์
@HiltViewModel
class ProductionViewModel @Inject constructor(
    private val repository: DataRepository,
    private val analyticsService: AnalyticsService,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) : ViewModel() {
    
    private val _state = MutableStateFlow(ScreenState.Loading)
    val state: StateFlow<ScreenState> = _state.asStateFlow()
    
    private val _events = Channel<UiEvent>(Channel.BUFFERED)
    val events: Flow<UiEvent> = _events.receiveAsFlow()
    
    init {
        loadInitialData()
    }
    
    private fun loadInitialData() {
        viewModelScope.launch {
            try {
                _state.value = ScreenState.Loading
                val data = withContext(ioDispatcher) {
                    repository.getInitialData()
                }
                _state.value = ScreenState.Success(data)
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                _state.value = ScreenState.Error(e.message ?: "Unknown error")
                analyticsService.logError(e)
            }
        }
    }
    
    fun refresh() {
        viewModelScope.launch {
            if (_state.value is ScreenState.Loading) return@launch
            loadInitialData()
        }
    }
    
    fun sendEvent(event: UiEvent) {
        viewModelScope.launch {
            _events.send(event)
        }
    }
}

sealed class ScreenState {
    object Loading : ScreenState()
    data class Success(val data: Any) : ScreenState()
    data class Error(val message: String) : ScreenState()
}

sealed class UiEvent {
    data class ShowMessage(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
    object Refresh : UiEvent()
}
```

---

## แบบฝึกหัด Part 15

```kotlin
// แบบฝึกหัด: Implement a simple worker pool using channels

class WorkerPool<T, R>(
    private val numWorkers: Int,
    private val worker: suspend (T) -> R
) {
    suspend fun processAll(items: List<T>): List<R> = coroutineScope {
        val inputChannel = Channel<T>(Channel.UNLIMITED)
        val outputChannel = Channel<R>(Channel.UNLIMITED)
        
        // Send all items
        items.forEach { inputChannel.send(it) }
        inputChannel.close()
        
        // Launch workers
        val workers = List(numWorkers) {
            launch {
                for (item in inputChannel) {
                    outputChannel.send(worker(item))
                }
            }
        }
        
        // Wait for all workers, then close output
        launch {
            workers.forEach { it.join() }
            outputChannel.close()
        }
        
        // Collect results
        outputChannel.toList()
    }
}

// ทดสอบ
suspend fun main() {
    val pool = WorkerPool<Int, Int>(numWorkers = 4) { num ->
        delay(100)  // simulate work
        num * num
    }
    
    val results = pool.processAll((1..20).toList())
    println(results.sorted())  // [1, 4, 9, 16, 25, ...]
}
```

---

*Part 15 จบแล้ว | ก่อนหน้า: [Part 14](../part14/README.md) | ถัดไป: [Part 16](../part16/README.md)*
