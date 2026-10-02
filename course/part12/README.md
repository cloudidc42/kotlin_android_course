# Part 12: Data Class, Sealed Class, Enum
## ขั้นตอนที่ 251-275

---

## ขั้นตอนที่ 251: Data Class

```kotlin
// ============================================
// Data Class - สร้าง class สำหรับเก็บข้อมูล
// ============================================

// data class สร้าง equals(), hashCode(), toString(), copy(), componentN() ให้อัตโนมัติ
data class Person(
    val name: String,
    val age: Int,
    val email: String
)

// เทียบกับ class ธรรมดา
class PersonNormal(
    val name: String,
    val age: Int,
    val email: String
) {
    // ต้องเขียนเองทั้งหมด!
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is PersonNormal) return false
        return name == other.name && age == other.age && email == other.email
    }
    
    override fun hashCode(): Int {
        var result = name.hashCode()
        result = 31 * result + age
        result = 31 * result + email.hashCode()
        return result
    }
    
    override fun toString(): String = "PersonNormal(name=$name, age=$age, email=$email)"
}

fun main() {
    val p1 = Person("สมชาย", 25, "somchai@example.com")
    val p2 = Person("สมชาย", 25, "somchai@example.com")
    val p3 = Person("สมหญิง", 30, "somying@example.com")
    
    // toString() - อัตโนมัติ!
    println(p1)  // Person(name=สมชาย, age=25, email=somchai@example.com)
    
    // equals() - เปรียบเทียบค่า
    println("p1 == p2: ${p1 == p2}")  // true (ค่าเหมือนกัน)
    println("p1 == p3: ${p1 == p3}")  // false
    
    // hashCode() - เท่ากันถ้าค่าเท่ากัน
    println("p1.hashCode() == p2.hashCode(): ${p1.hashCode() == p2.hashCode()}")  // true
    
    // copy() - สร้าง copy พร้อมเปลี่ยนบางค่า
    val p4 = p1.copy(age = 26)
    println("p4 = $p4")  // Person(name=สมชาย, age=26, email=somchai@example.com)
    
    val p5 = p1.copy(name = "สมชาย Junior", email = "new@example.com")
    println("p5 = $p5")
    
    // ============================================
    // Destructuring (componentN())
    // ============================================
    
    val (name, age, email) = p1
    println("\nDestructuring:")
    println("name = $name, age = $age, email = $email")
    
    // ใน for loop
    val people = listOf(
        Person("สมชาย", 25, "a@test.com"),
        Person("สมหญิง", 30, "b@test.com"),
        Person("สมศักดิ์", 28, "c@test.com")
    )
    
    println("\nPeople list:")
    for ((personName, personAge) in people) {
        println("  $personName: $personAge ปี")
    }
    
    // ============================================
    // Data class ใน Set และ Map
    // ============================================
    
    val personSet = setOf(p1, p2, p3)
    println("\nSet size: ${personSet.size}")  // 2 (p1 == p2)
    
    val scoreMap = mapOf(p1 to 90, p3 to 85)
    println("p1's score: ${scoreMap[p1]}")       // 90
    println("p2's score: ${scoreMap[p2]}")       // 90 (p1 == p2)
}
```

---

## ขั้นตอนที่ 252: Data Class ขั้นสูง

