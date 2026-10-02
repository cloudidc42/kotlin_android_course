# Part 06: OOP Basics - Class and Object

## ภาพรวม (Overview)
ในส่วนนี้เราจะเรียนรู้พื้นฐานของ Object-Oriented Programming (OOP) ใน Kotlin
ครอบคลุมการสร้างคลาส, constructor, properties, companion object, singleton และ data class

**Steps ที่ครอบคลุม:** 91-120

---

## Step 91: Class Declaration - การประกาศคลาส

```kotlin
// คลาสพื้นฐานใน Kotlin
class Person {
    // Properties (ตัวแปรของคลาส)
    var name: String = ""
    var age: Int = 0

    // Methods (ฟังก์ชันของคลาส)
    fun greet() {
        println("สวัสดี ฉันชื่อ $name อายุ $age ปี")
    }

    fun birthday() {
        age++
        println("Happy Birthday $name! อายุ $age แล้ว")
    }
}

// คลาสที่มี properties เป็น private
class BankAccount {
    private var balance: Double = 0.0
    var owner: String = ""

    fun deposit(amount: Double) {
        if (amount > 0) {
            balance += amount
            println("ฝากเงิน $amount บาท ยอดคงเหลือ: $balance บาท")
        }
    }

    fun withdraw(amount: Double): Boolean {
        return if (amount <= balance) {
            balance -= amount
            println("ถอนเงิน $amount บาท ยอดคงเหลือ: $balance บาท")
            true
        } else {
            println("ยอดเงินไม่เพียงพอ")
            false
        }
    }

    fun getBalance() = balance
}

fun main() {
    // สร้าง instance ของ Person
    val person = Person()
    person.name = "สมชาย"
    person.age = 25
    person.greet()
    person.birthday()

    println()

    // สร้าง BankAccount
    val account = BankAccount()
    account.owner = "สมหญิง"
    account.deposit(5000.0)
    account.deposit(3000.0)
    account.withdraw(2000.0)
    account.withdraw(10000.0)
    println("ยอดเงินของ ${account.owner}: ${account.getBalance()} บาท")
}
```

---

## Step 92: Primary Constructor - Constructor หลัก

```kotlin
// Primary Constructor - ประกาศพร้อมกับชื่อคลาส
class Student(val name: String, val id: Int, var grade: Double) {
    
    fun info() {
        println("ID: $id, ชื่อ: $name, เกรด: $grade")
    }
}

// Primary Constructor พร้อม default values
class Car(
    val brand: String,
    val model: String,
    val year: Int = 2024,
    var mileage: Double = 0.0
) {
    fun drive(km: Double) {
        mileage += km
        println("$brand $model ขับได้ $km กม. รวม: $mileage กม.")
    }
    
    override fun toString() = "$brand $model ($year)"
}

// Constructor ที่มี visibility modifier
class Config private constructor(
    val host: String,
    val port: Int,
    val database: String
) {
    companion object {
        fun create(host: String, port: Int, database: String): Config {
            require(host.isNotBlank()) { "host ต้องไม่ว่าง" }
            require(port in 1..65535) { "port ต้องอยู่ระหว่าง 1-65535" }
            return Config(host, port, database)
        }
    }
    
    override fun toString() = "Config($host:$port/$database)"
}

fun main() {
    // สร้าง Student
    val s1 = Student("สมชาย", 1001, 3.5)
    val s2 = Student("สมหญิง", 1002, 3.8)
    s1.info()
    s2.info()

    // สร้าง Car พร้อม default values
    val car1 = Car("Toyota", "Camry")
    val car2 = Car("Honda", "Civic", 2023, 5000.0)
    println("$car1")
    println("$car2")
    car1.drive(150.5)
    car2.drive(80.0)

    // สร้าง Config ผ่าน factory method
    val config = Config.create("localhost", 5432, "mydb")
    println(config)
    
    // จะ throw exception
    try {
        Config.create("", 5432, "mydb")
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }
}
```

---

## Step 93: Secondary Constructor - Constructor รอง

```kotlin
class Rectangle {
    val width: Double
    val height: Double

    // Primary constructor
    constructor(width: Double, height: Double) {
        this.width = width
        this.height = height
    }

    // Secondary constructors
    constructor(side: Double) : this(side, side)  // สี่เหลี่ยมจัตุรัส

    constructor(widthInt: Int, heightInt: Int) : this(widthInt.toDouble(), heightInt.toDouble())

    fun area() = width * height
    fun perimeter() = 2 * (width + height)

    override fun toString() = "Rectangle($width x $height)"
}

// คลาสที่มีทั้ง primary และ secondary constructor
class Connection(val host: String, val port: Int) {
    var timeout: Int = 30
    var maxRetries: Int = 3
    var secure: Boolean = false

    // Secondary constructor พร้อม extra configuration
    constructor(host: String, port: Int, timeout: Int) : this(host, port) {
        this.timeout = timeout
    }

    constructor(host: String, port: Int, timeout: Int, secure: Boolean) : this(host, port, timeout) {
        this.secure = secure
    }

    override fun toString() = "Connection($host:$port, timeout=$timeout, secure=$secure)"
}

// ใช้ init block กับ constructor
class Product(
    val name: String,
    val price: Double,
    val quantity: Int
) {
    val totalValue: Double
    val category: String

    init {
        // ตรวจสอบค่า
        require(price >= 0) { "ราคาต้องไม่ติดลบ" }
        require(quantity >= 0) { "จำนวนต้องไม่ติดลบ" }
        
        totalValue = price * quantity
        category = when {
            price < 100 -> "ราคาถูก"
            price < 1000 -> "ราคากลาง"
            else -> "ราคาสูง"
        }
    }

    override fun toString() = "$name: ราคา $price, จำนวน $quantity (หมวด: $category)"
}

fun main() {
    val r1 = Rectangle(5.0, 3.0)
    val r2 = Rectangle(4.0)  // สี่เหลี่ยมจัตุรัส
    val r3 = Rectangle(6, 4)

    println("$r1: พื้นที่=${r1.area()}, เส้นรอบ=${r1.perimeter()}")
    println("$r2: พื้นที่=${r2.area()}")
    println("$r3: พื้นที่=${r3.area()}")

    val conn1 = Connection("localhost", 8080)
    val conn2 = Connection("db.example.com", 5432, 60)
    val conn3 = Connection("api.example.com", 443, 30, true)
    println("\n$conn1")
    println("$conn2")
    println("$conn3")

    val p1 = Product("iPhone", 35000.0, 5)
    val p2 = Product("น้ำดื่ม", 15.0, 100)
    println("\n$p1")
    println("$p2")
    println("มูลค่ารวม iPhone: ${p1.totalValue}")
}
```

