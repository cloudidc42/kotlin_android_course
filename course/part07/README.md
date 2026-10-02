# Part 07: Inheritance and Polymorphism

## ภาพรวม (Overview)
ในส่วนนี้เราจะเรียนรู้เกี่ยวกับ Inheritance (การสืบทอด) และ Polymorphism (พหุสัณฐาน) ใน Kotlin
ครอบคลุม open class, abstract class, override, super และ sealed class เบื้องต้น

**Steps ที่ครอบคลุม:** 121-150

---

## Step 121: การสืบทอด (Inheritance) พื้นฐาน

```kotlin
// ใน Kotlin ทุก class เป็น final โดย default
// ต้องใส่ 'open' ถ้าต้องการให้ class อื่น inherit ได้

open class Animal(val name: String, val sound: String) {
    open fun makeSound() {
        println("$name พูดว่า: $sound")
    }
    
    open fun describe() {
        println("ฉันคือ $name")
    }
}

// Dog สืบทอดจาก Animal
class Dog(name: String) : Animal(name, "โฮ่ง") {
    override fun makeSound() {
        println("$name เห่าว่า: $sound $sound!")
    }
    
    fun fetch(item: String) {
        println("$name วิ่งไปเอา $item มา")
    }
}

// Cat สืบทอดจาก Animal
class Cat(name: String, val isIndoor: Boolean) : Animal(name, "เมี้ยว") {
    override fun makeSound() {
        println("$name ร้องว่า: $sound~")
    }
    
    override fun describe() {
        super.describe()  // เรียก parent's describe()
        println("ฉันเป็นแมว${if (isIndoor) "ในบ้าน" else "นอกบ้าน"}")
    }
}

// Bird สืบทอดจาก Animal
class Bird(name: String, val canFly: Boolean) : Animal(name, "จิ๊บ") {
    override fun describe() {
        super.describe()
        println("ฉัน${if (canFly) "บินได้" else "บินไม่ได้"}")
    }
}

fun main() {
    val dog = Dog("บัดดี้")
    val cat = Cat("วิสกี้", true)
    val bird = Bird("ทวีทวี", true)
    val penguin = Bird("เพนกวิน", false)

    // เรียกใช้ method ที่ override
    dog.makeSound()
    cat.makeSound()
    bird.makeSound()
    
    println()
    
    // เรียก describe ที่มี super call
    cat.describe()
    println()
    bird.describe()
    
    println()
    
    // Polymorphism - ตัวแปรประเภท Animal ชี้ไปที่ object ใดก็ได้
    val animals: List<Animal> = listOf(dog, cat, bird, penguin, Dog("แม็กซ์"))
    animals.forEach { animal ->
        animal.makeSound()
    }
    
    // specific method เรียกได้เฉพาะถ้า cast แล้ว
    animals.filterIsInstance<Dog>().forEach { it.fetch("ลูกบอล") }
}
```

---

## Step 122: open, override, final

```kotlin
open class Shape {
    open val name: String = "รูปร่าง"
    open fun area(): Double = 0.0
    open fun perimeter(): Double = 0.0
    
    // final method - ห้าม override ต่อ
    final fun info() {
        println("$name: พื้นที่=${"%.2f".format(area())}, เส้นรอบ=${"%.2f".format(perimeter())}")
    }
    
    override fun toString() = "$name (area=${"%.2f".format(area())})"
}

open class Rectangle(val width: Double, val height: Double) : Shape() {
    override val name = "สี่เหลี่ยม"
    override fun area() = width * height
    override fun perimeter() = 2 * (width + height)
}

// Square สืบทอดจาก Rectangle
class Square(side: Double) : Rectangle(side, side) {
    override val name = "สี่เหลี่ยมจัตุรัส"
    val side get() = width
}

open class Circle(val radius: Double) : Shape() {
    override val name = "วงกลม"
    override fun area() = Math.PI * radius * radius
    override fun perimeter() = 2 * Math.PI * radius
}

// ColoredCircle - override ได้อีก
class ColoredCircle(radius: Double, val color: String) : Circle(radius) {
    override val name = "วงกลมสี$color"
    // area() และ perimeter() ยังคงเป็นของ Circle
}

fun main() {
    val shapes: List<Shape> = listOf(
        Rectangle(5.0, 3.0),
        Square(4.0),
        Circle(5.0),
        ColoredCircle(3.0, "แดง")
    )

    shapes.forEach { shape ->
        shape.info()  // final method ทุก shape ใช้ได้
    }

    println()

    // ตรวจสอบประเภท
    shapes.forEach { shape ->
        when (shape) {
            is Square -> println("${shape.name}: side=${shape.side}")
            is Rectangle -> println("${shape.name}: ${shape.width}x${shape.height}")
            is ColoredCircle -> println("${shape.name}: r=${shape.radius}, สี=${shape.color}")
            is Circle -> println("${shape.name}: r=${shape.radius}")
        }
    }

    // คำนวณรวม
    val totalArea = shapes.sumOf { it.area() }
    println("\nพื้นที่รวม: ${"%.2f".format(totalArea)}")
    
    val largest = shapes.maxByOrNull { it.area() }
    println("รูปที่ใหญ่สุด: ${largest?.name}")
}
```

---

## Step 123: super Keyword

