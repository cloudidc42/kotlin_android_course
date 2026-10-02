# Part 11: Extension Functions
## ขั้นตอนที่ 231-250

---

## ขั้นตอนที่ 231: Extension Functions คืออะไร?

Extension Functions ช่วยให้เราเพิ่มฟังก์ชันให้กับ class ที่มีอยู่แล้ว **โดยไม่ต้องแก้ไข source code ของ class นั้น**

```kotlin
// ============================================
// รูปแบบ: fun ClassName.functionName(params): ReturnType { }
// ============================================

// เพิ่มฟังก์ชันให้ String
fun String.addExclamation(): String = "$this!"

fun String.wordCount(): Int = if (this.isBlank()) 0 else this.trim().split("\\s+".toRegex()).size

fun String.isPalindrome(): Boolean {
    val clean = this.lowercase().filter { it.isLetterOrDigit() }
    return clean == clean.reversed()
}

fun String.capitalizeWords(): String {
    return this.split(" ").joinToString(" ") { word ->
        word.replaceFirstChar { it.uppercase() }
    }
}

fun String.truncate(maxLength: Int, suffix: String = "..."): String {
    return if (this.length <= maxLength) this
    else this.take(maxLength - suffix.length) + suffix
}

// เพิ่มฟังก์ชันให้ Int
fun Int.factorial(): Long {
    if (this < 0) throw IllegalArgumentException("ต้องไม่ติดลบ")
    return (1..this).fold(1L) { acc, n -> acc * n }
}

fun Int.isPrime(): Boolean {
    if (this < 2) return false
    if (this == 2) return true
    if (this % 2 == 0) return false
    for (i in 3..Math.sqrt(this.toDouble()).toInt() step 2) {
        if (this % i == 0) return false
    }
    return true
}

fun Int.toBinary(): String = Integer.toBinaryString(this)
fun Int.toHex(): String = Integer.toHexString(this).uppercase()

// เพิ่มฟังก์ชันให้ Double
fun Double.round(decimals: Int): Double {
    val factor = Math.pow(10.0, decimals.toDouble())
    return Math.round(this * factor) / factor
}

fun Double.toCurrency(symbol: String = "฿"): String = 
    "$symbol${"%.2f".format(this)}"

fun main() {
    // String extensions
    println("Hello".addExclamation())              // Hello!
    println("สวัสดี Kotlin".wordCount())           // 2
    println("racecar".isPalindrome())              // true
    println("hello world kotlin".capitalizeWords()) // Hello World Kotlin
    println("This is a very long text here".truncate(20))  // This is a very l...
    
    // Int extensions
    println(5.factorial())    // 120
    println(7.isPrime())      // true
    println(42.toBinary())    // 101010
    println(255.toHex())      // FF
    
    // Double extensions
    println(3.14159265.round(2))   // 3.14
    println(1234.56.toCurrency())  // ฿1234.56
    println(99.99.toCurrency("$")) // $99.99
}
```

---

## ขั้นตอนที่ 232: Extension Functions กับ Collections

```kotlin
// ============================================
// Extension Functions สำหรับ Collections
// ============================================

// Extension ให้ List
fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null

fun <T> List<T>.swap(i: Int, j: Int): List<T> {
    val result = this.toMutableList()
    val temp = result[i]
    result[i] = result[j]
    result[j] = temp
    return result
}

fun <T> List<T>.chunk(size: Int): List<List<T>> {
    return this.chunked(size)
}

fun <T> List<T>.rotate(positions: Int): List<T> {
    if (isEmpty()) return this
    val n = positions % size
    return if (n == 0) this
    else drop(n) + take(n)
}

// Extension ให้ Map
fun <K, V> Map<K, V>.getOrDefault(key: K, default: V): V = this[key] ?: default

fun <K, V> Map<K, V>.filterValues(predicate: (V) -> Boolean): Map<K, V> =
    this.filter { (_, v) -> predicate(v) }

fun <K, V, R> Map<K, V>.mapValues(transform: (V) -> R): Map<K, R> =
    this.mapValues { (_, v) -> transform(v) }

// Statistics extensions
fun List<Int>.median(): Double {
    val sorted = this.sorted()
    return if (size % 2 == 0) {
        (sorted[size / 2 - 1] + sorted[size / 2]) / 2.0
    } else {
        sorted[size / 2].toDouble()
    }
}

fun List<Double>.standardDeviation(): Double {
    val mean = this.average()
    val variance = this.map { (it - mean) * (it - mean) }.average()
    return Math.sqrt(variance)
}

fun List<Int>.mode(): List<Int> {
    val freq = this.groupBy { it }.mapValues { it.value.size }
    val maxFreq = freq.values.maxOrNull() ?: return emptyList()
    return freq.filter { it.value == maxFreq }.keys.toList().sorted()
}

fun main() {
    val numbers = listOf(5, 2, 8, 1, 9, 3, 7, 4, 6, 10)
    
    println("List: $numbers")
    println("second: ${numbers.secondOrNull()}")
    println("swap(0,9): ${numbers.swap(0, 9)}")
    println("chunk(3): ${numbers.chunk(3)}")
    println("rotate(3): ${numbers.rotate(3)}")
    
    val scores = listOf(85, 92, 78, 95, 88, 72, 95, 88, 65)
    println("\nScores: $scores")
    println("Median: ${scores.median()}")
    println("Mode: ${scores.mode()}")
    println("StdDev: ${"%.2f".format(scores.map { it.toDouble() }.standardDeviation())}")
    
    val config = mapOf("host" to "localhost", "port" to "8080", "debug" to "true")
    println("\nConfig port: ${config.getOrDefault("port", "80")}")
    println("Missing key: ${config.getOrDefault("timeout", "30")}")
}
```