---

## Step 94: Properties - val, var และ Backing Field

```kotlin
class Temperature(celsius: Double) {
    // Computed properties
    val celsius: Double = celsius
    
    val fahrenheit: Double
        get() = celsius * 9 / 5 + 32
    
    val kelvin: Double
        get() = celsius + 273.15
    
    override fun toString() = "$celsius°C = $fahrenheit°F = $kelvin K"
}

class Circle {
    var radius: Double = 0.0
        set(value) {
            require(value >= 0) { "รัศมีต้องไม่ติดลบ" }
            field = value  // backing field
        }
    
    val area: Double
        get() = Math.PI * radius * radius
    
    val circumference: Double
        get() = 2 * Math.PI * radius
}

class Person {
    var firstName: String = ""
    var lastName: String = ""
    
    val fullName: String
        get() = "$firstName $lastName"
    
    var age: Int = 0
        set(value) {
            require(value >= 0) { "อายุต้องไม่ติดลบ" }
            field = value
        }
    
    val isAdult: Boolean
        get() = age >= 18
    
    private var _email: String = ""
    var email: String
        get() = _email
        set(value) {
            require(value.contains("@")) { "Email ไม่ถูกต้อง" }
            _email = value.lowercase()
        }
}

// Lazy properties
class DatabaseConfig {
    val connectionString: String by lazy {
        println("กำลังโหลด connection string...")
        "postgresql://localhost:5432/mydb"
    }
    
    val tables: List<String> by lazy {
        println("กำลังโหลดรายชื่อตาราง...")
        listOf("users", "products", "orders")
    }
}

fun main() {
    val temp = Temperature(100.0)
    println(temp)
    
    val t2 = Temperature(-40.0)
    println(t2)

    val circle = Circle()
    circle.radius = 5.0
    println("\nวงกลมรัศมี ${circle.radius}")
    println("พื้นที่: ${"%.2f".format(circle.area)}")
    println("เส้นรอบ: ${"%.2f".format(circle.circumference)}")
    
    try {
        circle.radius = -1.0
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }

    val person = Person()
    person.firstName = "สมชาย"
    person.lastName = "ใจดี"
    person.age = 25
    person.email = "SOMCHAI@EXAMPLE.COM"
    
    println("\n${person.fullName}")
    println("email: ${person.email}")
    println("เป็นผู้ใหญ่: ${person.isAdult}")

    // Lazy property
    println("\n--- Lazy Properties ---")
    val db = DatabaseConfig()
    println("สร้าง config แล้ว แต่ยังไม่ได้โหลด")
    println(db.connectionString)  // โหลดตอนนี้
    println(db.connectionString)  // ใช้ค่าที่ cache แล้ว
    println(db.tables)
}
```

---

## Step 95: init Block - บล็อก Initialization

```kotlin
class User(
    val username: String,
    val email: String,
    private val passwordHash: String
) {
    val id: Int
    val createdAt: String
    val displayName: String

    // init block ทำงานหลัง constructor
    init {
        println("กำลังสร้าง User...")
        
        // Validation
        require(username.length >= 3) { "username ต้องมีอย่างน้อย 3 ตัวอักษร" }
        require(email.contains("@")) { "email ไม่ถูกต้อง" }
        
        // Initialize computed values
        id = generateId()
        createdAt = getCurrentTime()
        displayName = username.capitalize()
        
        println("สร้าง User '$displayName' (id=$id) สำเร็จ")
    }

    private fun generateId(): Int = (1000..9999).random()
    private fun getCurrentTime() = "2024-01-01 12:00:00"
    
    fun changePassword(newHash: String): User {
        return User(username, email, newHash)
    }
}

// หลาย init blocks
class DataProcessor(val data: List<Int>) {
    val sum: Int
    val average: Double
    val max: Int
    val min: Int
    val sorted: List<Int>

    init {
        println("init 1: ตรวจสอบข้อมูล")
        require(data.isNotEmpty()) { "data ต้องไม่ว่าง" }
    }

    init {
        println("init 2: คำนวณค่าต่างๆ")
        sum = data.sum()
        average = data.average()
        max = data.max()
        min = data.min()
        sorted = data.sorted()
    }

    init {
        println("init 3: พร้อมใช้งาน")
        println("ข้อมูล: $data")
        println("ผลรวม: $sum, เฉลี่ย: $average")
    }
}

// init กับ inheritance
open class Animal(val name: String) {
    val sound: String

    init {
        println("Animal init: name=$name")
        sound = "..."
    }
}

class Dog(name: String, val breed: String) : Animal(name) {
    init {
        println("Dog init: breed=$breed")
        // sound ของ Animal ถูก set แล้ว แต่ Dog ไม่สามารถ override sound ที่นี่
    }
    
    fun bark() = println("$name ($breed): Woof!")
}

fun main() {
    val user = User("somchai", "somchai@example.com", "hashed_password_123")
    println("User ID: ${user.id}")
    
    println()

    try {
        User("ab", "test@example.com", "hash")
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }

    println()
    val processor = DataProcessor(listOf(5, 2, 8, 1, 9, 3))
    println("sorted: ${processor.sorted}")

    println()
    val dog = Dog("บัดดี้", "Labrador")
    dog.bark()
}
```