```kotlin
// ============================================
// Data Class กับ Properties นอก constructor
// ============================================

data class Product(
    val id: Int,
    val name: String,
    val price: Double,
    val category: String
) {
    // Properties นี้ไม่รวมใน equals/hashCode/toString/copy
    val displayPrice: String get() = "฿${"%.2f".format(price)}"
    val isExpensive: Boolean get() = price > 1000
    
    // Custom toString (override ที่ data class สร้าง)
    // ถ้าต้องการ format พิเศษ
    // override fun toString(): String = "[$id] $name ($displayPrice)"
    
    fun applyDiscount(percent: Double): Product {
        return copy(price = price * (1 - percent / 100))
    }
}

// ============================================
// Nested Data Classes
// ============================================

data class Address(
    val street: String,
    val city: String,
    val province: String,
    val postalCode: String,
    val country: String = "ไทย"
)

data class Customer(
    val id: Long,
    val name: String,
    val email: String,
    val phone: String,
    val address: Address,
    val loyaltyPoints: Int = 0
)

// ============================================
// Data Class กับ Inheritance
// ============================================

// data class ไม่สามารถ inherit จาก class อื่นที่ไม่ใช่ interface
// แต่ implement interface ได้

interface Printable {
    fun print()
}

data class Invoice(
    val invoiceId: String,
    val customer: Customer,
    val items: List<Product>,
    val discount: Double = 0.0
) : Printable {
    val subtotal: Double get() = items.sumOf { it.price }
    val discountAmount: Double get() = subtotal * discount / 100
    val total: Double get() = subtotal - discountAmount
    
    override fun print() {
        println("=".repeat(40))
        println("Invoice: $invoiceId")
        println("Customer: ${customer.name}")
        println("-".repeat(40))
        items.forEachIndexed { i, item ->
            println("${i+1}. ${item.name.padEnd(20)} ${item.displayPrice}")
        }
        println("-".repeat(40))
        println("Subtotal: ${"%.2f".format(subtotal)}")
        if (discount > 0) println("Discount: -${"%.2f".format(discountAmount)} ($discount%)")
        println("Total: ${"%.2f".format(total)}")
        println("=".repeat(40))
    }
}

fun main() {
    val products = listOf(
        Product(1, "MacBook Pro", 85000.0, "Electronics"),
        Product(2, "Magic Mouse", 3500.0, "Accessories"),
        Product(3, "USB Hub", 1200.0, "Accessories")
    )
    
    println("Products:")
    products.forEach { p ->
        println("  ${p.name}: ${p.displayPrice} | Expensive: ${p.isExpensive}")
    }
    
    // apply discount
    val discounted = products.map { it.applyDiscount(10.0) }
    println("\nAfter 10% discount:")
    discounted.forEach { println("  ${it.name}: ${it.displayPrice}") }
    
    // Nested data class
    val address = Address("123 ถ.สุขุมวิท", "กรุงเทพฯ", "กรุงเทพมหานคร", "10110")
    val customer = Customer(1, "สมชาย ใจดี", "somchai@example.com", "0812345678", address)
    
    println("\nCustomer: $customer")
    println("City: ${customer.address.city}")
    
    // Invoice
    val invoice = Invoice("INV-001", customer, products, discount = 5.0)
    println()
    invoice.print()
    
    // Copy with modification
    val updatedCustomer = customer.copy(
        loyaltyPoints = 500,
        address = customer.address.copy(city = "เชียงใหม่")
    )
    println("\nUpdated city: ${updatedCustomer.address.city}")
}
```

---

## ขั้นตอนที่ 253: Enum Class

