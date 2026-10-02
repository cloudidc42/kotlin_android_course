# Part 08: Interface and Abstract Class

## ภาพรวม (Overview)
ในส่วนนี้เราจะเรียนรู้เกี่ยวกับ Interface และ Abstract Class อย่างละเอียด
รวมถึง default implementations, multiple interface, interface delegation และ SAM conversions

**Steps ที่ครอบคลุม:** 151-175

---

## Step 151: Interface พื้นฐาน

```kotlin
// Interface - กำหนด "สัญญา" ว่า class ต้องมีอะไร
// ไม่มี state (ไม่มี backing field ปกติ)
// ทุก member เป็น abstract โดย default

interface Printable {
    fun print()
}

interface Saveable {
    fun save(filename: String)
    fun load(filename: String): Boolean
}

interface Describable {
    val description: String  // abstract property
    fun describe() = println(description)  // default implementation
}

// Implement หลาย interface
class Document(
    val title: String,
    val content: String
) : Printable, Saveable, Describable {
    
    override val description = "เอกสาร: $title (${content.length} ตัวอักษร)"
    
    override fun print() {
        println("=== $title ===")
        println(content)
        println("=" .repeat(20))
    }
    
    override fun save(filename: String) {
        println("บันทึก '$title' ไปยัง $filename")
    }
    
    override fun load(filename: String): Boolean {
        println("โหลดจาก $filename")
        return true
    }
}

// Interface สามารถ extend interface อื่นได้
interface Clickable {
    fun onClick()
    fun onLongClick() {
        println("Long click (default)")
    }
}

interface Focusable {
    fun onFocus()
    fun onBlur()
}

interface UIComponent : Clickable, Focusable {
    val id: String
    fun render()
}

class Button(override val id: String, val label: String) : UIComponent {
    override fun onClick() = println("Button '$label' clicked")
    override fun onFocus() = println("Button '$label' focused")
    override fun onBlur() = println("Button '$label' blurred")
    override fun render() = println("<button id='$id'>$label</button>")
}

fun main() {
    val doc = Document("คู่มือ Kotlin", "Kotlin เป็นภาษาที่ยอดเยี่ยม...")
    
    doc.print()
    doc.save("manual.txt")
    doc.describe()
    
    println()
    
    val btn = Button("btn-1", "คลิกฉัน")
    btn.render()
    btn.onClick()
    btn.onLongClick()  // default implementation
    btn.onFocus()
    btn.onBlur()
    
    // Polymorphism ผ่าน interface
    println()
    val printables: List<Printable> = listOf(doc)
    printables.forEach { it.print() }
    
    val describables: List<Describable> = listOf(doc)
    describables.forEach { it.describe() }
}
```

---

## Step 152: Interface with Default Implementations

```kotlin
interface Validator<T> {
    // Abstract
    fun validate(value: T): Boolean
    fun getErrorMessage(value: T): String
    
    // Default implementation
    fun validateAndReport(value: T): ValidationResult {
        return if (validate(value)) {
            ValidationResult.Valid
        } else {
            ValidationResult.Invalid(getErrorMessage(value))
        }
    }
}

sealed class ValidationResult {
    object Valid : ValidationResult()
    data class Invalid(val message: String) : ValidationResult()
    
    val isValid get() = this is Valid
    
    override fun toString() = when (this) {
        is Valid -> "Valid"
        is Invalid -> "Invalid: $message"
    }
}

class EmailValidator : Validator<String> {
    override fun validate(value: String) = 
        value.matches(Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"))
    
    override fun getErrorMessage(value: String) = "'$value' ไม่ใช่ email ที่ถูกต้อง"
}

class PasswordValidator(
    private val minLength: Int = 8,
    private val requireUppercase: Boolean = true,
    private val requireDigit: Boolean = true
) : Validator<String> {
    override fun validate(value: String): Boolean {
        if (value.length < minLength) return false
        if (requireUppercase && !value.any { it.isUpperCase() }) return false
        if (requireDigit && !value.any { it.isDigit() }) return false
        return true
    }
    
    override fun getErrorMessage(value: String): String {
        val errors = mutableListOf<String>()
        if (value.length < minLength) errors.add("ต้องมีอย่างน้อย $minLength ตัวอักษร")
        if (requireUppercase && !value.any { it.isUpperCase() }) errors.add("ต้องมีตัวพิมพ์ใหญ่")
        if (requireDigit && !value.any { it.isDigit() }) errors.add("ต้องมีตัวเลข")
        return errors.joinToString(", ")
    }
}

class RangeValidator(private val min: Int, private val max: Int) : Validator<Int> {
    override fun validate(value: Int) = value in min..max
    override fun getErrorMessage(value: Int) = "$value ต้องอยู่ระหว่าง $min-$max"
}

// Interface พร้อม companion object
interface Repository<T, ID> {
    fun findById(id: ID): T?
    fun findAll(): List<T>
    fun save(entity: T): T
    fun delete(id: ID): Boolean
    fun count() = findAll().size
    fun exists(id: ID) = findById(id) != null
}

fun main() {
    val emailValidator = EmailValidator()
    val passwordValidator = PasswordValidator(minLength = 8)
    val ageValidator = RangeValidator(0, 150)

    val testEmails = listOf("test@example.com", "invalid-email", "user@domain.co.th")
    val testPasswords = listOf("weak", "Strong1", "VeryStr0ng!")
    val testAges = listOf(-5, 25, 200, 18)

    println("=== Email Validation ===")
    testEmails.forEach { email ->
        println("$email: ${emailValidator.validateAndReport(email)}")
    }

    println("\n=== Password Validation ===")
    testPasswords.forEach { pwd ->
        println("$pwd: ${passwordValidator.validateAndReport(pwd)}")
    }

    println("\n=== Age Validation ===")
    testAges.forEach { age ->
        println("$age: ${ageValidator.validateAndReport(age)}")
    }

    // Composite validator
    data class UserRegistration(val email: String, val password: String, val age: Int)
    
    fun validateUser(reg: UserRegistration): Map<String, ValidationResult> {
        return mapOf(
            "email" to emailValidator.validateAndReport(reg.email),
            "password" to passwordValidator.validateAndReport(reg.password),
            "age" to ageValidator.validateAndReport(reg.age)
        )
    }
    
    val reg = UserRegistration("bad-email", "weak", -1)
    println("\n=== User Registration ===")
    validateUser(reg).forEach { (field, result) ->
        println("$field: $result")
    }
}
```

