# Part 54: Performance Optimization
## ขั้นตอนที่ 826-850

---

## ขั้นตอนที่ 826: Performance Metrics

```kotlin
// เครื่องมือวัด Performance:
// 1. Android Profiler (Android Studio)
//    - CPU Profiler
//    - Memory Profiler
//    - Network Profiler
//    - Energy Profiler
//
// 2. Systrace / Perfetto
// 3. Baseline Profiles
// 4. Macrobenchmark

// สิ่งที่ต้องดู:
// - App Startup Time (cold, warm, hot)
// - Frame rate (เป้าหมาย 60fps = 16ms ต่อ frame)
// - Memory usage (heap, native)
// - Battery drain
// - Network efficiency
```

---

## ขั้นตอนที่ 827: Compose Performance

```kotlin
// ============================================
// Recomposition Optimization
// ============================================

// ❌ สร้าง lambda ใหม่ทุก recomposition
@Composable
fun BadList(items: List<String>) {
    LazyColumn {
        items(items) { item ->
            Text(
                text = item,
                modifier = Modifier.clickable {  // ❌ lambda ใหม่ทุกครั้ง
                    println("Clicked: $item")
                }
            )
        }
    }
}

// ✅ ใช้ remember หรือ hoisting
@Composable
fun GoodList(
    items: List<String>,
    onItemClick: (String) -> Unit  // ✅ hoist callback
) {
    LazyColumn {
        items(items, key = { it }) { item ->  // ✅ key ช่วย reuse
            ItemRow(
                item = item,
                onClick = { onItemClick(item) }
            )
        }
    }
}

// ============================================
// derivedStateOf - คำนวณเมื่อ dependencies เปลี่ยนเท่านั้น
// ============================================

@Composable
fun SearchScreen() {
    val items = remember { mutableStateListOf("apple", "banana", "cherry") }
    var query by remember { mutableStateOf("") }
    
    // ❌ คำนวณทุก recomposition
    // val filtered = items.filter { it.contains(query) }
    
    // ✅ คำนวณเฉพาะเมื่อ items หรือ query เปลี่ยน
    val filtered by remember(items, query) {
        derivedStateOf { items.filter { it.contains(query, ignoreCase = true) } }
    }
    
    Column {
        TextField(value = query, onValueChange = { query = it }, label = { Text("Search") })
        LazyColumn {
            items(filtered) { item -> Text(item) }
        }
    }
}

// ============================================
// Stable Classes - ป้องกัน unnecessary recomposition
// ============================================

// ❌ Unstable class - Compose ไม่รู้ว่า equal หรือเปล่า
class UserProfile(val name: String, val email: String)

// ✅ Stable options:
// 1. Data class (Compose จะ check equals)
data class UserProfileData(val name: String, val email: String)

// 2. @Stable annotation
@Stable
class UserProfileStable(val name: String, val email: String) {
    override fun equals(other: Any?): Boolean {
        if (other !is UserProfileStable) return false
        return name == other.name && email == other.email
    }
    override fun hashCode() = 31 * name.hashCode() + email.hashCode()
}

// 3. @Immutable annotation (ทุก property เป็น val และ immutable)
@Immutable
data class UserProfileImmutable(val name: String, val email: String)

// ============================================
// remember กับ key
// ============================================

@Composable
fun ProfileImage(userId: Long) {
    // ❌ สร้าง Painter ใหม่ทุกครั้ง
    // val painter = rememberAsyncImagePainter(...)
    
    // ✅ Recreate เฉพาะเมื่อ userId เปลี่ยน
    val painter = remember(userId) {
        // create painter for this userId
    }
}
```

---

## ขั้นตอนที่ 828: Lazy Loading และ Pagination

```kotlin
// ============================================
// Paging 3 Library
// ============================================

// build.gradle.kts
// implementation("androidx.paging:paging-runtime:3.3.x")
// implementation("androidx.paging:paging-compose:3.3.x")

// PagingSource
class PostPagingSource(
    private val apiService: ApiService
) : PagingSource<Int, Post>() {
    
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Post> {
        val page = params.key ?: 1
        
        return try {
            val response = apiService.getPosts(page = page, limit = params.loadSize)
            
            LoadResult.Page(
                data = response.items,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.items.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
    
    override fun getRefreshKey(state: PagingState<Int, Post>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }
}

// Repository
class PostRepository(private val apiService: ApiService) {
    
    fun getPostsPaged(): Flow<PagingData<Post>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            enablePlaceholders = false,
            prefetchDistance = 5,
            initialLoadSize = 40
        ),
        pagingSourceFactory = { PostPagingSource(apiService) }
    ).flow
}

// ViewModel
@HiltViewModel
class PostListViewModel @Inject constructor(
    private val repository: PostRepository
) : ViewModel() {
    
    val posts: Flow<PagingData<Post>> = repository
        .getPostsPaged()
        .cachedIn(viewModelScope)  // cache ใน ViewModel scope
}

// Composable
@Composable
fun PagedPostList(viewModel: PostListViewModel = hiltViewModel()) {
    val posts = viewModel.posts.collectAsLazyPagingItems()
    
    LazyColumn {
        items(
            count = posts.itemCount,
            key = posts.itemKey { it.id }
        ) { index ->
            val post = posts[index]
            if (post != null) {
                PostCard(post = post)
            } else {
                // Placeholder
                Box(modifier = Modifier
                    .fillMaxWidth()
                    .height(80.dp)
                    .background(MaterialTheme.colorScheme.surfaceVariant))
            }
        }
        
        // Loading state
        when (posts.loadState.append) {
            is LoadState.Loading -> {
                item {
                    CircularProgressIndicator(modifier = Modifier.fillMaxWidth().padding(16.dp))
                }
            }
            is LoadState.Error -> {
                item {
                    TextButton(onClick = { posts.retry() }) {
                        Text("ลองใหม่")
                    }
                }
            }
            else -> {}
        }
    }
}
```

