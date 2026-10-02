# Part 93: App Performance Optimization
## ขั้นตอนที่ 1801-1825

---

## ขั้นตอนที่ 1801: R8 & ProGuard Optimization

```
R8 เป็น next-gen ProGuard:
- Code shrinking (ลบ unused code)
- Obfuscation (rename identifiers)
- Optimization (bytecode-level)
- Resource shrinking (ลบ unused resources)

เปิดใช้งาน:
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

```kotlin
// ============================================
// proguard-rules.pro
// ============================================

/*
# Keep data classes for Gson/Moshi/kotlinx.serialization
-keep class com.myapp.data.model.** { *; }
-keep class com.myapp.domain.model.** { *; }

# Keep sealed classes
-keep class com.myapp.domain.result.** { *; }

# Retrofit
-keepattributes Signature
-keepattributes Exceptions
-dontwarn retrofit2.**
-keep class retrofit2.** { *; }

# OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**
-keep class okhttp3.** { *; }

# Room
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *
-dontwarn androidx.room.paging.**

# Hilt
-keep class dagger.hilt.** { *; }
-keep class javax.inject.** { *; }
-keepclasseswithmembernames class * { @javax.inject.* <fields>; }

# Firebase
-keep class com.google.firebase.** { *; }
-dontwarn com.google.firebase.**

# Kotlin
-keep class kotlin.Metadata { *; }
-keepclassmembers class kotlin.Metadata {
    public <methods>;
}

# Coroutines
-dontwarn kotlinx.coroutines.**

# Enums
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# Parcelable
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# Serializable
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}
*/
```

---

## ขั้นตอนที่ 1802: APK/AAB Size Reduction

```kotlin
// ============================================
// Reduce APK size
// ============================================

// build.gradle.kts (app)
android {
    
    // 1. ABI Splits - generate separate APK per CPU architecture
    splits {
        abi {
            isEnable = true
            reset()
            include("arm64-v8a", "x86_64")  // Target modern devices
            isUniversalApk = false
        }
    }
    
    // 2. Density splits
    splits {
        density {
            isEnable = true
            reset()
            include("hdpi", "xhdpi", "xxhdpi", "xxxhdpi")
            isUniversalApk = false
        }
    }
    
    // 3. Resource configuration restrictions
    defaultConfig {
        resourceConfigurations += listOf("th", "en", "xxhdpi", "xxxhdpi")
    }
}

// build.gradle.kts (app)
dependencies {
    // Use vector drawables (smaller than PNGs)
    implementation("androidx.vectordrawable:vectordrawable:1.2.0")
    
    // Dynamic feature modules - load on demand
    // implementation(project(":feature:ar"))  // Heavy feature
}
```

```
APK Size Reduction Checklist:
✅ Enable R8 minification
✅ Enable resource shrinking
✅ Use WebP instead of PNG (30% smaller)
✅ Use vector drawables
✅ Remove unused dependencies
✅ Use Play App Signing (enables split APKs)
✅ Dynamic feature modules for heavy features
✅ Analyze with Build > Analyze APK

Tools:
- Build > Analyze APK → inspect content
- ./gradlew :app:bundleRelease → create AAB
```

---

## ขั้นตอนที่ 1803: Compose Performance

```kotlin
// ============================================
// Compose Recomposition Optimization
// ============================================

// 1. Use stable types
@Stable
data class ProductUiModel(
    val id: Long,
    val name: String,
    val price: Double,
    val imageUrl: String
)

// 2. Remember expensive computations
@Composable
fun ProductScreen(products: List<ProductUiModel>) {
    
    // ✅ Remember sorted list
    val sortedProducts = remember(products) {
        products.sortedBy { it.price }
    }
    
    // ✅ Remember derived state
    val totalPrice by remember(products) {
        derivedStateOf { products.sumOf { it.price } }
    }
    
    // ✅ Stable keys in LazyList
    LazyColumn {
        items(
            items = sortedProducts,
            key = { product -> product.id }  // Stable key
        ) { product ->
            ProductItem(product)
        }
    }
}

// 3. Avoid unnecessary recompositions with lambdas
@Composable
fun ProductList(
    products: List<ProductUiModel>,
    onProductClick: (Long) -> Unit  // This lambda is recreated every recomposition!
) {
    // ...
}

// Better: use remembered lambda
@Composable
fun ProductScreen(viewModel: ProductViewModel = hiltViewModel()) {
    
    // ✅ Stable callback reference
    val onProductClick = remember<(Long) -> Unit>(viewModel) {
        { productId -> viewModel.onProductClick(productId) }
    }
    
    ProductList(
        products = products,
        onProductClick = onProductClick
    )
}

// 4. Skip unnecessary params with @Immutable
@Immutable
data class ProductFilter(
    val categoryId: Long?,
    val minPrice: Double?,
    val maxPrice: Double?,
    val sortOrder: SortOrder
)