```kotlin
open class Vehicle(
    val brand: String,
    val model: String,
    val year: Int
) {
    open var speed: Double = 0.0
    
    open fun accelerate(amount: Double) {
        speed += amount
        println("$brand $model เร่งความเร็ว +$amount กม./ชม. (ปัจจุบัน: $speed)")
    }
    
    open fun brake(amount: Double) {
        speed = maxOf(0.0, speed - amount)
        println("$brand $model เบรก -$amount กม./ชม. (ปัจจุบัน: $speed)")
    }
    
    open fun describe() {
        println("$brand $model ($year)")
    }
}

open class ElectricVehicle(
    brand: String,
    model: String,
    year: Int,
    val batteryCapacity: Double  // kWh
) : Vehicle(brand, model, year) {
    var batteryLevel: Double = 100.0  // %
    
    override fun accelerate(amount: Double) {
        // เรียก parent accelerate ก่อน
        super.accelerate(amount)
        // แล้วเพิ่ม EV-specific behavior
        val energyUsed = amount * 0.1
        batteryLevel -= energyUsed
        println("  Battery: ${"%.1f".format(batteryLevel)}% (-${"%.1f".format(energyUsed)})")
    }
    
    override fun describe() {
        super.describe()  // เรียก Vehicle.describe()
        println("  Battery: ${batteryCapacity}kWh, Level: ${"%.1f".format(batteryLevel)}%")
    }
    
    fun charge(percent: Double) {
        batteryLevel = minOf(100.0, batteryLevel + percent)
        println("ชาร์จแบต: ${"%.1f".format(batteryLevel)}%")
    }
}

class Tesla(model: String) : ElectricVehicle("Tesla", model, 2024, 100.0) {
    var autopilotEnabled = false
    
    override fun accelerate(amount: Double) {
        if (autopilotEnabled) {
            println("Autopilot กำลังควบคุม...")
        }
        super.accelerate(amount)  // เรียก ElectricVehicle.accelerate()
    }
    
    override fun describe() {
        super.describe()  // เรียก ElectricVehicle.describe()
        println("  Autopilot: ${if (autopilotEnabled) "เปิด" else "ปิด"}")
    }
}

fun main() {
    val car = Vehicle("Toyota", "Camry", 2022)
    car.accelerate(60.0)
    car.brake(20.0)
    car.describe()

    println()

    val ev = ElectricVehicle("Nissan", "Leaf", 2023, 62.0)
    ev.accelerate(80.0)
    ev.accelerate(40.0)
    ev.describe()
    ev.charge(15.0)

    println()

    val tesla = Tesla("Model 3")
    tesla.describe()
    tesla.accelerate(100.0)
    tesla.autopilotEnabled = true
    tesla.accelerate(50.0)
    tesla.describe()

    // Polymorphism
    println("\n--- Fleet ---")
    val fleet: List<Vehicle> = listOf(car, ev, tesla)
    fleet.forEach { vehicle ->
        print("${vehicle.brand} ${vehicle.model}: ")
        when (vehicle) {
            is Tesla -> println("Tesla EV, battery=${vehicle.batteryLevel}%")
            is ElectricVehicle -> println("EV, battery=${vehicle.batteryLevel}%")
            else -> println("Conventional")
        }
    }
}
```

---

## Step 124: Abstract Class

```kotlin
// Abstract class - ไม่สามารถ instantiate ได้
// ใช้เป็น template สำหรับ subclasses

abstract class DatabaseRepository<T> {
    protected val items = mutableListOf<T>()
    
    // Abstract methods - subclass ต้องกำหนด
    abstract fun getById(id: Int): T?
    abstract fun save(item: T): T
    abstract fun delete(id: Int): Boolean
    abstract fun getAll(): List<T>
    
    // Concrete methods - ทุก subclass ใช้ร่วมกัน
    fun count() = items.size
    
    fun isEmpty() = items.isEmpty()
    
    fun clear() {
        items.clear()
        println("ล้างข้อมูลทั้งหมดแล้ว")
    }
    
    // Template method pattern
    fun saveAll(newItems: List<T>): List<T> {
        return newItems.map { save(it) }
    }
}

data class User(val id: Int, val name: String, val email: String)

class UserRepository : DatabaseRepository<User>() {
    private var nextId = 1
    
    override fun getById(id: Int) = items.find { it.id == id }
    
    override fun save(item: User): User {
        val existing = items.indexOfFirst { it.id == item.id }
        return if (existing >= 0) {
            items[existing] = item
            println("อัพเดท User: ${item.name}")
            item
        } else {
            val newUser = item.copy(id = nextId++)
            items.add(newUser)
            println("เพิ่ม User: ${newUser.name} (id=${newUser.id})")
            newUser
        }
    }
    
    override fun delete(id: Int): Boolean {
        val removed = items.removeIf { it.id == id }
        if (removed) println("ลบ User id=$id แล้ว")
        return removed
    }
    
    override fun getAll() = items.toList()
    
    fun findByEmail(email: String) = items.find { it.email == email }
    
    fun searchByName(query: String) = items.filter { 
        it.name.contains(query, ignoreCase = true) 
    }
}

// Abstract class สำหรับ shapes
abstract class Shape2D(val color: String) {
    abstract val name: String
    abstract fun area(): Double
    abstract fun perimeter(): Double
    
    // Concrete method
    open fun draw() {
        println("วาด $name สี$color (พื้นที่=${"%.2f".format(area())})")
    }
    
    // ทุก shape ต้องสามารถ scale ได้
    abstract fun scale(factor: Double): Shape2D
}

class Triangle(
    val base: Double,
    val height: Double,
    val side1: Double,
    val side2: Double,
    color: String
) : Shape2D(color) {
    override val name = "สามเหลี่ยม"
    override fun area() = 0.5 * base * height
    override fun perimeter() = base + side1 + side2
    override fun scale(factor: Double) = Triangle(
        base * factor, height * factor, side1 * factor, side2 * factor, color
    )
}

class RegularPolygon(val sides: Int, val sideLength: Double, color: String) : Shape2D(color) {
    override val name = "รูป $sides เหลี่ยม"
    
    override fun area(): Double {
        val a = sideLength / (2 * Math.tan(Math.PI / sides))
        return sides * sideLength * a / 2
    }
    
    override fun perimeter() = sides * sideLength
    
    override fun scale(factor: Double) = RegularPolygon(sides, sideLength * factor, color)
}

fun main() {
    val repo = UserRepository()
    
    repo.save(User(0, "สมชาย", "somchai@email.com"))
    repo.save(User(0, "สมหญิง", "somying@email.com"))
    repo.save(User(0, "สมศักดิ์", "somsak@email.com"))
    
    println("จำนวน: ${repo.count()}")
    println("หา id=2: ${repo.getById(2)}")
    println("หา email: ${repo.findByEmail("somchai@email.com")}")
    
    repo.save(User(2, "สมหญิง แก้ไขแล้ว", "somying@email.com"))
    repo.delete(3)
    
    println("ผู้ใช้ทั้งหมด: ${repo.getAll()}")

    println()

    // Abstract shapes
    val shapes: List<Shape2D> = listOf(
        Triangle(6.0, 4.0, 5.0, 5.0, "แดง"),
        RegularPolygon(6, 4.0, "น้ำเงิน"),
        RegularPolygon(8, 3.0, "เขียว")
    )
    
    shapes.forEach { it.draw() }
    
    println("\nScale 2x:")
    shapes.map { it.scale(2.0) }.forEach { it.draw() }
}
```