---

## Step 96: Companion Object - Object ของคลาส

```kotlin
class MathUtils {
    companion object {
        // Constants
        const val PI = 3.14159265358979
        const val E = 2.71828182845905
        
        // Utility functions
        fun square(n: Double) = n * n
        fun cube(n: Double) = n * n * n
        fun factorial(n: Int): Long {
            require(n >= 0) { "n ต้องไม่ติดลบ" }
            return if (n <= 1) 1L else n * factorial(n - 1)
        }
        fun isPrime(n: Int): Boolean {
            if (n < 2) return false
            return (2..Math.sqrt(n.toDouble()).toInt()).none { n % it == 0 }
        }
    }
}

// Companion object เป็น Factory
class Color private constructor(
    val r: Int,
    val g: Int,
    val b: Int
) {
    val hex: String
        get() = "#%02X%02X%02X".format(r, g, b)

    override fun toString() = "Color(r=$r, g=$g, b=$b) = $hex"

    companion object {
        // Named factories
        val RED = Color(255, 0, 0)
        val GREEN = Color(0, 255, 0)
        val BLUE = Color(0, 0, 255)
        val WHITE = Color(255, 255, 255)
        val BLACK = Color(0, 0, 0)

        fun from(r: Int, g: Int, b: Int): Color {
            require(r in 0..255 && g in 0..255 && b in 0..255) {
                "ค่าสี RGB ต้องอยู่ระหว่าง 0-255"
            }
            return Color(r, g, b)
        }

        fun fromHex(hex: String): Color {
            val h = hex.trimStart('#')
            return Color(
                h.substring(0, 2).toInt(16),
                h.substring(2, 4).toInt(16),
                h.substring(4, 6).toInt(16)
            )
        }

        fun mix(c1: Color, c2: Color): Color {
            return Color(
                (c1.r + c2.r) / 2,
                (c1.g + c2.g) / 2,
                (c1.b + c2.b) / 2
            )
        }
    }
}

// Companion object implement interface
interface JsonSerializable {
    fun toJson(): String
}

class ApiResponse(
    val status: Int,
    val message: String,
    val data: Any?
) {
    companion object : JsonSerializable {
        override fun toJson(): String = "{\"type\": \"ApiResponse\"}"
        
        fun success(data: Any?) = ApiResponse(200, "success", data)
        fun error(message: String) = ApiResponse(500, message, null)
        fun notFound() = ApiResponse(404, "Not Found", null)
    }
    
    fun toJson(): String {
        return """{"status": $status, "message": "$message", "data": "$data"}"""
    }
}

fun main() {
    // ใช้ MathUtils
    println("PI = ${MathUtils.PI}")
    println("5! = ${MathUtils.factorial(5)}")
    println("square(7) = ${MathUtils.square(7.0)}")
    println("isPrime(17) = ${MathUtils.isPrime(17)}")

    // ใช้ Color
    println("\n${Color.RED}")
    println("${Color.BLUE}")
    val custom = Color.from(128, 64, 200)
    println("$custom")
    val fromHex = Color.fromHex("#FF8040")
    println("$fromHex")
    val mixed = Color.mix(Color.RED, Color.BLUE)
    println("mix RED+BLUE: $mixed")

    // ใช้ ApiResponse
    val ok = ApiResponse.success(mapOf("user" to "สมชาย"))
    val err = ApiResponse.error("Database error")
    val nf = ApiResponse.notFound()
    println("\n${ok.toJson()}")
    println("${err.toJson()}")
    println("${nf.toJson()}")
}
```

---

## Step 97: Object Declaration - Singleton