---

## Step 153: Multiple Interface Implementation

```kotlin
// ตัวอย่างระบบ Media Player

interface AudioPlayer {
    val currentTrack: String?
    fun play(track: String)
    fun pause()
    fun stop()
    fun setVolume(level: Int)
}

interface VideoPlayer {
    val currentVideo: String?
    fun playVideo(file: String)
    fun pauseVideo()
    fun setResolution(width: Int, height: Int)
    fun setFullscreen(enabled: Boolean)
}

interface Playlist {
    val tracks: List<String>
    fun addTrack(track: String)
    fun removeTrack(track: String)
    fun next(): String?
    fun previous(): String?
    fun shuffle()
}

interface Equalizer {
    fun setBass(level: Int)
    fun setTreble(level: Int)
    fun setMid(level: Int)
    fun resetEqualizer() {
        setBass(50)
        setTreble(50)
        setMid(50)
    }
}

// MediaCenter implement ทุก interface
class MediaCenter : AudioPlayer, VideoPlayer, Playlist, Equalizer {
    private val _tracks = mutableListOf<String>()
    override val tracks: List<String> get() = _tracks.toList()
    
    private var currentIndex = -1
    override var currentTrack: String? = null
    override var currentVideo: String? = null
    
    private var volume = 50
    private var bass = 50
    private var treble = 50
    private var mid = 50
    
    // AudioPlayer
    override fun play(track: String) {
        currentTrack = track
        println("กำลังเล่น: $track (volume: $volume)")
    }
    
    override fun pause() = println("หยุดชั่วคราว: $currentTrack")
    override fun stop() { currentTrack = null; println("หยุดเล่น") }
    override fun setVolume(level: Int) {
        volume = level.coerceIn(0, 100)
        println("ระดับเสียง: $volume")
    }
    
    // VideoPlayer
    override fun playVideo(file: String) {
        currentVideo = file
        println("เล่นวิดีโอ: $file")
    }
    override fun pauseVideo() = println("หยุดวิดีโอชั่วคราว")
    override fun setResolution(width: Int, height: Int) = println("ความละเอียด: ${width}x$height")
    override fun setFullscreen(enabled: Boolean) = println("เต็มหน้าจอ: $enabled")
    
    // Playlist
    override fun addTrack(track: String) { _tracks.add(track); println("เพิ่ม: $track") }
    override fun removeTrack(track: String) { _tracks.remove(track); println("ลบ: $track") }
    
    override fun next(): String? {
        if (_tracks.isEmpty()) return null
        currentIndex = (currentIndex + 1) % _tracks.size
        return _tracks[currentIndex].also { play(it) }
    }
    
    override fun previous(): String? {
        if (_tracks.isEmpty()) return null
        currentIndex = if (currentIndex <= 0) _tracks.size - 1 else currentIndex - 1
        return _tracks[currentIndex].also { play(it) }
    }
    
    override fun shuffle() {
        _tracks.shuffle()
        currentIndex = -1
        println("สุ่มลำดับเพลง: $_tracks")
    }
    
    // Equalizer
    override fun setBass(level: Int) { bass = level; println("Bass: $bass") }
    override fun setTreble(level: Int) { treble = level; println("Treble: $treble") }
    override fun setMid(level: Int) { mid = level; println("Mid: $mid") }
}

fun main() {
    val player = MediaCenter()
    
    // Setup playlist
    listOf("Song A", "Song B", "Song C", "Song D").forEach { player.addTrack(it) }
    
    // Play
    player.setVolume(75)
    player.next()
    player.next()
    player.previous()
    
    // Video
    player.playVideo("movie.mp4")
    player.setResolution(1920, 1080)
    player.setFullscreen(true)
    
    // Equalizer
    player.setBass(70)
    player.setTreble(40)
    player.setMid(60)
    player.resetEqualizer()  // default implementation
    
    // Polymorphism
    println("\n--- Polymorphism ---")
    val audio: AudioPlayer = player
    audio.play("Jazz Track")
    
    val eq: Equalizer = player
    eq.setBass(80)
}
```

---

## Step 154: Abstract Class vs Interface