---

## Step 125: Polymorphism - พหุสัณฐาน

```kotlin
// Polymorphism - object ประเภทเดียวกันทำงานต่างกัน

abstract class PaymentMethod(val name: String) {
    abstract fun pay(amount: Double): Boolean
    abstract fun refund(amount: Double, reason: String): Boolean
    open fun getDescription() = "Payment via $name"
}

class CreditCard(
    val cardNumber: String,
    val holderName: String,
    private val creditLimit: Double
) : PaymentMethod("Credit Card") {
    private var usedCredit = 0.0

    override fun pay(amount: Double): Boolean {
        return if (usedCredit + amount <= creditLimit) {
            usedCredit += amount
            println("Credit Card ${cardNumber.takeLast(4)}: จ่าย $amount บาท (ใช้ไป: $usedCredit/$creditLimit)")
            true
        } else {
            println("Credit limit เกิน! (ต้องการ: $amount, เหลือ: ${creditLimit - usedCredit})")
            false
        }
    }

    override fun refund(amount: Double, reason: String): Boolean {
        usedCredit -= amount
        println("คืนเงิน $amount บาท เหตุผล: $reason")
        return true
    }

    override fun getDescription() = "Credit Card (*${cardNumber.takeLast(4)}) ของ $holderName"
}

class DigitalWallet(val username: String, initialBalance: Double) : PaymentMethod("Digital Wallet") {
    private var balance = initialBalance

    override fun pay(amount: Double): Boolean {
        return if (balance >= amount) {
            balance -= amount
            println("Wallet '$username': จ่าย $amount บาท (คงเหลือ: $balance)")
            true
        } else {
            println("ยอดเงินไม่พอ! (มี: $balance, ต้องการ: $amount)")
            false
        }
    }

    override fun refund(amount: Double, reason: String): Boolean {
        balance += amount
        println("คืนเงิน $amount บาท เข้า wallet (ยอดใหม่: $balance)")
        return true
    }

    fun topUp(amount: Double) {
        balance += amount
        println("เติมเงิน $amount บาท (ยอดใหม่: $balance)")
    }
}

class BankTransfer(val accountNumber: String, val bankName: String) : PaymentMethod("Bank Transfer") {
    override fun pay(amount: Double): Boolean {
        println("โอนเงิน $amount บาท ไปยัง $accountNumber ($bankName)")
        return true
    }

    override fun refund(amount: Double, reason: String): Boolean {
        println("คืนเงิน $amount บาท จาก bank transfer: $reason")
        return true
    }
}

// Order ที่ใช้ PaymentMethod ได้ทุกชนิด
class Order(val items: Map<String, Double>) {
    val total = items.values.sum()
    
    fun checkout(payment: PaymentMethod): Boolean {
        println("\n--- Checkout ---")
        println("รายการ: ${items.entries.joinToString { "${it.key}: ฿${it.value}" }}")
        println("รวม: ฿$total")
        println("ชำระด้วย: ${payment.getDescription()}")
        
        return if (payment.pay(total)) {
            println("ชำระเงินสำเร็จ!")
            true
        } else {
            println("ชำระเงินไม่สำเร็จ")
            false
        }
    }
}

fun main() {
    val card = CreditCard("1234567890123456", "สมชาย", 50000.0)
    val wallet = DigitalWallet("somying", 5000.0)
    val bank = BankTransfer("123-456-789", "กสิกร")

    val order1 = Order(mapOf("iPhone" to 35000.0, "Case" to 500.0))
    val order2 = Order(mapOf("AirPods" to 8000.0))
    val order3 = Order(mapOf("Coffee" to 150.0, "Cake" to 200.0))
    val bigOrder = Order(mapOf("MacBook" to 65000.0))

    // Polymorphism - ใช้ payment ต่างชนิดได้
    order1.checkout(card)
    order2.checkout(wallet)
    order3.checkout(wallet)
    
    // wallet ไม่พอ - จ่ายด้วย card แทน
    val result = order3.checkout(wallet)
    if (!result) {
        println("เปลี่ยนไปใช้ credit card...")
        order3.checkout(card)
    }

    bigOrder.checkout(card)  // เกิน limit
    
    // Polymorphic list
    println("\n--- Payment Methods ---")
    val methods: List<PaymentMethod> = listOf(card, wallet, bank)
    methods.forEach { println(it.getDescription()) }
}
```