```kotlin
// ============================================
// Enum Class พื้นฐาน
// ============================================

enum class Direction {
    NORTH, SOUTH, EAST, WEST
}

enum class DayOfWeek {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY;
    
    val isWeekend: Boolean get() = this == SATURDAY || this == SUNDAY
    val isWorkday: Boolean get() = !isWeekend
    
    fun next(): DayOfWeek {
        val values = values()
        return values[(ordinal + 1) % values.size]
    }
}

// Enum ที่มี properties
enum class Planet(val mass: Double, val radius: Double) {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6),
    MARS(6.421e+23, 3.3972e6),
    JUPITER(1.9e+27, 7.1492e7),
    SATURN(5.688e+26, 6.0268e7),
    URANUS(8.686e+25, 2.5559e7),
    NEPTUNE(1.024e+26, 2.4746e7);
    
    companion object {
        const val G = 6.67300E-11  // gravitational constant
    }
    
    fun surfaceGravity(): Double = G * mass / (radius * radius)
    
    fun surfaceWeight(otherMass: Double): Double = otherMass * surfaceGravity()
}

// Enum สำหรับ HTTP Status Codes
enum class HttpStatus(val code: Int, val message: String) {
    OK(200, "OK"),
    CREATED(201, "Created"),
    NO_CONTENT(204, "No Content"),
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    FORBIDDEN(403, "Forbidden"),
    NOT_FOUND(404, "Not Found"),
    INTERNAL_SERVER_ERROR(500, "Internal Server Error"),
    SERVICE_UNAVAILABLE(503, "Service Unavailable");
    
    val isSuccess: Boolean get() = code in 200..299
    val isClientError: Boolean get() = code in 400..499
    val isServerError: Boolean get() = code in 500..599
    
    override fun toString(): String = "$code $message"
    
    companion object {
        fun fromCode(code: Int): HttpStatus? = values().find { it.code == code }
    }
}

fun main() {
    // Direction
    println("Directions: ${Direction.values().toList()}")
    val dir = Direction.NORTH
    println("Direction: $dir (ordinal: ${dir.ordinal}, name: ${dir.name})")
    
    // DayOfWeek
    println("\nDays of week:")
    DayOfWeek.values().forEach { day ->
        val type = if (day.isWeekend) "หยุด" else "ทำงาน"
        println("  $day: $type")
    }
    
    println("\nNext day after FRIDAY: ${DayOfWeek.FRIDAY.next()}")
    
    // Planet weights
    val earthWeight = 75.0
    val mass = earthWeight / Planet.EARTH.surfaceGravity()
    println("\nน้ำหนัก ${earthWeight}kg บนโลก บนดาวอื่น:")
    Planet.values().forEach { planet ->
        println("  ${planet.name}: ${"%.2f".format(planet.surfaceWeight(mass))}kg")
    }
    
    // HTTP Status
    println("\nHTTP Status Codes:")
    val statuses = listOf(HttpStatus.OK, HttpStatus.NOT_FOUND, HttpStatus.INTERNAL_SERVER_ERROR)
    statuses.forEach { status ->
        val type = when {
            status.isSuccess     -> "✅ Success"
            status.isClientError -> "⚠️ Client Error"
            status.isServerError -> "❌ Server Error"
            else -> "ℹ️ Info"
        }
        println("  $status - $type")
    }
    
    // Find by code
    println("\nFind status 404: ${HttpStatus.fromCode(404)}")
    println("Find status 200: ${HttpStatus.fromCode(200)}")
    println("Find status 999: ${HttpStatus.fromCode(999)}")
    
    // Enum ใน when
    fun handleStatus(status: HttpStatus): String = when (status) {
        HttpStatus.OK, HttpStatus.CREATED -> "สำเร็จ!"
        HttpStatus.NOT_FOUND -> "ไม่พบข้อมูล"
        HttpStatus.UNAUTHORIZED, HttpStatus.FORBIDDEN -> "ไม่มีสิทธิ์"
        HttpStatus.INTERNAL_SERVER_ERROR -> "เซิร์ฟเวอร์มีปัญหา"
        else -> "เกิดข้อผิดพลาด: $status"
    }
    
    println("\nHandle statuses:")
    statuses.forEach { println("  ${handleStatus(it)}") }
}
```

---

## ขั้นตอนที่ 254: Sealed Class

