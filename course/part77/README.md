# Part 77: Memory Profiling & LeakCanary
## ขั้นตอนที่ 1401-1425

---

## ขั้นตอนที่ 1401: Memory Management ใน Android

```
Memory Issues ที่พบบ่อย:

1. Memory Leak - object ไม่ถูก GC เพราะมี reference ค้างอยู่
   - Activity/Fragment referenced by static field
   - Inner class holding outer class reference
   - Lambda/anonymous class capturing context
   - Listener not removed

2. OOM (Out of Memory) - allocate memory มากเกินไป
   - Loading bitmap ขนาดใหญ่
   - Cache ไม่มี size limit
   - Recursive calls

3. Excessive GC - GC ทำงานบ่อยเกิน → jank
   - Allocating objects ใน onDraw() หรือ scroll listener
   - String concatenation ใน loop

Tools:
- Android Profiler (Android Studio)
- LeakCanary (automatic leak detection)
- StrictMode (detect disk/network on main thread)
- Memory Profiler (heap dump analysis)
```

---

## ขั้นตอนที่ 1402: LeakCanary Setup

```kotlin
// build.gradle.kts
// debugImplementation("com.squareup.leakcanary:leakcanary-android:2.x")
// เฉพาะ debug build เท่านั้น - ไม่ include ใน release

// LeakCanary ทำงานอัตโนมัติ - ไม่ต้อง init
// เมื่อ detect leak → แสดง notification → เปิด LeakCanary Activity

// Custom watched objects (นอกจาก Activity/Fragment ที่ LeakCanary watch อัตโนมัติ)
class MyViewModel : ViewModel() {
    override fun onCleared() {
        super.onCleared()
        AppWatcher.objectWatcher.watch(
            watchedObject = this,
            description = "MyViewModel should be garbage collected"
        )
    }
}
```

---

## ขั้นตอนที่ 1403: Common Memory Leaks & Fixes

```kotlin
// ============================================
// LEAK 1: Static reference to Context
// ============================================

// BAD: static context reference
object BadSingleton {
    var context: Context? = null  // LEAK!
    
    fun init(ctx: Context) {
        context = ctx  // holds Activity
    }
}

// GOOD: Use ApplicationContext
object GoodSingleton {
    private lateinit var appContext: Context
    
    fun init(ctx: Context) {
        appContext = ctx.applicationContext  // safe
    }
}

// ============================================
// LEAK 2: Inner class holding outer reference
// ============================================

// BAD: Non-static inner class (Handler, AsyncTask)
class LeakyActivity : Activity() {
    private val handler = object : Handler(Looper.getMainLooper()) {  // holds LeakyActivity
        override fun handleMessage(msg: Message) { /* ... */ }
    }
}

// GOOD: Static inner class + WeakReference
class SafeActivity : Activity() {
    private val handler = SafeHandler(this)
    
    class SafeHandler(activity: SafeActivity) : Handler(Looper.getMainLooper()) {
        private val weakRef = WeakReference(activity)
        
        override fun handleMessage(msg: Message) {
            weakRef.get()?.let { activity ->
                // handle message
            }
        }
    }
}

// ============================================
// LEAK 3: Callback not removed
// ============================================

// BAD: Register callback, never remove
class EventBusLeakActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        EventBus.default.register(this)  // Activity never unregistered!
    }
}

// GOOD: Register/unregister in lifecycle
class EventBusSafeActivity : ComponentActivity() {
    override fun onStart() {
        super.onStart()
        EventBus.default.register(this)
    }
    
    override fun onStop() {
        super.onStop()
        EventBus.default.unregister(this)
    }
}

// ============================================
// LEAK 4: Coroutine Job leak
// ============================================

// BAD: Launch coroutine without proper scope
class LeakyViewModel : ViewModel() {
    fun loadData() {
        // GlobalScope keeps coroutine alive even after ViewModel cleared
        GlobalScope.launch {
            delay(10_000)
            // this might crash if ViewModel is cleared
        }
    }
}

// GOOD: Use viewModelScope
class SafeViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {  // auto-cancelled when ViewModel cleared
            delay(10_000)
            // safe
        }
    }
}
```

---

## ขั้นตอนที่ 1404: Bitmap Memory Management

