# Part 16: Flow และ StateFlow
## ขั้นตอนที่ 361-390

---

## ขั้นตอนที่ 361: Flow คืออะไร?

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// ============================================
// Flow - Asynchronous Stream of Values
// เหมือน Sequence แต่ async
// ============================================

// Simple Flow
fun numberFlow(): Flow<Int> = flow {
    println("Flow เริ่มต้น")
    for (i in 1..5) {
        delay(100)
        println("Emit: $i")
        emit(i)  // ส่งค่าออกไป
    }
    println("Flow เสร็จสิ้น")
}

// Flow แบบ cold - เริ่มทำงานเมื่อ collect
fun main() = runBlocking {
    println("=== Cold Flow ===")
    
    val flow = numberFlow()
    
    println("ก่อน collect - Flow ยังไม่ทำงาน")
    
    // Collect #1
    println("\nCollect #1:")
    flow.collect { value ->
        println("Received: $value")
    }
    
    // Collect #2 - เริ่มใหม่ตั้งแต่ต้น!
    println("\nCollect #2:")
    flow.collect { value ->
        println("Received: $value")
    }
    
    // ============================================
    // Flow Builders
    // ============================================
    
    println("\n=== Flow Builders ===")
    
    // flowOf - จาก values
    flowOf(1, 2, 3).collect { print("$it ") }
    println()
    
    // asFlow - จาก Collection
    listOf("A", "B", "C").asFlow().collect { print("$it ") }
    println()
    
    // range as flow
    (1..5).asFlow().collect { print("$it ") }
    println()
}
```

---

## ขั้นตอนที่ 362: Flow Operators

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val numbers = (1..10).asFlow()
    
    // ============================================
    // Intermediate Operators (Lazy - ไม่ทำงานจนกว่า collect)
    // ============================================
    
    println("=== map ===")
    numbers.map { it * it }
           .collect { print("$it ") }
    println()
    
    println("=== filter ===")
    numbers.filter { it % 2 == 0 }
           .collect { print("$it ") }
    println()
    
    println("=== map + filter ===")
    numbers.filter { it % 2 == 0 }
           .map { it * 10 }
           .collect { print("$it ") }
    println()
    
    println("=== take ===")
    numbers.take(3)
           .collect { print("$it ") }
    println()
    
    println("=== drop ===")
    numbers.drop(7)
           .collect { print("$it ") }
    println()
    
    println("=== transform ===")
    numbers.transform { n ->
        emit("[$n]")
        if (n % 3 == 0) emit(" <-- divisible by 3")
    }.collect { print(it) }
    println()
    
    println("=== flatMapConcat ===")
    (1..3).asFlow()
          .flatMapConcat { n ->
              flowOf("$n-A", "$n-B")
          }
          .collect { print("$it ") }
    println()
    
    println("=== zip ===")
    val letters = flowOf("A", "B", "C")
    val nums = flowOf(1, 2, 3)
    
    letters.zip(nums) { letter, num -> "$letter$num" }
           .collect { print("$it ") }
    println()
    
    println("=== combine ===")
    // combine - เมื่อไหนก็ตามที่ใด flow emit ค่าใหม่
    val flow1 = flow {
        delay(100); emit("A1")
        delay(300); emit("A2")
    }
    val flow2 = flow {
        delay(200); emit("B1")
        delay(100); emit("B2")
    }
    
    flow1.combine(flow2) { a, b -> "$a + $b" }
         .collect { println("  combined: $it") }
    
    println("\n=== Terminal Operators ===")
    
    val nums2 = (1..5).asFlow()
    
    println("toList: ${nums2.toList()}")
    println("first: ${nums2.first()}")
    println("firstOrNull: ${nums2.firstOrNull { it > 3 }}")
    println("last: ${nums2.last()}")
    println("count: ${nums2.count()}")
    println("sum: ${nums2.map { it.toLong() }.reduce { acc, n -> acc + n }}")
    println("fold: ${nums2.fold(0) { acc, n -> acc + n }}")
    println("any even: ${nums2.any { it % 2 == 0 }}")
    println("all positive: ${nums2.all { it > 0 }}")
}
```

---

## ขั้นตอนที่ 363: StateFlow และ SharedFlow

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// ============================================
// StateFlow - Hot Flow ที่มีค่า state ปัจจุบัน
// ใช้สำหรับ UI State
// ============================================

class CounterViewModel {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
    
    fun increment() { _count.value++ }
    fun decrement() { _count.value-- }
    fun reset() { _count.value = 0 }
    
    fun incrementBy(n: Int) {
        _count.update { currentValue -> currentValue + n }
    }
}