---

## Step 126: Method Overriding ลึกขึ้น

```kotlin
open class Logger {
    open fun log(message: String) {
        println("[LOG] $message")
    }
    
    open fun logError(message: String) {
        log("ERROR: $message")
    }
    
    open fun logWarning(message: String) {
        log("WARN: $message")
    }
}

class TimestampLogger : Logger() {
    private fun timestamp() = "2024-01-01 12:00:00"
    
    override fun log(message: String) {
        println("[${timestamp()}] $message")
    }
    // logError และ logWarning จะใช้ log() ที่ override แล้ว
}

class ColoredLogger : Logger() {
    private val colors = mapOf(
        "ERROR" to "\u001B[31m",  // สีแดง
        "WARN" to "\u001B[33m",   // สีเหลือง
        "LOG" to "\u001B[0m"      // ปกติ
    )
    private val reset = "\u001B[0m"
    
    override fun log(message: String) {
        println("${colors["LOG"]}[LOG] $message$reset")
    }
    
    override fun logError(message: String) {
        println("${colors["ERROR"]}[ERROR] $message$reset")
    }
    
    override fun logWarning(message: String) {
        println("${colors["WARN"]}[WARN] $message$reset")
    }
}

// Property overriding
open class Config {
    open val timeout: Int = 30
    open val maxRetries: Int = 3
    open val baseUrl: String = "http://localhost"
    
    open fun describe() {
        println("Config: timeout=$timeout, retries=$maxRetries, url=$baseUrl")
    }
}

class ProductionConfig : Config() {
    override val timeout: Int = 60
    override val maxRetries: Int = 5
    override val baseUrl: String = "https://api.production.com"
}

class TestConfig : Config() {
    override val timeout: Int = 5
    override val maxRetries: Int = 1
    override val baseUrl: String = "http://test.localhost"
}

// var override เป็น val ไม่ได้, แต่ val override เป็น var ได้
open class Base {
    open val readOnly: String = "base"
    open var readWrite: String = "base"
}

class Derived : Base() {
    // override val เป็น var ได้
    override var readOnly: String = "derived (now mutable!)"
    override var readWrite: String = "derived"
}

fun main() {
    // Logger polymorphism
    val loggers: List<Logger> = listOf(Logger(), TimestampLogger(), ColoredLogger())
    
    println("=== Loggers ===")
    loggers.forEach { logger ->
        logger.log("ข้อความทดสอบ")
        logger.logError("เกิดข้อผิดพลาด")
        logger.logWarning("คำเตือน")
        println()
    }
    
    // Config polymorphism
    println("=== Configs ===")
    val configs: List<Config> = listOf(Config(), ProductionConfig(), TestConfig())
    configs.forEach { config ->
        config.describe()
    }
    
    // Property override
    val base = Base()
    val derived = Derived()
    println("\nBase: readOnly=${base.readOnly}, readWrite=${base.readWrite}")
    println("Derived: readOnly=${derived.readOnly}, readWrite=${derived.readWrite}")
    
    derived.readOnly = "changed!"  // ทำได้เพราะ Derived override เป็น var
    println("Derived after change: ${derived.readOnly}")
}
```

---

## Step 127: Calling Super Method