```kotlin
// Interface: ควรใช้เมื่อ
// - กำหนด "สามารถทำอะไรได้" (capabilities)
// - ต้องการ multiple inheritance
// - ไม่มี state

// Abstract Class: ควรใช้เมื่อ
// - มี common state หรือ constructor
// - มี concrete methods ที่ใช้ร่วมกัน
// - อยาก model "เป็นอะไร" (is-a relationship)

// ตัวอย่าง: Template Method ด้วย Abstract Class
abstract class ReportGenerator {
    // Template method
    fun generateReport(data: List<Map<String, Any>>): String {
        val sb = StringBuilder()
        sb.append(getTitle())
        sb.append("\n")
        sb.append(formatHeader(data))
        sb.append("\n")
        data.forEach { row ->
            sb.append(formatRow(row))
            sb.append("\n")
        }
        sb.append(formatSummary(data))
        return sb.toString()
    }
    
    // Abstract
    abstract fun getTitle(): String
    abstract fun formatHeader(data: List<Map<String, Any>>): String
    abstract fun formatRow(row: Map<String, Any>): String
    
    // Concrete with default (can override)
    open fun formatSummary(data: List<Map<String, Any>>) = "Total: ${data.size} records"
}

class TextReportGenerator : ReportGenerator() {
    override fun getTitle() = "=== TEXT REPORT ==="
    
    override fun formatHeader(data: List<Map<String, Any>>): String {
        return data.firstOrNull()?.keys?.joinToString("\t") ?: ""
    }
    
    override fun formatRow(row: Map<String, Any>) = row.values.joinToString("\t")
}

class MarkdownReportGenerator : ReportGenerator() {
    override fun getTitle() = "# Report"
    
    override fun formatHeader(data: List<Map<String, Any>>): String {
        val keys = data.firstOrNull()?.keys ?: return ""
        val header = keys.joinToString(" | ")
        val separator = keys.joinToString(" | ") { "---" }
        return "$header\n$separator"
    }
    
    override fun formatRow(row: Map<String, Any>) = 
        row.values.joinToString(" | ")
    
    override fun formatSummary(data: List<Map<String, Any>>) = 
        "\n*Total: ${data.size} records*"
}

// Interface สำหรับ capabilities
interface Cacheable {
    val cacheKey: String
    val cacheDuration: Long  // milliseconds
    
    fun shouldCache() = cacheDuration > 0
}

interface Loggable {
    val logPrefix: String
    
    fun logInfo(msg: String) = println("[$logPrefix] INFO: $msg")
    fun logError(msg: String) = println("[$logPrefix] ERROR: $msg")
    fun logDebug(msg: String) = println("[$logPrefix] DEBUG: $msg")
}

// Abstract class พร้อม interfaces
abstract class BaseService(val serviceName: String) : Loggable {
    override val logPrefix = serviceName
    
    protected var isRunning = false
    
    fun start() {
        logInfo("Starting...")
        isRunning = true
        onStart()
        logInfo("Started")
    }
    
    fun stop() {
        logInfo("Stopping...")
        onStop()
        isRunning = false
        logInfo("Stopped")
    }
    
    protected abstract fun onStart()
    protected abstract fun onStop()
}

class UserService : BaseService("UserService"), Cacheable {
    override val cacheKey = "users"
    override val cacheDuration = 5 * 60 * 1000L  // 5 minutes
    
    private val users = mutableListOf<String>()
    
    override fun onStart() {
        logInfo("Loading users...")
        users.addAll(listOf("user1", "user2", "user3"))
        logInfo("Loaded ${users.size} users")
    }
    
    override fun onStop() {
        logInfo("Saving users...")
        users.clear()
    }
    
    fun getUsers(): List<String> {
        if (shouldCache()) logDebug("Using cache ($cacheKey)")
        return users.toList()
    }
}

fun main() {
    val data = listOf(
        mapOf("Name" to "สมชาย", "Age" to 25, "City" to "กรุงเทพ"),
        mapOf("Name" to "สมหญิง", "Age" to 30, "City" to "เชียงใหม่"),
        mapOf("Name" to "สมศักดิ์", "Age" to 22, "City" to "ขอนแก่น")
    )

    val textReport = TextReportGenerator()
    val mdReport = MarkdownReportGenerator()

    println(textReport.generateReport(data))
    println()
    println(mdReport.generateReport(data))

    println()

    val userService = UserService()
    userService.start()
    println("Users: ${userService.getUsers()}")
    println("Cache key: ${userService.cacheKey}")
    println("Should cache: ${userService.shouldCache()}")
    userService.stop()
}
```

---

## Step 155: Interface Delegation - by keyword