// ============================================
// SharedFlow - Hot Flow สำหรับ Events
// ไม่มี initial value, สามารถ replay ได้
// ============================================

class EventBus {
    private val _events = MutableSharedFlow<String>(
        replay = 0,           // ไม่ replay event เก่า
        extraBufferCapacity = 100  // buffer สำหรับ slow collectors
    )
    val events: SharedFlow<String> = _events.asSharedFlow()
    
    suspend fun emit(event: String) {
        _events.emit(event)
    }
    
    fun tryEmit(event: String): Boolean {
        return _events.tryEmit(event)  // non-suspend version
    }
}

fun main() = runBlocking {
    // ============================================
    // StateFlow Demo
    // ============================================
    
    println("=== StateFlow ===")
    
    val viewModel = CounterViewModel()
    
    // Collector
    val collectJob = launch {
        viewModel.count.collect { count ->
            println("Count: $count")
        }
    }
    
    delay(100)
    viewModel.increment()
    delay(100)
    viewModel.increment()
    delay(100)
    viewModel.increment()
    delay(100)
    viewModel.decrement()
    delay(100)
    viewModel.reset()
    delay(100)
    
    collectJob.cancel()
    
    // StateFlow ใช้สำหรับ UI State
    println("\n=== StateFlow for UI State ===")
    
    data class User(val id: Int, val name: String)
    
    sealed class UiState<out T> {
        object Loading : UiState<Nothing>()
        data class Success<T>(val data: T) : UiState<T>()
        data class Error(val message: String) : UiState<Nothing>()
    }
    
    class UserViewModel {
        private val _state = MutableStateFlow<UiState<User>>(UiState.Loading)
        val state: StateFlow<UiState<User>> = _state.asStateFlow()
        
        suspend fun loadUser(id: Int) {
            _state.value = UiState.Loading
            delay(200)
            _state.value = UiState.Success(User(id, "User#$id"))
        }
    }
    
    val userVM = UserViewModel()
    
    val stateJob = launch {
        userVM.state.collect { state ->
            when (state) {
                is UiState.Loading -> println("Loading...")
                is UiState.Success -> println("Success: ${state.data}")
                is UiState.Error -> println("Error: ${state.message}")
            }
        }
    }
    
    userVM.loadUser(42)
    delay(500)
    stateJob.cancel()
    
    // ============================================
    // SharedFlow Demo
    // ============================================
    
    println("\n=== SharedFlow ===")
    
    val eventBus = EventBus()
    
    // Multiple collectors
    val collector1 = launch {
        eventBus.events.collect { event ->
            println("Collector1: $event")
        }
    }
    
    val collector2 = launch {
        eventBus.events.collect { event ->
            println("Collector2: $event")
        }
    }
    
    delay(100)
    
    launch {
        eventBus.emit("Login event")
        delay(100)
        eventBus.emit("Page view: Home")
        delay(100)
        eventBus.emit("Purchase: Item123")
    }
    
    delay(500)
    collector1.cancel()
    collector2.cancel()
}
```

---

## ขั้นตอนที่ 364: Flow ใน Android (ViewModel + Compose)

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// ============================================
// Pattern: Repository + ViewModel + Flow
// ============================================

// Domain Model
data class Post(
    val id: Int,
    val title: String,
    val content: String,
    val authorId: Int,
    val likes: Int = 0,
    val createdAt: Long = System.currentTimeMillis()
)

// Repository
class PostRepository {
    private val posts = mutableListOf(
        Post(1, "Hello Kotlin", "เรียน Kotlin มาก", 1),
        Post(2, "Coroutines", "เรียน Coroutines", 1),
        Post(3, "Android Dev", "พัฒนา Android", 2),
        Post(4, "Compose UI", "Jetpack Compose", 2)
    )
    
    private val _postsFlow = MutableStateFlow(posts.toList())
    
    fun observePosts(): Flow<List<Post>> = _postsFlow.asStateFlow()
    
    fun observePostsByAuthor(authorId: Int): Flow<List<Post>> = 
        _postsFlow.map { posts -> posts.filter { it.authorId == authorId } }
    
    suspend fun addPost(post: Post) {
        delay(100)  // simulate network
        posts.add(post)
        _postsFlow.value = posts.toList()
    }
    
    suspend fun likePost(postId: Int) {
        delay(50)
        val index = posts.indexOfFirst { it.id == postId }
        if (index >= 0) {
            posts[index] = posts[index].copy(likes = posts[index].likes + 1)
            _postsFlow.value = posts.toList()
        }
    }
    
    suspend fun deletePost(postId: Int) {
        delay(100)
        posts.removeIf { it.id == postId }
        _postsFlow.value = posts.toList()
    }
    
    // Search
    fun searchPosts(query: String): Flow<List<Post>> = 
        _postsFlow.map { posts ->
            if (query.isBlank()) posts
            else posts.filter { it.title.contains(query, ignoreCase = true) }
        }
}

// ViewModel
class PostViewModel(private val repository: PostRepository) {
    private val _searchQuery = MutableStateFlow("")
    
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    val filteredPosts: Flow<List<Post>> = _searchQuery
        .debounce(300)  // รอ 300ms หลัง user หยุดพิมพ์
        .flatMapLatest { query ->
            repository.searchPosts(query)
        }
    
    fun search(query: String) { _searchQuery.value = query }
    
    suspend fun addPost(title: String, content: String) {
        val newPost = Post(
            id = (Math.random() * 1000).toInt(),
            title = title,
            content = content,
            authorId = 1
        )
        repository.addPost(newPost)
    }
    
    suspend fun likePost(postId: Int) { repository.likePost(postId) }
    suspend fun deletePost(postId: Int) { repository.deletePost(postId) }
}

fun main() = runBlocking {
    val repository = PostRepository()
    val viewModel = PostViewModel(repository)
    
    // Observe posts
    val observeJob = launch {
        viewModel.filteredPosts.collect { posts ->
            println("Posts (${posts.size}):")
            posts.forEach { post ->
                println("  [${post.id}] ${post.title} by Author#${post.authorId} (${post.likes} likes)")
            }
            println()
        }
    }
    
    delay(100)
    
    // Add post
    println("Adding new post...")
    viewModel.addPost("New Post Title", "New post content")
    delay(200)
    
    // Like a post
    println("Liking post 1...")
    viewModel.likePost(1)
    viewModel.likePost(1)
    delay(200)
    
    // Search
    println("Searching for 'kotlin'...")
    viewModel.search("kotlin")
    delay(500)
    
    println("Clear search...")
    viewModel.search("")
    delay(400)
    
    // Delete
    println("Deleting post 2...")
    viewModel.deletePost(2)
    delay(200)
    
    observeJob.cancel()
}
```