---

## ขั้นตอนที่ 233: Extension Properties

```kotlin
// ============================================
// Extension Properties
// ============================================

// Extension Properties สำหรับ String
val String.firstChar: Char? get() = if (isNotEmpty()) this[0] else null
val String.lastChar: Char? get() = if (isNotEmpty()) this[length - 1] else null
val String.isEmail: Boolean get() = matches(Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"))
val String.isPhoneNumber: Boolean get() = matches(Regex("^[0-9]{9,10}$"))
val String.isUrl: Boolean get() = startsWith("http://") || startsWith("https://")

// Extension Properties สำหรับ Int
val Int.isEven: Boolean get() = this % 2 == 0
val Int.isOdd: Boolean get() = this % 2 != 0
val Int.isPositive: Boolean get() = this > 0
val Int.isNegative: Boolean get() = this < 0
val Int.absoluteValue: Int get() = Math.abs(this)

// Unit conversion extensions
val Int.seconds: Long get() = this * 1000L
val Int.minutes: Long get() = this * 60 * 1000L
val Int.hours: Long get() = this * 60 * 60 * 1000L
val Int.days: Long get() = this * 24 * 60 * 60 * 1000L

val Double.kg: Double get() = this
val Double.pounds: Double get() = this * 0.453592
val Double.grams: Double get() = this * 1000

val Double.km: Double get() = this
val Double.miles: Double get() = this * 1.60934
val Double.meters: Double get() = this * 1000

fun main() {
    println("'K'.firstChar: ${"Kotlin".firstChar}")   // K
    println("'n'.lastChar: ${"Kotlin".lastChar}")      // n
    println("email valid: ${"user@example.com".isEmail}")  // true
    println("phone valid: ${"0812345678".isPhoneNumber}")   // true
    
    println("\n5.isEven: ${5.isEven}")     // false
    println("4.isEven: ${4.isEven}")       // true
    println("-3.absoluteValue: ${(-3).absoluteValue}")  // 3
    
    println("\nTime conversions:")
    println("2 hours = ${2.hours} ms")    // 7200000
    println("30 minutes = ${30.minutes} ms")  // 1800000
    
    println("\nWeight conversions:")
    println("70 kg = ${70.0.kg} kg")
    println("70 kg = ${"%.2f".format(70.0.kg / 0.453592)} pounds")
    
    println("\nDistance conversions:")
    println("5 km = ${5.0.km} km")
    println("5 km = ${"%.2f".format(5.0.km / 1.60934)} miles")
}
```

---

## ขั้นตอนที่ 234: Extension Functions สำหรับ Android

