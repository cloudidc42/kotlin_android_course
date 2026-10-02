# Part 80: Macrobenchmark & Performance Testing
## ขั้นตอนที่ 1476-1500

---

## ขั้นตอนที่ 1476: Performance Testing Overview

```
Performance Testing Tools:

1. Macrobenchmark - measure app startup, scroll, interaction
   - Runs on real device or emulator
   - Produces detailed metrics
   - Integrates with CI

2. Microbenchmark - measure specific code snippets
   - High precision (nanoseconds)
   - Eliminates JIT/GC noise

3. Baseline Profiles - pre-compile hot code paths
   - Reduces startup time
   - Improves scroll performance

Metrics ที่สำคัญ:
- Time to Initial Display (TTID)
- Time to Full Display (TTFD)
- Frame timing (jank)
- Memory allocation rate
```

---

## ขั้นตอนที่ 1477: Macrobenchmark Setup

```kotlin
// benchmark/build.gradle.kts (separate module)
plugins {
    id("com.android.test")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.myapp.benchmark"
    compileSdk = 35
    
    defaultConfig {
        minSdk = 23
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
    
    targetProjectPath = ":app"
    
    // Benchmark builds must be non-debuggable
    buildTypes {
        create("benchmark") {
            isDebuggable = false
            signingConfig = signingConfigs.getByName("debug")
            matchingFallbacks += listOf("release")
        }
    }
}

dependencies {
    implementation("androidx.test.ext:junit:1.x")
    implementation("androidx.test.espresso:espresso-core:3.x")
    implementation("androidx.test.uiautomator:uiautomator:2.x")
    implementation("androidx.benchmark:benchmark-macro-junit4:1.x")
}

// app/build.gradle.kts - add benchmark build type
buildTypes {
    create("benchmark") {
        signingConfig = signingConfigs.getByName("debug")
        matchingFallbacks += listOf("release")
        isDebuggable = false
    }
}
```

---

## ขั้นตอนที่ 1478: Startup Benchmark

```kotlin
// benchmark/src/androidTest/kotlin/

@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()
    
    // Cold start - app ยังไม่อยู่ใน memory
    @Test
    fun startupCold() = benchmarkRule.measureRepeated(
        packageName = "com.myapp",
        metrics = listOf(
            StartupTimingMetric(),
            MemoryUsageMetric(MemoryUsageMetric.Mode.LAST)
        ),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }
    
    // Warm start - process exists but Activity was destroyed
    @Test
    fun startupWarm() = benchmarkRule.measureRepeated(
        packageName = "com.myapp",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.WARM
    ) {
        startActivityAndWait()
    }
    
    // Hot start - Activity still in back stack
    @Test
    fun startupHot() = benchmarkRule.measureRepeated(
        packageName = "com.myapp",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.HOT
    ) {
        startActivityAndWait()
    }
}
```

---

## ขั้นตอนที่ 1479: Scroll Performance Benchmark

```kotlin
@RunWith(AndroidJUnit4::class)
class ScrollBenchmark {
    
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()
    
    @Test
    fun scrollProductList() = benchmarkRule.measureRepeated(
        packageName = "com.myapp",
        metrics = listOf(
            FrameTimingMetric(),
            MemoryUsageMetric(MemoryUsageMetric.Mode.MAX)
        ),
        iterations = 5,
        startupMode = StartupMode.WARM,
        setupBlock = {
            // Navigate to product list before measuring
            startActivityAndWait()
            device.findObject(By.res("com.myapp:id/products_tab")).click()
            device.waitForIdle()
        }
    ) {
        // Measure scrolling performance
        val scrollable = device.findObject(
            UiSelector().resourceId("com.myapp:id/product_list")
        )
        
        // Scroll from top to bottom
        repeat(3) {
            scrollable.flingForward()
            device.waitForIdle()
        }
        
        // Scroll back
        repeat(3) {
            scrollable.flingBackward()
            device.waitForIdle()
        }
    }
    
    @Test
    fun scrollWithFiltering() = benchmarkRule.measureRepeated(
        packageName = "com.myapp",
        metrics = listOf(FrameTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.WARM
    ) {
        startActivityAndWait()
        
        // Open filter
        device.findObject(By.res("com.myapp:id/filter_button")).click()
        device.waitForIdle()
        
        // Apply filter
        device.findObject(By.text("Electronics")).click()
        device.findObject(By.res("com.myapp:id/apply_filter")).click()
        device.waitForIdle()
        
        // Scroll filtered results
        val list = device.findObject(UiSelector().resourceId("com.myapp:id/product_list"))
        list.flingForward()
        device.waitForIdle()
    }
}
```

---

## ขั้นตอนที่ 1480: Baseline Profiles Generation