```kotlin
// Object declaration = singleton (มีเพียงตัวเดียวในโปรแกรม)

object Logger {
    private val logs = mutableListOf<String>()
    
    enum class Level { DEBUG, INFO, WARN, ERROR }
    
    var minLevel: Level = Level.DEBUG
    
    fun log(level: Level, message: String) {
        if (level >= minLevel) {
            val entry = "[${level.name}] $message"
            logs.add(entry)
            println(entry)
        }
    }
    
    fun debug(msg: String) = log(Level.DEBUG, msg)
    fun info(msg: String) = log(Level.INFO, msg)
    fun warn(msg: String) = log(Level.WARN, msg)
    fun error(msg: String) = log(Level.ERROR, msg)
    
    fun getLogs() = logs.toList()
    fun clearLogs() = logs.clear()
}

// Singleton สำหรับ Configuration
object AppConfig {
    val appName = "MyApp"
    val version = "1.0.0"
    
    private val properties = mutableMapOf<String, String>(
        "db.host" to "localhost",
        "db.port" to "5432",
        "api.timeout" to "30"
    )
    
    fun get(key: String): String? = properties[key]
    fun set(key: String, value: String) { properties[key] = value }
    fun getAll() = properties.toMap()
}

// Singleton Counter
object Counter {
    private var count = 0
    
    fun increment() = ++count
    fun decrement() = --count
    fun reset() { count = 0 }
    fun value() = count
}

// Object implement interface
interface Greeting {
    fun greet(name: String): String
}

object ThaiGreeting : Greeting {
    override fun greet(name: String) = "สวัสดีครับ/ค่ะ คุณ$name"
}

object EnglishGreeting : Greeting {
    override fun greet(name: String) = "Hello, $name!"
}

fun main() {
    // ใช้ Logger
    Logger.info("โปรแกรมเริ่มต้น")
    Logger.debug("กำลัง debug...")
    Logger.warn("คำเตือน: พื้นที่น้อย")
    Logger.error("เกิดข้อผิดพลาด!")
    
    println("\nLogs ทั้งหมด:")
    Logger.getLogs().forEach { println("  $it") }

    // ใช้ AppConfig
    println("\n--- AppConfig ---")
    println("App: ${AppConfig.appName} v${AppConfig.version}")
    println("DB Host: ${AppConfig.get("db.host")}")
    AppConfig.set("db.host", "db.production.com")
    println("DB Host (updated): ${AppConfig.get("db.host")}")

    // ใช้ Counter
    println("\n--- Counter ---")
    repeat(5) { println("count: ${Counter.increment()}") }
    Counter.decrement()
    println("after decrement: ${Counter.value()}")

    // ใช้ Greeting
    println("\n--- Greetings ---")
    listOf(ThaiGreeting, EnglishGreeting).forEach { greeting ->
        println(greeting.greet("สมชาย"))
    }
    
    // ตรวจสอบว่าเป็น singleton จริงๆ
    println("\nLogger is same instance: ${Logger === Logger}")
    println("AppConfig hashCode: ${AppConfig.hashCode()}")
}
```

---

## Step 98: Object Expression - Anonymous Object

```kotlin
interface EventListener {
    fun onClick(x: Int, y: Int)
    fun onHover(x: Int, y: Int)
}

interface Transformer<T, R> {
    fun transform(input: T): R
}

fun setupButton(listener: EventListener) {
    println("Button setup with listener")
    // จำลองการ trigger events
    listener.onClick(100, 200)
    listener.onHover(150, 250)
}

fun <T, R> processData(data: List<T>, transformer: Transformer<T, R>): List<R> {
    return data.map { transformer.transform(it) }
}

fun main() {
    // Object expression - สร้าง anonymous object
    val listener = object : EventListener {
        override fun onClick(x: Int, y: Int) {
            println("คลิกที่ ($x, $y)")
        }
        
        override fun onHover(x: Int, y: Int) {
            println("hover ที่ ($x, $y)")
        }
    }
    
    setupButton(listener)

    // Anonymous object กับ Transformer
    val numbers = listOf(1, 2, 3, 4, 5)
    val doubled = processData(numbers, object : Transformer<Int, Int> {
        override fun transform(input: Int) = input * 2
    })
    println("\ndoubled: $doubled")

    val words = listOf("hello", "world", "kotlin")
    val upperWords = processData(words, object : Transformer<String, String> {
        override fun transform(input: String) = input.uppercase()
    })
    println("upper: $upperWords")

    // Anonymous object ไม่ inherit interface (plain object expression)
    val point = object {
        val x = 10
        val y = 20
        fun distance() = Math.sqrt((x * x + y * y).toDouble())
    }
    println("\npoint: (${point.x}, ${point.y}), distance=${point.distance()}")

    // Object expression ใน local scope
    fun createComparator(): Comparator<String> {
        return object : Comparator<String> {
            override fun compare(o1: String, o2: String): Int {
                return o1.length.compareTo(o2.length)
            }
        }
    }
    
    val byLength = createComparator()
    val sortedByLength = listOf("kotlin", "is", "awesome", "hi").sortedWith(byLength)
    println("sorted by length: $sortedByLength")

    // ใช้ lambda แทนได้ (SAM conversion - ดูใน Part 10)
    val sortedByLengthLambda = listOf("kotlin", "is", "awesome", "hi")
        .sortedWith(Comparator { a, b -> a.length.compareTo(b.length) })
    println("sorted by length (lambda): $sortedByLengthLambda")
}
```

---

## Step 99: Data Class - คลาสข้อมูล

```kotlin
// Data class - สร้างสำหรับเก็บข้อมูล
// auto-generate: toString, equals, hashCode, copy, componentN

data class Point(val x: Double, val y: Double) {
    fun distanceTo(other: Point): Double {
        val dx = x - other.x
        val dy = y - other.y
        return Math.sqrt(dx * dx + dy * dy)
    }
    
    operator fun plus(other: Point) = Point(x + other.x, y + other.y)
    operator fun minus(other: Point) = Point(x - other.x, y - other.y)
}

data class Address(
    val street: String,
    val city: String,
    val country: String = "Thailand",
    val zipCode: String = ""
)

data class Employee(
    val id: Int,
    val name: String,
    val email: String,
    val address: Address,
    val salary: Double
) {
    val formattedSalary: String
        get() = "฿${"%.2f".format(salary)}"
}

fun main() {
    val p1 = Point(0.0, 0.0)
    val p2 = Point(3.0, 4.0)
    
    // toString auto-generated
    println("p1 = $p1")
    println("p2 = $p2")
    
    // equals auto-generated
    val p3 = Point(3.0, 4.0)
    println("p2 == p3: ${p2 == p3}")
    println("p1 == p3: ${p1 == p3}")
    
    // copy
    val p4 = p2.copy(x = 6.0)
    println("p4 (copy of p2 with x=6): $p4")
    
    // component functions (destructuring)
    val (x, y) = p2
    println("x=$x, y=$y")
    
    // custom functions
    println("distance p1 to p2: ${p1.distanceTo(p2)}")
    println("p1 + p2 = ${p1 + p2}")

    // ซับซ้อนขึ้น
    val emp = Employee(
        id = 1,
        name = "สมชาย ใจดี",
        email = "somchai@company.com",
        address = Address("123 ถ.สุขุมวิท", "กรุงเทพ"),
        salary = 75000.0
    )
    
    println("\n$emp")
    println("เงินเดือน: ${emp.formattedSalary}")
    
    // copy nested data class
    val promoted = emp.copy(
        salary = 90000.0,
        address = emp.address.copy(city = "เชียงใหม่")
    )
    println("\nหลัง promote:")
    println("$promoted")
    println("เงินเดือนใหม่: ${promoted.formattedSalary}")

    // ใช้ใน collection
    val employees = listOf(emp, promoted)
    val byCity = employees.groupBy { it.address.city }
    byCity.forEach { (city, emps) ->
        println("$city: ${emps.map { it.name }}")
    }
}
```