```kotlin
open class HttpClient(val baseUrl: String) {
    protected val headers = mutableMapOf<String, String>()
    
    init {
        headers["Content-Type"] = "application/json"
        headers["Accept"] = "application/json"
    }
    
    open fun get(path: String): String {
        println("GET $baseUrl$path")
        println("Headers: $headers")
        return "Response from $path"
    }
    
    open fun post(path: String, body: String): String {
        println("POST $baseUrl$path")
        println("Body: $body")
        return "Created at $path"
    }
}

class AuthenticatedClient(baseUrl: String, private val token: String) : HttpClient(baseUrl) {
    init {
        headers["Authorization"] = "Bearer $token"
    }
    
    override fun get(path: String): String {
        println("=== Authenticated GET ===")
        return super.get(path)  // เรียก HttpClient.get()
    }
    
    override fun post(path: String, body: String): String {
        println("=== Authenticated POST ===")
        return super.post(path, body)
    }
}

class CachingClient(baseUrl: String) : HttpClient(baseUrl) {
    private val cache = mutableMapOf<String, String>()
    
    override fun get(path: String): String {
        val cached = cache[path]
        if (cached != null) {
            println("Cache hit: $path")
            return cached
        }
        
        println("Cache miss: $path")
        val response = super.get(path)  // เรียก parent
        cache[path] = response
        return response
    }
    
    fun clearCache() {
        cache.clear()
        println("Cache cleared")
    }
}

class LoggingAuthCachingClient(
    baseUrl: String,
    token: String
) : AuthenticatedClient(baseUrl, token) {
    private val requestLog = mutableListOf<String>()
    private val cache = mutableMapOf<String, String>()
    
    override fun get(path: String): String {
        requestLog.add("GET $path")
        
        val cached = cache[path]
        if (cached != null) {
            println("📦 Cache: $path")
            return cached
        }
        
        val result = super.get(path)  // เรียก AuthenticatedClient.get()
        cache[path] = result
        return result
    }
    
    fun getRequestLog() = requestLog.toList()
}

fun main() {
    val basic = HttpClient("https://api.example.com")
    println("=== Basic Client ===")
    basic.get("/users")
    
    println()
    
    val auth = AuthenticatedClient("https://api.example.com", "my-secret-token")
    println("=== Auth Client ===")
    auth.get("/users/me")
    
    println()
    
    val caching = CachingClient("https://api.example.com")
    println("=== Caching Client ===")
    caching.get("/products")
    caching.get("/products")  // cache hit
    caching.get("/users")
    caching.clearCache()
    caching.get("/products")  // cache miss again
    
    println()
    
    val full = LoggingAuthCachingClient("https://api.example.com", "token-123")
    println("=== Full Featured Client ===")
    full.get("/dashboard")
    full.get("/dashboard")  // cached
    full.get("/settings")
    
    println("\nRequest log: ${full.getRequestLog()}")
}
```

---

## Step 128: Abstract Class กับ Template Method Pattern

```kotlin
abstract class DataExporter(val filename: String) {
    // Template method - กำหนดขั้นตอน แต่ให้ subclass กำหนดรายละเอียด
    fun export(data: List<Map<String, Any>>) {
        println("เริ่ม export ไปยัง $filename")
        val header = createHeader(data)
        val rows = data.map { formatRow(it) }
        val footer = createFooter(data)
        writeToFile(header, rows, footer)
        println("Export สำเร็จ")
    }
    
    // Abstract - subclass ต้องกำหนด
    abstract fun createHeader(data: List<Map<String, Any>>): String
    abstract fun formatRow(row: Map<String, Any>): String
    abstract fun createFooter(data: List<Map<String, Any>>): String
    
    // Concrete - subclass ใช้ร่วมกัน
    private fun writeToFile(header: String, rows: List<String>, footer: String) {
        println("--- $filename ---")
        println(header)
        rows.forEach { println(it) }
        println(footer)
    }
}

class CsvExporter(filename: String) : DataExporter("$filename.csv") {
    override fun createHeader(data: List<Map<String, Any>>): String {
        return data.firstOrNull()?.keys?.joinToString(",") ?: ""
    }
    
    override fun formatRow(row: Map<String, Any>): String {
        return row.values.joinToString(",") { 
            val s = it.toString()
            if (s.contains(",")) "\"$s\"" else s
        }
    }
    
    override fun createFooter(data: List<Map<String, Any>>) = "# ${data.size} records"
}

class HtmlExporter(filename: String) : DataExporter("$filename.html") {
    override fun createHeader(data: List<Map<String, Any>>): String {
        val headers = data.firstOrNull()?.keys?.joinToString("") { "<th>$it</th>" } ?: ""
        return "<table><thead><tr>$headers</tr></thead><tbody>"
    }
    
    override fun formatRow(row: Map<String, Any>): String {
        val cells = row.values.joinToString("") { "<td>$it</td>" }
        return "  <tr>$cells</tr>"
    }
    
    override fun createFooter(data: List<Map<String, Any>>) = "</tbody></table>"
}

class JsonExporter(filename: String) : DataExporter("$filename.json") {
    private var rowIndex = 0
    
    override fun createHeader(data: List<Map<String, Any>>): String {
        rowIndex = 0
        return "["
    }
    
    override fun formatRow(row: Map<String, Any>): String {
        rowIndex++
        val fields = row.entries.joinToString(",\n    ") { (k, v) ->
            when (v) {
                is String -> "\"$k\": \"$v\""
                else -> "\"$k\": $v"
            }
        }
        return "  {\n    $fields\n  }${if (rowIndex < 3) "," else ""}"
    }
    
    override fun createFooter(data: List<Map<String, Any>>) = "]"
}

fun main() {
    val salesData = listOf(
        mapOf("name" to "iPhone", "quantity" to 5, "price" to 35000),
        mapOf("name" to "AirPods", "quantity" to 10, "price" to 8000),
        mapOf("name" to "MacBook", "quantity" to 2, "price" to 65000)
    )

    val exporters: List<DataExporter> = listOf(
        CsvExporter("sales"),
        HtmlExporter("sales"),
        JsonExporter("sales")
    )

    exporters.forEach { exporter ->
        println("\n" + "=".repeat(40))
        exporter.export(salesData)
    }
}
```

---

## Step 129: Sealed Class Preview