```kotlin
// Interface Delegation: ใช้ 'by' keyword
// Class สามารถ delegate การ implement interface ให้ object อื่น

interface Printer {
    fun print(text: String)
    fun printLine(text: String) = println(text)
}

class ConsolePrinter : Printer {
    override fun print(text: String) = kotlin.io.print(text)
}

class FilePrinter(val filename: String) : Printer {
    val output = StringBuilder()
    override fun print(text: String) { output.append(text) }
    override fun printLine(text: String) { output.appendLine(text) }
    fun getContent() = output.toString()
}

// Delegation ด้วย 'by'
class PrefixPrinter(
    private val prefix: String,
    printer: Printer  // delegate ให้ printer นี้
) : Printer by printer {  // delegate Printer methods ให้ printer
    
    // Override ที่ต้องการ custom behavior
    override fun print(text: String) {
        // ก่อน delegate ให้เพิ่ม prefix
        // ต้องเรียกแบบนี้เพราะเข้า printer ไม่ได้ตรงๆ
        println("[$prefix] $text")
    }
}

// Decorator ด้วย delegation
interface Stack<T> {
    fun push(item: T)
    fun pop(): T?
    fun peek(): T?
    fun isEmpty(): Boolean
    fun size(): Int
}

class SimpleStack<T> : Stack<T> {
    private val items = mutableListOf<T>()
    
    override fun push(item: T) { items.add(item) }
    override fun pop(): T? = if (items.isEmpty()) null else items.removeLast()
    override fun peek(): T? = items.lastOrNull()
    override fun isEmpty() = items.isEmpty()
    override fun size() = items.size
}

// LoggingStack delegate ให้ SimpleStack แล้ว add logging
class LoggingStack<T>(
    private val inner: Stack<T> = SimpleStack()
) : Stack<T> by inner {
    
    override fun push(item: T) {
        println("PUSH: $item")
        inner.push(item)
    }
    
    override fun pop(): T? {
        val item = inner.pop()
        println("POP: $item")
        return item
    }
}

// Mixin pattern ด้วย delegation
interface Serializable {
    fun serialize(): String
}

interface Deserializable<T> {
    fun deserialize(data: String): T
}

class JsonSerializer : Serializable {
    override fun serialize(): String = "json implementation"
}

data class Config2(val host: String, val port: Int) {
    private val serializer = JsonSerializer()
    
    // Delegate serialize ให้ JsonSerializer
    fun toJson(): String = """{"host":"$host","port":$port}"""
    
    companion object {
        fun fromJson(json: String): Config2 {
            // simplified parsing
            val host = json.substringAfter("host\":\"").substringBefore("\"")
            val port = json.substringAfter("port\":").substringBefore("}").trim().toInt()
            return Config2(host, port)
        }
    }
}

fun main() {
    // Delegation example
    val consolePrinter = ConsolePrinter()
    val prefixPrinter = PrefixPrinter("INFO", consolePrinter)
    
    consolePrinter.printLine("Direct output")
    prefixPrinter.print("With prefix")
    prefixPrinter.printLine("Line with prefix")  // ใช้ default impl จาก ConsolePrinter
    
    println()
    
    // Stack delegation
    val stack: Stack<Int> = LoggingStack()
    stack.push(1)
    stack.push(2)
    stack.push(3)
    println("Peek: ${stack.peek()}")
    stack.pop()
    stack.pop()
    println("Size: ${stack.size()}")
    println("Empty: ${stack.isEmpty()}")
    
    println()
    
    // Config serialization
    val config = Config2("localhost", 8080)
    val json = config.toJson()
    println("Serialized: $json")
    
    val restored = Config2.fromJson(json)
    println("Deserialized: $restored")
    println("Equal: ${config == restored}")
}
```

---

## Step 156: SAM Conversions

```kotlin
// SAM = Single Abstract Method
// Interface ที่มีเพียง 1 abstract method
// สามารถใช้ lambda แทน object expression ได้

// Java-style functional interface
fun interface Transformer {
    fun transform(input: String): String
}

// Kotlin functional interface
fun interface NumberPredicate {
    fun test(n: Int): Boolean
}

fun interface Action {
    fun execute()
}

// Function ที่รับ SAM interface
fun applyTransform(text: String, transformer: Transformer): String {
    return transformer.transform(text)
}

fun filter(numbers: List<Int>, predicate: NumberPredicate): List<Int> {
    return numbers.filter { predicate.test(it) }
}

fun runAction(times: Int, action: Action) {
    repeat(times) { action.execute() }
}

// Event system ด้วย SAM
fun interface EventHandler<T> {
    fun handle(event: T)
}

class EventBus<T> {
    private val handlers = mutableListOf<EventHandler<T>>()
    
    fun subscribe(handler: EventHandler<T>) = handlers.add(handler)
    fun unsubscribe(handler: EventHandler<T>) = handlers.remove(handler)
    fun emit(event: T) = handlers.forEach { it.handle(event) }
}

data class UserEvent(val type: String, val userId: Int, val data: Map<String, Any> = emptyMap())

fun main() {
    // SAM ด้วย lambda
    val upper = Transformer { it.uppercase() }
    val reverse = Transformer { it.reversed() }
    val trim = Transformer { it.trim() }
    
    println(applyTransform("  hello world  ", trim))
    println(applyTransform("hello world", upper))
    println(applyTransform("hello", reverse))
    
    // Chain transformations
    fun String.transform(vararg transformers: Transformer): String {
        return transformers.fold(this) { acc, t -> t.transform(acc) }
    }
    
    val result = "  Hello World  ".transform(trim, upper, reverse)
    println("Chained: $result")
    
    // NumberPredicate
    val numbers = (1..20).toList()
    val isEven = NumberPredicate { it % 2 == 0 }
    val isPrime = NumberPredicate { n ->
        n > 1 && (2..Math.sqrt(n.toDouble()).toInt()).none { n % it == 0 }
    }
    val isLarge = NumberPredicate { it > 10 }
    
    println("\nเลขคู่: ${filter(numbers, isEven)}")
    println("เลขเฉพาะ: ${filter(numbers, isPrime)}")
    println("ตัวเลขใหญ่: ${filter(numbers, isLarge)}")
    
    // Action
    var count = 0
    runAction(5) { count++ }
    println("\ncount: $count")
    
    runAction(3) { println("hello!") }
    
    // EventBus
    println("\n--- EventBus ---")
    val userBus = EventBus<UserEvent>()
    
    // subscribe ด้วย lambda (SAM conversion)
    userBus.subscribe { event ->
        println("Logger: ${event.type} - userId=${event.userId}")
    }
    
    userBus.subscribe { event ->
        if (event.type == "login") {
            println("Security: User ${event.userId} logged in from ${event.data["ip"]}")
        }
    }
    
    val analyticsHandler = EventHandler<UserEvent> { event ->
        println("Analytics: tracking ${event.type}")
    }
    userBus.subscribe(analyticsHandler)
    
    userBus.emit(UserEvent("login", 1, mapOf("ip" to "192.168.1.1")))
    userBus.emit(UserEvent("logout", 1))
    userBus.emit(UserEvent("purchase", 2, mapOf("amount" to 1500)))
    
    // unsubscribe
    userBus.unsubscribe(analyticsHandler)
    println("\nหลัง unsubscribe analytics:")
    userBus.emit(UserEvent("login", 3))
}
```