---

## Step 100: Data Class Advanced

```kotlin
data class Version(
    val major: Int,
    val minor: Int,
    val patch: Int = 0
) : Comparable<Version> {
    override fun compareTo(other: Version): Int {
        return when {
            major != other.major -> major.compareTo(other.major)
            minor != other.minor -> minor.compareTo(other.minor)
            else -> patch.compareTo(other.patch)
        }
    }
    
    override fun toString() = "$major.$minor.$patch"
    
    fun isCompatibleWith(other: Version) = major == other.major
}

// Data class ที่มี list
data class ShoppingCart(
    val userId: Int,
    val items: List<CartItem> = emptyList()
) {
    val total: Double get() = items.sumOf { it.total }
    val itemCount: Int get() = items.sumOf { it.quantity }
    
    fun addItem(item: CartItem): ShoppingCart {
        val existing = items.find { it.productId == item.productId }
        return if (existing != null) {
            copy(items = items.map { 
                if (it.productId == item.productId) 
                    it.copy(quantity = it.quantity + item.quantity)
                else it
            })
        } else {
            copy(items = items + item)
        }
    }
    
    fun removeItem(productId: Int): ShoppingCart {
        return copy(items = items.filter { it.productId != productId })
    }
}

data class CartItem(
    val productId: Int,
    val name: String,
    val price: Double,
    val quantity: Int
) {
    val total: Double get() = price * quantity
    
    override fun toString() = "$name x$quantity = ฿${"%.2f".format(total)}"
}

fun main() {
    // Version comparison
    val v1 = Version(1, 0)
    val v2 = Version(1, 2, 3)
    val v3 = Version(2, 0)
    
    val versions = listOf(v3, v1, v2, Version(1, 1))
    println("Versions: ${versions.sorted()}")
    println("Latest: ${versions.max()}")
    println("v1 compatible with v2: ${v1.isCompatibleWith(v2)}")
    println("v1 compatible with v3: ${v1.isCompatibleWith(v3)}")

    // Shopping cart - immutable updates
    var cart = ShoppingCart(userId = 1)
    
    cart = cart.addItem(CartItem(1, "iPhone", 35000.0, 1))
    cart = cart.addItem(CartItem(2, "AirPods", 8000.0, 2))
    cart = cart.addItem(CartItem(1, "iPhone", 35000.0, 1))  // เพิ่ม iPhone อีก 1
    
    println("\nตะกร้าสินค้า:")
    cart.items.forEach { println("  $it") }
    println("จำนวนสินค้า: ${cart.itemCount}")
    println("ยอดรวม: ฿${"%.2f".format(cart.total)}")
    
    cart = cart.removeItem(2)
    println("\nหลังลบ AirPods:")
    cart.items.forEach { println("  $it") }
    println("ยอดรวม: ฿${"%.2f".format(cart.total)}")
    
    // hashCode ใน data class
    val item1 = CartItem(1, "iPhone", 35000.0, 1)
    val item2 = CartItem(1, "iPhone", 35000.0, 1)
    println("\nitem1 == item2: ${item1 == item2}")
    println("hash เท่ากัน: ${item1.hashCode() == item2.hashCode()}")
    println("ใช้ใน Set: ${setOf(item1, item2).size}")  // 1 เพราะเท่ากัน
}
```

---

## Step 101: Nested Class และ Inner Class