```kotlin
// ============================================
// Baseline Profile Generator
// ============================================

@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    
    @get:Rule
    val rule = BaselineProfileRule()
    
    @Test
    fun generate() = rule.collect(
        packageName = "com.myapp"
    ) {
        // Define critical user journeys
        
        // Journey 1: App startup → Home
        pressHome()
        startActivityAndWait()
        
        // Journey 2: Navigate to product list
        device.findObject(By.res("com.myapp:id/products_tab")).click()
        device.waitForIdle()
        
        // Journey 3: Open product detail
        device.findObject(
            UiSelector().resourceId("com.myapp:id/product_item").instance(0)
        ).click()
        device.waitForIdle()
        
        // Journey 4: Add to cart
        device.findObject(By.res("com.myapp:id/add_to_cart")).click()
        device.waitForIdle()
        
        // Journey 5: Go to checkout
        device.findObject(By.res("com.myapp:id/cart_icon")).click()
        device.waitForIdle()
        device.findObject(By.res("com.myapp:id/checkout_button")).click()
        device.waitForIdle()
        
        // Journey 6: Search
        device.pressBack()
        device.pressBack()
        device.findObject(By.res("com.myapp:id/search_bar")).click()
        device.findObject(By.res("com.myapp:id/search_input")).setText("iPhone")
        device.waitForIdle()
    }
}

// Copy generated profile to:
// app/src/main/baseline-prof.txt
```

---

## ขั้นตอนที่ 1481: Microbenchmark

```kotlin
// Microbenchmark สำหรับ specific operations
// เพิ่ม dependency: androidTestImplementation("androidx.benchmark:benchmark-junit4:1.x")

@RunWith(AndroidJUnit4::class)
class JsonParsingBenchmark {
    
    @get:Rule
    val benchmarkRule = BenchmarkRule()
    
    private val gson = Gson()
    private val moshi = Moshi.Builder().build()
    private val moshiAdapter = moshi.adapter(ProductDto::class.java)
    
    private val sampleJson = """
    {"id": 1, "name": "iPhone 16", "price": 35000.0, "category": "Electronics"}
    """
    
    @Test
    fun parseWithGson() = benchmarkRule.measureRepeated {
        gson.fromJson(sampleJson, ProductDto::class.java)
    }
    
    @Test
    fun parseWithMoshi() = benchmarkRule.measureRepeated {
        moshiAdapter.fromJson(sampleJson)
    }
    
    @Test
    fun parseWithKotlinxSerialization() = benchmarkRule.measureRepeated {
        Json.decodeFromString<ProductDto>(sampleJson)
    }
}

@RunWith(AndroidJUnit4::class)
class DatabaseBenchmark {
    
    @get:Rule
    val benchmarkRule = BenchmarkRule()
    
    private lateinit var db: AppDatabase
    private lateinit var dao: ProductDao
    
    @Before
    fun setup() {
        val context = InstrumentationRegistry.getInstrumentation().context
        db = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java).build()
        dao = db.productDao()
        
        // Pre-populate
        runBlocking {
            val products = (1..1000).map { ProductEntity(id = it.toLong(), name = "Product $it", price = it.toDouble()) }
            dao.insertAll(products)
        }
    }
    
    @Test
    fun queryAllProducts() = benchmarkRule.measureRepeated {
        runBlocking { dao.getAll() }
    }
    
    @Test
    fun searchProducts() = benchmarkRule.measureRepeated {
        runBlocking { dao.search("Product 5%") }
    }
    
    @After
    fun teardown() {
        db.close()
    }
}
```

---

## แบบฝึกหัด Part 80

```kotlin
// แบบฝึกหัด: Performance Audit

// 1. เขียน StartupBenchmark สำหรับ app ของคุณ
// 2. เขียน ScrollBenchmark สำหรับ list ที่ใช้บ่อยที่สุด
// 3. Generate Baseline Profile
// 4. วัด Before/After performance improvement
// 5. Report ผล (startup time ลดลงกี่ ms?)

// Performance checklist
data class PerformanceReport(
    val coldStartMs: Double,
    val warmStartMs: Double,
    val hotStartMs: Double,
    val avgFrameTimeMs: Double,
    val jankyFramePercent: Double,
    val p99FrameTimeMs: Double,
    val memoryUsageMb: Double
) {
    fun summary(): String = """
        === Performance Report ===
        Cold Start: ${coldStartMs}ms
        Warm Start: ${warmStartMs}ms
        Hot Start: ${hotStartMs}ms
        Avg Frame Time: ${avgFrameTimeMs}ms (target: <16.7ms)
        Janky Frames: ${jankyFramePercent}% (target: <5%)
        P99 Frame Time: ${p99FrameTimeMs}ms
        Memory Usage: ${memoryUsageMb}MB
    """.trimIndent()
}
```

---

*Part 80 จบแล้ว | ก่อนหน้า: [Part 79](../part79/README.md) | ถัดไป: [Part 81](../part81/README.md)*