---

## Step 157: Interface Properties

```kotlin
// Interface สามารถมี properties ได้
// แต่ไม่มี backing field

interface Shape {
    // Abstract properties
    val area: Double
    val perimeter: Double
    val name: String
    
    // Property พร้อม default (computed จาก abstract properties)
    val isLargeShape: Boolean get() = area > 100.0
    val description: String get() = "$name: area=${"%.2f".format(area)}, perimeter=${"%.2f".format(perimeter)}"
    
    fun printInfo() = println(description)
}

class Circle2(val radius: Double) : Shape {
    override val area = Math.PI * radius * radius
    override val perimeter = 2 * Math.PI * radius
    override val name = "Circle"
}

class Rectangle2(val width: Double, val height: Double) : Shape {
    override val area = width * height
    override val perimeter = 2 * (width + height)
    override val name = "Rectangle"
}

// Interface ที่ inherit แล้วเพิ่ม properties
interface ColoredShape : Shape {
    val color: String
    override val description: String get() = "${super.description}, color=$color"
}

class ColoredCircle2(radius: Double, override val color: String) : ColoredShape {
    private val circle = Circle2(radius)
    override val area = circle.area
    override val perimeter = circle.perimeter
    override val name = "ColoredCircle"
}

// Interface สำหรับ Observable pattern
interface Observable<T> {
    val observers: List<(T) -> Unit>
    fun addObserver(observer: (T) -> Unit)
    fun removeObserver(observer: (T) -> Unit)
    fun notifyObservers(value: T)
}

class ObservableList<T> : Observable<List<T>> {
    private val _items = mutableListOf<T>()
    private val _observers = mutableListOf<(List<T>) -> Unit>()
    
    override val observers: List<(List<T>) -> Unit> get() = _observers.toList()
    
    override fun addObserver(observer: (List<T>) -> Unit) { _observers.add(observer) }
    override fun removeObserver(observer: (List<T>) -> Unit) { _observers.remove(observer) }
    override fun notifyObservers(value: List<T>) = _observers.forEach { it(value) }
    
    fun add(item: T) {
        _items.add(item)
        notifyObservers(_items.toList())
    }
    
    fun remove(item: T) {
        _items.remove(item)
        notifyObservers(_items.toList())
    }
    
    fun getItems() = _items.toList()
}

fun main() {
    val shapes: List<Shape> = listOf(
        Circle2(5.0),
        Circle2(15.0),
        Rectangle2(8.0, 10.0),
        Rectangle2(3.0, 4.0),
        ColoredCircle2(7.0, "แดง")
    )

    shapes.forEach { shape ->
        shape.printInfo()
        println("  isLarge: ${shape.isLargeShape}")
    }

    println("\nรูปขนาดใหญ่:")
    shapes.filter { it.isLargeShape }.forEach { it.printInfo() }

    println("\nเรียงตามพื้นที่:")
    shapes.sortedByDescending { it.area }.forEach { 
        println("  ${it.name}: ${"%.2f".format(it.area)}")
    }

    // Observable
    println("\n--- Observable List ---")
    val list = ObservableList<String>()
    
    list.addObserver { items ->
        println("Observer 1: items changed -> $items")
    }
    
    list.addObserver { items ->
        println("Observer 2: count = ${items.size}")
    }
    
    list.add("Kotlin")
    list.add("Java")
    list.add("Python")
    list.remove("Java")
}
```

---

## Step 158: Combining Abstract Class and Interface