```kotlin
// extensions/ContextExtensions.kt
// (ตัวอย่าง Android Extensions - ใช้ใน Android project)

/*
// Context extensions
fun Context.showToast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}

fun Context.showLongToast(message: String) = showToast(message, Toast.LENGTH_LONG)

fun Context.getColorCompat(@ColorRes colorRes: Int): Int {
    return ContextCompat.getColor(this, colorRes)
}

fun Context.getDrawableCompat(@DrawableRes drawableRes: Int): Drawable? {
    return ContextCompat.getDrawable(this, drawableRes)
}

fun Context.dpToPx(dp: Float): Float {
    return dp * resources.displayMetrics.density
}

fun Context.pxToDp(px: Float): Float {
    return px / resources.displayMetrics.density
}

fun Context.isNetworkAvailable(): Boolean {
    val connectivityManager = getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    val network = connectivityManager.activeNetwork ?: return false
    val capabilities = connectivityManager.getNetworkCapabilities(network) ?: return false
    return capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
}

// View extensions
fun View.show() { visibility = View.VISIBLE }
fun View.hide() { visibility = View.GONE }
fun View.invisible() { visibility = View.INVISIBLE }
fun View.isVisible(): Boolean = visibility == View.VISIBLE

fun View.setOnSafeClickListener(debounceMs: Long = 500, onClick: (View) -> Unit) {
    var lastClickTime = 0L
    setOnClickListener {
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastClickTime >= debounceMs) {
            lastClickTime = currentTime
            onClick(it)
        }
    }
}

// EditText extensions
fun EditText.text(): String = this.text.toString()
fun EditText.isEmpty(): Boolean = text().isEmpty()
fun EditText.isNotEmpty(): Boolean = text().isNotEmpty()
fun EditText.clear() { setText("") }
fun EditText.trimText(): String = text().trim()

// ImageView extensions
fun ImageView.loadUrl(url: String) {
    Glide.with(this).load(url).into(this)
}

// Fragment extensions
fun Fragment.showToast(message: String) {
    context?.showToast(message)
}

// Activity extensions
fun AppCompatActivity.replaceFragment(containerId: Int, fragment: Fragment) {
    supportFragmentManager.beginTransaction()
        .replace(containerId, fragment)
        .commit()
}
*/

// จำลองการใช้งานใน Pure Kotlin
class MockView(var visibility: String = "VISIBLE")
class MockContext

fun MockView.show() { visibility = "VISIBLE" }
fun MockView.hide() { visibility = "GONE" }
fun MockView.isVisible(): Boolean = visibility == "VISIBLE"

fun MockContext.dpToPx(dp: Float, density: Float = 3.0f): Float = dp * density
fun MockContext.pxToDp(px: Float, density: Float = 3.0f): Float = px / density

fun main() {
    val view = MockView()
    println("Initial visibility: ${view.visibility}")
    
    view.hide()
    println("After hide(): ${view.visibility}")
    println("isVisible: ${view.isVisible()}")
    
    view.show()
    println("After show(): ${view.visibility}")
    
    val context = MockContext()
    println("\n24dp in px: ${context.dpToPx(24f)} px")
    println("72px in dp: ${context.pxToDp(72f)} dp")
}
```

---

## ขั้นตอนที่ 235: Extension Functions ขั้นสูง

```kotlin
// ============================================
// Generic Extension Functions
// ============================================

// Print with label
fun <T> T.printLabeled(label: String): T {
    println("$label: $this")
    return this  // return ตัวเอง เพื่อ chaining
}

// Conditional apply
fun <T> T.applyIf(condition: Boolean, block: T.() -> T): T {
    return if (condition) block() else this
}

// Transform with fallback
fun <T, R> T?.mapOrDefault(default: R, transform: (T) -> R): R {
    return if (this != null) transform(this) else default
}

// Validate and transform
fun <T> T.validate(
    predicate: (T) -> Boolean,
    errorMessage: String = "Validation failed"
): T {
    if (!predicate(this)) throw IllegalArgumentException(errorMessage)
    return this
}

// Try or null
fun <T> T.tryOrNull(block: T.() -> T): T? {
    return try { block() } catch (e: Exception) { null }
}

// ============================================
// Builder pattern ด้วย Extension Functions
// ============================================

data class HttpRequest(
    val url: String = "",
    val method: String = "GET",
    val headers: Map<String, String> = emptyMap(),
    val body: String? = null,
    val timeout: Int = 30
)

class HttpRequestBuilder {
    var url: String = ""
    var method: String = "GET"
    val headers: MutableMap<String, String> = mutableMapOf()
    var body: String? = null
    var timeout: Int = 30
    
    fun header(key: String, value: String) { headers[key] = value }
    
    fun build(): HttpRequest = HttpRequest(url, method, headers.toMap(), body, timeout)
}

fun httpRequest(block: HttpRequestBuilder.() -> Unit): HttpRequest {
    return HttpRequestBuilder().apply(block).build()
}

fun main() {
    // Generic extensions
    val result = 42
        .printLabeled("Original")
        .applyIf(true) { this * 2 }
        .printLabeled("After *2")
        .applyIf(false) { this + 100 }  // ไม่ทำ เพราะ condition false
        .printLabeled("Final")
    
    println()
    
    val nullStr: String? = null
    val validStr: String? = "hello"
    
    println(nullStr.mapOrDefault("default") { it.uppercase() })   // default
    println(validStr.mapOrDefault("default") { it.uppercase() })  // HELLO
    
    // Validate
    try {
        val age = 25.validate({ it >= 18 }, "ต้องอายุ 18 ปีขึ้นไป")
        println("\nAge $age is valid")
        
        val youngAge = 15.validate({ it >= 18 }, "ต้องอายุ 18 ปีขึ้นไป")
    } catch (e: IllegalArgumentException) {
        println("Validation error: ${e.message}")
    }
    
    // Builder pattern
    val request = httpRequest {
        url = "https://api.example.com/users"
        method = "POST"
        header("Content-Type", "application/json")
        header("Authorization", "Bearer token123")
        body = """{"name": "สมชาย", "age": 25}"""
        timeout = 60
    }
    
    println("\nHTTP Request:")
    println("  URL: ${request.url}")
    println("  Method: ${request.method}")
    println("  Headers: ${request.headers}")
    println("  Body: ${request.body}")
    println("  Timeout: ${request.timeout}s")
}
```