```kotlin
class OuterClass(val outerProp: String) {
    
    // Nested class - ไม่เข้าถึง outer class
    class NestedClass {
        fun nested() = println("Nested class (ไม่เห็น outer)")
        // ไม่สามารถเข้าถึง outerProp ได้
    }
    
    // Inner class - เข้าถึง outer class ได้
    inner class InnerClass {
        fun inner() = println("Inner class เห็น outer: $outerProp")
        
        fun outerRef() = this@OuterClass  // อ้างอิง outer
    }
    
    fun createNested() = NestedClass()
    fun createInner() = InnerClass()
}

// Nested class ใช้งานจริง
class Matrix(val rows: Int, val cols: Int) {
    private val data = Array(rows) { DoubleArray(cols) }
    
    operator fun get(row: Int, col: Int) = data[row][col]
    operator fun set(row: Int, col: Int, value: Double) {
        data[row][col] = value
    }
    
    // Nested class เป็น Builder
    class Builder(private val rows: Int, private val cols: Int) {
        private val matrix = Matrix(rows, cols)
        
        fun set(row: Int, col: Int, value: Double): Builder {
            matrix[row, col] = value
            return this
        }
        
        fun identity(): Builder {
            require(rows == cols) { "Identity matrix ต้องเป็น square" }
            for (i in 0 until rows) matrix[i, i] = 1.0
            return this
        }
        
        fun build() = matrix
    }
    
    // Inner class Iterator
    inner class RowIterator : Iterator<DoubleArray> {
        private var currentRow = 0
        
        override fun hasNext() = currentRow < rows
        override fun next() = data[currentRow++]
    }
    
    fun rowIterator() = RowIterator()
    
    override fun toString(): String {
        return data.joinToString("\n") { row ->
            row.joinToString("\t") { "%.1f".format(it) }
        }
    }
}

fun main() {
    // Nested vs Inner
    val outer = OuterClass("ค่าจาก Outer")
    
    val nested = OuterClass.NestedClass()  // สร้างตรงๆ ไม่ต้องผ่าน outer
    nested.nested()
    
    val inner = outer.InnerClass()  // ต้องสร้างผ่าน outer
    inner.inner()

    // Matrix Builder
    val matrix = Matrix.Builder(3, 3)
        .identity()
        .set(0, 1, 5.0)
        .set(1, 2, 3.0)
        .build()
    
    println("\nMatrix:")
    println(matrix)
    
    // ใช้ inner Iterator
    val iter = matrix.rowIterator()
    println("\nRows:")
    while (iter.hasNext()) {
        println(iter.next().toList())
    }
    
    println("\nmatrix[1,2] = ${matrix[1, 2]}")
}
```

---

## Step 102: Extension Functions บนคลาส

```kotlin
data class Money(val amount: Double, val currency: String = "THB") {
    override fun toString() = "%.2f $currency".format(amount)
}

// Extension functions
operator fun Money.plus(other: Money): Money {
    require(currency == other.currency) { "ไม่สามารถบวกเงินต่าง currency" }
    return Money(amount + other.amount, currency)
}

operator fun Money.times(factor: Double) = Money(amount * factor, currency)

fun Money.toUSD(rate: Double = 35.0) = Money(amount / rate, "USD")

// Extension บน String
fun String.toMoney(currency: String = "THB") = Money(toDouble(), currency)

// Extension บน Number
val Double.thb get() = Money(this, "THB")
val Int.thb get() = Money(this.toDouble(), "THB")

data class Email(val to: String, val subject: String, val body: String) {
    companion object {
        fun builder(to: String) = EmailBuilder(to)
    }
}

class EmailBuilder(private val to: String) {
    private var subject: String = ""
    private var body: String = ""
    
    fun subject(s: String): EmailBuilder { subject = s; return this }
    fun body(b: String): EmailBuilder { body = b; return this }
    fun build() = Email(to, subject, body)
}

// Extension สำหรับ builder
fun Email.send() = println("ส่ง email ถึง $to: $subject")
fun Email.preview() = println("Preview:\nTo: $to\nSubject: $subject\n$body")

fun main() {
    val price1 = 150.thb
    val price2 = 250.thb
    val total = price1 + price2
    println("ราคารวม: $total")
    println("ลด 20%: ${total * 0.8}")
    println("เป็น USD: ${total.toUSD()}")
    
    val salary = "75000".toMoney()
    println("เงินเดือน: $salary")

    val email = Email.builder("boss@company.com")
        .subject("รายงานประจำเดือน")
        .body("สวัสดีครับ ขอส่งรายงานประจำเดือนมาด้วยครับ")
        .build()
    
    email.preview()
    email.send()
}
```

---

## Step 103: Class Visibility Modifiers

```kotlin
// Visibility modifiers:
// public (default) - เข้าถึงได้จากทุกที่
// private - เข้าถึงได้เฉพาะในคลาสเดียวกัน
// protected - เข้าถึงได้ในคลาสและ subclasses
// internal - เข้าถึงได้ใน module เดียวกัน

class SecureUser {
    // Public properties
    val username: String
    val email: String
    
    // Private - ซ่อนจากภายนอก
    private val passwordHash: String
    private val secretToken: String
    
    // Internal - ใช้ใน module เดียวกัน
    internal var lastLoginTime: Long = 0
    
    constructor(username: String, email: String, password: String) {
        this.username = username
        this.email = email
        this.passwordHash = hashPassword(password)
        this.secretToken = generateToken()
    }
    
    // Public method
    fun login(password: String): Boolean {
        return if (verifyPassword(password)) {
            lastLoginTime = System.currentTimeMillis()
            println("$username login สำเร็จ")
            true
        } else {
            println("รหัสผ่านไม่ถูกต้อง")
            false
        }
    }
    
    // Private methods - ใช้ภายในคลาสเท่านั้น
    private fun hashPassword(pwd: String) = "hashed_${pwd.length}"
    private fun verifyPassword(pwd: String) = hashPassword(pwd) == passwordHash
    private fun generateToken() = "token_${(1000..9999).random()}"
    
    // Internal method
    internal fun getDebugInfo() = "User($username, token=$secretToken)"
    
    override fun toString() = "User($username, $email)"
}

// Private constructor - ใช้ Factory pattern
class Connection private constructor(
    private val url: String,
    private val poolSize: Int
) {
    private var isConnected = false
    
    companion object {
        private const val MAX_POOL_SIZE = 100
        private const val DEFAULT_POOL_SIZE = 10
        
        fun create(url: String, poolSize: Int = DEFAULT_POOL_SIZE): Connection {
            require(url.isNotBlank()) { "URL ต้องไม่ว่าง" }
            require(poolSize in 1..MAX_POOL_SIZE) { "Pool size ต้องอยู่ระหว่าง 1-$MAX_POOL_SIZE" }
            return Connection(url, poolSize)
        }
    }
    
    fun connect() {
        isConnected = true
        println("Connected to $url (pool: $poolSize)")
    }
    
    fun disconnect() {
        isConnected = false
        println("Disconnected")
    }
    
    val status get() = if (isConnected) "Connected" else "Disconnected"
}

fun main() {
    val user = SecureUser("somchai", "somchai@example.com", "mypassword")
    println("User: $user")
    user.login("wrongpassword")
    user.login("mypassword")
    // user.passwordHash  // Error: private
    // user.secretToken   // Error: private

    val conn = Connection.create("postgresql://localhost/mydb", 20)
    conn.connect()
    println("Status: ${conn.status}")
    conn.disconnect()
    println("Status: ${conn.status}")
}
```