```kotlin
// ตัวอย่าง: Game Entity System

interface Renderable {
    fun render()
}

interface Collidable {
    val boundingBox: BoundingBox
    fun onCollision(other: Entity)
}

interface Updatable {
    fun update(deltaTime: Double)
}

data class BoundingBox(val x: Double, val y: Double, val width: Double, val height: Double) {
    fun intersects(other: BoundingBox): Boolean {
        return x < other.x + other.width &&
               x + width > other.x &&
               y < other.y + other.height &&
               y + height > other.y
    }
    
    override fun toString() = "BB($x, $y, ${width}x$height)"
}

// Abstract class พร้อม common state
abstract class Entity(
    var x: Double,
    var y: Double,
    val name: String
) : Updatable, Renderable {
    abstract val width: Double
    abstract val height: Double
    
    var isActive = true
    var health = 100.0
    
    val isAlive get() = health > 0 && isActive
    
    override fun update(deltaTime: Double) {
        // default: ไม่ทำอะไร
    }
    
    fun takeDamage(amount: Double) {
        health -= amount
        println("$name รับ damage $amount (health: $health)")
        if (!isAlive) onDeath()
    }
    
    protected open fun onDeath() {
        println("$name ตายแล้ว!")
        isActive = false
    }
}

class Player(x: Double, y: Double) : Entity(x, y, "Player"), Collidable {
    override val width = 32.0
    override val height = 48.0
    
    override val boundingBox get() = BoundingBox(x, y, width, height)
    
    var speed = 150.0
    var score = 0
    
    override fun update(deltaTime: Double) {
        // จำลองการเคลื่อนที่
        x += speed * deltaTime * 0.01
    }
    
    override fun render() {
        println("Rendering Player at ($x, $y) health=$health score=$score")
    }
    
    override fun onCollision(other: Entity) {
        when (other) {
            is Enemy -> {
                takeDamage(10.0)
                println("Player ชนกับ Enemy!")
            }
            is Coin -> {
                score += 10
                println("Player เก็บ Coin! score=$score")
                other.isActive = false
            }
        }
    }
}

class Enemy(x: Double, y: Double, val type: String) : Entity(x, y, "Enemy-$type"), Collidable {
    override val width = 24.0
    override val height = 32.0
    override val boundingBox get() = BoundingBox(x, y, width, height)
    
    private var direction = 1.0
    
    override fun update(deltaTime: Double) {
        x += 50.0 * direction * deltaTime * 0.01
        if (x > 400 || x < 0) direction *= -1
    }
    
    override fun render() {
        println("Rendering $name at ($x, $y)")
    }
    
    override fun onCollision(other: Entity) {
        println("$name ชนกับ ${other.name}")
    }
}

class Coin(x: Double, y: Double) : Entity(x, y, "Coin") {
    override val width = 16.0
    override val height = 16.0
    
    override fun render() {
        if (isActive) println("Rendering Coin at ($x, $y)")
    }
}

class GameWorld {
    private val entities = mutableListOf<Entity>()
    
    fun addEntity(entity: Entity) = entities.add(entity)
    
    fun update(deltaTime: Double) {
        entities.filter { it.isAlive }.forEach { it.update(deltaTime) }
        checkCollisions()
    }
    
    fun render() {
        entities.filter { it.isActive }.forEach { it.render() }
    }
    
    private fun checkCollisions() {
        val collidables = entities.filterIsInstance<Collidable>()
        for (i in collidables.indices) {
            for (j in i + 1 until collidables.size) {
                val a = collidables[i] as Entity
                val b = collidables[j] as Entity
                if (a.isActive && b.isActive && collidables[i].boundingBox.intersects(collidables[j].boundingBox)) {
                    collidables[i].onCollision(b)
                    collidables[j].onCollision(a)
                }
            }
        }
    }
}

fun main() {
    val world = GameWorld()
    val player = Player(100.0, 100.0)
    
    world.addEntity(player)
    world.addEntity(Enemy(100.0, 100.0, "Slime"))  // ชนกับ player ทันที
    world.addEntity(Coin(200.0, 200.0))
    world.addEntity(Coin(300.0, 100.0))
    
    println("=== Frame 1 ===")
    world.update(16.0)  // 16ms = ~60fps
    world.render()
    
    // เลื่อน player ไปเก็บ coin
    player.x = 200.0
    player.y = 200.0
    
    println("\n=== Frame 2 ===")
    world.update(16.0)
    world.render()
}
```

---

## Step 159: Functional Interfaces ขั้นสูง

```kotlin
// Functional interfaces ใน Kotlin

fun interface Predicate<T> {
    fun test(value: T): Boolean
    
    // Default methods
    fun and(other: Predicate<T>): Predicate<T> = Predicate { test(it) && other.test(it) }
    fun or(other: Predicate<T>): Predicate<T> = Predicate { test(it) || other.test(it) }
    fun negate(): Predicate<T> = Predicate { !test(it) }
}

fun interface Mapper<T, R> {
    fun map(value: T): R
    
    fun <V> andThen(after: Mapper<R, V>): Mapper<T, V> = Mapper { after.map(map(it)) }
}

fun interface Reducer<T, R> {
    fun reduce(acc: R, value: T): R
}

// Pipeline ด้วย functional interfaces
class Pipeline<T> private constructor(private val data: List<T>) {
    fun <R> map(mapper: Mapper<T, R>): Pipeline<R> {
        return Pipeline(data.map { mapper.map(it) })
    }
    
    fun filter(predicate: Predicate<T>): Pipeline<T> {
        return Pipeline(data.filter { predicate.test(it) })
    }
    
    fun <R> reduce(initial: R, reducer: Reducer<T, R>): R {
        return data.fold(initial) { acc, item -> reducer.reduce(acc, item) }
    }
    
    fun toList() = data.toList()
    
    companion object {
        fun <T> of(vararg items: T) = Pipeline(items.toList())
        fun <T> of(items: List<T>) = Pipeline(items)
    }
}

fun main() {
    // Predicate composition
    val isEven: Predicate<Int> = Predicate { it % 2 == 0 }
    val isPositive: Predicate<Int> = Predicate { it > 0 }
    val isLarge: Predicate<Int> = Predicate { it > 10 }
    
    val isEvenAndPositive = isEven.and(isPositive)
    val isEvenOrLarge = isEven.or(isLarge)
    val isOdd = isEven.negate()
    
    val numbers = (-5..20).toList()
    println("isEven: ${numbers.filter { isEven.test(it) }}")
    println("isEvenAndPositive: ${numbers.filter { isEvenAndPositive.test(it) }}")
    println("isOdd: ${numbers.filter { isOdd.test(it) }}")

    // Mapper composition
    val toDouble: Mapper<Int, Double> = Mapper { it.toDouble() }
    val square: Mapper<Double, Double> = Mapper { it * it }
    val toString: Mapper<Double, String> = Mapper { "%.2f".format(it) }
    
    val squaredString = toDouble.andThen(square).andThen(toString)
    
    println("\nSquared strings:")
    (1..5).map { squaredString.map(it) }.forEach { println("  $it") }

    // Pipeline
    val result = Pipeline.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .filter(isEven)
        .filter(isPositive)
        .map(Mapper { it * it })
        .filter(Predicate { it > 10 })
        .toList()
    
    println("\nPipeline result: $result")
    
    val sum = Pipeline.of(1, 2, 3, 4, 5)
        .reduce(0, Reducer { acc, v -> acc + v })
    println("Sum: $sum")
    
    // String pipeline
    val words = Pipeline.of("hello", "world", "kotlin", "is", "awesome")
        .filter(Predicate { it.length > 4 })
        .map(Mapper { it.uppercase() })
        .toList()
    println("Words: $words")
}
```