---

## ขั้นตอนที่ 365: Advanced Flow Patterns

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    // ============================================
    // buffer - เพิ่ม performance
    // ============================================
    
    println("=== buffer ===")
    
    fun slowProducer(): Flow<Int> = flow {
        for (i in 1..3) {
            delay(100)
            println("Produced $i")
            emit(i)
        }
    }
    
    // ไม่มี buffer - producer รอ consumer
    val timeWithout = measureTimeMillis {
        slowProducer().collect { value ->
            delay(200)  // slow consumer
            println("Consumed $value")
        }
    }
    println("Without buffer: ${timeWithout}ms")  // ~900ms
    
    // มี buffer - producer ไม่ต้องรอ
    val timeWith = measureTimeMillis {
        slowProducer().buffer()
                      .collect { value ->
                          delay(200)
                          println("Buffered: $value")
                      }
    }
    println("With buffer: ${timeWith}ms")  // ~700ms
    
    // ============================================
    // conflate - ข้าม intermediate values
    // ============================================
    
    println("\n=== conflate ===")
    
    (1..10).asFlow()
           .onEach { delay(10) }
           .conflate()  // ข้ามค่าที่ collector ยังไม่พร้อมรับ
           .collect { value ->
               delay(100)
               println("Conflated: $value")
           }
    
    // ============================================
    // collectLatest - cancel ถ้ามีค่าใหม่มา
    // ============================================
    
    println("\n=== collectLatest ===")
    
    (1..5).asFlow()
          .onEach { delay(50) }
          .collectLatest { value ->
              println("Processing: $value")
              delay(100)  // processing time
              println("Done: $value")  // อาจ cancel ก่อนถึงบรรทัดนี้
          }
    
    // ============================================
    // flatMapLatest - เมื่อค่าใหม่มา cancel flow เก่า
    // ============================================
    
    println("\n=== flatMapLatest (search) ===")
    
    val searchQueries = flowOf("a", "ab", "abc", "abcd")
    
    searchQueries.flatMapLatest { query ->
        flow {
            delay(200)  // simulate search debounce
            emit("Results for: $query")
        }
    }.collect { println(it) }
    // จะเห็นแค่ผลลัพธ์สุดท้าย เพราะ query ใหม่มาเร็ว
    
    // ============================================
    // catch - handle exceptions
    // ============================================
    
    println("\n=== catch ===")
    
    flow {
        emit(1)
        emit(2)
        throw RuntimeException("Something went wrong!")
        emit(3)  // จะไม่ถูก emit
    }.catch { e ->
        println("Caught: ${e.message}")
        emit(-1)  // สามารถ emit ค่า fallback ได้
    }.collect { println("Value: $it") }
    
    // ============================================
    // onCompletion
    // ============================================
    
    println("\n=== onCompletion ===")
    
    flow {
        emit("A")
        emit("B")
        emit("C")
    }.onCompletion { cause ->
        if (cause != null) println("Flow failed: $cause")
        else println("Flow completed normally")
    }.collect { println("Got: $it") }
    
    // ============================================
    // retry
    // ============================================
    
    println("\n=== retry ===")
    
    var attempt = 0
    flow {
        attempt++
        println("Attempt $attempt")
        if (attempt < 3) throw RuntimeException("Failed!")
        emit("Success!")
    }.retry(3) { e ->
        delay(100)
        true  // return true เพื่อ retry
    }.collect { println(it) }
    
    // ============================================
    // flowOn - เปลี่ยน dispatcher ของ upstream
    // ============================================
    
    println("\n=== flowOn ===")
    
    flow {
        println("Producing on: ${Thread.currentThread().name}")
        emit(1)
        emit(2)
        emit(3)
    }.map {
        println("Mapping on: ${Thread.currentThread().name}")
        it * 2
    }.flowOn(Dispatchers.Default)  // producer + map runs on Default
     .collect { value ->
         println("Collecting on: ${Thread.currentThread().name}")
         println("Got: $value")
     }
}
```

---

## แบบฝึกหัด Part 16

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// แบบฝึกหัดที่ 1: Stock Price Simulator
// สร้าง flow ที่ emit ราคาหุ้นแบบ real-time

fun stockPriceFlow(symbol: String, initialPrice: Double): Flow<Pair<String, Double>> {
    // TODO: สร้าง flow ที่:
    // - emit ราคาหุ้นทุก 500ms
    // - ราคาเปลี่ยนแบบ random (±5%)
    // - emit Pair(symbol, price)
    TODO()
}

// แบบฝึกหัดที่ 2: Search with Debounce
// จำลอง search ที่มี debounce และ loading state

sealed class SearchState {
    object Idle : SearchState()
    object Loading : SearchState()
    data class Results(val items: List<String>) : SearchState()
    data class Error(val message: String) : SearchState()
}

fun createSearchFlow(queries: Flow<String>): Flow<SearchState> {
    // TODO: สร้าง flow ที่:
    // - debounce 300ms
    // - emit Loading ก่อน
    // - simulate search delay 200ms
    // - emit Results
    TODO()
}

fun main() = runBlocking {
    // Test stockPriceFlow
    println("Stock Prices:")
    stockPriceFlow("AAPL", 150.0)
        .take(5)
        .collect { (symbol, price) ->
            println("  $symbol: $${"%.2f".format(price)}")
        }
    
    // Test search
    println("\nSearch:")
    val queries = flowOf("", "k", "ko", "kot", "kotl", "kotlin")
        .onEach { delay(100) }
    
    createSearchFlow(queries)
        .collect { state ->
            when (state) {
                is SearchState.Idle -> {}
                is SearchState.Loading -> println("  Loading...")
                is SearchState.Results -> println("  Results: ${state.items}")
                is SearchState.Error -> println("  Error: ${state.message}")
            }
        }
}
```

### เฉลย

```kotlin
fun stockPriceFlow(symbol: String, initialPrice: Double): Flow<Pair<String, Double>> = flow {
    var currentPrice = initialPrice
    while (true) {
        val change = (Math.random() - 0.5) * 0.1  // ±5%
        currentPrice *= (1 + change)
        emit(Pair(symbol, currentPrice))
        delay(500)
    }
}

fun createSearchFlow(queries: Flow<String>): Flow<SearchState> = queries
    .debounce(300)
    .filter { it.isNotBlank() }
    .flatMapLatest { query ->
        flow {
            emit(SearchState.Loading)
            delay(200)  // simulate search
            val results = listOf("$query 1", "$query 2", "$query 3")
            emit(SearchState.Results(results))
        }
    }
    .onStart { emit(SearchState.Idle) }
    .catch { emit(SearchState.Error(it.message ?: "Unknown error")) }
```

---

*Part 16 จบแล้ว | ก่อนหน้า: [Part 15](../part15/README.md) | ถัดไป: [Part 17](../part17/README.md)*