```kotlin
// ============================================
// Sealed Class - Restricted Class Hierarchy
// ============================================

// sealed class จำกัดให้ subclass อยู่ในไฟล์เดียวกัน (หรือ same package ใน Kotlin 1.5+)
// ทำให้ when สามารถ exhaustive ได้

sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
    object Loading : Result<Nothing>()
    object Empty : Result<Nothing>()
}

// Handle Result
fun <T> handleResult(result: Result<T>, onSuccess: (T) -> Unit = {}, onError: (String) -> Unit = {}) {
    when (result) {
        is Result.Success -> {
            println("✅ Success: ${result.data}")
            onSuccess(result.data)
        }
        is Result.Error -> {
            println("❌ Error: ${result.message}")
            result.cause?.let { println("  Caused by: ${it.message}") }
            onError(result.message)
        }
        is Result.Loading -> println("⏳ Loading...")
        is Result.Empty -> println("📭 Empty result")
    }
}

// ============================================
// Sealed Class สำหรับ UI State
// ============================================

sealed class UiState<out T> {
    object Idle : UiState<Nothing>()
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
    
    val isLoading: Boolean get() = this is Loading
    val isSuccess: Boolean get() = this is Success
    val isError: Boolean get() = this is Error
}

fun <T> UiState<T>.getDataOrNull(): T? = when (this) {
    is UiState.Success -> data
    else -> null
}

// ============================================
// Sealed Class สำหรับ Navigation Events
// ============================================

sealed class NavigationEvent {
    data class Navigate(val route: String, val args: Map<String, Any> = emptyMap()) : NavigationEvent()
    data class NavigateBack(val result: Any? = null) : NavigationEvent()
    object NavigateToHome : NavigationEvent()
    data class ShowDialog(val title: String, val message: String) : NavigationEvent()
    data class OpenUrl(val url: String) : NavigationEvent()
}

// ============================================
// Sealed Class สำหรับ Domain Events
// ============================================

sealed class UserEvent {
    data class Register(val name: String, val email: String, val password: String) : UserEvent()
    data class Login(val email: String, val password: String) : UserEvent()
    object Logout : UserEvent()
    data class UpdateProfile(val name: String?, val phone: String?) : UserEvent()
    data class ChangePassword(val oldPassword: String, val newPassword: String) : UserEvent()
}

fun processUserEvent(event: UserEvent): String = when (event) {
    is UserEvent.Register -> "สมัครสมาชิก: ${event.name} (${event.email})"
    is UserEvent.Login -> "เข้าสู่ระบบ: ${event.email}"
    is UserEvent.Logout -> "ออกจากระบบ"
    is UserEvent.UpdateProfile -> "อัพเดทโปรไฟล์: ${event.name ?: "ไม่เปลี่ยน"}"
    is UserEvent.ChangePassword -> "เปลี่ยนรหัสผ่าน"
}

fun main() {
    // Result usage
    println("=== Result Sealed Class ===")
    
    val successResult: Result<List<String>> = Result.Success(listOf("A", "B", "C"))
    val errorResult: Result<String> = Result.Error("Network timeout", RuntimeException("Connection refused"))
    val loadingResult: Result<String> = Result.Loading
    val emptyResult: Result<List<String>> = Result.Empty
    
    listOf(successResult, errorResult, loadingResult, emptyResult).forEach { result ->
        handleResult(result)
    }
    
    // UiState
    println("\n=== UiState ===")
    
    val states: List<UiState<String>> = listOf(
        UiState.Idle,
        UiState.Loading,
        UiState.Success("ข้อมูลผู้ใช้"),
        UiState.Error("ไม่สามารถโหลดข้อมูลได้")
    )
    
    states.forEach { state ->
        when (state) {
            is UiState.Idle -> println("Idle - รอรับคำสั่ง")
            is UiState.Loading -> println("Loading...")
            is UiState.Success -> println("Success: ${state.data}")
            is UiState.Error -> println("Error: ${state.message}")
        }
        println("  getDataOrNull(): ${state.getDataOrNull()}")
    }
    
    // Navigation events
    println("\n=== Navigation Events ===")
    
    val events = listOf(
        NavigationEvent.Navigate("profile/123", mapOf("tab" to "posts")),
        NavigationEvent.NavigateBack("result_data"),
        NavigationEvent.NavigateToHome,
        NavigationEvent.ShowDialog("แจ้งเตือน", "คุณต้องการออกจากระบบหรือไม่?")
    )
    
    events.forEach { event ->
        val description = when (event) {
            is NavigationEvent.Navigate -> "Navigate to: ${event.route}"
            is NavigationEvent.NavigateBack -> "Back with: ${event.result}"
            is NavigationEvent.NavigateToHome -> "Go home"
            is NavigationEvent.ShowDialog -> "Dialog: ${event.title}"
            is NavigationEvent.OpenUrl -> "Open: ${event.url}"
        }
        println(description)
    }
    
    // User events
    println("\n=== User Events ===")
    
    val userEvents = listOf(
        UserEvent.Register("สมชาย", "somchai@example.com", "password123"),
        UserEvent.Login("somchai@example.com", "password123"),
        UserEvent.UpdateProfile("สมชาย ใจดี", null),
        UserEvent.Logout
    )
    
    userEvents.forEach { event -> println(processUserEvent(event)) }
}
```