---

## Step 104: ตัวอย่างโปรแกรมจริง - Library System

```kotlin
data class Book(
    val isbn: String,
    val title: String,
    val author: String,
    val year: Int,
    val genre: String
)

data class Member(
    val id: Int,
    val name: String,
    val email: String
)

data class Loan(
    val memberId: Int,
    val isbn: String,
    val borrowDate: String,
    val dueDate: String,
    var returnDate: String? = null
) {
    val isReturned get() = returnDate != null
    val isOverdue get() = !isReturned && dueDate < getCurrentDate()
    
    private fun getCurrentDate() = "2024-01-15"  // จำลองวันที่ปัจจุบัน
}

class Library(val name: String) {
    private val books = mutableMapOf<String, Book>()
    private val members = mutableMapOf<Int, Member>()
    private val loans = mutableListOf<Loan>()
    private var nextMemberId = 1

    // Book management
    fun addBook(book: Book) {
        books[book.isbn] = book
        println("เพิ่มหนังสือ: ${book.title}")
    }

    fun getBook(isbn: String) = books[isbn]
    
    fun searchByTitle(query: String) = books.values.filter { 
        it.title.contains(query, ignoreCase = true) 
    }
    
    fun searchByAuthor(author: String) = books.values.filter {
        it.author.contains(author, ignoreCase = true)
    }
    
    fun getByGenre(genre: String) = books.values.filter { it.genre == genre }

    // Member management
    fun registerMember(name: String, email: String): Member {
        val member = Member(nextMemberId++, name, email)
        members[member.id] = member
        println("ลงทะเบียนสมาชิก: $name (ID: ${member.id})")
        return member
    }

    // Loan management
    fun borrowBook(memberId: Int, isbn: String): Boolean {
        val member = members[memberId] ?: return false.also {
            println("ไม่พบสมาชิก ID: $memberId")
        }
        val book = books[isbn] ?: return false.also {
            println("ไม่พบหนังสือ ISBN: $isbn")
        }
        
        val currentLoan = loans.find { it.isbn == isbn && !it.isReturned }
        if (currentLoan != null) {
            println("${book.title} ถูกยืมอยู่แล้ว")
            return false
        }
        
        loans.add(Loan(memberId, isbn, "2024-01-01", "2024-01-15"))
        println("${member.name} ยืมหนังสือ: ${book.title}")
        return true
    }

    fun returnBook(memberId: Int, isbn: String): Boolean {
        val loan = loans.find { it.memberId == memberId && it.isbn == isbn && !it.isReturned }
            ?: return false.also { println("ไม่พบการยืม") }
        
        loan.returnDate = "2024-01-10"
        val book = books[isbn]!!
        println("${members[memberId]?.name} คืนหนังสือ: ${book.title}")
        return true
    }

    fun getMemberLoans(memberId: Int) = loans.filter { it.memberId == memberId }
    
    fun getAvailableBooks() = books.values.filter { book ->
        loans.none { it.isbn == book.isbn && !it.isReturned }
    }

    fun statistics() {
        println("\n=== $name Statistics ===")
        println("หนังสือทั้งหมด: ${books.size}")
        println("สมาชิกทั้งหมด: ${members.size}")
        println("การยืมทั้งหมด: ${loans.size}")
        println("หนังสือที่ว่าง: ${getAvailableBooks().size}")
        
        val genreCount = books.values.groupBy { it.genre }
            .mapValues { it.value.size }
        println("หนังสือตามหมวด: $genreCount")
    }
}

fun main() {
    val library = Library("ห้องสมุดกลาง")
    
    library.addBook(Book("978-0-07-001-234-5", "Clean Code", "Robert Martin", 2008, "Programming"))
    library.addBook(Book("978-0-07-001-234-6", "The Pragmatic Programmer", "Hunt & Thomas", 1999, "Programming"))
    library.addBook(Book("978-0-07-001-234-7", "Design Patterns", "Gang of Four", 1994, "Programming"))
    library.addBook(Book("978-0-07-001-234-8", "สัตว์หิมะ", "นักเขียนไทย", 2020, "นิยาย"))
    
    val member1 = library.registerMember("สมชาย", "somchai@email.com")
    val member2 = library.registerMember("สมหญิง", "somying@email.com")
    
    library.borrowBook(member1.id, "978-0-07-001-234-5")
    library.borrowBook(member2.id, "978-0-07-001-234-6")
    library.borrowBook(member1.id, "978-0-07-001-234-5")  // ยืมแล้ว
    
    println("\nหนังสือเกี่ยวกับ Programming:")
    library.getByGenre("Programming").forEach { println("  ${it.title}") }
    
    library.returnBook(member1.id, "978-0-07-001-234-5")
    
    library.statistics()
}
```

---

## Step 105: แบบฝึกหัด OOP Basics