---

## Step 160: Design Patterns ด้วย Interface

```kotlin
// Observer Pattern
interface Observer<T> {
    fun update(value: T)
}

interface Subject<T> {
    fun attach(observer: Observer<T>)
    fun detach(observer: Observer<T>)
    fun notify(value: T)
}

// Command Pattern
interface Command {
    fun execute()
    fun undo()
}

class TextEditor {
    private val text = StringBuilder()
    private val history = ArrayDeque<Command>()
    
    fun executeCommand(command: Command) {
        command.execute()
        history.addLast(command)
    }
    
    fun undo() {
        if (history.isNotEmpty()) {
            history.removeLast().undo()
        }
    }
    
    inner class InsertCommand(val position: Int, val chars: String) : Command {
        override fun execute() {
            text.insert(position, chars)
            println("Insert '$chars' at $position -> '$text'")
        }
        
        override fun undo() {
            text.delete(position, position + chars.length)
            println("Undo insert -> '$text'")
        }
    }
    
    inner class DeleteCommand(val position: Int, val count: Int) : Command {
        private var deleted = ""
        
        override fun execute() {
            deleted = text.substring(position, position + count)
            text.delete(position, position + count)
            println("Delete '$deleted' at $position -> '$text'")
        }
        
        override fun undo() {
            text.insert(position, deleted)
            println("Undo delete -> '$text'")
        }
    }
    
    fun getText() = text.toString()
}

// Strategy Pattern
interface CompressionStrategy {
    val name: String
    fun compress(data: String): String
    fun decompress(data: String): String
}

class NoCompression : CompressionStrategy {
    override val name = "None"
    override fun compress(data: String) = data
    override fun decompress(data: String) = data
}

class RunLengthEncoding : CompressionStrategy {
    override val name = "RLE"
    
    override fun compress(data: String): String {
        if (data.isEmpty()) return ""
        val sb = StringBuilder()
        var count = 1
        for (i in 1 until data.length) {
            if (data[i] == data[i - 1]) {
                count++
            } else {
                sb.append("${data[i-1]}$count")
                count = 1
            }
        }
        sb.append("${data.last()}$count")
        return sb.toString()
    }
    
    override fun decompress(data: String): String {
        val sb = StringBuilder()
        var i = 0
        while (i < data.length) {
            val char = data[i]
            val countStr = StringBuilder()
            i++
            while (i < data.length && data[i].isDigit()) {
                countStr.append(data[i])
                i++
            }
            repeat(countStr.toString().toInt()) { sb.append(char) }
        }
        return sb.toString()
    }
}

fun main() {
    // Command Pattern
    println("=== Text Editor ===")
    val editor = TextEditor()
    
    editor.executeCommand(editor.InsertCommand(0, "Hello"))
    editor.executeCommand(editor.InsertCommand(5, " World"))
    editor.executeCommand(editor.InsertCommand(11, "!"))
    editor.executeCommand(editor.DeleteCommand(5, 6))
    
    println("Text: '${editor.getText()}'")
    
    editor.undo()
    editor.undo()
    
    println("After 2 undos: '${editor.getText()}'")

    // Strategy Pattern
    println("\n=== Compression ===")
    val strategies: List<CompressionStrategy> = listOf(NoCompression(), RunLengthEncoding())
    
    val testData = "AAABBBCCCCDDDDDEEE"
    strategies.forEach { strategy ->
        val compressed = strategy.compress(testData)
        val decompressed = strategy.decompress(compressed)
        println("${strategy.name}:")
        println("  Original: $testData (${testData.length})")
        println("  Compressed: $compressed (${compressed.length})")
        println("  Decompressed: $decompressed")
        println("  Match: ${testData == decompressed}")
    }
}
```

---

## Step 161: แบบฝึกหัด Interface ขั้นสูง