// 5. Use key() to control identity
@Composable
fun AnimatedList(items: List<Item>) {
    items.forEach { item ->
        key(item.id) {  // Preserve state per item
            AnimatedVisibility(visible = item.isVisible) {
                ItemRow(item)
            }
        }
    }
}

// 6. Defer state reads for frequent updates
@Composable
fun ScrollableHeader(scrollState: ScrollState) {
    
    // ❌ Causes recomposition on every scroll
    // val alpha = 1f - (scrollState.value / 500f).coerceIn(0f, 1f)
    
    // ✅ Only measures at draw phase, no recomposition
    Box(
        modifier = Modifier.graphicsLayer {
            alpha = 1f - (scrollState.value / 500f).coerceIn(0f, 1f)
        }
    )
}
```

---

## ขั้นตอนที่ 1804: Startup Performance

```kotlin
// ============================================
// App Startup Performance
// ============================================

// 1. App Startup Library - lazy initialize
class TimberInitializer : Initializer<Timber.Forest> {
    
    override fun create(context: Context): Timber.Forest {
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
        return Timber
    }
    
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

class FirebaseInitializer : Initializer<FirebaseApp> {
    
    override fun create(context: Context): FirebaseApp {
        return FirebaseApp.initializeApp(context)!!
    }
    
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// AndroidManifest.xml
/*
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.myapp.TimberInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="com.myapp.FirebaseInitializer"
        android:value="androidx.startup" />
</provider>
*/

// 2. Baseline Profiles (covered in Part 80)
// 3. Deferring non-critical work
class MyApplication : Application() {
    
    override fun onCreate() {
        super.onCreate()
        
        // Critical path - init synchronously
        initCrashReporting()
        
        // Non-critical - defer to after UI is interactive
        ProcessLifecycleOwner.get().lifecycle.addObserver(object : DefaultLifecycleObserver {
            override fun onStart(owner: LifecycleOwner) {
                owner.lifecycle.removeObserver(this)
                
                // Runs after first activity starts
                initAnalytics()
                prefetchData()
            }
        })
    }
}

// 4. Splash Screen API
class MainActivity : ComponentActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()
        super.onCreate(savedInstanceState)
        
        // Keep splash until data is loaded
        var keepSplash = true
        splashScreen.setKeepOnScreenCondition { keepSplash }
        
        lifecycleScope.launch {
            // Load initial data
            delay(1000)  // simulate
            keepSplash = false
        }
        
        setContent {
            AppTheme { MainScreen() }
        }
    }
}
```

---

## ขั้นตอนที่ 1805: Memory Optimization

```kotlin
// ============================================
// Memory Optimization Strategies
// ============================================

// 1. Image loading with Coil - auto memory management
@Composable
fun OptimizedImage(url: String, size: Dp) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .size(size.value.toInt())  // Load at display size, not full resolution
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .crossfade(true)
            .build(),
        contentDescription = null,
        modifier = Modifier.size(size)
    )
}

// 2. LRU Cache for custom data
class DataCache<K, V>(maxSize: Int) {
    private val cache = object : LruCache<K, V>(maxSize) {
        override fun sizeOf(key: K, value: V): Int = 1
    }
    
    fun get(key: K): V? = cache.get(key)
    fun put(key: K, value: V) = cache.put(key, value)
    fun remove(key: K) = cache.remove(key)
    fun evictAll() = cache.evictAll()
}

// 3. onTrimMemory callback
class MyApplication : Application() {
    
    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        
        when (level) {
            TRIM_MEMORY_RUNNING_LOW,
            TRIM_MEMORY_RUNNING_CRITICAL -> {
                // App is in foreground but memory is low
                clearNonEssentialCaches()
            }
            TRIM_MEMORY_UI_HIDDEN -> {
                // App moved to background
                clearUiCaches()
            }
            TRIM_MEMORY_BACKGROUND,
            TRIM_MEMORY_MODERATE,
            TRIM_MEMORY_COMPLETE -> {
                // App in background, clear as much as possible
                clearAllCaches()
            }
        }
    }
}

// 4. Avoid memory leaks with coroutines
class DataRepository @Inject constructor(
    private val api: DataApi,
    @ApplicationScope private val scope: CoroutineScope  // Application-scoped, not ViewModel
) {
    // Application-level background work that should outlive ViewModel
    fun startPeriodicRefresh() {
        scope.launch {
            while (isActive) {
                refreshData()
                delay(60_000)
            }
        }
    }
}

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class ApplicationScope

@Module
@InstallIn(SingletonComponent::class)
object CoroutineScopeModule {
    
    @Provides
    @Singleton
    @ApplicationScope
    fun provideApplicationScope(): CoroutineScope =
        CoroutineScope(SupervisorJob() + Dispatchers.Default)
}
```

---

*Part 93 จบแล้ว | ก่อนหน้า: [Part 92](../part92/README.md) | ถัดไป: [Part 94](../part94/README.md)*