---

## ขั้นตอนที่ 255: Sealed Interface (Kotlin 1.5+)

```kotlin
// ============================================
// Sealed Interface
// ============================================

sealed interface ValidationResult {
    object Valid : ValidationResult
    data class Invalid(val errors: List<String>) : ValidationResult
}

sealed interface NetworkResult<out T> {
    data class Success<T>(val data: T, val statusCode: Int = 200) : NetworkResult<T>
    data class HttpError(val statusCode: Int, val message: String) : NetworkResult<Nothing>
    data class NetworkError(val cause: Throwable) : NetworkResult<Nothing>
    object Timeout : NetworkResult<Nothing>
    object Cancelled : NetworkResult<Nothing>
}

fun <T> NetworkResult<T>.isSuccess(): Boolean = this is NetworkResult.Success
fun <T> NetworkResult<T>.dataOrNull(): T? = (this as? NetworkResult.Success)?.data
fun <T> NetworkResult<T>.errorMessage(): String? = when (this) {
    is NetworkResult.HttpError -> "HTTP $statusCode: $message"
    is NetworkResult.NetworkError -> "Network error: ${cause.message}"
    is NetworkResult.Timeout -> "Request timed out"
    is NetworkResult.Cancelled -> "Request cancelled"
    else -> null
}

// ============================================
// Combining sealed classes
// ============================================

sealed class ApiResponse<out T> {
    data class Success<T>(
        val data: T,
        val meta: Meta = Meta()
    ) : ApiResponse<T>()
    
    data class Failure(
        val error: ApiError
    ) : ApiResponse<Nothing>()
    
    data class Meta(
        val page: Int = 1,
        val total: Int = 0,
        val hasMore: Boolean = false
    )
    
    data class ApiError(
        val code: String,
        val message: String,
        val details: Map<String, String> = emptyMap()
    )
}

fun <T, R> ApiResponse<T>.map(transform: (T) -> R): ApiResponse<R> = when (this) {
    is ApiResponse.Success -> ApiResponse.Success(transform(data), meta)
    is ApiResponse.Failure -> this
}

fun <T> ApiResponse<T>.getOrElse(default: T): T = when (this) {
    is ApiResponse.Success -> data
    else -> default
}

fun main() {
    // ValidationResult
    println("=== Validation ===")
    
    fun validateAge(age: Int): ValidationResult {
        val errors = mutableListOf<String>()
        if (age < 0) errors.add("อายุต้องไม่ติดลบ")
        if (age > 150) errors.add("อายุไม่สมเหตุสมผล")
        return if (errors.isEmpty()) ValidationResult.Valid
        else ValidationResult.Invalid(errors)
    }
    
    listOf(-5, 25, 200).forEach { age ->
        when (val result = validateAge(age)) {
            is ValidationResult.Valid -> println("อายุ $age: ✅ ถูกต้อง")
            is ValidationResult.Invalid -> println("อายุ $age: ❌ ${result.errors}")
        }
    }
    
    // NetworkResult
    println("\n=== Network Results ===")
    
    val results: List<NetworkResult<List<String>>> = listOf(
        NetworkResult.Success(listOf("Item1", "Item2")),
        NetworkResult.HttpError(404, "Not Found"),
        NetworkResult.NetworkError(RuntimeException("Connection refused")),
        NetworkResult.Timeout
    )
    
    results.forEach { result ->
        if (result.isSuccess()) {
            println("✅ Data: ${result.dataOrNull()}")
        } else {
            println("❌ Error: ${result.errorMessage()}")
        }
    }
    
    // ApiResponse
    println("\n=== API Response ===")
    
    val successResponse: ApiResponse<List<String>> = ApiResponse.Success(
        data = listOf("สมชาย", "สมหญิง"),
        meta = ApiResponse.Meta(page = 1, total = 50, hasMore = true)
    )
    
    val failureResponse: ApiResponse<List<String>> = ApiResponse.Failure(
        ApiResponse.ApiError("UNAUTHORIZED", "กรุณาเข้าสู่ระบบ")
    )
    
    listOf(successResponse, failureResponse).forEach { response ->
        when (response) {
            is ApiResponse.Success -> {
                println("Success: ${response.data}")
                println("Meta: page=${response.meta.page}, total=${response.meta.total}")
            }
            is ApiResponse.Failure -> {
                println("Failure: ${response.error.code} - ${response.error.message}")
            }
        }
    }
    
    // map transformation
    val upperResponse = successResponse.map { names -> names.map { it.uppercase() } }
    println("\nMapped: ${upperResponse.getOrElse(emptyList())}")
}
```