```kotlin
// Sealed class - จำกัดชนิดของ subclass
// ทุก subclass ต้องอยู่ใน file เดียวกัน

sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val code: Int = 0) : Result<Nothing>()
    object Loading : Result<Nothing>()
    
    val isSuccess get() = this is Success
    val isError get() = this is Error
    val isLoading get() = this is Loading
}

// Network response
sealed class NetworkState {
    object Idle : NetworkState()
    object Loading : NetworkState()
    data class Success(val data: String) : NetworkState()
    data class Error(val message: String) : NetworkState()
    data class Timeout(val secondsWaited: Int) : NetworkState()
}

fun processResult(result: Result<String>) {
    // when กับ sealed class ต้องครอบคลุมทุก case
    when (result) {
        is Result.Success -> println("สำเร็จ: ${result.data}")
        is Result.Error -> println("ผิดพลาด (${result.code}): ${result.message}")
        is Result.Loading -> println("กำลังโหลด...")
    }
}

fun handleNetworkState(state: NetworkState): String {
    return when (state) {
        is NetworkState.Idle -> "รอการเชื่อมต่อ"
        is NetworkState.Loading -> "กำลังโหลด..."
        is NetworkState.Success -> "ข้อมูล: ${state.data}"
        is NetworkState.Error -> "เกิดข้อผิดพลาด: ${state.message}"
        is NetworkState.Timeout -> "หมดเวลา (${state.secondsWaited}s)"
    }
}

// Simulating API call
fun fetchUser(id: Int): Result<String> {
    return when (id) {
        1 -> Result.Success("สมชาย")
        2 -> Result.Error("User not found", 404)
        3 -> Result.Loading
        else -> Result.Error("Unknown error", 500)
    }
}

fun main() {
    // ทดสอบ Result
    for (id in 1..4) {
        val result = fetchUser(id)
        print("fetchUser($id): ")
        processResult(result)
    }

    println()

    // ทดสอบ NetworkState
    val states: List<NetworkState> = listOf(
        NetworkState.Idle,
        NetworkState.Loading,
        NetworkState.Success("""{"user": "somchai"}"""),
        NetworkState.Error("Connection refused"),
        NetworkState.Timeout(30)
    )

    states.forEach { state ->
        println(handleNetworkState(state))
    }

    // Sealed class กับ extension function
    fun Result<Int>.map(transform: (Int) -> Int): Result<Int> {
        return when (this) {
            is Result.Success -> Result.Success(transform(data))
            is Result.Error -> this
            is Result.Loading -> this
        }
    }

    val r1: Result<Int> = Result.Success(5)
    val r2 = r1.map { it * 2 }
    println("\n$r1 -> $r2")
}
```

---

## Step 130: Inheritance Hierarchy

```kotlin
// ตัวอย่างจริง: ระบบพนักงาน

open class Employee(
    val id: Int,
    val name: String,
    val department: String
) {
    open val baseSalary: Double get() = 25000.0
    open val bonus: Double get() = 0.0
    val totalSalary get() = baseSalary + bonus

    open fun describe() {
        println("พนักงาน: $name ($department)")
        println("เงินเดือนพื้นฐาน: $baseSalary บาท")
        println("โบนัส: $bonus บาท")
        println("รวม: $totalSalary บาท")
    }

    open fun work() {
        println("$name กำลังทำงาน")
    }
}

open class Manager(
    id: Int,
    name: String,
    department: String,
    val teamSize: Int
) : Employee(id, name, department) {
    override val baseSalary = 60000.0
    override val bonus get() = baseSalary * 0.2  // 20% bonus

    override fun describe() {
        super.describe()
        println("ขนาดทีม: $teamSize คน")
    }

    override fun work() {
        println("$name กำลังบริหารทีม $teamSize คน")
    }

    open fun conductMeeting() {
        println("$name จัดประชุมทีม $department")
    }
}

class Director(
    id: Int,
    name: String,
    val division: String,
    teamSize: Int
) : Manager(id, name, division, teamSize) {
    override val baseSalary = 150000.0
    override val bonus get() = baseSalary * 0.5  // 50% bonus

    override fun describe() {
        super.describe()
        println("ฝ่าย: $division")
    }

    override fun work() {
        println("$name วางแผนกลยุทธ์ฝ่าย $division")
    }

    fun presentToBoard() {
        println("$name นำเสนอต่อคณะกรรมการ")
    }
}

open class Specialist(
    id: Int,
    name: String,
    department: String,
    val skill: String
) : Employee(id, name, department) {
    override val baseSalary = 45000.0
    override val bonus get() = baseSalary * 0.1

    override fun work() {
        println("$name ใช้ทักษะ $skill")
    }
}

class SeniorSpecialist(
    id: Int,
    name: String,
    department: String,
    skill: String,
    val yearsOfExperience: Int
) : Specialist(id, name, department, skill) {
    override val baseSalary = 70000.0 + yearsOfExperience * 2000.0
    override val bonus get() = baseSalary * 0.15

    fun mentor(junior: Specialist) {
        println("$name สอน ${junior.name} ด้าน $skill")
    }
}

fun main() {
    val employees: List<Employee> = listOf(
        Employee(1, "สมชาย", "IT"),
        Specialist(2, "สมหญิง", "IT", "Kotlin"),
        SeniorSpecialist(3, "สมศักดิ์", "IT", "Architecture", 8),
        Manager(4, "สมใจ", "IT", 5),
        Director(5, "สมพร", "Technology", 20)
    )

    // Polymorphism
    println("=== พนักงานทั้งหมด ===")
    employees.forEach { emp ->
        emp.work()
    }

    println("\n=== รายได้ ===")
    employees.forEach { emp ->
        println("${emp.name}: ฿${"%.2f".format(emp.totalSalary)}")
    }

    // Stats
    val totalPayroll = employees.sumOf { it.totalSalary }
    println("\nเงินเดือนรวม: ฿${"%.2f".format(totalPayroll)}")
    println("เฉลี่ย: ฿${"%.2f".format(totalPayroll / employees.size)}")

    println("\n=== ผู้บริหาร ===")
    employees.filterIsInstance<Manager>().forEach { mgr ->
        mgr.conductMeeting()
    }

    println("\n=== รายละเอียด Director ===")
    employees.filterIsInstance<Director>().firstOrNull()?.describe()

    // Type checking
    println("\n=== Type Checking ===")
    employees.forEach { emp ->
        val type = when (emp) {
            is Director -> "Director"
            is Manager -> "Manager"
            is SeniorSpecialist -> "Senior Specialist"
            is Specialist -> "Specialist"
            else -> "Employee"
        }
        println("${emp.name}: $type")
    }
}
```

