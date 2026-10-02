# Part 90: World-Class Developer Practices
## ขั้นตอนที่ 1726-1750

---

## ขั้นตอนที่ 1726: Code Review Excellence

```
Code Review Best Practices:

ในฐานะ Author:
1. PR ขนาดเล็ก (<400 LOC) - ง่ายต่อการ review
2. เขียน description ที่ชัดเจน
3. Self-review ก่อนส่ง
4. ตอบทุก comment ก่อน merge
5. ไม่ตั้งรับ - feedback คือของขวัญ

ในฐานะ Reviewer:
1. Review ภายใน 24 ชั่วโมง
2. ให้ context ใน comment เสมอ (ทำไม ไม่ใช่แค่อะไร)
3. แยก blocking vs non-blocking
4. ชมเมื่อเห็นดี
5. ตั้งคำถามแทนคำสั่ง

Google Engineering Practices:
- LGTM = Looks Good To Me
- nit: = small style suggestion
- optional: = nice to have
- blocking: = must fix before merge
```

---

## ขั้นตอนที่ 1727: Kotlin Code Style

```kotlin
// ============================================
// Idiomatic Kotlin - ตัวอย่างที่ดี
// ============================================

// 1. Use data classes for value objects
data class Money(val amount: BigDecimal, val currency: Currency) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }
}

// 2. Prefer expression body
fun formatPrice(amount: Double, currency: String = "THB"): String =
    NumberFormat.getCurrencyInstance(Locale("th", "TH")).format(amount)

// 3. Use scope functions correctly
// let - transform nullable
val userName = user?.let { "${it.firstName} ${it.lastName}" } ?: "Guest"

// run - execute block and return result  
val result = userInput.run {
    if (isBlank()) return@run null
    trim().lowercase()
}

// apply - configure object, return same object
val intent = Intent(context, MainActivity::class.java).apply {
    putExtra("userId", userId)
    putExtra("source", "notification")
    flags = Intent.FLAG_ACTIVITY_CLEAR_TOP
}

// also - perform side effect, return same object
val user = createUser()
    .also { newUser -> analytics.track("user_created", newUser.id) }
    .also { newUser -> sendWelcomeEmail(newUser.email) }

// with - call multiple methods on same object
val summary = with(order) {
    "Order #${id}: ${items.size} items, Total: ${total}"
}

// 4. Destructuring
val (width, height) = view.size
val (first, *rest) = list
val (status, body) = response

// 5. Extension functions
fun String.toSlug(): String = lowercase().replace(" ", "-").replace(Regex("[^a-z0-9-]"), "")
fun Double.toCurrency(currency: String = "THB"): String = "฿${String.format("%.2f", this)}"
fun Int.dp(context: Context): Int = (this * context.resources.displayMetrics.density).toInt()

// 6. Lazy initialization
class ExpensiveObject {
    val data: List<Item> by lazy {
        // computed once, on first access
        loadExpensiveData()
    }
}

// 7. Sealed classes > enums for behavior
sealed class Shape {
    abstract fun area(): Double
    
    data class Circle(val radius: Double) : Shape() {
        override fun area() = Math.PI * radius * radius
    }
    
    data class Rectangle(val width: Double, val height: Double) : Shape() {
        override fun area() = width * height
    }
    
    data class Triangle(val base: Double, val height: Double) : Shape() {
        override fun area() = 0.5 * base * height
    }
}

// 8. Prefer immutable
val items = listOf(1, 2, 3)  // not mutableListOf unless needed
var count = 0  // use val when possible
```

---

## ขั้นตอนที่ 1728: API Design