---

## แบบฝึกหัด Part 12

```kotlin
// แบบฝึกหัดที่ 1: สร้าง sealed class สำหรับ Payment
sealed class PaymentResult {
    // TODO: สร้าง Success, Declined, InsufficientFunds, NetworkError
}

// แบบฝึกหัดที่ 2: Enum สำหรับ Thai Provinces
enum class ThaiRegion(val thaiName: String) {
    // TODO: สร้าง enum ของภาคในไทย
    // ภาคเหนือ, ภาคกลาง, ภาคใต้, ภาคตะวันออกเฉียงเหนือ, ภาคตะวันออก, ภาคตะวันตก
}

// แบบฝึกหัดที่ 3: Data class Chain
data class CartItem(val product: String, val price: Double, val quantity: Int)
data class Cart(val items: List<CartItem>, val discount: Double = 0.0)

// TODO: Extension function: Cart.total() - คำนวณราคารวมหลังหักส่วนลด
// TODO: Extension function: Cart.summary() - แสดงสรุปตะกร้าสินค้า

fun main() {
    val cart = Cart(
        items = listOf(
            CartItem("MacBook", 85000.0, 1),
            CartItem("Magic Mouse", 3500.0, 2),
            CartItem("USB Hub", 1200.0, 1)
        ),
        discount = 10.0
    )
    // TODO: println(cart.total())
    // TODO: cart.summary()
}
```

### เฉลย

```kotlin
// เฉลย 1
sealed class PaymentResult {
    data class Success(val transactionId: String, val amount: Double) : PaymentResult()
    data class Declined(val reason: String) : PaymentResult()
    data class InsufficientFunds(val balance: Double, val required: Double) : PaymentResult()
    data class NetworkError(val message: String) : PaymentResult()
}

// เฉลย 2
enum class ThaiRegion(val thaiName: String) {
    NORTH("ภาคเหนือ"),
    CENTRAL("ภาคกลาง"),
    SOUTH("ภาคใต้"),
    NORTHEAST("ภาคตะวันออกเฉียงเหนือ"),
    EAST("ภาคตะวันออก"),
    WEST("ภาคตะวันตก")
}

// เฉลย 3
fun Cart.total(): Double {
    val subtotal = items.sumOf { it.price * it.quantity }
    return subtotal * (1 - discount / 100)
}

fun Cart.summary() {
    println("=== สรุปตะกร้าสินค้า ===")
    items.forEach { item ->
        println("${item.product}: ฿${item.price} × ${item.quantity} = ฿${"%.2f".format(item.price * item.quantity)}")
    }
    val subtotal = items.sumOf { it.price * it.quantity }
    println("ราคารวม: ฿${"%.2f".format(subtotal)}")
    if (discount > 0) println("ส่วนลด $discount%: -฿${"%.2f".format(subtotal * discount / 100)}")
    println("สุทธิ: ฿${"%.2f".format(total())}")
}
```

---

*Part 12 จบแล้ว | ก่อนหน้า: [Part 11](../part11/README.md) | ถัดไป: [Part 13](../part13/README.md)*