---

## Step 131: Multiple Inheritance ด้วย Interface

```kotlin
// Kotlin ไม่รองรับ multiple inheritance จาก class
// แต่รองรับ multiple interface implementation

interface Flyable {
    val maxAltitude: Double
    fun fly() = println("บินที่ความสูง $maxAltitude เมตร")
    fun land() = println("ลงจอด")
}

interface Swimmable {
    val maxDepth: Double
    fun swim() = println("ว่ายน้ำลึก $maxDepth เมตร")
    fun surface() = println("ขึ้นมาผิวน้ำ")
}

interface Huntable {
    fun hunt(prey: String) = println("ล่า $prey")
}

open class Animal2(val name: String)

// Duck สามารถทำได้ทั้งหมด
class Duck(name: String) : Animal2(name), Flyable, Swimmable, Huntable {
    override val maxAltitude = 1000.0
    override val maxDepth = 2.0

    override fun fly() {
        println("$name บินด้วยปีก")
        super<Flyable>.fly()
    }

    override fun swim() {
        println("$name ว่ายน้ำด้วยเท้า")
        super<Swimmable>.swim()
    }
    
    override fun hunt(prey: String) {
        println("$name ดำน้ำล่า $prey")
    }
}

class Eagle(name: String) : Animal2(name), Flyable, Huntable {
    override val maxAltitude = 5000.0

    override fun fly() {
        println("$name บินร่อนสูง")
        super.fly()
    }

    override fun hunt(prey: String) {
        fly()
        println("$name โฉบลง ล่า $prey")
        land()
    }
}

class Shark(name: String) : Animal2(name), Swimmable, Huntable {
    override val maxDepth = 500.0
}

fun main() {
    val duck = Duck("เป็ดทอง")
    duck.fly()
    duck.swim()
    duck.hunt("ปลา")

    println()

    val eagle = Eagle("นกอินทรี")
    eagle.hunt("กระต่าย")

    println()

    val shark = Shark("ฉลาม")
    shark.swim()
    shark.hunt("ปลาทูน่า")

    // Polymorphism ผ่าน interfaces
    println("\n=== Flyers ===")
    val flyers: List<Flyable> = listOf(duck, eagle)
    flyers.forEach { it.fly() }

    println("\n=== Swimmers ===")
    val swimmers: List<Swimmable> = listOf(duck, shark)
    swimmers.forEach { it.swim() }

    println("\n=== Hunters ===")
    val hunters: List<Huntable> = listOf(duck, eagle, shark)
    hunters.forEach { it.hunt("เหยื่อ") }
}
```

---

## Step 132: Polymorphism ขั้นสูง

```kotlin
// Strategy Pattern ด้วย Polymorphism

abstract class SortStrategy<T : Comparable<T>> {
    abstract fun sort(list: MutableList<T>)
    abstract val name: String
    
    fun sortAndTime(list: MutableList<T>): Long {
        val start = System.currentTimeMillis()
        sort(list)
        return System.currentTimeMillis() - start
    }
}

class BubbleSort<T : Comparable<T>> : SortStrategy<T>() {
    override val name = "Bubble Sort"
    
    override fun sort(list: MutableList<T>) {
        for (i in list.indices) {
            for (j in 0 until list.size - i - 1) {
                if (list[j] > list[j + 1]) {
                    val temp = list[j]
                    list[j] = list[j + 1]
                    list[j + 1] = temp
                }
            }
        }
    }
}

class SelectionSort<T : Comparable<T>> : SortStrategy<T>() {
    override val name = "Selection Sort"
    
    override fun sort(list: MutableList<T>) {
        for (i in list.indices) {
            var minIdx = i
            for (j in i + 1 until list.size) {
                if (list[j] < list[minIdx]) minIdx = j
            }
            if (minIdx != i) {
                val temp = list[i]
                list[i] = list[minIdx]
                list[minIdx] = temp
            }
        }
    }
}

class BuiltinSort<T : Comparable<T>> : SortStrategy<T>() {
    override val name = "Built-in Sort"
    override fun sort(list: MutableList<T>) = list.sort()
}

class Sorter<T : Comparable<T>>(var strategy: SortStrategy<T>) {
    fun sort(data: List<T>): Pair<List<T>, Long> {
        val mutable = data.toMutableList()
        val time = strategy.sortAndTime(mutable)
        return Pair(mutable, time)
    }
    
    fun changeStrategy(newStrategy: SortStrategy<T>) {
        strategy = newStrategy
        println("เปลี่ยน strategy เป็น ${strategy.name}")
    }
}

fun main() {
    val data = (1..10).map { (1..100).random() }
    println("ข้อมูลดิบ: $data")

    val strategies: List<SortStrategy<Int>> = listOf(
        BubbleSort(),
        SelectionSort(),
        BuiltinSort()
    )

    strategies.forEach { strategy ->
        val mutableData = data.toMutableList()
        val time = strategy.sortAndTime(mutableData)
        println("${strategy.name}: $mutableData (${time}ms)")
    }

    println()

    val sorter = Sorter(BubbleSort<Int>())
    val (sorted1, time1) = sorter.sort(data)
    println("Bubble: $sorted1 (${time1}ms)")

    sorter.changeStrategy(BuiltinSort())
    val (sorted2, time2) = sorter.sort(data)
    println("Built-in: $sorted2 (${time2}ms)")
}
```