```kotlin
// ============================================
// Kotlin API Design Principles
// ============================================

// 1. Named arguments ทำให้ชัดเจน
val circle = Circle(radius = 5.0)  // ดีกว่า Circle(5.0)

// 2. Default parameters ลด overloading
fun sendEmail(
    to: String,
    subject: String,
    body: String,
    cc: List<String> = emptyList(),
    attachments: List<File> = emptyList(),
    priority: Priority = Priority.NORMAL
)

// 3. Builder pattern เมื่อมี optional params มาก
class HttpRequestBuilder {
    private var method: String = "GET"
    private var url: String = ""
    private val headers = mutableMapOf<String, String>()
    private var body: String? = null
    private var timeout: Long = 30_000
    
    fun method(method: String) = apply { this.method = method }
    fun url(url: String) = apply { this.url = url }
    fun header(name: String, value: String) = apply { headers[name] = value }
    fun body(body: String) = apply { this.body = body }
    fun timeout(ms: Long) = apply { this.timeout = ms }
    fun build() = HttpRequest(method, url, headers, body, timeout)
}

// DSL style (elegant)
fun httpRequest(block: HttpRequestBuilder.() -> Unit): HttpRequest {
    return HttpRequestBuilder().apply(block).build()
}

// Usage
val request = httpRequest {
    method("POST")
    url("https://api.example.com/users")
    header("Content-Type", "application/json")
    body("""{"name": "Alice"}""")
    timeout(60_000)
}

// 4. Type-safe builders
class Menu {
    val items = mutableListOf<MenuItem>()
    
    fun item(title: String, action: () -> Unit) {
        items.add(MenuItem(title, action))
    }
    
    fun separator() {
        items.add(MenuItem.Separator)
    }
}

fun menu(block: Menu.() -> Unit): Menu = Menu().apply(block)

// Usage
val contextMenu = menu {
    item("Copy") { clipboard.copy(selected) }
    item("Paste") { clipboard.paste() }
    separator()
    item("Delete") { deleteSelected() }
}
```

---

## ขั้นตอนที่ 1729: Documentation & Knowledge Sharing

```kotlin
// ============================================
// KDoc Comments (เฉพาะ public API)
// ============================================

/**
 * Repository สำหรับจัดการ product data
 *
 * ใช้ offline-first strategy:
 * - Return cached data ทันที
 * - Refresh จาก network ใน background เมื่อ online
 *
 * Example:
 * ```
 * val products by productRepository.observeProducts(categoryId).collectAsStateWithLifecycle()
 * ```
 */
interface ProductRepository {
    
    /**
     * Observe products สำหรับ category ที่กำหนด
     *
     * @param categoryId ID ของ category (0 = ทุก category)
     * @param sortBy ลำดับการเรียง (default: ราคา น้อย → มาก)
     * @return Flow ที่ emit list ของ products เมื่อข้อมูลเปลี่ยน
     */
    fun observeProducts(
        categoryId: Long = 0L,
        sortBy: SortOrder = SortOrder.PRICE_ASC
    ): Flow<List<Product>>
    
    /**
     * เพิ่มสินค้าลงตะกร้า
     *
     * @throws StockException เมื่อสินค้าไม่มีในสต็อก
     * @throws AuthException เมื่อไม่ได้ login
     */
    @Throws(StockException::class, AuthException::class)
    suspend fun addToCart(productId: Long, quantity: Int = 1): Cart
}

// ============================================
// Architecture Decision Records (ADR)
// ============================================

/*
# ADR-001: ใช้ Room แทน Realm

## Status: Accepted

## Context
ต้องการ local database สำหรับ offline-first app

## Decision
ใช้ Room เพราะ:
1. Official Jetpack library - long-term support guaranteed
2. Type-safe queries ด้วย @Query annotation + KSP
3. Integration กับ Flow และ Coroutines
4. Easier testing ด้วย inMemoryDatabaseBuilder

## Consequences
- ต้องเขียน SQL (ไม่ใช่ object-based)
- Schema migration ต้องทำ manually
- Trade-off: มากกว่า Realm สำหรับ complex queries

## Alternatives Considered
- Realm: object-based, auto-sync กับ backend
- SQLDelight: type-safe SQL, KMP compatible
*/
```

---

## ขั้นตอนที่ 1730: Course Summary - World-Class Checklist