```kotlin
// ============================================================
// โจทย์ 1: Plugin System
// ============================================================

interface Plugin {
    val name: String
    val version: String
    val description: String
    
    fun initialize(context: PluginContext)
    fun shutdown()
    fun isCompatible(appVersion: String): Boolean = true
}

class PluginContext(
    val appName: String,
    val appVersion: String,
    val config: Map<String, String>
) {
    private val services = mutableMapOf<String, Any>()
    
    fun registerService(name: String, service: Any) {
        services[name] = service
    }
    
    fun getService(name: String) = services[name]
}

class LogPlugin : Plugin {
    override val name = "Logger Plugin"
    override val version = "1.0.0"
    override val description = "เพิ่มความสามารถในการ log"
    
    private var logLevel = "INFO"
    
    override fun initialize(context: PluginContext) {
        logLevel = context.config["log.level"] ?: "INFO"
        context.registerService("logger", this)
        println("$name initialized (level=$logLevel)")
    }
    
    override fun shutdown() = println("$name shutting down")
    
    fun log(level: String, msg: String) {
        println("[$level] $msg")
    }
}

class MetricsPlugin : Plugin {
    override val name = "Metrics Plugin"
    override val version = "2.0.0"
    override val description = "เก็บ metrics ต่างๆ"
    
    private val metrics = mutableMapOf<String, Long>()
    
    override fun initialize(context: PluginContext) {
        context.registerService("metrics", this)
        println("$name initialized")
    }
    
    override fun shutdown() {
        println("$name: final metrics = $metrics")
    }
    
    fun increment(key: String) { metrics[key] = (metrics[key] ?: 0) + 1 }
    fun get(key: String) = metrics[key] ?: 0
}

class PluginManager {
    private val plugins = mutableListOf<Plugin>()
    private var context: PluginContext? = null
    
    fun registerPlugin(plugin: Plugin) {
        plugins.add(plugin)
        println("Registered: ${plugin.name} v${plugin.version}")
    }
    
    fun initialize(appName: String, appVersion: String, config: Map<String, String> = emptyMap()) {
        context = PluginContext(appName, appVersion, config)
        println("Initializing plugins for $appName v$appVersion...")
        
        plugins.filter { it.isCompatible(appVersion) }.forEach { plugin ->
            plugin.initialize(context!!)
        }
    }
    
    fun shutdown() {
        println("Shutting down plugins...")
        plugins.reversed().forEach { it.shutdown() }
    }
    
    @Suppress("UNCHECKED_CAST")
    fun <T> getService(name: String): T? = context?.getService(name) as? T
}

fun main() {
    val manager = PluginManager()
    manager.registerPlugin(LogPlugin())
    manager.registerPlugin(MetricsPlugin())
    
    manager.initialize(
        appName = "MyApp",
        appVersion = "2.0.0",
        config = mapOf("log.level" to "DEBUG")
    )
    
    // ใช้ services จาก plugins
    val logger = manager.getService<LogPlugin>("logger")
    val metrics = manager.getService<MetricsPlugin>("metrics")
    
    logger?.log("INFO", "Application started")
    metrics?.increment("requests")
    metrics?.increment("requests")
    metrics?.increment("errors")
    
    logger?.log("DEBUG", "requests=${metrics?.get("requests")}, errors=${metrics?.get("errors")}")
    
    manager.shutdown()
    
    // ============================================================
    // โจทย์ 2: Builder Pattern ด้วย Interface
    // ============================================================

    interface Builder<T> {
        fun build(): T
    }
    
    interface Validatable {
        fun validate(): List<String>  // returns error messages
        fun isValid() = validate().isEmpty()
    }

    data class HttpRequest(
        val method: String,
        val url: String,
        val headers: Map<String, String>,
        val body: String?,
        val timeout: Int
    )

    class HttpRequestBuilder : Builder<HttpRequest>, Validatable {
        private var method = "GET"
        private var url = ""
        private val headers = mutableMapOf<String, String>()
        private var body: String? = null
        private var timeout = 30
        
        fun method(m: String) = apply { method = m.uppercase() }
        fun url(u: String) = apply { url = u }
        fun header(key: String, value: String) = apply { headers[key] = value }
        fun authorization(token: String) = header("Authorization", "Bearer $token")
        fun json() = apply { headers["Content-Type"] = "application/json" }
        fun body(b: String) = apply { body = b }
        fun timeout(t: Int) = apply { timeout = t }
        
        override fun validate(): List<String> {
            val errors = mutableListOf<String>()
            if (url.isBlank()) errors.add("URL ต้องไม่ว่าง")
            if (method !in listOf("GET", "POST", "PUT", "DELETE", "PATCH")) 
                errors.add("Method ไม่ถูกต้อง: $method")
            if (method in listOf("POST", "PUT") && body == null)
                errors.add("$method ต้องมี body")
            if (timeout <= 0) errors.add("Timeout ต้องมากกว่า 0")
            return errors
        }
        
        override fun build(): HttpRequest {
            val errors = validate()
            if (errors.isNotEmpty()) {
                throw IllegalStateException("Invalid request:\n${errors.joinToString("\n")}")
            }
            return HttpRequest(method, url, headers.toMap(), body, timeout)
        }
    }

    println("\n=== HTTP Request Builder ===")

    val request = HttpRequestBuilder()
        .method("POST")
        .url("https://api.example.com/users")
        .json()
        .authorization("my-token-123")
        .body("""{"name": "สมชาย", "email": "test@example.com"}""")
        .timeout(60)
        .build()
    
    println("Method: ${request.method}")
    println("URL: ${request.url}")
    println("Headers: ${request.headers}")
    println("Body: ${request.body}")
    println("Timeout: ${request.timeout}s")
    
    // Invalid request
    val invalidBuilder = HttpRequestBuilder()
        .method("POST")
        .url("")  // ว่าง
        // ไม่มี body สำหรับ POST
    
    val errors = invalidBuilder.validate()
    println("\nValidation errors:")
    errors.forEach { println("  - $it") }
}
```

---

## สรุปส่วนที่ 8 (Summary)

| Concept | ใช้เมื่อไหร่ |
|---------|------------|
| `interface` | กำหนด capabilities, multiple behavior |
| `abstract class` | shared state + behavior template |
| default implementation | optional behavior ที่ subclass ไม่ต้อง override |
| `interface by delegation` | reuse implementation โดยไม่ต้อง inherit |
| `fun interface` | SAM interface สำหรับ lambda |
| SAM conversion | ใช้ lambda แทน anonymous object |

### เมื่อไหร่ใช้อะไร
- **Interface**: ไม่มี state, multiple inheritance ต้องการ
- **Abstract Class**: มี state หรือ constructor, "is-a" relationship
- **Interface Delegation**: Decorator/Mixin pattern
- **SAM/fun interface**: Callbacks, event handlers

---

[← Part 07: Inheritance](../part07/README.md) | [Part 09: Null Safety →](../part09/README.md)