---

## Step 133: แบบฝึกหัดและโจทย์ Part 07

```kotlin
// ============================================================
// แบบฝึกหัดที่ 1: ระบบสัตว์ที่ครบถ้วน
// ============================================================

abstract class Wildlife(
    val name: String,
    val species: String,
    val weight: Double  // kg
) {
    abstract val habitat: String
    abstract val diet: String
    
    open fun eat() = println("$name ($species) กินอาหาร ($diet)")
    open fun sleep() = println("$name นอนหลับ")
    abstract fun move()
    abstract fun communicate()
    
    override fun toString() = "$name - $species (${weight}kg) อาศัยใน $habitat"
}

class Lion(name: String, weight: Double) : Wildlife(name, "Panthera leo", weight) {
    override val habitat = "ทุ่งหญ้าสะวันนา"
    override val diet = "เนื้อสัตว์"
    
    override fun move() = println("$name วิ่งด้วยความเร็ว 80km/h")
    override fun communicate() = println("$name คำราม!")
    
    override fun eat() {
        println("$name ล่าสัตว์ก่อนกิน")
        super.eat()
    }
}

class Dolphin(name: String, weight: Double) : Wildlife(name, "Tursiops truncatus", weight) {
    override val habitat = "มหาสมุทร"
    override val diet = "ปลาและหมึก"
    
    override fun move() = println("$name ว่ายน้ำด้วยความเร็ว 40km/h")
    override fun communicate() = println("$name ส่งเสียงอัลตราโซนิก click click~")
}

class Eagle2(name: String, weight: Double) : Wildlife(name, "Aquila chrysaetos", weight) {
    override val habitat = "ภูเขาสูง"
    override val diet = "สัตว์เล็กและปลา"
    
    override fun move() = println("$name บินร่อนที่ความสูง 3000m")
    override fun communicate() = println("$name ส่งเสียงร้องแหลม")
}

fun main() {
    val animals: List<Wildlife> = listOf(
        Lion("ซิมบ้า", 190.0),
        Dolphin("ฟลิปเปอร์", 150.0),
        Eagle2("ซูส", 6.0)
    )

    println("=== ชีวิตสัตว์ป่า ===")
    animals.forEach { animal ->
        println("\n$animal")
        animal.move()
        animal.communicate()
        animal.eat()
        animal.sleep()
    }

    println("\n=== สถิติ ===")
    println("น้ำหนักเฉลี่ย: ${"%.1f".format(animals.map { it.weight }.average())} kg")
    println("หนักที่สุด: ${animals.maxByOrNull { it.weight }?.name}")

    // ============================================================
    // แบบฝึกหัดที่ 2: Design Pattern - Decorator
    // ============================================================

    interface Coffee {
        val description: String
        val cost: Double
    }

    class SimpleCoffee : Coffee {
        override val description = "กาแฟดำ"
        override val cost = 30.0
    }

    abstract class CoffeeDecorator(protected val coffee: Coffee) : Coffee {
        override val description get() = coffee.description
        override val cost get() = coffee.cost
    }

    class MilkDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
        override val description get() = "${coffee.description} + นม"
        override val cost get() = coffee.cost + 15.0
    }

    class SugarDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
        override val description get() = "${coffee.description} + น้ำตาล"
        override val cost get() = coffee.cost + 5.0
    }

    class WhipDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
        override val description get() = "${coffee.description} + วิปครีม"
        override val cost get() = coffee.cost + 20.0
    }

    println("\n=== Coffee Shop ===")

    val coffee1: Coffee = SimpleCoffee()
    val coffee2: Coffee = MilkDecorator(SimpleCoffee())
    val coffee3: Coffee = WhipDecorator(MilkDecorator(SugarDecorator(SimpleCoffee())))

    listOf(coffee1, coffee2, coffee3).forEach { c ->
        println("${c.description}: ฿${c.cost}")
    }
}
```

---

## สรุปส่วนที่ 7 (Summary)

| Keyword | ความหมาย |
|---------|---------|
| `open class` | class ที่ inherit ได้ |
| `override` | override method/property จาก parent |
| `super` | เรียก method ของ parent |
| `abstract class` | class ที่ instantiate ไม่ได้ มี abstract methods |
| `abstract fun/val` | method/property ที่ subclass ต้องกำหนด |
| `final` | ห้าม override ต่อ |
| `sealed class` | จำกัด subclass hierarchy |

### หลัก Polymorphism
- Method Overriding: subclass เปลี่ยนพฤติกรรมของ parent
- Runtime Polymorphism: เลือก method ตาม actual type ตอน runtime
- `is` / `when` / `filterIsInstance`: ตรวจสอบและ cast ประเภท

---

[← Part 06: OOP Basics](../part06/README.md) | [Part 08: Interface →](../part08/README.md)