---

## ขั้นตอนที่ 829: App Startup Optimization

```kotlin
// ============================================
// Baseline Profiles - ปรับ JIT Compilation
// ============================================

// build.gradle.kts
// implementation("androidx.profileinstaller:profileinstaller:1.3.x")
// baselineProfile(project(":baseline-profile"))

// BaselineProfileGenerator.kt (ใน :baseline-profile module)
@ExperimentalBaselineProfilesApi
class BaselineProfileGenerator {
    
    @get:Rule
    val baselineProfileRule = BaselineProfileRule()
    
    @Test
    fun startup() = baselineProfileRule.collect(
        packageName = "com.example.app"
    ) {
        pressHome()
        startActivityAndWait()
    }
    
    @Test
    fun scrollFeed() = baselineProfileRule.collect(
        packageName = "com.example.app"
    ) {
        pressHome()
        startActivityAndWait()
        
        device.wait(Until.hasObject(By.res("feed_list")), 3_000)
        val feed = device.findObject(By.res("feed_list"))
        feed.setGestureMargin(device.displayWidth / 5)
        
        repeat(3) { feed.fling(Direction.DOWN) }
    }
}

// ============================================
// App Startup Library
// ============================================

// สำหรับ initialize libraries แบบ lazy
class MyInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        // Initialize lazily
        FirebaseApp.initializeApp(context)
    }
    
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// AndroidManifest.xml
// <provider
//     android:name="androidx.startup.InitializationProvider"
//     android:authorities="${applicationId}.androidx-startup"
//     android:exported="false"
//     tools:node="merge">
//     <meta-data
//         android:name="com.example.MyInitializer"
//         android:value="androidx.startup" />
// </provider>
```

---

## ขั้นตอนที่ 830: Memory Optimization

```kotlin
// ============================================
// Bitmap Memory Management
// ============================================

// ใช้ Coil หรือ Glide แทนการจัดการ Bitmap เอง
@Composable
fun OptimizedImage(
    imageUrl: String,
    modifier: Modifier = Modifier
) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .crossfade(true)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .size(Size.ORIGINAL)
            .build(),
        contentDescription = null,
        contentScale = ContentScale.Crop,
        modifier = modifier
    )
}

// ============================================
// Avoiding Memory Leaks
// ============================================

// ❌ Memory leak - Activity context ใน singleton
object BadSingleton {
    var context: Context? = null  // ❌ leaks Activity
}

// ✅ ใช้ Application Context
object GoodSingleton {
    private lateinit var appContext: Context
    
    fun init(context: Context) {
        appContext = context.applicationContext  // ✅ safe
    }
}

// ❌ Memory leak - listener ไม่ถูก remove
class LeakyActivity : AppCompatActivity() {
    override fun onStart() {
        super.onStart()
        someManager.addListener(this)  // ❌ ไม่ remove ใน onStop
    }
}

// ✅ ใช้ lifecycle-aware
class GoodActivity : AppCompatActivity() {
    override fun onStart() {
        super.onStart()
        someManager.addListener(this)
    }
    
    override fun onStop() {
        super.onStop()
        someManager.removeListener(this)  // ✅
    }
}

// ✅ ดียิ่งขึ้นด้วย Compose DisposableEffect
@Composable
fun LifecycleAwareListener() {
    DisposableEffect(Unit) {
        val listener = someManager.addListener { /* handle */ }
        onDispose { listener.remove() }
    }
}

// ============================================
// WeakReference สำหรับ callbacks
// ============================================

class MyCallback(activity: MainActivity) {
    private val weakActivity = WeakReference(activity)
    
    fun onResult(result: String) {
        weakActivity.get()?.handleResult(result)
            ?: println("Activity was garbage collected")
    }
}
```

---

## แบบฝึกหัด Part 54

```kotlin
// แบบฝึกหัด: Optimize a slow scrolling list

// ปัญหา: List ที่ scroll ช้า
@Composable
fun SlowList(items: List<Post>) {
    LazyColumn {
        items(items) { post ->
            Card(modifier = Modifier.padding(8.dp)) {
                Column(modifier = Modifier.padding(16.dp)) {
                    // ❌ ปัญหา 1: สร้าง Date object ใน composition
                    val date = java.util.Date(post.createdAt)
                    Text("${date.toString().take(10)}")
                    
                    // ❌ ปัญหา 2: Complex calculation ใน composition
                    val wordCount = post.content.split(" ").size
                    Text("$wordCount words - estimated ${wordCount / 200} min read")
                    
                    // ❌ ปัญหา 3: ไม่มี key
                    // items(items) ← ควรใช้ key
                    
                    Text(post.title, style = MaterialTheme.typography.titleMedium)
                    Text(post.content, maxLines = 2)
                }
            }
        }
    }
}

// TODO: แก้ไขให้ list scroll ได้อย่าง smooth
// 1. ย้าย calculations ออกจาก composition
// 2. เพิ่ม key ให้ LazyColumn
// 3. ใช้ remember สำหรับ derived state
// 4. สร้าง stable data class สำหรับ Post UI model
```

---

*Part 54 จบแล้ว | ก่อนหน้า: [Part 53](../part53/README.md) | ถัดไป: [Part 55](../part55/README.md)*