```kotlin
// ============================================
// Efficient Bitmap Loading
// ============================================

// Use Coil (ทำ memory management ให้อัตโนมัติ)
// implementation("io.coil-kt:coil-compose:2.x")

@Composable
fun OptimizedImage(url: String, modifier: Modifier = Modifier) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .size(Size.ORIGINAL)      // หรือกำหนด size ที่ต้องการ
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .build(),
        contentDescription = null,
        contentScale = ContentScale.Crop,
        modifier = modifier
    )
}

// กำหนด memory cache size
val imageLoader = ImageLoader.Builder(context)
    .memoryCache {
        MemoryCache.Builder(context)
            .maxSizePercent(0.25)  // 25% of available memory
            .build()
    }
    .diskCache {
        DiskCache.Builder()
            .directory(context.cacheDir.resolve("image_cache"))
            .maxSizeBytes(100 * 1024 * 1024)  // 100MB
            .build()
    }
    .build()

// Bitmap decode ด้วยตัวเอง
fun decodeSampledBitmapFromResource(
    res: Resources,
    resId: Int,
    reqWidth: Int,
    reqHeight: Int
): Bitmap {
    return BitmapFactory.Options().run {
        inJustDecodeBounds = true
        BitmapFactory.decodeResource(res, resId, this)
        
        inSampleSize = calculateInSampleSize(this, reqWidth, reqHeight)
        inJustDecodeBounds = false
        
        BitmapFactory.decodeResource(res, resId, this)
    }
}

fun calculateInSampleSize(options: BitmapFactory.Options, reqWidth: Int, reqHeight: Int): Int {
    val (height, width) = options.outHeight to options.outWidth
    var inSampleSize = 1
    
    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        
        while (halfHeight / inSampleSize >= reqHeight && halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    
    return inSampleSize
}
```

---

## ขั้นตอนที่ 1405: StrictMode & Profiling

```kotlin
// ============================================
// StrictMode - detect bad practices
// ============================================

class MyApplication : Application() {
    
    override fun onCreate() {
        super.onCreate()
        
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .detectCustomSlowCalls()
                    .penaltyLog()         // log ใน Logcat
                    .penaltyFlashScreen() // flash screen สีแดง
                    // .penaltyDeath()    // crash เมื่อ detect (aggressive)
                    .build()
            )
            
            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedSqlLiteObjects()
                    .detectLeakedClosableObjects()
                    .detectActivityLeaks()
                    .penaltyLog()
                    .build()
            )
        }
    }
}

// ============================================
// Memory Profiling Annotations
// ============================================

// Trace sections ใน Android Profiler
fun heavyOperation() {
    Trace.beginSection("HeavyOperation")
    try {
        // code ที่ต้องการ profile
        Thread.sleep(100)
    } finally {
        Trace.endSection()
    }
}

// หรือใช้ traceAsync
inline fun <T> trace(label: String, block: () -> T): T {
    Trace.beginSection(label)
    return try {
        block()
    } finally {
        Trace.endSection()
    }
}

// ใช้งาน
suspend fun loadProducts() = trace("loadProducts") {
    withContext(Dispatchers.IO) {
        productApi.getProducts()
    }
}
```

---

## แบบฝึกหัด Part 77

```kotlin
// แบบฝึกหัด: Fix Memory Leaks

// โค้ดด้านล่างมี memory leaks อยู่หลายจุด
// หา และแก้ไข leaks ทั้งหมด

class LeakyProductAdapter(private val context: Context) :
    RecyclerView.Adapter<LeakyProductAdapter.ViewHolder>() {
    
    companion object {
        // LEAK 1: static list holds Product objects indefinitely
        val allProducts = mutableListOf<Product>()
    }
    
    private var listener: (() -> Unit)? = null
    
    // LEAK 2: context stored in instance
    fun setListener(l: () -> Unit) { listener = l }
    
    inner class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        // LEAK 3: inner class holds adapter reference
        
        fun bind(product: Product) {
            itemView.setOnClickListener {
                // LEAK 4: closure captures 'context' which may be Activity
                val intent = Intent(context, DetailActivity::class.java)
                context.startActivity(intent)
            }
        }
    }
    
    // TODO: Fix all 4 memory leaks
    override fun getItemCount() = allProducts.size
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder =
        ViewHolder(LayoutInflater.from(parent.context).inflate(R.layout.item_product, parent, false))
    override fun onBindViewHolder(holder: ViewHolder, position: Int) =
        holder.bind(allProducts[position])
}
```

---

*Part 77 จบแล้ว | ก่อนหน้า: [Part 76](../part76/README.md) | ถัดไป: [Part 78](../part78/README.md)*