---

## ขั้นตอนที่ 236-240: Extension ที่ใช้บ่อยใน Production

```kotlin
import java.text.SimpleDateFormat
import java.util.Date

// ============================================
// Date Extensions
// ============================================

fun Date.format(pattern: String = "dd/MM/yyyy"): String {
    return SimpleDateFormat(pattern).format(this)
}

fun Date.toISOString(): String {
    return SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'").format(this)
}

// ============================================
// Number Formatting Extensions
// ============================================

fun Number.formatWithCommas(): String {
    return String.format("%,.0f", this.toDouble())
}

fun Double.toPercent(decimals: Int = 1): String {
    return "${"%.${decimals}f".format(this * 100)}%"
}

fun Long.toReadableSize(): String = when {
    this < 1024 -> "$this B"
    this < 1024 * 1024 -> "${"%.1f".format(this / 1024.0)} KB"
    this < 1024 * 1024 * 1024 -> "${"%.1f".format(this / (1024.0 * 1024))} MB"
    else -> "${"%.1f".format(this / (1024.0 * 1024 * 1024))} GB"
}

fun Long.toReadableDuration(): String {
    val seconds = this / 1000
    val minutes = seconds / 60
    val hours = minutes / 60
    val days = hours / 24
    
    return when {
        days > 0 -> "${days}d ${hours % 24}h"
        hours > 0 -> "${hours}h ${minutes % 60}m"
        minutes > 0 -> "${minutes}m ${seconds % 60}s"
        else -> "${seconds}s"
    }
}

// ============================================
// Collection Extensions สำหรับ Production
// ============================================

fun <T> List<T>.safeGet(index: Int): T? =
    if (index in 0 until size) this[index] else null

fun <K, V> MutableMap<K, MutableList<V>>.addToList(key: K, value: V) {
    getOrPut(key) { mutableListOf() }.add(value)
}

fun <T> List<T>.sumOf(selector: (T) -> Double): Double =
    fold(0.0) { acc, t -> acc + selector(t) }

fun <T> Iterable<T>.countBy(selector: (T) -> Boolean): Int =
    count(selector)

// ============================================
// String Extensions สำหรับ Production
// ============================================

fun String.toSlug(): String {
    return this.lowercase()
        .replace("[^a-z0-9\\s-]".toRegex(), "")
        .replace("\\s+".toRegex(), "-")
        .replace("-+".toRegex(), "-")
        .trim('-')
}

fun String.maskEmail(): String {
    if (!this.isEmail) return this
    val (local, domain) = this.split("@")
    val masked = if (local.length <= 2) local
    else local[0] + "*".repeat(local.length - 2) + local.last()
    return "$masked@$domain"
}

val String.isEmail: Boolean get() = 
    matches(Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"))

fun String.toTitleCase(): String = split(" ").joinToString(" ") { word ->
    word.lowercase().replaceFirstChar { it.uppercase() }
}

fun main() {
    // Date
    val now = Date()
    println("Date: ${now.format()}")
    println("ISO: ${now.toISOString()}")
    
    // Number formatting
    println("\nNumber Formatting:")
    println("1234567.formatWithCommas() = ${1234567.formatWithCommas()}")
    println("0.1567.toPercent() = ${0.1567.toPercent()}")
    println("0.1567.toPercent(2) = ${0.1567.toPercent(2)}")
    
    println("\nFile Sizes:")
    println("512.toReadableSize() = ${512L.toReadableSize()}")
    println("1536.toReadableSize() = ${1536L.toReadableSize()}")
    println("(1.5MB).toReadableSize() = ${(1536L * 1024).toReadableSize()}")
    println("(2GB).toReadableSize() = ${(2L * 1024 * 1024 * 1024).toReadableSize()}")
    
    println("\nDurations:")
    println("90000ms = ${90000L.toReadableDuration()}")
    println("3661000ms = ${3661000L.toReadableDuration()}")
    
    // String
    println("\nString Extensions:")
    println("\"Hello World Kotlin\".toSlug() = ${"Hello World Kotlin!".toSlug()}")
    println("email mask: ${"user@example.com".maskEmail()}")
    println("'the quick brown fox'.toTitleCase() = ${"the quick brown fox".toTitleCase()}")
}
```