```kotlin
// ============================================================
// Exercise 1: สร้างระบบ Inventory
// ============================================================

data class Item(
    val id: Int,
    val name: String,
    val category: String,
    val price: Double,
    var stock: Int
)

class Inventory {
    private val items = mutableMapOf<Int, Item>()
    private var nextId = 1

    fun addItem(name: String, category: String, price: Double, stock: Int): Item {
        val item = Item(nextId++, name, category, price, stock)
        items[item.id] = item
        return item
    }

    fun restock(id: Int, quantity: Int) {
        items[id]?.also { 
            it.stock += quantity
            println("เติมสินค้า ${it.name}: +$quantity = ${it.stock}")
        } ?: println("ไม่พบสินค้า ID: $id")
    }

    fun sell(id: Int, quantity: Int): Boolean {
        val item = items[id] ?: run {
            println("ไม่พบสินค้า ID: $id")
            return false
        }
        return if (item.stock >= quantity) {
            item.stock -= quantity
            println("ขาย ${item.name} x$quantity ยอดคงเหลือ: ${item.stock}")
            true
        } else {
            println("สินค้าไม่พอ (มี: ${item.stock}, ต้องการ: $quantity)")
            false
        }
    }

    fun getLowStock(threshold: Int = 5) = items.values.filter { it.stock <= threshold }
    
    fun getByCategory(category: String) = items.values.filter { it.category == category }
    
    fun totalValue() = items.values.sumOf { it.price * it.stock }
    
    fun report() {
        println("\n=== Inventory Report ===")
        items.values.groupBy { it.category }.forEach { (cat, catItems) ->
            println("$cat:")
            catItems.forEach { println("  ${it.name}: ${it.stock} ชิ้น (฿${it.price})") }
        }
        println("มูลค่ารวม: ฿${"%.2f".format(totalValue())}")
        
        val lowStock = getLowStock()
        if (lowStock.isNotEmpty()) {
            println("\nสินค้าใกล้หมด: ${lowStock.map { it.name }}")
        }
    }
}

fun main() {
    val inv = Inventory()
    
    inv.addItem("iPhone 15", "Mobile", 35000.0, 20)
    inv.addItem("Samsung S24", "Mobile", 28000.0, 15)
    inv.addItem("MacBook Pro", "Laptop", 65000.0, 5)
    inv.addItem("AirPods Pro", "Accessories", 8500.0, 30)
    inv.addItem("iPad Air", "Tablet", 25000.0, 8)
    
    inv.sell(1, 5)
    inv.sell(3, 4)
    inv.sell(3, 3)  // เหลือน้อยกว่า threshold
    inv.restock(3, 10)
    
    inv.report()
    
    println("\nสินค้ามือถือ:")
    inv.getByCategory("Mobile").forEach { println("  ${it.name}: ${it.stock}") }
    
    // ============================================================
    // Exercise 2: Builder Pattern
    // ============================================================
    
    data class Pizza(
        val size: String,
        val crust: String,
        val toppings: List<String>,
        val extraCheese: Boolean,
        val price: Double
    ) {
        override fun toString() = buildString {
            append("Pizza $size ($crust crust)")
            if (toppings.isNotEmpty()) append(", toppings: ${toppings.joinToString()}")
            if (extraCheese) append(", +extra cheese")
            append(", price: ฿$price")
        }
    }

    class PizzaBuilder {
        private var size: String = "Medium"
        private var crust: String = "Thin"
        private val toppings = mutableListOf<String>()
        private var extraCheese = false

        fun small() = apply { size = "Small" }
        fun medium() = apply { size = "Medium" }
        fun large() = apply { size = "Large" }
        fun thinCrust() = apply { crust = "Thin" }
        fun thickCrust() = apply { crust = "Thick" }
        fun stuffedCrust() = apply { crust = "Stuffed" }
        fun topping(t: String) = apply { toppings.add(t) }
        fun extraCheese() = apply { extraCheese = true }
        
        fun build(): Pizza {
            val basePrice = when (size) { "Small" -> 150.0; "Large" -> 350.0; else -> 250.0 }
            val crustPrice = if (crust == "Stuffed") 50.0 else 0.0
            val toppingPrice = toppings.size * 30.0
            val cheesePrice = if (extraCheese) 40.0 else 0.0
            return Pizza(size, crust, toppings.toList(), extraCheese, 
                        basePrice + crustPrice + toppingPrice + cheesePrice)
        }
    }

    println("\n--- Pizza Builder ---")
    val pizza1 = PizzaBuilder()
        .large()
        .stuffedCrust()
        .topping("Pepperoni")
        .topping("Mushroom")
        .topping("Olive")
        .extraCheese()
        .build()
    
    val pizza2 = PizzaBuilder()
        .small()
        .thinCrust()
        .topping("Cheese")
        .build()
    
    println("Pizza 1: $pizza1")
    println("Pizza 2: $pizza2")
}
```

---

## สรุปส่วนที่ 6 (Summary)

| Concept | ใช้เมื่อไหร่ |
|---------|------------|
| `class` | สร้าง blueprint ของ object |
| Primary constructor | parameter หลักของคลาส |
| Secondary constructor | สร้าง alternative ways ในการสร้าง object |
| `init` block | code ที่รันหลัง constructor |
| `val/var` property | เก็บ state ของ object |
| `companion object` | static members, factory methods |
| `object` declaration | Singleton |
| `object` expression | Anonymous object (implement interface) |
| `data class` | DTO, value objects |
| Nested class | Helper class ที่ไม่ต้องการ outer reference |
| `inner class` | Helper class ที่ต้องการ outer reference |

---

[← Part 05: Collections](../part05/README.md) | [Part 07: Inheritance →](../part07/README.md)