```kotlin
object WorldClassAndroidDeveloper {
    
    val foundations = listOf(
        "✅ Kotlin syntax, idioms, null safety",
        "✅ OOP, Functional Programming ใน Kotlin",
        "✅ Coroutines, Flow, StateFlow, SharedFlow",
        "✅ Kotlin Contracts, value classes, KSP"
    )
    
    val android = listOf(
        "✅ Android Lifecycle, Activity/Fragment",
        "✅ Jetpack Compose, state management",
        "✅ Navigation Compose, deep links",
        "✅ ViewModel, LiveData vs StateFlow",
        "✅ Room, DataStore",
        "✅ WorkManager, Foreground Service",
        "✅ CameraX, ML Kit",
        "✅ Google Maps, Location, Geofencing",
        "✅ Firebase FCM, Crashlytics"
    )
    
    val architecture = listOf(
        "✅ Clean Architecture",
        "✅ MVVM / MVI",
        "✅ Repository Pattern, Offline-first",
        "✅ Dependency Injection with Hilt",
        "✅ Modularization",
        "✅ Feature Flags, Analytics",
        "✅ State Machine"
    )
    
    val quality = listOf(
        "✅ Unit Testing, TDD",
        "✅ Integration Testing",
        "✅ UI Testing (Compose)",
        "✅ Screenshot Testing",
        "✅ Property-based Testing",
        "✅ Performance Testing (Macrobenchmark)",
        "✅ Memory Profiling, LeakCanary"
    )
    
    val advanced = listOf(
        "✅ Custom Compose layouts",
        "✅ Custom animations, Shared Element",
        "✅ Kotlin Symbol Processing (KSP)",
        "✅ Custom Gradle Plugins",
        "✅ BLE, NFC",
        "✅ AR Core, On-device ML",
        "✅ Compose Multiplatform",
        "✅ WebSocket, SSE",
        "✅ Payment integration"
    )
    
    val operations = listOf(
        "✅ CI/CD with GitHub Actions",
        "✅ App signing, ProGuard/R8",
        "✅ Fastlane deployment",
        "✅ Baseline Profiles",
        "✅ App security (KeyStore, Biometric)",
        "✅ Certificate pinning",
        "✅ Accessibility (a11y)",
        "✅ Localization"
    )
    
    fun status(): String {
        val total = foundations.size + android.size + architecture.size +
                   quality.size + advanced.size + operations.size
        return """
            🎓 World-Class Android Developer Curriculum
            ==========================================
            Total topics covered: $total
            
            📚 Foundations: ${foundations.size} topics
            📱 Android Specific: ${android.size} topics
            🏛️ Architecture: ${architecture.size} topics
            🧪 Quality: ${quality.size} topics
            ⚡ Advanced: ${advanced.size} topics
            🚀 Operations: ${operations.size} topics
            
            🎉 หลักสูตรระดับ World-Class เสร็จสมบูรณ์!
        """.trimIndent()
    }
}
```

---

## ยินดีด้วย! 🎉

```
คุณได้เรียนรู้ครบ 1750 steps ของหลักสูตร Kotlin & Android Development

จาก:
- Part 1: Hello World
- Part 10: OOP
- Part 20: KMP
- Part 30: Advanced Kotlin
- Part 50: Jetpack Compose
- Part 60: Architecture
- Part 70: World-Class
- Part 80: Advanced
- Part 90: Expert Level

สิ่งที่ต้องทำต่อ:
1. สร้าง Portfolio App จริงๆ ที่ใช้ทุก concept ที่เรียนมา
2. Contribute to Open Source projects
3. สอนคนอื่น (the best way to learn)
4. Follow Android Engineering blog
5. เข้าร่วม community (Kotlin Slack, Thai Android Dev)

เส้นทางต่อไป:
- Android Team Lead
- Staff Engineer  
- Open Source maintainer
- Conference speaker
- Tech blog author
```

---

*Part 90 จบแล้ว - หลักสูตร World-Class เสร็จสมบูรณ์! 🎊*

*ก่อนหน้า: [Part 89](../part89/README.md)*