---

## แบบฝึกหัด Part 11

```kotlin
// แบบฝึกหัดที่ 1: String Validation Extensions
// สร้าง extension properties สำหรับ String

// TODO: isValidPassword - ต้องยาวอย่างน้อย 8 ตัวอักษร มีตัวพิมพ์ใหญ่ ตัวพิมพ์เล็ก ตัวเลข และอักขระพิเศษ
// TODO: isThaiText - ตรวจสอบว่าเป็นตัวอักษรไทยทั้งหมด
// TODO: removeSpecialChars - ลบอักขระพิเศษทั้งหมด

// แบบฝึกหัดที่ 2: Number Extensions
// TODO: Int.toRoman() - แปลง Int เป็น Roman numerals (1-3999)
// TODO: Int.toBinaryString() - แปลงเป็น binary string พร้อม padding
// TODO: Double.clamp(min, max) - จำกัดค่าระหว่าง min และ max

// แบบฝึกหัดที่ 3: Collection Extensions
// TODO: List<T>.circularGet(index) - ดึงค่าแบบ circular (index เกินขนาด list ก็ยังทำงานได้)
// TODO: List<T>.cartesianProduct(other) - Cartesian product ของสอง lists

fun main() {
    // ทดสอบเมื่อทำแบบฝึกหัดเสร็จ
    // println("StrongP@ss1".isValidPassword)  // true
    // println("weak".isValidPassword)         // false
    // println(14.toRoman())                    // XIV
    // println(2024.toRoman())                  // MMXXIV
    // println(3.14.clamp(0.0, 3.0))           // 3.0
    // println(listOf("A","B","C").circularGet(5))  // C (5 % 3 = 2)
}
```

### เฉลย

```kotlin
// เฉลย 1: String Extensions
val String.isValidPassword: Boolean get() {
    return length >= 8 &&
        any { it.isUpperCase() } &&
        any { it.isLowerCase() } &&
        any { it.isDigit() } &&
        any { "!@#$%^&*()_+-=[]{}|;':\",./<>?".contains(it) }
}

val String.isThaiText: Boolean get() =
    this.all { it in '฀'..'๿' || it.isWhitespace() }

fun String.removeSpecialChars(): String =
    this.filter { it.isLetterOrDigit() || it.isWhitespace() }

// เฉลย 2: Number Extensions
fun Int.toRoman(): String {
    val values = intArrayOf(1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1)
    val symbols = arrayOf("M", "CM", "D", "CD", "C", "XC", "L", "XL", "X", "IX", "V", "IV", "I")
    
    var remaining = this
    return buildString {
        for (i in values.indices) {
            while (remaining >= values[i]) {
                append(symbols[i])
                remaining -= values[i]
            }
        }
    }
}

fun Double.clamp(min: Double, max: Double): Double = 
    maxOf(min, minOf(max, this))

// เฉลย 3: Collection Extensions
fun <T> List<T>.circularGet(index: Int): T {
    if (isEmpty()) throw IndexOutOfBoundsException("List is empty")
    return this[((index % size) + size) % size]
}

fun <A, B> List<A>.cartesianProduct(other: List<B>): List<Pair<A, B>> =
    this.flatMap { a -> other.map { b -> Pair(a, b) } }
```

---

*Part 11 จบแล้ว | ก่อนหน้า: [Part 10](../part10/README.md) | ถัดไป: [Part 12](../part12/README.md)*
